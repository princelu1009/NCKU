---
title: "STARVERI: Efficient and Accurate Verification for Risk-Avoidance Routing in LEO Satellite Networks"
authors: [Chenwei Gu, Qian Wu, Zeqi Lai, Hewu Li, Jihao Li, Weisen Liu, Qi Zhang, Jun Liu, Yuanjie Li]
venue: "IEEE ICNP 2024（致謝提到 ICNP reviewers；arXiv:2412.16496v1, 2024-12-21）"
affiliation: "清華大學網路科學與網路空間研究院、中關村實驗室（通訊作者 Zeqi Lai）"
tags: [LEO, satellite-routing, security, path-verification, risk-avoidance, source-routing, relay, MAC, delay-based, Alibi-Routing, StarryNet, Starlink, Kuiper, +Grid]
---

## Citation
C. Gu, Q. Wu, Z. Lai, H. Li, J. Li, W. Liu, Q. Zhang, J. Liu, Y. Li, "STARVERI: Efficient and Accurate Verification for Risk-Avoidance Routing in LEO Satellite Networks," *IEEE ICNP*, 2024. arXiv:2412.16496.

## 一句話總結
這篇**不是在找「最快的路」**，而是在處理**安全**：衛星會飛過不受控的「風險區」（例子用 Egypt、North Korea），那裡的人可能竊聽或劫持（hijack）流量。營運商（SNO）可以規劃一條**繞開風險區的路**，但還要能**證明封包真的沒經過那裡**（path verification，路徑驗證）。現有兩類方法在 LEO 都不好用：**crypto-based**（每一跳都算 MAC）太吃衛星算力和 header；**delay-based**（Alibi Routing：如果繞進風險區，延遲會明顯變大）依賴「延遲 ≈ 距離的線性函數」，但 LEO 拓樸一直變，這個假設不成立，誤判率 > 25%。STARVERI 把兩者合起來：(1) SNO 控制中心用軌道預測**動態挑選最多 2 顆衛星當 relay**，把路切成幾段；(2) **只有 relay 算 MAC**（證明封包有經過我），並對**每一段**做 delay 檢查：實測段延遲 ≤「定期探測到的段延遲 + 繞路門檻 δ」，其中 δ = 段上任一點到任一風險衛星最短傳播延遲的 **2 倍**（去一趟再回來）。在 StarryNet 上（Starlink Shell 1 的 1584 顆、Kuiper 1156 顆、197 組城市對）驗證準確率 **100%**（Alibi：FP 38.7%、FN 27.3%），header 最多 **48 B**，35 跳時驗證延遲比 EPIC 少 **97.2%**。

**English:** STARVERI is a path-verification framework, not a shortest-path protocol. Satellites inevitably pass over risky regions where attackers may eavesdrop or hijack routes, so an operator wants traffic to avoid those regions and wants proof that it did. Per-hop cryptographic verification (ICING, OPT, EPIC) is too heavy for satellites, and delay-based verification (Alibi Routing) breaks because LEO delay is not a linear function of ground distance. STARVERI lets the operations centre pick at most two satellite relays per flow from predicted topology (using "Nearest Low-Risk Planes", NLRP), splits the path into segments, and has only relays add a MAC. Each relay checks that its segment delay stays under the probed delay plus a detour threshold equal to twice the minimum propagation delay from the segment to any risk satellite. On StarryNet with Starlink and Kuiper it reports 100% accuracy, at most 48 bytes of header and much lower verification delay than EPIC.

## 0. 背景知識
### 這篇用到的名詞
| 名詞 | 意思 |
| --- | --- |
| **Risk area（RA）／risk satellites（RS）** | 地面上的一塊危險區域；當下在它上空的衛星就是 risk satellites，**每個時刻都不同** |
| **SNO** | Satellite Network Operator，例如 SpaceX。有控制中心、知道全網軌道 |
| **Path verification** | 不只「規劃」路，還要讓目的端能**驗證**封包實際走的路符合規劃 |
| **MAC** | Message Authentication Code，用共享金鑰算的短簽章（本文用 HMAC-SHA256 取前 4 B） |
| **Crypto-based 驗證** | ICING / OPT / EPIC：路徑上**每一跳**都算 MAC、塞進 header，層層巢狀，目的端檢查整條鏈 |
| **Delay-based 驗證** | Alibi Routing（SIGCOMM 2015）：強迫路經過一個遠離風險區的 relay（alibi，不在場證明）；繞進風險區就會多很多延遲 |
| **FP / FN** | False positive：沒經過卻被判經過；False negative：經過了卻沒抓到（安全上更嚴重） |
| **PAN / source routing** | Path-Aware Networking（如 SCION）：來源端把整條路寫進封包，中間節點照著轉送。STARVERI 建在這上面 |
| **+Grid** | 每顆衛星連同軌前後 2 顆 + 左右相鄰軌 2 顆（和 OPSPF、StableRoute 等筆記同一個拓樸） |
| **NE-bound / SE-bound** | 往東北飛的升軌段 / 往東南飛的降軌段。Walker-Delta 星座在同一地區上空有兩組交錯的網格 |

### 為什麼 delay-based 在 LEO 不準（§II-C，Fig. 2–3）⭐
地面網路：兩點延遲大約和大圓距離成正比 → 延遲突然變大 = 繞路了。
LEO：Src/Dst 都不動，路也會因為**換入口衛星、拓樸變動**而變長，延遲本來就會跳。作者在 Starlink Shell 1（72 軌 × 22 顆）+ 165 地面站、約 2000 組城市對、24 小時、每秒一次：
* Fig. 2：大圓距離和 RTT **不是線性關係**。
* Fig. 3：用 Alibi Routing（風險區 Egypt、地面固定 relay），**多數城市對的 FP 和 FN 比例都 > 25%**。
→ 「延遲變大」可能只是拓樸變了，不是被劫持；反之也可能被劫持卻沒變大到門檻。

## 1. Problem
* **威脅模型（§II-B，Fig. 1）**：
    * **Information stealing**：風險區上空的衛星被竊聽（例：S7→S9 被聽）。
    * **Traffic hijacking**：攻擊者發假路由廣告，把流量引到自己（例：路被改成 S7→S4→S5→S9），進行審查、注入、竄改。
* SNO 要做兩件事：(i) 讓流量**繞開**風險區；(ii) **驗證**實際路徑真的繞開。
* 現有方法的缺點：
    * Crypto-based：每跳算 MAC → 衛星 CPU 吃不消；header 隨跳數變長（LEO 路常有幾十跳）→ goodput 下降；每跳多處理延遲，在 LEO 路徑本來就常變的情況下更糟。PPV / MASK 用機率抽驗，流量大時還是重。
    * Delay-based：線性假設不成立 → 不準；地面固定 relay 數量少、分布不均，常常找不到。

## 2. Solution：STARVERI 架構（§III，Fig. 4）
### 2.1 三個核心想法
1. **用 relay 把路切成段**：驗證整條路 = 驗證每一段。
2. **只有 relay 做密碼運算**：證明封包確實經過這些 relay（MAC 鏈），其他衛星照常轉送、什麼都不做。
3. **段延遲比全程延遲穩定**：一段很短，拓樸變動對它的影響小，所以 delay-based 檢查在段上就準了。

### 2.2 工作流程（§III-B）
1. **Dynamic relay selection**（在 SNO 控制中心）：用可預測的軌道，**預先算好未來一段時間**每個時槽的 relay 序列和每段的繞路門檻 δ，下發給 Src（終端存在本地快取，減少向 SNO 查詢）。
2. **Intra-segment state probing**：每個 relay（含 Dst 的接入衛星 sat_d）**每幾十秒**向前一個 relay 發探測包，量這段的實際延遲 d（含排隊、處理）。探測包每跳用鄰居共享金鑰認證，防竄改。
3. **Authentication fields initiation**：Src 送出前填入修正後的時間戳、HASH、每個 relay 預留的 AUTH 欄位。
4. **Relay-based segment verification**：relay 收到封包，檢查段延遲是否超標；沒超標就算 MAC、填自己的 AUTH 欄位。
5. **Destination verification**：Dst 檢查 HASH、整條 MAC 鏈、最後一段延遲，全過才收。

## 3. 方法細節（§IV）
### 3.1 網路模型（§IV-A）
* 時間切成快照 t1, t2, …，快照內拓樸固定；G^t = (V, L^t)。
* 衛星用邏輯座標 (p, n)：p = 軌道編號（< P）、n = 軌內位置（< H）。
* 每段路 = Dijkstra 最短路；相鄰兩點延遲 = 直線距離 / 光速。
* **EP^t_ab**：a、b 之間**所有延遲差不多的路**經過的節點集合。這是重點：+Grid 上有很多條一樣短的路，只要其中一條穿過風險區，延遲檢查就分不出「走了好路」還是「走了壞路」。

### 3.2 AVDRS 問題（§IV-B a）
**Avoidance-Verifiable Dynamic Relay Selection**
* 輸入：拓樸 L^t、接入衛星 sat_s、sat_d、風險衛星 RS^t。輸出：relay 序列 RE^t。
* 目標（式 1）：最小化 Src→Dst 端到端延遲。
* 限制：
    * (2) relay 有順序；
    * (3) 路上任一點到任一風險衛星距離 **≥ θ**（θ 越大越安全但越繞）；
    * (4) 路要連通；
    * (5) ⭐ **EP ∩ RS = ∅**：每段的「所有等長路」都不能碰到風險衛星（否則驗證會被等長的壞路混淆）；
    * (6) relay 數 **≤ σ**（每多一個 relay，header 多 16 B）。
* 複雜度：可用 relay 約 10³ 顆，排列數 O(A^σ_ψ)；作者把它類比成 Hamiltonian Path Problem（NP-hard）。⚠️ 這個「類比」證明很鬆，只說明可以化成找一條經過每個 relay 一次的序列，不是嚴格的 NP-hard 歸約。

### 3.3 DRSA：啟發式解法（§IV-B b，Alg. 1，Fig. 5–6）⭐
**NLRP（Nearest Low-Risk Planes）**：把風險衛星框起來的「外框」。在 NE-bound 和 SE-bound 兩個方向各取**上、下、左、右**四條低風險平面（共 8 條，式 7），每條距離最外圈風險衛星 **θ hop**。
* 想像成在 +Grid 上把風險區用一個長方形框住，框外 θ 格都是緩衝區。

**Frame F^t_sd**：sat_s 和 sat_d 所在的軌道面/軌內位置圍成的長方形（+Grid 上所有最短路都在這個框內）。r* = (p_s, n_d) 或 (p_d, n_s) 是框的角。

**同方向（dir(sat_s) = dir(sat_d)）的三種情況：**
| 情況 | F_sd 和 NLRP 的關係 | DRSA 怎麼選 relay | Fig. 5 例子 |
| --- | --- | --- | --- |
| 不重疊 | 框和風險框分開 | 選離 NLRP 較遠的那個角 r* 當唯一 relay（確保兩段各在同一軌道面內，沒有其他等長路） | s6→s14，relay = s7 |
| 半重疊 | 有一條最短路可以不碰風險區 | 選那條安全最短路上的角當唯一 relay | s1→s8，relay = s7 |
| 全重疊 | 所有最短路都會穿過風險區 | 把框**放大**到包含 NLRP，選 NLRP 邊界和 s/d 軌道面交點的**兩個角**當 relay | s10→s9，relay = (s1, s7) |
| 其他 | 例如 sat_s 或 sat_d 本身就在 RS 裡 | 回傳 Error | — |

**不同方向**（Fig. 6）：把 3D 拓樸近似攤平成 2D，同樣判斷框和 NLRP 的交集。

**為什麼 relay 放在角上**：relay 和前後節點在**同一個軌道面**時，那一段只有一條直線路（沿軌或跨軌），**不會有其他等長路**，延遲檢查就不會被混淆。

**為什麼最多 2 個 relay**（「Discussion of σ」）：多放 relay 看起來驗證更細，但 (i) 選錯 relay（如 s11, s3）路更長、還更靠近風險區；(ii) 在同一條直線上多插一個（s4）兩段的門檻幾乎一樣，沒增益。所以 DRSA 用 NLRP 把 σ 壓到 ≤ 2；同一軌道面又不是全重疊時甚至**不需要 relay**。

### 3.4 繞路門檻 δ（式 8）⭐
δ^t_ij = min over (a ∈ 段 ij 上的點, b ∈ RS) 的 **2 · D^t_ab**
* 直覺：封包要被誤轉進風險區，至少要從路上某點**去到最近的風險衛星再回來**，所以額外延遲 ≥ 2 倍最短傳播延遲。
* 假設：沒有攻擊能讓延遲**低於**探測值（因為段本來就是最短路）。
* **段延遲上限 = 探測到的段延遲 d + δ**。δ 同時當作抖動緩衝（jitter buffer），所以 θ 越大，δ 越大，對排隊抖動越耐受。

### 3.5 封包處理（§IV-C～E，Alg. 2，Fig. 7）
**Src（Step I）**
* 先探測接入延遲 D_access，修正送出時間戳 ts0 = ts_send + D_access + α（α 為處理時間）→ 消除第一跳（上行 GSL）誤差。
* HASH = H(Payload ‖ PATH ‖ Δ ‖ ts0 ‖ RE)，取最低 l = 4 B。
* 每個 relay 的 AUTH 欄位 16 B：relay ID r_i、段門檻 δ、時間戳 ts_i、MAC（各 4 B）。
* 放在 **IPv6 hop-by-hop extension header**。

**Relay（Step II）**
* 非 relay 的衛星：只轉送（或處理探測包）。
* Relay：(1) 檢查前一個 relay 有沒有填 AUTH（只檢查「有填」，**不驗證**，驗證留給 Dst）；(2) 檢查 ts_i − ts_{i−1} ≤ δ + d（式 9）。都過 → 算 MAC_K(HASH ‖ ts_i ‖ r_i)，填欄位、轉送；不過 → 丟棄。
* sat_d 只填自己的到達時間 ts_d，並探測最後一段延遲報給 Dst。

**Dst（Step III）** 四個條件全過才接受：(1) 來源認證（論文 out of scope）；(2) HASH 正確（PATH 沒被改）；(3) 每個 relay 的 MAC 都對（真的經過這些 relay）；(4) 最後一段延遲 ≤ δ + d。

⚠️ 時間戳檢查需要 relay 之間**時間同步**，論文沒討論同步誤差。

## 4. 安全分析（§V）
* **Hijacking**：被劫進風險區的封包最終還是要回到一個正常節點（至少 sat_d），多出的延遲會超過段上限。
* **竄改 PATH**：Dst 重算 HASH 會發現。
* **跳過 relay**：下游 relay 或 Dst 發現前一個 AUTH 沒更新 → 丟棄。跳過一般節點則延遲會變長。
* **偽造時間戳/MAC**：攻擊者沒有 session key，Dst 重算 MAC 鏈會發現。
* **Replay**：HASH 含時間戳；過期封包也過不了延遲檢查。
* ⚠️ 沒有討論的：**relay 本身被入侵**的情況（它持有 session key，可以替繞路的封包簽 MAC），以及攻擊者在段延遲門檻 δ 內就能完成的短繞路。

## 5. Evaluation（§VI）
### 5.1 設定
* 模擬器：**StarryNet**（NSDI 2023，同一個清華團隊）—— 每顆衛星是一個 Docker 容器，完整 TCP/IP。驗證邏輯用 **Netfilter** 佇列逐包處理；Src 持續 ping Dst **6000 s**（超過一個軌道週期）。
* 星座：**Starlink Phase 1 Shell 1**（72 × 22 = 1584）和 **Kuiper**（34 軌、1156 顆）；+Grid；接入用最近衛星。
* 197 組城市對（主要在美洲和部分南亞）；風險區 **Egypt**（大）和 **North Korea**（小）。
* θ = 1/2/3 hop；對手 Alibi（f = 0/0.5，地面站當 relay）；開銷比較對象 ICING、OPT、EPIC。
* 硬體：兩台 Xeon Gold 5215、512 GB。

### 5.2 結果
| 問題 | 結果 |
| --- | --- |
| Q1 準確率 | STARVERI 劫持場景下 **100%**；Alibi 在 Egypt：**FP 38.7%、FN 27.3%** |
| Q2 延遲代價（Table I） | Starlink：Egypt θ=1/2/3 = **12.4% / 23.3% / 36.4%**；North Korea = 3.6% / 5.4% / 9.0%。Kuiper：Egypt 10.6% / 15.7% / 24.7%；NK 2.4% / 3.0% / 4.4%。Alibi 最大 inflation **156.5% / 175.5%** |
| Alibi 找不到 relay | NK 時 197 組只有 **112 組**找得到固定地面 relay；STARVERI 只要接入衛星不在 NLRP 內就能選 |
| 繞路門檻（Fig. 9） | 幾十～幾百 ms；Egypt 時 θ 越大 δ 越大；NK（小區域）θ 不同 δ 都約 **75 ms** → 小風險區用 θ=1 就夠 |
| Header（Table II） | STARVERI **16 / 32 / 48 B**（σ=0/1/2，和跳數無關）；30 跳時 ICING 1273 B、OPT 532 B、EPIC 174 B |
| Goodput（Fig. 10） | 30 跳、1024 B payload、σ=2：比 ICING / OPT / EPIC 高 **110.2% / 43.5% / 11.3%** |
| 驗證延遲（Table III） | STARVERI 172 / 308 / 550 µs vs EPIC 約 6 ms；**35 跳時少 97.2%** |
| NLRP 變動頻率（Fig. 11） | Starlink 避開 Egypt：NLRP 上下平面平均 **260.7 s** 才變一次，比 RS 變化慢很多 → relay 和金鑰協商不用常做 |
| 擴展性（Fig. 12） | 700～1000 條平行 flow 時平均驗證延遲 86 / 92 / 134 ms（σ=0/1/2） |
| relay 數（Fig. 13） | θ=1 時平均只有 **29.6%** 的時間需要 2 個 relay |

### 5.3 討論（§VII）
* **探測週期**：路徑平均每 **26.2 s（Starlink）/ 26.7 s（Kuiper）** 變一次，最長 124 s / 148 s。路變的原因：接入衛星換了，或風險衛星換了。合適的探測週期留作 future work。
* **部署**：SNO 擁有整個網路，可以在自家衛星部署；終端、衛星、地面站之間已有加密通道；建在 PAN / source routing 上。
* **多個風險區**：在風險區之間設中繼 waypoint，每兩個 waypoint 之間當成一組獨立城市對；風險區很近時可以把 NLRP 合併成一個大框。
* **限制**：終端本身在 NLRP 裡的情況還沒處理（future work）。

## 6. 值得讀的段落
1. **§II-C + Fig. 2–3**：為什麼「延遲 ∝ 距離」在 LEO 失效。這個觀察和 [[The_Nearest_Is_Not_The_Fastest]] 一樣，對任何用延遲推論路徑的方法都有用。
2. **§IV-B b + Fig. 5 + Alg. 1**：NLRP 和三種重疊情況，是整篇最核心、也最有 +Grid 幾何味道的部分。建議對著 Fig. 5 自己走一次 s6→s14、s1→s8、s10→s9。
3. **式 8 和「Discussion of number of relays σ」**：繞路門檻 2·D 的直覺，以及為什麼 relay 不是越多越好。
4. **Alg. 2 + Fig. 7**：封包欄位和三方各自檢查什麼，16 B/relay 的 header 怎麼來。
5. **Table I～III**：延遲代價、header、驗證延遲的數字。
6. **§VII**：路徑平均 26 s 變一次的觀察，可以和其他筆記裡的路由更新頻率對照。

## 7. 和專案裡其他 routing 筆記的關係
* **[[OPSPF_note_latest]]（OPSPF）**：
    * **目標完全不同**：OPSPF 解決的是**可靠性/收斂**（極區 ISL 可預期斷線不要觸發 OSPF 泛洪）；STARVERI 解決的是**安全**（繞開風險區並證明有繞開）。
    * **共同的核心手法：軌道可預測**。OPSPF 每顆衛星用軌道參數預測拓樸、自己跑 Dijkstra；STARVERI 由 **SNO 控制中心**預測拓樸、預先算 relay 和門檻（集中式）。兩者都把「可預測的變化」從即時協定中拿掉。
    * **路由模式不同**：OPSPF 是分散式 hop-by-hop link-state（每顆查自己的路由表）；STARVERI 假設 **source routing / PAN**（路寫在封包裡）。STARVERI 不能直接疊在 OPSPF/OSPF 上，因為 OSPF 每跳自己決定 next hop，來源端無法保證路徑。若要結合，可想成「OPSPF 給出拓樸 → 控制中心排除 RS 後算路 → 以 source route 下發」。
    * OPSPF 用 LEO-48 Walker-Star；STARVERI 用 Starlink/Kuiper 的 Walker-Delta（有 NE/SE 兩個方向的交錯網格），規模大很多。
* **[[OSPF_in_Low_Earth_Orbit_Stability_overhead_and_routing_dynamics]]（notes/routing/ 現有筆記）**：
    * 兩篇都強調 **+Grid 上有大量等長最短路（ECMP）**，但立場相反：OSPF 那篇把它當**好事**（免費備用 next hop，斷線快速恢復）；STARVERI 把它當**麻煩**——等長路裡只要有一條經過風險區，延遲驗證就分不出來，所以限制 (5) 要求 EP ∩ RS = ∅，relay 才會放在角上讓每段只剩一條直線路。
    * 規模/平台：OSPF 那篇 288 顆 Teledesic + FRR 容器；STARVERI 1584/1156 顆 + StarryNet 容器。兩篇都是「每顆衛星一個 Linux 容器」的模擬方式。
    * 路徑變動頻率可以對照：OSPF 那篇 SPF 事件平均約 23 s 一次；STARVERI 量到路徑平均約 26 s 變一次（不同星座、不同定義，只能粗略參考）。
* **[[StableRoute_note_latest]]（StableRoute）**、**[[SIDA_note]]**：都在處理 +Grid「很多條一樣短的路」。StableRoute 認為不必換、SIDA 拿來分流、OSPF 論文當備用、**STARVERI 則刻意消除等長路的歧義**以便驗證。四篇放在一起看很有意思。
* **[[StepClimb_note_latest]]（StepClimb）**：同樣利用 Walker 網格的 (p, n) 座標規律。StepClimb 用座標直接推最短路；STARVERI 的 frame F_sd 其實就是「+Grid 上 s→d 所有最短路都在這個長方形裡」這個性質，和 StepClimb 的分析是同一件事。
* **[[On_distributed_geographic-based_packet_routing_for_LEO_satellite_networks]]（Henderson）**：地理路由每跳貪婪決定，來源無法控制路徑，很難像 STARVERI 這樣證明避開某區域。STARVERI §VIII 也提到 location-based routing（ref [21]，同團隊 MobiCom 2024 的另一篇，不是 Henderson）能避開風險區但「很難證明真的避開」。
* **[[ASER_Scalable_Distributed_Routing_Protocol_for_LEO_Satellite_Networks]]**、**[[A_Disruption_Tolerant_Distributed_Routing_Algorithm_in_LEO_Satellite_Networks]]（DTDR）**、**[[A_Distributed_Survivable_Routing_Algorithm_for_Mega-Constellations_With_Inclined_Orbits]]（Qi 2020）**：這些都是分散式、追求收斂或抗故障的路由；STARVERI 是另一個維度（安全、可驗證），且依賴集中式控制。Qi 2020 和 STARVERI 都處理傾斜軌道（Walker-Delta）。
* **[[The_Nearest_Is_Not_The_Fastest]]**：STARVERI Fig. 2 的「距離和 RTT 非線性」與接入衛星選擇造成的路徑變化，跟這篇的論點相呼應；STARVERI 用最近衛星接入，Src 還特別先探測 D_access 來扣掉第一跳誤差。
* **[[LEO_Satellite_Networking_Relaunched_Survey_and_Current_Research_Challenges]]**：survey 的網路層部分偏重路由效能；STARVERI 補上了「路由安全/路徑驗證」這個 survey 較少著墨的方向。

## 8. 讀後評論
* **優點**：
    * 問題定義清楚，且是 LEO 特有的（路由器本身會飛進別人的國境上空）。
    * 把 crypto-based 和 delay-based 的優點結合得很自然：MAC 只證明「經過 relay」，延遲只負責「段內沒繞路」。
    * NLRP + 角落 relay 很漂亮地利用了 +Grid 幾何，σ ≤ 2 讓 header 固定、和跳數無關。
    * 真實星座參數 + 容器模擬，數字可信度比純數值模擬高。
* **限制 / 質疑點**：
    * **100% 準確率**只在「封包被轉到風險節點」這種劫持場景下測，沒測 FP（正常路但因擁塞抖動被誤判丟包）在高負載下的比例；Fig. 12 顯示大量 flow 時驗證延遲達 ~100 ms 等級，可能接近 δ。
    * **時間同步**：段延遲用不同衛星的時間戳相減，需要精確時鐘同步，論文沒討論。
    * 探測週期（幾十秒）vs 路徑平均 26 s 變一次：探測值可能常常過時，作者自己也留作 future work。
    * NP-hard 的「類比」不嚴謹；DRSA 只是啟發式，沒和最佳解比較延遲差距。
    * 終端在 NLRP 內、多風險區都只有文字討論；relay 本身被入侵則完全沒提。
    * 假設 SNO 可信、建在 source routing 上；和現行 Starlink 的實際路由方式（未公開）是否相容不明。
    * 比較對象 Alibi 用**地面站**當 relay，STARVERI 用**衛星**當 relay；部分優勢來自 relay 位置選擇更自由，不完全是驗證方法本身。
* **對研究的啟示**：如果研究方向是 OSPF/OPSPF 這類分散式路由，這篇提醒了一個常被忽略的需求：**路由不只要快、要穩，在 LEO 還可能要「可驗證地避開某些區域」**。分散式 hop-by-hop 路由要做到這點很難，這是集中式/source routing 的一個優勢。
