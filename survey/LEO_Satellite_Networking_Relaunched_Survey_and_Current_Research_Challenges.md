---
title: "LEO Satellite Networking Relaunched: Survey and Current Research Challenges"
authors: [Cedric Westphal, Lin Han, Richard Li]
venue: "arXiv 2310.07646v1 [cs.NI], 11 Oct 2023（Futurewei；PDF 上沒有期刊／會議資訊）"
DOI: "arXiv:2310.07646"
tags: [LEO, survey, satellite-routing, SDN, load-balancing, 3GPP-NTN, IETF, TVR, standardization]
---

## Citation
C. Westphal, L. Han, R. Li, "LEO Satellite Networking Relaunched: Survey and Current Research Challenges," arXiv:2310.07646v1 [cs.NI], Oct. 2023.

## 一句話摘要
這是一篇**網路層的入門 survey**，不是新方法。它把 LEO 衛星網路從「bent-pipe（衛星只當反射鏡）」走到「有 ISL、資料在太空中多跳轉送」之後的研究，分成 11 個主題整理（IP、負載平衡、SDN、機器學習、容錯、地理路由、低延遲、多層衛星、ICN／MEC、模擬平台），每節最後有一段作者自己的 **"What next?"** 看法；後半段介紹 **3GPP NTN** 和 **IETF**（特別是 **TVR，Time-Variant Routing 工作組**）的標準化進度。對這個專案最有用的地方：它**直接點名了 OPSPF 和 StableRoute**，並把它們放進「IP／OSPF 怎麼改才適合衛星」這條脈絡裡；也提供了很多值得追的 reference（Bhattacherjee 的 motif 拓樸設計、Hypatia 模擬器、Handley 的低延遲路由）。

**比喻**：如果 OPSPF、StableRoute、StepClimb 是三間各自專精的餐廳，這篇就是一本美食地圖：每間只寫兩三行，但告訴你它們在哪一區、旁邊還有哪些店。

## 0. 背景知識
### Bent-pipe vs ISL（摘要第一段）
| | Bent-pipe（透明轉發） | 有 ISL（多跳轉送） |
|---|---|---|
| 資料怎麼走 | 地面 → 衛星 → **馬上回地面** | 地面 → 衛星 → 衛星 → … → 衛星 → 地面 |
| 衛星要不要「路由」 | 不用，只做 RF 濾波、變頻、放大 | 要，衛星變成太空中的 router |
| 對地面站的依賴 | 高：衛星底下一定要有地面站 | 低：可以飛過海洋再下來 |
| 3GPP 對應 | transparent payload（§5.1 第一型） | regenerative payload（§5.1 第二、三型） |

整個專案的路由論文（OPSPF、StableRoute、StepClimb）都假設是右邊這種。

### Snapshot routing vs Dynamic routing（§2）
這篇把衛星路由分成兩大類，和 OPSPF 筆記的分法一致：
- **Snapshot routing**：把時間切成一段段，每段內拓樸當作不變，事先算好路由表上傳給衛星。理由是封包傳一次只要幾 ms，而衛星在地面站視野內可以待好幾分鐘，所以「一段時間內不變」是合理的。
- **Dynamic routing**：衛星依當下的網路狀態轉送，協定要自己適應拓樸變化。

作者在 §4.4 補了一句：**SDN 可以看成兩者的混合**，因為轉送規則本身是靜態的，但可以被 controller 動態改寫。

### 為什麼 LEO 能比光纖快（§3）
- 地面到 1,000 km 軌道只要約 3 ms。
- 在 1,000 km 高度走兩點的最短路徑，只比地面上對應的路徑長約 15%（就是半徑比 (6371+1000)/6371 ≈ 1.157）。
- 光在光纖裡只有真空的約 2/3（論文用 1.47 倍），所以「多走 15%，但快 1.47 倍」，長距離會淨賺。
- 註腳的經驗法則：地面距離大約超過**衛星高度的兩倍**就開始划算（500 km 高度 → 約 1,000 km）。這個數字我覺得偏樂觀，見 §7。

### Walker Star / Walker Delta / Rosette（§3，Fig. 2–4）
- Fig. 2 畫的是 20 個軌道面 × 每面 20 顆的 Polar Star 星座，一半衛星往北、一半往南（就是 OPSPF 筆記裡講的 **counter-rotating seam**）。
- Fig. 3 對比 Polar Star 和 Walker Delta（Rosette）。Walker Star／Delta 的差別細節見 [[OPSPF_Orbit_Prediction_Shortest_Path_First_Routing_for_Resilient_LEO_Satellite_Networks]] 的背景知識。
- 另外提到 **F-Rosette [101]**：利用「衛星地面軌跡固定且週期重複」來設計 ISL 連法、定址和路由。

## 1. 這篇要做什麼（定位）
- **目標讀者**：想快速進入 LEO 網路層研究的人。作者明說它和 [61]（MAC／PHY）、[34]（3GPP）、[25]（dynamic routing）、[169]（1990 年代以來的完整歷史）互補，自己的重點是「**近年**的新趨勢＋標準化」。
- **沒有 problem formulation、沒有實驗**。每個主題就是「列幾篇代表作＋一段 What next?」。
- **作者背景**：三位都在 Futurewei（華為美國研發）。Lin Han、Richard Li 是 New IP 和衛星 semantic addressing 的 IETF draft 作者，所以 §4.2 和 §5.2 有自家工作的推廣成分（見 §7）。

## 2. 全文架構

| 節 | 內容 | 和本專案的相關度 |
|---|---|---|
| §3 | LEO 基本概念、星座類型、Starlink／OneWeb／Telesat 現況 | 中（背景） |
| §4.1 | 早期里程碑：FSA、snapshot、ATM、datagram routing、LAOR | **高** |
| §4.2 | IP-based：OSPF 改良（**OPSPF、StableRoute**、ASER）、BGP、New IP | **最高** |
| §4.3 | 負載平衡（ELB、SIDA、HLBR、DBPR、segment routing…） | 高 |
| §4.4 | SDN-based（controller 放 GEO／地面／衛星上） | 中 |
| §4.5 | 機器學習路由（DDQN、Q-learning、MARL） | 中 |
| §4.6 | 容錯／DTN（DODR、DTDR、survivable routing） | 高（對照 OPSPF 動態模組） |
| §4.7 | 地理路由 | 中 |
| §4.8 | 低延遲網路 | 中 |
| §4.9 | 多層衛星（LEO/MEO/GEO） | 低 |
| §4.10 | 其他：應用、節能、ICN、MEC | 低 |
| §4.11 | 模擬平台與模型（SaVi、**Hypatia**、**motif 拓樸設計**） | **高** |
| §5.1 | 3GPP NTN（transparent vs regenerative payload） | 低～中 |
| §5.2 | IETF（**TVR WG**、LISP、semantic addressing、instructive routing） | **高** |
| §6 | 未來挑戰：可靠度、垂直整合、應用支援、互通性 | 中 |

## 3. 重點內容（只挑和路由／拓樸有關的）
### 3.1 早期里程碑（§4.1）：「拓樸可預測」這件事 1995 年就有人用了
- **FSA（Finite State Automaton）[27, 28]，1995/1998**：星座拓樸只會在有限個狀態之間輪換，所以可以事先把每個狀態的「拓樸設計＋路由」算好，衛星只要依目前狀態載入就好，**完全不用在軌計算**。
- **Snapshot [58]，1999**：每顆衛星存幾個 snapshot 的路由表，定期由地面上傳，用 label switching（受 ATM 啟發）。
- **ATM VPC [152]**：直接為整個軌道週期預算路徑，等於一組「隨時間變化的路由表」。
- **Datagram routing [41, 42]，2000**：分散式、逐封包決定，以傳播延遲為目標，同時考慮壅塞。因為是逐封包，**鏈路斷了也沒有 handover 問題**。
- **LAOR [121]**：把 AODV 搬上衛星，按需求找路；成本是每個請求都要做一次路徑發現。

**我的觀察**：OPSPF 的「用軌道預測取代 flooding」和 StableRoute SR_G 的「一次規劃整個週期」，本質上都是 FSA／snapshot 這條老路線的現代版本。差別在於 OPSPF 是**每顆衛星自己算**，不是地面上傳。

### 3.2 IP-based 解法（§4.2）：本專案的論文出現在這裡
作者的立場：TCP/IP、BGP、OSPF 都是為地面設計的，直接搬上衛星不適合。
- **[100]** 分析：IP 協定照原樣用，網路可用率會掉到 **20% 以下**（但解法只適用 F-Rosette）。
- **ASER [178]**：把 OSPF 分區（area），區內拓樸當靜態，區之間用 OSPF 動態連接。
- **OPSPF [119]**：「把 OSPF 改成考慮拓樸週期可預測，解決 OSPF 無止盡收斂、浪費 ISL 頻寬的問題。」→ 見 [[OPSPF_Orbit_Prediction_Shortest_Path_First_Routing_for_Resilient_LEO_Satellite_Networks]]
- **StableRoute [111]**：「改良 Dijkstra，在等成本的路徑中選更新次數最少的。」→ 見 [[StableRoute_When_Dijkstras_Algorithm_Meets_Topology-Varying_Satellite_Networks]]
- **[108]**：也是改 Dijkstra，但針對**沒有 ISL** 的大型網路（作者特別註明：這篇 survey 裡其他所有論文都假設有 ISL）。
- **[177]**：link-state 演算法，分三階段：建立拓樸 → 算路由 → 回應故障。
- **BGP 的問題 [32, 57, 44]**：太空／地面鏈路斷斷續續，BGP peering 效能掉得很嚴重；BGP-S 是早期修補版本。
- **New IP [81, 95, 97, 66]**：用可變長度地址，依「軌道殼層、軌道面、在軌道上的位置」幫衛星定址（作者自家工作）。

**What next?（作者觀點）**：IP 用在衛星上有張力。現在的星座都是**私有、不互通**的；Starlink 的商業優勢是發射比別人便宜，互通反而會削弱先行者優勢。但監管機關可能會強制互通。

### 3.3 負載平衡（§4.3）
**為什麼需要**：衛星均勻分布在全球，但人口不是。太平洋上空的衛星幾乎沒有本地流量（頂多有上海–洛杉磯的過境流量）；人口密集區之間的鏈路則很擠。

| 方法                    | 核心做法                                       | 和其他筆記的關係                                   |
| --------------------- | ------------------------------------------ | ------------------------------------------ |
| ELB [137]             | 快壅塞時通知鄰居少送一點，鄰居繞過去                         | 分散式、反應式                                    |
| SIDA＋SSLB [103]       | 改 Dijkstra：多條等長最短路時，**避免共用同一條鏈路**          | 和 StableRoute 一樣是在「等長路徑裡挑」，但挑的標準是分散流量，不是穩定 |
| 全域＋本地混合 [109]         | 可預測的長期流量用全域最佳化事先處理；短期波動在本地即時處理             | 和 OPSPF「可預測交給預算，不可預測交給 OSPF」同一個思路          |
| HLBR [105]            | 預測區域狀態＋多路徑，用長距離繞道（LTD）避免連鎖壅塞               |                                            |
| DBPR [38]             | Back-pressure；**把路由限制在起終點圍成的矩形內**，控制 hop 數 | 這個矩形就是 StepClimb 的 **MHP 區域**              |
| LBRA-CP [147]         | 蟻群演算法解多目標（路徑成本＋預測壅塞）                       |                                            |
| Segment routing [106] | 分輕載區／重載區；輕載區用最短路，重載區用壅塞指標                  |                                            |
| LCRA [107]            | 一步分散式計算，依目前位置、目的地和鄰居壅塞度                    |                                            |
| 流量分類 [33]             | 分成延遲敏感、吞吐量敏感、一般三類，分別路由；並考慮拓樸密度隨緯度變化        | 緯度效應和 StepClimb 的「高緯度 ISL 較短」相關            |

**What next?**：沒有營運商的實際數據，很難證明負載平衡真的必要。作者認為 [33] 的「依流量類型給不同路徑」是最務實的方向。

### 3.4 SDN（§4.4）
- 理由：衛星網路和資料中心一樣邊界清楚，適合集中控制；而且地面算力便宜，可以在地面預算路由和拓樸，衛星只執行簡單的轉送規則。
- Controller 放哪裡是主要差異：**GEO 衛星**（OpenSAN [15]、[140]）、**地面站**（[167]，避免 GEO 成為瓶頸）、**衛星自己**（[120]，依需求動態決定哪些衛星當 controller）。
- **[90]** 用 Bresenham 畫線演算法（螢幕畫直線用的）找出最接近「起點到終點直線」的那串衛星。
- **What next?**：作者認為 controller 應該在**地面**，把衛星段視為一條邏輯鏈路來決定路由。

### 3.5 容錯（§4.6）：對照 OPSPF 的動態模組
- 作者指出：snapshot（如 FSA）和 dynamic（如 LAOR）的方法**對鏈路故障都反應不夠快**。例子：太陽能板在某段軌道會擋住天線。
- **DODR [80]**：類 AODV，利用 RREP 和本地修復快速修補斷路。
- **DTDR [84]**：只廣播「哪條鏈路壞了」，而不是所有鏈路狀態更新，降低開銷；靜態路由＋動態演算法混合。→ 這和 OPSPF「平常用預測、故障才發 LSU」非常像。
- **[123]**：針對**傾斜軌道（inclined，Walker Delta）**的大型星座，事先算好每個目的地的主要和備援路徑，故障時用有限範圍的 flooding 和預算繞道。
- **What next?**：作者認為「設計成收斂夠快的路由」是最好的方向。

### 3.6 地理路由（§4.7）
- **[70]，2000**：下一跳選「離目的地歐氏距離最近」的鄰居。平常和最佳路徑的延遲差不超過 10 ms，但在 **counter-rotating seam、極區、接近目的地**時會失效。
- 作者的評論：在衛星網路裡，目的地（地面）相對於星座是會動的，上升段和下降段衛星的拓樸也不同，所以要能快速算出鄰居位置才實用。

### 3.7 模擬平台與模型（§4.11）：可以直接拿來用的工具
- **SaVi [165]**：開源的星座視覺化工具，2006 年起持續更新。
- **Hypatia [85]**：Python 函式庫＋ns-3，產生 LEO 路由結果（Kassing et al., IMC 2020）。如果要重現 StableRoute／StepClimb 的實驗，這個比 OPSPF 用的商用 Exata 容易取得。
- **[135]**：最早期的簡單模型，N×N 網格、每顆 4 個鄰居（就是 +Grid）。
- **[18] Bhattacherjee & Singla, CoNEXT 2019（"Network Topology Design at 27,000 km/hour"）**：
  - 先證明大型星座的 ISL 拓樸設計**無法**用整數規劃、隨機圖或蟻群解決（規模太大）。
  - 觀察到：每顆衛星「局部看出去」的可能連法是有限的，而且**所有衛星都一樣**，稱為 **motif**；靠近軌道頂點和赤道時可用的 motif 又不同。
  - 用 motif 建拓樸，在 Starlink、Kuiper 上都比 +Grid（neighbor-grid）好很多。
  - → 這篇是 DoTD／LAMP 那條「拓樸設計」路線的重要前身。
- **[168]**：容量模型，指出**連到反方向運行的衛星**可以增加潛在鏈路數。
- **[149]**：第一篇用 stochastic geometry（二項點過程）分析衛星路由。作者提醒：真實星座是規則排列的，不是隨機點。
- **[60]**：把路由建模成 Unsplittable Multi-Commodity Flow（UMCF），比最短路少壅塞。
- **[13] "The Nearest Is Not The Fastest"**：只改**第一顆和最後一顆衛星**的選擇（星座內部路由不動）。因為衛星只連固定的幾個鄰居，選對上下行衛星就能得到更短的路徑，不一定是離地面站最近那顆。→ 和 StepClimb 的「路徑形狀由起點相位 u0 決定」以及 ISL vision 論文提到的「起點不同可差到 45 hops」是同一件事。

### 3.8 標準化（§5）
**3GPP（§5.1）**：從 Rel-15 開始研究 NTN（Non-Terrestrial Network），列了 8 份 TR。TR 38.821 定義三種衛星：

| 類型 | 衛星上做什麼 | 有沒有 ISL |
|---|---|---|
| Transparent payload | 只做 RF 濾波、變頻、放大；基地台在地面 | 無 |
| Regenerative（gNB on board） | 整個基地台在衛星上；**星座用 ISL 連成 IP 網路**，承載 NG／Xn 介面 | 可有可無 |
| Regenerative（gNB-DU on board，gNB-CU on ground） | 衛星只做資料單元，控制單元在地面 | 可有可無 |

另外兩個整合情境：**衛星當 5G 存取網路**（手機直連衛星，作者認為還很遠，因為手機功率不夠）和**衛星當 5G 回程（backhaul）**（手機連地面 gNB，gNB 經地面站連衛星網路，比較實際）。

**IETF（§5.2）**：
- **TCPSat WG（1997–2000）**：GEO 時代研究 TCP 在衛星鏈路上的行為，產出 RFC 2488、RFC 2760。
- **LISP**：overlay 在地面，衛星網路當 underlay。
- **TVR（Time-Variant Routing）WG**：2022 年 IETF 115 BOF 後成立，目標是定義「**排程好的、隨時間改變**的網路」的資訊和資料模型（鏈路、鄰接、cost、甚至流量都可以隨時間變）。衛星路由是主要應用。→ 這基本上就是 OPSPF 和 StableRoute SR_G 想做的事的標準化版本。
- **Semantic addressing + instructive routing [63–65]**（作者自家 draft）：用衛星的「語意地址」（殼層、軌道、位置），封包轉送變成一條簡單指令，例如「**沿某個方向一直轉送，直到某顆衛星**」。屬於 source routing，比 SRv6 開銷小；和「路徑怎麼決定」是正交的，可以搭配修改過的 OSPF [127] 或任何方法。
- **DTN（Bundle Protocol，RFC 9171–9173）**：源自 IPNSIG（行星際網路），不是 LEO 專用，但證明 IETF 可以標準化太空協定。

### 3.9 未來挑戰（§6）
1. **可靠度**：衛星有很多時間飛在極區或海洋上空（覆蓋過剩），人口密集區又飽和；太空碎片也是問題。Starlink 發射的 3,055 顆中，2023 年初還有 2,793 顆在軌（作者說約 10% 損耗，我算是 8.6%）。
2. **垂直整合**：地面微波塔（陸地上）＋衛星（海上）；UAV、熱氣球（Loon → Aalyria，用 SDN 管理隨時間變化的拓樸）。
3. **應用支援**：Starlink 實測 [116]（比利時到德國）：輕載掉包率 0.4%、壅塞時 1.5–2%；RTT 中位數從 50 ms 增加到約 100 ms。
4. **互通性**：現在衛星的 ISL 數量有限（距離可達 2,000 英里，只能指向同軌或鄰軌的衛星，因為相對運動小），一家營運商自己就用完了，沒有動機跟別人互連。但光害、碎片需要協調，**ISL 數量增加後就會需要全球標準**。

## 4. 作者的 "What next?" 一覽

| 主題 | 作者看法 |
|---|---|
| IP | 互通性可能被監管機關強制，IP 整合可能變成必要 |
| 負載平衡 | 缺營運數據，無法證明需要；依流量類型分路最務實 |
| SDN | Controller 應放地面，衛星段當作一條邏輯鏈路 |
| ML | 百家爭鳴、沒有明顯贏家；衛星網路週期性強，適合學習；但缺標準資料模型 |
| 容錯 | 快速收斂的路由設計是最佳方向 |
| 地理路由 | 簡單，但目的地相對星座會動，上升／下降段拓樸不同 |
| 低延遲 | 衛星本來就比光纖快、也不會有大 buffer，**最佳化空間有限** |
| 多層衛星 | 重要但已經冷下來，目前商用星座都是單層 |

## 5. Novelty／貢獻
- 不是新方法，價值在**整理**和**連到標準化**。大部分 survey 只講學術，這篇把 3GPP、IETF（特別是 TVR）一起講，是比較少見的角度。
- 每節的 What next? 有作者自己的立場，比純列表的 survey 好讀。

## 6. 這篇和專案的關係
### 和 OPSPF 的關係
- Survey 直接把 OPSPF [119] 列在 §4.2，描述是「考慮拓樸週期可預測，修正 OSPF 無止盡收斂」。和 OPSPF 筆記的理解一致。
- OPSPF 的「可預測靠預算、不可預測靠 OSPF」在 survey 裡有好幾個同類：FSA（純預算）、DTDR [84]（只廣播故障）、[109]（長期全域＋短期本地）、[123]（預算主備路徑＋有限 flooding）。
- Survey 提到地理路由在 **counter-rotating seam 和極區**會失效，和 OPSPF 關閉極區跨軌 ISL 是同一個痛點。

### 和 StableRoute 的關係
- Survey 列為 [111]，描述「等成本路徑中選更新最少的」。
- 同一節旁邊的 **SIDA [103]** 也是在「等長最短路」裡做選擇，但目的是分散流量。可以想成：同一群等長路徑，StableRoute 挑「和上次一樣的」，SIDA 挑「和別人不重疊的」，StepClimb 挑「實際公里數最短的」。
- 作者提到 [113] 想讓路由和 TCP 相容，這和 StableRoute 的動機（換路會讓 TCP 誤判掉包）相同。

### 和 StepClimb 的關係
- Survey 沒有提到 StepClimb（2024 年才發表）。
- 但 **DBPR [38] 把路由限制在起終點的矩形內**，就是 StepClimb 的 MHP 區域；**[13]** 討論起終點衛星的選擇，對應 StepClimb 的 u0。
- **Instructive routing（§5.2）**：「沿某方向走到某顆衛星」這種指令，剛好可以很精簡地編碼 StepClimb 算出來的路徑（同軌走幾步 → 一口氣跨軌 → 再同軌走幾步）。這是我自己的推論，論文沒有這樣說。

### 和 DoTD／LAMP 的關係（拓樸設計）
- Survey 的 [18]（motif）和 [168]（連反方向衛星）是拓樸設計的前身。DoTD 和 LAMP 的「動態／臨時 ISL」可以看成 motif 想法的延伸。見 [[Time-_Dependent_Network_Topology_Optimization_for_LEO_Satellite_Constellations]]、[[LAMP_Low-Latency_Dynamic_Topology_for_LEO_Satellite_Constellations]]。

### 和 Starlink Self-Driving（MobiCom 2023）的關係
- 兩篇都在 2023 年，都指出 Starlink 的 ISL 網路還在發展中。Survey 把 Starlink 歸在 transparent payload（bent-pipe）那一型，Self-Driving 那篇則討論「未來有 ISL 的 Starlink」受軌道機動影響。見 [[A_Networking_Perspective_on_Starlinks_Self-Driving_LEO_Mega-Constellation]]。

### 和 ISL vision（2406.05078）的關係
- 另一篇概覽，偏通訊和最佳化（鏈路指派、資源分配），並直接引用 StepClimb 團隊的 SDP⊂MHP 論文。兩篇的比較表在那篇筆記最後。見 [[Enhancing_LEO_Mega-Constellations_with_Inter-Satellite_Links_Vision_and_Challenges]]。

### 小對照表：專案裡的方法在 survey 分類中的位置

| 方法 | Survey 分類 | Snapshot / Dynamic | 決策在哪裡 |
|---|---|---|---|
| OPSPF | §4.2 IP-based（OSPF 改良） | 混合（預測＋OSPF） | 每顆衛星本地 |
| StableRoute | §4.2 IP-based（Dijkstra 改良） | SR_L/SR_K 偏 dynamic；SR_G 偏 snapshot | 每顆衛星本地（SR_G 要全週期資訊） |
| StepClimb | （未列；最接近 §4.11 模型＋§4.7 地理路由） | 都不是：直接用公式 | 任何知道星座參數的節點 |
| DoTD／LAMP | §4.11 拓樸設計（[18] 的延伸） | 拓樸層，不是路由層 | 離線（地面） |

## 7. Limitations／質疑點
1. **深度很淺**：每篇論文只有兩三行，沒有量化比較，也沒有統一的分類軸。OPSPF 和 StableRoute 各只有一句話。要真正理解還是要讀原文。
2. **自家工作的推廣**：New IP、semantic addressing、instructive routing 都是作者（Futurewei）的 IETF draft，篇幅明顯比其他方法多，而且沒有和其他方法比較。
3. **低延遲的經驗法則偏樂觀**：註腳說地面距離超過衛星高度兩倍（500 km → 1,000 km）就開始划算。我用論文自己的數字（高度比 1+h/R、光纖慢 1.47 倍、上下各 h）算損益平衡點：d = 2h / (0.47 − h/R)。
   - h = 500 km → **約 2,550 km**
   - h = 550 km → 約 2,870 km
   - h = 1,000 km → 約 6,400 km

   所以光纖如果走直線，要比註腳說的遠 2.5 倍以上才划算。註腳自己也承認光纖會繞路，所以實際可能提早，但「兩倍高度」只在光纖繞很多路時成立。另外 ISL 路徑在 +Grid 上是折線，不是大圓弧，實際還會更長。
4. **Starlink 的描述有點矛盾**：摘要說通訊已經進化到有 ISL，但 §5.1 又說 Starlink「現在的工作模式」是 transparent payload，衛星只做訊號中繼。2023 年 Starlink 已有部分衛星搭載雷射 ISL，這裡應該是指主要使用模式，作者沒說清楚。
5. **"What next? 低延遲最佳化空間有限" 值得懷疑**：作者的理由是衛星本來就比光纖快、也沒有大 buffer。但 StepClimb 顯示同樣 hop 數，跨軌位置不同每次可以差 200+ km；LAMP 也顯示拓樸設計能降低 18.5% 延遲。空間其實不小。
6. **參考文獻有排版錯誤**：例如 [60] 的 bibtex 被另一篇（Wang et al. 2023, multi-tier reliability）插進去，[113] 的描述「maximizes the RTT delays compared to the TCP timer granularity」意思不清楚（應該是讓 RTT 變化相對 TCP timer 不要太大）。
7. **v1 預印本**：PDF 上沒有任何發表資訊，可能沒有經過同儕審查。

## 8. 值得追的 references
| Ref | 為什麼 |
|---|---|
| [18] Bhattacherjee & Singla, CoNEXT 2019 | Motif 拓樸設計，DoTD／LAMP 的前身 |
| [85] Kassing et al., Hypatia, IMC 2020 | 開源 LEO 模擬器（ns-3），可用來重現實驗 |
| [67] Handley, HotNets 2018 | Starlink 低延遲路由的經典分析 |
| [17] Bhattacherjee et al., HotNets 2018 | 衛星比光纖快多少的詳細評估 |
| [13] Auddino et al., VTC 2022 | 起終點衛星選擇的影響 |
| [25] Cao et al., Sensors 2022 | 完整的 dynamic routing survey |
| [70] Henderson & Katz, Globecom 2000 | 地理路由在 seam／極區失效的原始分析 |
| [178] ASER, LCN 2021 | 分區 OSPF，和 OPSPF 是另一種改法 |
| [103] SIDA | 等長最短路裡分散流量，可對照 StableRoute |
| TVR WG（IETF） | 時變路由的標準化，對應 OPSPF／SR_G |
