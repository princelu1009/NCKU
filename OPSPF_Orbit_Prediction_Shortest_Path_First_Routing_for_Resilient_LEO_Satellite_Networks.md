---
title: "OPSPF: Orbit Prediction Shortest Path First Routing for Resilient LEO Satellite Networks"
authors: [Tian Pan, Tao Huang, Xingchen Li, Yujie Chen, Wenhao Xue, Yunjie Liu]
venue: "IEEE ICC 2019, pp. 1–6 (BUPT；依 STARCURE 論文 ref [14])"
DOI: "未印於 PDF，待查"
tags: [LEO, satellite-routing, OSPF, orbit-prediction, link-state, resilience]
---

## Citation
T. Pan, T. Huang, X. Li, Y. Chen, W. Xue, Y. Liu, "OPSPF: Orbit Prediction Shortest Path First Routing for Resilient LEO Satellite Networks," IEEE, 2019.

## 一句話摘要
LEO 衛星網路的 ISL 會規律地斷線、重連（例如進入極區時）。傳統 OSPF 每次都要等偵測、全網 flood 才能收斂，太慢；snapshot-based 方法要事先存一大堆路由表，又無法處理突發故障。OPSPF 讓每顆衛星用**自己的 GPS 位置推算整個星座的拓樸**，每秒在本地重算路由，因此規律變化完全不需要 flooding；只有**突發故障**時才退回 OSPF 的 Hello/LSU 機制。在 48 顆衛星的模擬中，與 OSPF 相比，封包數少約 57%，收斂時間少約 82%。

## 0. 背景知識
### ISL（Inter-Satellite Link）
- 衛星之間直接通訊的鏈路，現代多用雷射，不必經過地面站轉送。沒有 ISL 時資料只能「上去再下來」（bent-pipe），海洋或偏遠地區難以覆蓋。
- **Intra-plane ISL**：連同一軌道上前後兩顆衛星，相對位置固定，很穩定。
- **Inter-plane ISL**：連相鄰軌道上的衛星，距離和角度一直變，是拓樸變化的主因。
- 一般每顆衛星有 4 條（2 intra＋2 inter），形成 +Grid。
- 「連哪顆衛星」屬於拓樸設計（DoTD、LAMP、Minimum-hop）；「封包走哪條鏈路」屬於路由（OPSPF、StableRoute、SHORT）。

### OSPF（Open Shortest Path First）
地面網路常用的 link-state 路由協定：
1. **Hello**：定期和鄰居打招呼，太久沒回應就判定鏈路斷了。
2. **LSU flooding**：鏈路有變化時，把更新廣播給全網。
3. **LSDB**：每台路由器因此都有一張相同的全網拓樸圖。
4. **SPF**：各自用 Dijkstra 算出 routing table。

### 為什麼極區（緯度 > 70°）的 inter-plane ISL 會斷
- **Antenna tracking（天線追蹤）**：雷射光束極細，傳幾千公里後光點只有幾十公尺寬；兩顆衛星又都以約 7.5 km/s 移動，所以終端必須不斷轉動才能一直對準對方。轉動速度的上限稱為 **slew rate**。
- **Polar orbit 就像地球儀上的經線**：在赤道，相鄰軌道離得最遠且幾乎平行；越往北越靠近，到 90° 全部交會。相鄰軌道的距離大約和 cos(緯度) 成正比，70° 時只剩赤道的約三分之一，而且越往上縮得越快。
- 越靠近極點，鄰居方向變化越劇烈，過極點時還會左右互換，天線轉不過來，鏈路就斷。所以設計上在 |緯度| > 70° 時主動關閉 inter-plane ISL；intra-plane ISL 則不受影響。
- **比喻**：兩人在跑道上跑，你拿雷射筆照著朋友胸前硬幣大小的靶。並肩跑（赤道）時手幾乎不用動；跑道像 X 一樣交叉（極區）時，朋友一兩秒內從右前方跑到左後方，手甩不過來，雷射就照偏了。
- 70° 不是物理定律，是本文依天線追蹤能力選定的邊界值。因為只取決於緯度，所以**完全可預測**，這正是 OPSPF 能把它當成「規律變化」處理的原因。

## 1. Problem & Motivation
- LEO topology 週期性變化：衛星進入 polar region 後，因 antenna tracking 限制，inter-plane ISL 斷開。
- 直接用 OSPF：每次變化都經 Hello timeout 偵測再 flood 全網，造成無止境的 route convergence，耗用 ISL 頻寬。
- Snapshot-based 方法 [4–7]：衛星要在有限 memory 存大量 routing table snapshots，或頻繁與 ground station 互動上傳 snapshot。

## 2. Key Idea
- **Regular change**：每顆衛星只靠自己的 GPS 位置就能預測所有衛星位置，推出瞬時 topology，本地跑 SPF（預設每 1s）。不需 flooding、不存 snapshot。
- **Irregular change**（link failure/recovery）：把 OSPF Hello/LSU 以 on-demand 方式嵌入，只在衛星位於 polar region 外時偵測。
- 只有一張 routing table，由 static module 產生；dynamic module 只回饋 interface flag。

### OSPF vs Snapshot-based vs OPSPF
重新計算的是 **路由**（routing table），不是實體鏈路；ISL 的連上和斷開由軌道幾何決定。

| | Traditional OSPF | Snapshot-based | OPSPF |
|---|---|---|---|
| 核心概念 | 鏈路變了**之後**才重算路由 | **事先**把所有路由算好，時間到就切換 | 每顆衛星**即時**在本地預測拓樸並重算 |
| 流程 | 鏈路斷 → Hello timeout 偵測 → flood LSU → 全網跑 Dijkstra | 地面把一個週期切成多個時間片（snapshot），預算每片的 routing table 並上傳 → 衛星按時間表切換 | 由 GPS 位置推算所有衛星位置 → 推出當下拓樸 → 每 1s 跑 SPF |
| 是否需要軌道資訊 | 不需要 | 需要（在地面離線計算） | 需要（在衛星上即時計算） |
| 規律變化（極區斷線） | 每次都要 flood 和收斂，耗頻寬、會掉封包 | 不需偵測，也沒有收斂延遲 | 不需偵測，也沒有收斂延遲 |
| 突發故障 | 可以處理 | 無法處理（snapshot 裡沒有） | 退回使用 OSPF 的 Hello/LSU 處理 |
| 主要成本 | Flooding overhead、收斂時間（本文為 543 ms） | 衛星 memory 要存大量表格，且需反覆從地面上傳 | 每秒一次的 Dijkstra 運算 |

一句話總結：OSPF 是「鏈路變了**之後**才重算」；snapshot 是「**事先**全部算好，時間到就切換」；OPSPF 則是「可預測的變化用預測**即時**算，不可預測的故障**事後**用 OSPF 補」。

## 3. Method
四個部分的分工：static module 處理可預測的變化，dynamic module 處理突發故障，兩者寫入同一張 routing table。

**3.1 Static workflow（每秒跑一次）**：Geo data（GPS）→ 計算所有衛星位置 → 與 polar boundary（±70°）比對，把極區內的 inter-plane ISL 標記為斷 → 更新 LSDB → SPF（Dijkstra）→ routing table。這就是「即時計算」的部分。

**3.2 Orbit prediction（從自己的位置推出所有人的位置）**：$M$ 個 orbit、每 orbit $N$ 顆，等間距排列；local 為 $S(x,y)$；$d=\pm1$ 代表運動方向（北行／南行）。$c_0=360/N$，$c_1$ = 相鄰 orbit 的 latitude 相位差，$c_2=180/M$。
- 換 orbit：$S(i,y).\lambda=(S(i-1,y).\lambda+c_2+180)\bmod 360-180$，$S(i,y).\varphi=S(i-1,y).\varphi+c_1d$。白話：經度加 $180/M$（polar orbit 只需涵蓋 180° 經度），緯度加上相位差。
- 同 orbit：$S(i,j).\lambda=S(i,j-1).\lambda$，$S(i,j).\varphi=S(i,j-1).\varphi+c_0d$。白話：經度不變，緯度加 $360/N$（一圈平均分配）。
- 摺返：$\varphi>90\Rightarrow180-\varphi$；$\varphi<-90\Rightarrow-180-\varphi$。白話：越過極點後往回走。
- 只靠這三條規則，每顆衛星就能自己推出全部 48 顆的位置和拓樸。

**3.3 Hybrid architecture（銜接 static 和 dynamic）**：六個 module（static routing、dynamic routing、Geo data、LSDB & routing table、local flag、global flag）。
- Local flag ∈ {Failure, OK, Ineffective}：Ineffective 代表在 polar region 內，鏈路本來就該斷，因此不送 Hello；只有 OK 才送 Hello。
- Global flag ∈ {Failure, OK}：由 LSU 得知其他衛星鏈路的故障狀態，回饋給 static module。
- Static module 跑 Dijkstra 時會參考這些 flag，排除標為 Failure 的鏈路。

**3.4 Asynchronous events（類似 OSPF 的部分，處理突發故障）**
1. Failure：每送一個 Hello，Keepalive Counter 減 1，收到 Hello 就 reset；歸 0 → 設 Failure → SPF → flood LSU。
2. Recovery：本地只改 flag、reset、SPF，不主動回報；neighbor 收到來自 Failure/Ineffective interface 的 Hello 後設 OK → SPF → flood。
3. LSU：與 LSDB 一致就丟棄（避免無限 flooding），否則更新 → SPF → 再 flood。
- 最壞收斂時間 $T_c=T_d+T_p+T_s$（Keepalive 上限＋最長傳播時間＋SPF 時間）。

## 3.5 範例：三種方法如何處理同一事件
> 以本文的 LEO-48（6 個 polar orbit × 8 顆，邊界 ±70°）為背景，時間細節為說明用的假設。

封包從台北送到紐約，路徑經過 inter-plane ISL **A–B**。

**事件 1（規律變化）**：t = 100 s 時，A 進入 70°N 以上，A–B 必須斷開。
- **OSPF**：事先不知道。A–B 斷了之後，要等幾個 Hello 週期沒回應才判定斷線，再 flood LSU 給全部 48 顆衛星，大家重跑 Dijkstra。在這之前封包仍被送往 A–B 而遺失（本文：約 543 ms、235 個控制封包）。
- **Snapshot**：地面早已算好「100–130 s 用第 4 張表」，表 4 裡沒有 A–B。t = 100 s 時 A 直接換表，改走 C，不掉封包，但 A 必須存幾十張表。
- **OPSPF**：A 在每秒一次的更新中，由 GPS 算出「我已經超過 70°，A–B 不能用」，自己跑 Dijkstra 改走 C。不掉封包、不 flood（0 個封包），也不存表。

**事件 2（突發故障）**：t = 300 s 時，C–D 的雷射突然故障。
- **OSPF**：和事件 1 一樣，偵測 → flood → 重算。
- **Snapshot**：表上還寫著 C–D 可用，封包持續送往 C–D 而遺失，直到地面重新上傳表格。
- **OPSPF**：預測上 C–D 應該正常，但 Keepalive counter 歸零 → 設 Failure → 重算並 flood LSU（本文：約 100 ms、101 個封包）。

結論：OSPF 每次變化都慢；snapshot 遇到突發故障束手無策；OPSPF 結合兩者的優點。

## 4. Evaluation Setup
- Exata 5.1，4897 行 C（static 3853＋dynamic 1044）；STK 9.2.1 軌跡。
- LEO-48：6 polar orbit × 8 顆，週期約 6900s，polar boundary ±70°。
- Latency：inter-plane 5.6–11.7ms，intra-plane 16.6–16.8ms。
- CBR 100pps、512B；Hello 1 pkt/s；i7-6700、16GB。Baseline 只有 OSPF。

## 5. Main Results
- 單次變化封包數：OSPF 235；OPSPF regular 0；irregular 101（約 −57%）。
- 收斂時間：OSPF 543ms；regular 0；irregular 100ms（約 −82%）。
- 每 2s overhead：OPSPF 約 300（全是 Hello，75 ISL×2×2）；OSPF >700。
- 每 5s 一次 failure/recovery（200s 視窗）：OSPF overhead 與收斂時間都較高。
- Route calculation interval 越大，packet loss 非線性上升，故取 1s。
- End-to-end latency 隨 ISL hop 數波動；hop 相同時仍因 ISL 長度不同而有差異。

## 6. 這篇算是全新的方法嗎？
不完全是，比較像是把既有元件**重新組合**，創新點在整合方式。
- **不新的部分**：OSPF（Hello、LSU、LSDB）和 Dijkstra 原封不動沿用；用軌道可預測性處理拓樸變化，在 snapshot-based / virtual topology 路由中早已存在；polar 幾何（±70° 斷線）也是標準模型。
- **新的部分**：
  - 每顆衛星在**本地**由 GPS 即時推算拓樸，不用存 snapshot，也不用地面上傳，這是和既有 predictive routing 最大的差別。
  - **混合分工**：規律變化靠預測（零 flooding），突發故障才用 OSPF，並以 flag 機制銜接；極區內甚至不送 Hello。
  - 在 Exata 上完整實作並評估。
- 整體偏向「工程設計＋系統整合」型論文，沒有新的最佳化模型或理論證明。它的影響力主要在後續：StableRoute、LAMP 都把 OPSPF 當成 predictive routing 的代表。

## 7. Limitations / 質疑點
**作者自述**
- 沒有 LSU ack/retransmission；可靠版本 overhead 會接近 OSPF，所以 57%／82% 部分來自少做 reliability。
- Polar region 內的 failure 要等離開後才偵測到。

**我的質疑（inference）**
- 預測模型是理想化 polar 幾何（等間距、圓軌道、無 Earth rotation 與 J2），也沒處理 seam 反向 orbit；能否推廣到 Starlink 這類 inclined、多 shell 星座存疑。
- 只有 48 顆，baseline 只有 OSPF，沒和 snapshot-based 方法比 memory 或 latency。
- 每 1s 全網 Dijkstra 在數千顆規模的 CPU 成本與 route flapping 沒評估。
- Link cost 為靜態值，未考慮 congestion 或 load balancing。

## 8. 與其他筆記的關聯
- [[StableRoute_When_Dijkstras_Algorithm_Meets_Topology-Varying_Satellite_Networks]]：同樣在 time-varying topology 上跑 Dijkstra；OPSPF 每 1s 重算不顧路徑穩定性，StableRoute 正處理這個問題。
- [[Stable_Hierarchical_Routing_for_Operational_LEO_Networks]]：OPSPF 是 flat link-state，48 顆無法驗證 operational 規模；SHORT 以 hierarchy 處理可擴展性與穩定性。
- [[Time-_Dependent_Network_Topology_Optimization_for_LEO_Satellite_Constellations]]：OPSPF 把 topology 當給定；DoTD 主動設計 ISL topology，可作為 OPSPF 預測模組的上游。
- [[LAMP_Low-Latency_Dynamic_Topology_for_LEO_Satellite_Constellations]]：若 topology 改為動態設計，所有衛星需共享同一個 deterministic 規則，OPSPF「純幾何推得 topology」的前提才成立（inference；LAMP 原文也提到 offline links 適合 OPSPF 這類 predictive routing）。LAMP 的 slew 模型（1°/s，轉動期間鏈路不可用）正是對 antenna tracking 成本的建模。
