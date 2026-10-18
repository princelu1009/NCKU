---
title: "ASER: Scalable Distributed Routing Protocol for LEO Satellite Networks"
authors: [Xiang Zhang, Yuan Yang, Mingwei Xu, Jing Luo]
venue: "IEEE 46th Conference on Local Computer Networks (LCN), 2021, pp. 65–72"
doi: 10.1109/LCN52139.2021.9524989
survey_ref: "LEO survey 2310.07646 [178]（§4.2 IP-based routing，OSPF 改良）"
tags: [LEO, satellite-routing, ISL, ASER, hierarchical-routing, OSPF, link-state, area, Walker-Star, polar-orbit, scalability]
---

## Citation
X. Zhang, Y. Yang, M. Xu, J. Luo, "ASER: Scalable Distributed Routing Protocol for LEO Satellite Networks," in *Proc. IEEE LCN*, 2021, pp. 65–72. DOI: 10.1109/LCN52139.2021.9524989（清華大學計算機系、北京信息科學與技術國家研究中心、鵬城實驗室、ByteDance）

## 一句話總結
**ASER（Area-based SatellitE Routing）** 要解的問題是：大型 LEO 星座拓樸變化太頻繁（450 顆以上時，最長每不到 20 秒就變一次），傳統 OSPF 每次變化都要全網 flood LSA、全網重算，收斂慢、開銷大。做法是**階層式（hierarchical）OSPF**：把衛星切成固定的 **l × m 區（area）**（l 條相鄰軌道 × 每條軌道 m 顆連續衛星），區的成員永遠不變（跟著衛星一起移動）。關鍵設計是讓 **m > W**（W = 極區 link handover region 在一條軌道上最多同時涵蓋幾顆衛星），這樣不管衛星飛到哪，相鄰兩區之間**至少還有一條 IOL 是通的**，**區與區的鄰接關係永遠不因 handover 改變** → inter-area routing 幾乎不用動，handover 只觸發**區內**的 LSA flooding 和重算。最後用 ASER-FT 演算法把 inter-area 表和 intra-area 表**合併成一張普通的 IP forwarding table**，資料平面照用 longest-prefix match。NS2 模擬四個 Teledesic-like 極軌星座（288～1296 顆）：收斂時間約為 OSPF 的 50%～75%、計算和訊令開銷最低；但吞吐量、掉包、延遲只比 OSPF 好，比 ALBR/CEMR 差。

## 0. 背景知識
### Link handover 與 LHR（§I，Fig. 1–3）
* **Inter-orbit link（IOL，跨軌鏈路）**：同軌衛星彼此相對靜止，跨軌衛星不是；天線指向範圍是一個**圓錐**（半頂角 θ），追不上時只好關掉，進入極區後重新建立。作者把「關掉再重建」叫 **link handover**。
* Fig. 1/2：handover 發生的緯度隨軌道數、每軌衛星數**變高**（例：θ = 60°、每軌 10 顆時約 72°～85°；θ 越小緯度越低，θ = 30° 時可低到 45°～65°）。也就是衛星越多，IOL 要關的區域越小，**但永遠存在**。
* Fig. 3：拓樸變化的最大間隔隨總衛星數迅速下降：500 km、100 顆時約每 80 s 變一次；超過 450 顆時 < 20 s。這是本文的核心動機。
* 這和 [[OPSPF_Orbit_Prediction_Shortest_Path_First_Routing_for_Resilient_LEO_Satellite_Networks]] 的假設一致（極區 IOL 會斷），和 [[A_Disruption_Tolerant_Distributed_Routing_Algorithm_in_LEO_Satellite_Networks]]（DTDR，極區 IOL 不斷）相反。

### OSPF 的 area 是什麼（背景，非論文內容）
* 標準 OSPF 本來就有 area：area 內 flood 完整 LSA，area 之間只交換摘要（Type-3 summary LSA），所有 area 要接到 backbone area 0。ASER 借用「限制 flooding 範圍」的想法，但**不用 backbone**，改成新的 inter-area LSA 描述「區與區的鄰接關係」，全網 flood。

## 1. Problem
* **Problem**（§I Motivation）：
    * **Proactive（snapshot）路由**：拓樸變得越快，snapshot 越短、越多，預算和儲存開銷越大；而且 ISL 實際建立/關閉的時間可能比預測早或晚，預算的路由就失效（§II 對 [19][20] 的批評）。
    * **Reactive 路由（OSPF 這類）**：每次變化都要通知全網或集中控制器並重算，變化越頻繁開銷越大。
    * 根本原因：路由要在**全域拓樸**上算才最佳，但很多拓樸變化其實只影響少數衛星的路由 → 不需要全域感知。
    * 既有分散式方法（Ekici [21]、Henderson [22]、Soret [23]、Liu [24]）幾乎不用訊令，但對**非預期故障**韌性有限；既有階層式（DHRP [16]、CEMR [17] 用 plane speaker；NetGrid [18]）在大星座下因頻繁變化而效率差。
* **Target**：一個**完全分散式**、**intra-domain** 的路由協定，同時應付**規律的 handover** 和**非預期的鏈路/節點故障**，且收斂快、開銷低、可擴展。
* **有沒有正式 formulation**：只有區大小的最佳化問題（§III-B，Eq. 5–8），目標函數 f1、f2 沒有給具體形式。協定本身是設計＋模擬，沒有證明。

## 2. Method
### 2.1 全貌（§III-A）
* 每顆衛星（router）維護兩張表：
    1. **Intra-area routing table**：到同區每個 prefix 的 next hop（也包含**相鄰區邊界節點**的路由）。
    2. **Inter-area routing table**：到目的區要先往哪個「下一區」走。
* 兩張表都用 link-state 協定（改過的 OSPF）維護，最後合併成一張 forwarding table（§III-E）。
* 「區是虛擬的」：一顆衛星**永遠屬於同一區**。所以區在地理上會跟著衛星移動，但成員不變，不需要重新分區。
* 設計目標兩條：(1) **區之間的鄰接關係不因 link handover 改變**；(2) 每區夠小，區內路由能快速、低開銷地重建。

### 2.2 區的劃分（§III-B，Eq. 4–8）
* 每區是 **l × m 的 mesh**：**l 條相鄰、同向旋轉（co-rotating）的軌道**，每條軌道取 **m 顆相鄰衛星**。p 不能被 l 整除、q 不能被 m 整除時，允許少數區大一點。
* **l ≥ 2、m ≥ 2**：讓相鄰兩區之間至少有兩條 ISL，單一鏈路故障不會把兩區切開。
    * 沿軌道方向相鄰的兩區：由 l 條**同軌 ISL** 相連，同軌鏈路不會 handover。
    * 跨軌方向相鄰的兩區：由最多 **m 條 IOL** 相連，這些會在極區關掉。
* **LHR（link handover region）**：IOL 關閉的區域（極軌星座就是兩個極冠）。**LHR width W** = 同一條軌道上**同時**落在同一個 LHR 裡的衛星最多幾顆。極軌 Walker 星座：

  W = ⌊ q × 2(90° − δ) / 360° ⌋ + 1　　（Eq. 4）

  其中 q 是每軌衛星數、δ 是 link handover 緯度。白話：一個極冠在軌道上佔 2(90° − δ)/360° 的比例，乘上每軌顆數就是落在裡面的顆數，+1 是因為衛星位置和極冠邊界不一定對齊。
* **m > W 的條件（本篇的核心）**：跨軌相鄰的兩區最多由 m 條 IOL 相連，最壞情況下同時有 W 條在 LHR 裡被關掉。只要 **m > W**，就至少剩 m − W ≥ 1 條是通的 → **兩區永遠相連，鄰接關係不變，inter-area 路由不受 handover 影響**。同樣的理由也保證**區內**在最壞情況下仍連通（每對相鄰軌道之間至少一條 IOL 在 LHR 外）。
    * Fig. 4 例子：W = 3，每區 4 × 2（m = 4 > 3）。
    * m 越大韌性越好（可容忍更多同時故障），但區越大開銷越高。
    * ⚠ 文字寫 m > W，但最佳化問題的限制式 Eq. 7 寫 **m ≥ max{2, W}**。照文字的理由，m = W 時可能剛好 W 條全關、兩區斷開，所以應該是 **m ≥ W + 1**；Eq. 7 是寫錯或少了 +1（推論）。本文模擬的設定都滿足 m > W（見下表）。
    * ⚠ 論文方程式從 (4) 開始編號，沒有 (1)–(3)。
* 我用 Eq. 4 驗算 Table I/II 的設定（推論，論文沒列 W）：

| 星座 | q | δ | W | 預設區大小 l × m | m > W？ |
|---|---|---|---|---|---|
| 288（12 軌 × 24） | 24 | 60° | ⌊4⌋+1 = 5 | 6 × 8 | ✓ |
| 576（12 × 48） | 48 | 60° | ⌊8⌋+1 = 9 | 6 × 16 | ✓ |
| 864（36 × 24） | 24 | 75° | ⌊2⌋+1 = 3 | 12 × 12 | ✓ |
| 1296（18 × 72） | 72 | 75° | ⌊6⌋+1 = 7 | 9 × 24（另試 m = 36/24/12/8） | ✓（最小 m = 8 > 7） |

* **區大小最佳化**（Eq. 5–8）：min f1(m, l) + f2(n)，s.t. n·l·m = s，l ≥ 2，m ≥ max{2, W}，整數。f1 = 每個節點的區內開銷，f2 = 每個節點的區間開銷，n = 區數，s = 總衛星數。
    * 例：只想縮短區內收斂時間 → 最小化 l + m（區的直徑）。
    * 處理 LSA 的開銷：區內 O(m × l)、區間 O(n)。
    * 論文**沒有真的解這個問題**，模擬直接固定「6 區」。

### 2.3 Intra-area routing（§III-C）
* 和 OSPF 幾乎一樣：每個 router 產生 LSA，描述自己連的 prefix、鄰居、每條鏈路的 cost（可自訂，用來做 QoS）。
* **和 OSPF 不同的地方**：LSA 裡帶有**每個鄰居 router 的 area ID**（area ID 依分區事先設定）。這是後面合併 forwarding table 的關鍵：看到哪些鄰居屬於別區，就知道邊界在哪。
* **Intra-area LSA 只 flood 到同區**，別區收不到。每個 router 用 Dijkstra 在區內 LSDB 上算最短路。
* 拓樸變化的兩種情況（Fig. 5）：
    * (a) **區內**鏈路 A–B 斷/恢復 → A、B 產生新 LSA，只在該區 flood（只有 A、B、C、D 收到）。
    * (b) **跨區**鏈路 B–E 斷/恢復 → B 的 LSA 在 B 那區 flood、E 的 LSA 在 E 那區 flood，**各自分開**。
* Intra-area 表的每一筆是五元組 **⟨dst, mask, nexthop, area, cost⟩**；`area` = 該 prefix 屬於哪一區。表裡也有**相鄰區邊界節點**的條目（因為它們是本區 router 的鄰居，出現在 LSA 裡）。

### 2.4 Inter-area routing（§III-D）
* Router 看 LSA 裡鄰居的 area ID，就知道本區鄰接了哪些區。
* 新的 **inter-area LSA**：內容是 ⟨本區 ID, 鄰區 ID, 區間 cost⟩。任何 router 都能產生，但為了減少數量，每區**選一個 designated router（DR）當 speaker** 來發。
* **Inter-area LSA 全網 flood**，每個 router 在「區的圖」上算出 inter-area 表，每筆是 **⟨dst_area, next_area⟩**。
* 因為 m > W 保證鄰接關係不受 handover 影響，inter-area LSA **只在真的需要時**（例如非預期故障把兩區完全切斷）才重發，平常幾乎不動。
* ⚠ DR 怎麼選、DR 故障怎麼辦，論文沒說（推論：沿用 OSPF 的 DR 選舉）。

### 2.5 Packet forwarding 與定址（§III-E，Fig. 6）
* 轉送到**同區**：直接查 intra-area 表。
* 轉送到**別區**：(1) 查出目的地在哪一區 → (2) 查 inter-area 表得到 next area → (3) 在 intra-area 表裡找「通往 next area 的 next hop」。
* 「目的地在哪區」的兩種做法：靜態的 address→area 對照表（假設每區持有固定的 prefix 集合），或更漂亮的：**把 area ID 直接編進 IPv6 位址**：

  | Global routing prefix（32–48 bits）| Subnet ID（16–32 bits）= **Area ID** ＋ Satellite ID | Interface ID（64 bits）|

  這樣看位址就知道目的區，而且同一區的所有位址共用一個前綴 → 一條 forwarding entry 就能涵蓋整區。

### 2.6 Forwarding table 合併：ASER-FT（§III-E，Algorithm 1，Fig. 7）
* 目的：把三步查表**合成一次 longest-prefix match**，資料平面和地面 IP router 完全一樣。
* 輸入：global routing prefix `pref`、inter-area 表 P、intra-area 表 Q；輸出：forwarding table F。
* 步驟：
    1. F ← 空（Step 1）。
    2. **本區條目**：Q 裡每一筆 ⟨dst, mask, nexthop⟩ 直接放進 F（Step 2–5）。
    3. 算一次 `new_mask`（Step 6）：區外的目的前綴長度都一樣（global prefix ＋ area ID 的長度），所以只算一次。
    4. **區外條目**：對 P 的每一筆 ⟨dst_area, next_area⟩（Step 7–20）：
        * 掃 Q，在所有 `area == next_area` 的條目裡挑 **cost 最小**的那筆（Step 10–16）。
        * 目的前綴 = `pref ⊕ dst_area`（⊕ = 串接，Step 17）。
        * F 加入 ⟨pref ⊕ dst_area, new_mask, best_rt.nexthop⟩（Step 18–19）。
* 複雜度：P 有 O(n) 筆、Q 有 O(m × l) 筆 → **O(n × m × l) = O(s)**。
* **Fig. 7 例子**：global prefix `a:b::/32`，area ID 8 bits。inter-area 表：dst 0x0A → next 0x02。intra-area 表裡 area = 0x02 的有 d2（cost 3, h1）、d4（cost 4, h3）、d5（cost 2, h4），最小 cost 是 d5 → next hop **h4**。合併後的條目：**`a:b:0A00::/40 → h4`**（mask 32 + 8 = 40 bits）。
* 白話比喻：寄信到別縣市，郵局不需要知道對方門牌，只需要知道「往那個縣市要先送去哪個鄰縣」，再找「離我最近的那個鄰縣交界口」。
* 小細節：Step 11 的判斷是 `i.cost > min_cost` 才跳過，所以 cost 相同時會選**後面**那筆（不影響正確性）。

## 3. Evaluation Setup（§IV-A）
* 模擬器：**NS2.35**（Ubuntu 16.04 VM，i5 雙核 2 GHz、12 GB）。
* 星座：四個 **Teledesic-like 極軌**星座（Table I）；第一條和最後一條軌道只有 3 條 ISL（seam 兩側反向，不建鏈），其他 4 條。

| 規模 | 高度 | 軌道數 | 每軌顆數 | Handover 緯度 |
|---|---|---|---|---|
| 288 | 1375 km | 12 | 24 | 60° |
| 576 | 1375 km | 12 | 48 | 60° |
| 864 | 780 km | 36 | 24 | 75° |
| 1296 | 1375 km | 18 | 72 | 75° |

  Handover 緯度是「經驗決定」，作者說和 Fig. 1–2 的分析一致。
* ASER 預設 **6 區**（288: 6×8、576: 6×16、864: 12×12、1296: 9×24）。1296 另試 4/6/12/18 區（Table II：l 固定 9，m = 36/24/12/8）。
* **所有 link cost = 1**（區內和區間都是），也就是最少 hop。
* Baseline：**OSPF** [31]、**ALBR** [5]（agent-based load balancing）、**CEMR** [17]（plane speaker 的階層式多路徑）。
* 訊令大小的約定：OSPF/ASER 每條 ISL 的 LSA = 10 units；ALBR 每一跳路徑資訊 = 10 units；CEMR 每筆路由條目 = 20 units。
* 兩組實驗：
    1. **無使用者流量**，跑一個軌道週期：量收斂時間、路由計算時間、訊令量。
    2. **有流量**（只在 288 顆星座）：每個 flow 200～1200 kbps、封包 1000 bytes、**TCP**；使用者數和起終點依 [32] 的地理/時區分布隨機決定；每個 flow 持續一小段，**期間使用者和接入衛星的連線不變**；ISL FIFO queue 長度 700、ISL 25 Mbps、下行 30 Mbps；跑 24 小時。

## 4. Main Results（§IV-B；數字是我從圖讀的，近似值）
* **收斂時間 vs. 網路規模**（Fig. 8）：ASER 每次拓樸變化都在 **100 ms 內**收斂（約 65～70 ms）；OSPF 約 95～120 ms；ALBR 約 260～290 ms；CEMR 210 → **860 ms**（1296 顆時）。ASER 約為 OSPF 的 50%～75%（論文說法）。各方法的收斂時間幾乎不隨規模變，因為沒有背景流量、flood 很快。
* **收斂時間 vs. 區數**（Fig. 9，1296 顆）：ASER(4) ≈ 75 ms、ASER(6) ≈ 55 ms、ASER(12) ≈ 30 ms、ASER(18) ≈ 25 ms；OSPF ≈ 100 ms、ALBR ≈ 280 ms、CEMR ≈ 1000 ms。**區越多（越小）收斂越快**。
* **路由計算時間**（Fig. 10–11）：ASER 一律 **< 1 ms**（約 0.1～0.6 ms）；OSPF 約 0.4～2.5 ms；ALBR 約 250～300 ms（其實是找路時間，被傳播延遲主導）；CEMR 10 → 170 ms。1296 顆時 ASER(18) ≈ 0.3 ms vs. OSPF ≈ 2.5 ms。
* **訊令量**（Fig. 12–13，M units）：ASER 最少。1296 顆時：ASER(6) ≈ 0.65、OSPF ≈ 4、ALBR ≈ 9、CEMR ≈ 500；ASER(18) ≈ 0.2。
    * ⚠ 摘要說「computation and control overhead 降低好幾個數量級」，**對 CEMR/ALBR 成立，對 OSPF 只有約 4～20 倍**（訊令）和約 4～8 倍（計算）。
    * ALBR 的訊令量是作者簡化實作的結果（只有拓樸變化才發訊令，queue 變化不發），實際會更多（作者自己承認）。
* **吞吐量**（Fig. 14）：1200 kbps 時 CEMR ≈ 400、ALBR ≈ 365、ASER ≈ 355、OSPF ≈ 305 Mbps。作者說差距大致在 10% 內。
* **掉包率**（Fig. 15）：1200 kbps 時 OSPF ≈ 2.1%、**ASER ≈ 1.6%**、CEMR ≈ 0.35%、ALBR ≈ 0.15%。
* **端到端延遲**（Fig. 16）：200 kbps 時大家都約 75 ms；1200 kbps 時 OSPF ≈ 138、**ASER ≈ 118**、CEMR ≈ 93、ALBR ≈ 82 ms。
* 作者的解釋：ALBR/CEMR 用**多個 next hop** 做負載平衡（代價是亂序）；ASER 不同 flow 可以走不同的區間路徑，但**單一 flow 不拆**。

## 5. Novelty／貢獻
1. **用星座結構設計區**：區不是任意切，而是 l 條軌道 × m 顆，並用 **m > W** 保證「handover 永遠只影響區內」。這是和 DHRP/CEMR（以軌道為單位，handover 還是會影響上層）最大的不同。
2. **兩層都是 link state**：handover 之外的非預期故障也能處理（相對 Ekici/Henderson 這類無訊令方法）。
3. **改 OSPF 而不是另起爐灶**：LSA 加 area ID、新增 inter-area LSA、DR 當 speaker。
4. **ASER-FT**：把階層式路由壓回一張標準 forwarding table，O(s)，資料平面不用改。
5. **把 area ID 編進 IPv6 位址**，一區一條 forwarding entry。

## 6. Limitations
* **路徑不是全域最短**（推論）：inter-area 路由只在「區的圖」上算（cost 全 = 1），再挑「到下一區最近的邊界節點」（類似 hot-potato）。區越大、區間 cost 越粗，繞路越多。論文**沒有量 path stretch**（hop 數或傳播延遲和全域最短路的比較）；Fig. 16 低負載時延遲和其他方法差不多，但高負載時比 ALBR/CEMR 高約 25～35 ms。
* **掉包和延遲輸給 ALBR/CEMR 不少**：1200 kbps 時 ASER 掉包約為 ALBR 的 10 倍。作者用「a little worse」帶過，偏輕描淡寫。
* **區大小最佳化沒有真正做**：f1、f2 沒給具體形式，模擬直接用 6 區；Eq. 7 的 m ≥ W 和文字的 m > W 不一致。
* **只考慮極軌（Walker Star）**：Eq. 4 只對極軌成立。Walker Delta（例如 Starlink 53°）沒有極冠 LHR，IOL 的變化型態不同，論文沒討論怎麼算 W（推論：傾斜軌道可能根本不需要 m > W，但跨 seam 的問題也不同）。
* **沒處理地面–衛星 handover**：流量實驗刻意讓「每個 flow 期間使用者的接入衛星不變」。但實際上地面使用者幾分鐘就換一顆衛星，換到別區時位址前綴（area ID 編在位址裡）就對不上了，這正是 survey §4 討論的 IP 定址問題（推論）。
* **IOL 恢復時機**：handover 把 IOL 打開/關閉時還是要在區內 flood LSA 並重算。作者說「每次 < 100 ms」，但沒有量這段期間的封包遺失（OPSPF 則是用預測做到可預期變化 0 收斂時間）。
* **DR 選舉與 DR 故障、seam 兩側的區、q 不能被 m 整除時大區怎麼處理**：都只有一句話或沒寫。
* **模擬規模和速度偏小**：最大 1296 顆，ISL 只有 25 Mbps；流量實驗只在 288 顆跑。沒有和 OPSPF 比（雖然有引用 [25]）。
* 計算時間在單一 VM 上量，和衛星上的處理器差很多（推論）。

## 7. 與其他筆記的關聯
### 和 OPSPF（[[OPSPF_Orbit_Prediction_Shortest_Path_First_Routing_for_Resilient_LEO_Satellite_Networks]]）
* 兩篇都是「改 OSPF 去適應 LEO」，ASER 把 OPSPF 列為 [25]，但沒有拿來比。
* 兩種完全不同的切法：

| | OPSPF | ASER |
|---|---|---|
| 怎麼處理可預期的 handover | **時間**：用軌道預測，事先知道誰會斷，不發 LSA | **空間**：把影響關在區內，還是要發 LSA，但只 flood 一區 |
| 可預期變化的收斂 | 約 0 | < 100 ms（區內） |
| 非預期故障 | 全網 LSU | 區內 LSA；只有區被切斷才發 inter-area LSA |
| 路徑最佳性 | 全域最短 | 區階層下的近似 |
| 星座 | 極軌，±70° 斷 | 極軌，60°/75° 斷 |

* 兩者其實可以疊加（推論）：區內用 OPSPF 的預測，連區內 flood 都省掉。

### 和 survey（[[LEO_Satellite_Networking_Relaunched_Survey_and_Current_Research_Challenges]]、[[LEO_Survey_2310.07646_Network_Layer_Deep_Dive]]）
* Survey 把 ASER 列為 [178]，放在 §4.2 IP-based routing。
* ⚠ Survey 筆記裡寫「區內拓樸當靜態，區之間用 OSPF 動態連接」，讀完原文**剛好相反**：**區內**才是一直在跑 link state、會隨 handover 更新的部分；**區之間**的鄰接關係才是被 m > W 設計成「幾乎靜態」的部分。兩層都用 OSPF 式的 LSA。
* Deep Dive §2.6 表格寫 ASER 的弱點是「區的邊界本身會隨衛星移動」，要修正：區的**成員**是固定的（虛擬區），邊界在地理上移動不影響路由；真正的問題是**地面使用者換接入衛星時會跨區**（見 §6）。

### 和 DTDR、Qi 2020（[[A_Disruption_Tolerant_Distributed_Routing_Algorithm_in_LEO_Satellite_Networks]]、[[A_Distributed_Survivable_Routing_Algorithm_for_Mega-Constellations_With_Inclined_Orbits]]）
* 共同想法：**限制故障資訊的 flooding 範圍**。DTDR/Qi 是「以故障點為中心 flood n 跳」，ASER 是「flood 到整個區」。前者範圍跟著故障走，後者範圍固定、可以配 Dijkstra 和標準 OSPF。
* DTDR 假設極區 IOL 不斷，ASER 假設會斷，模型又不一致。

### 其他
* [[On_distributed_geographic-based_packet_routing_for_LEO_satellite_networks]]（Henderson [22]）：ASER 在 §II 把它歸為「分散式、幾乎不用訊令、但對故障韌性有限」的那一類。
* [[StableRoute_When_Dijkstras_Algorithm_Meets_Topology-Varying_Satellite_Networks]]：ASER 的 inter-area 路由幾乎不變，天生比較「穩定」；但區內每次 handover 重跑 Dijkstra，next hop 可能跳換，這正是 StableRoute 想避免的（推論，ASER 沒量換路次數）。
* [[SIDA_note]]（SIDA/SSLB）、[[Shortest_Path_in_LEO_Satellite_Constellation_Networks_An_Explicit_Analytic_Approach]]（StepClimb）：同樣是 +Grid 極軌模型，但都在單層網格上做，沒有分區。
