---
title: "Enhancing LEO Mega-Constellations with Inter-Satellite Links: Vision and Challenges"
authors: [Chenyu Wu, Shuai Han, Qian Chen, Yu Wang, Weixiao Meng, Abderrahim Benslimane]
venue: "arXiv 2406.05078v1 [cs.IT], 7 Jun 2024（HIT／HKU／Avignon；magazine 型短文，PDF 上沒有期刊資訊）"
DOI: "arXiv:2406.05078"
tags: [LEO, ISL, laser-ISL, vision, routing, load-balancing, link-scheduling, in-flight-connectivity, SAGIN]
---

## Citation
C. Wu, S. Han, Q. Chen, Y. Wang, W. Meng, A. Benslimane, "Enhancing LEO Mega-Constellations with Inter-Satellite Links: Vision and Challenges," arXiv:2406.05078v1 [cs.IT], Jun. 2024.

## 一句話摘要
一篇 7 頁的 **magazine 型願景文章**：地面站（GS）太少、feeder link 成為瓶頸，所以大型 LEO 星座需要 **ISL** 把衛星串成自主的太空網路。文章先列 ISL 的好處和 6 個應用（3 個「沒有 ISL 做不到」、3 個「有 ISL 會更好」），再從 **效能分析、路由與負載平衡、資源分配** 三個方向講挑戰，最後用一個**飛機上網（IFC）**的小模擬說明：允許的 ISL 數越多，平均下載延遲越低，但會飽和。對本專案最有用的是 §III-A（SDP／MHP 的理論分析，作者群和 StepClimb 是同一個 Quan Chen 團隊的延伸）和 §III-B／§III-C-1（路由分類、**同軌 ISL 比跨軌穩定**、動態 ISL 的鏈路指派）。

**比喻**：衛星像一群在天上飛的郵差，地面站是郵局。郵局太少，郵差大部分時間看不到郵局，信就送不出去。ISL 就是讓郵差在天上互相傳信，傳給剛好看得到郵局的那一位。

## 0. 背景知識
### ISL 的三個好處（§I）
| 好處 | 意思 |
|---|---|
| **Network autonomy（網路自主）** | 衛星上有處理器，直接用 ISL 轉送使用者資料，繞過地面站；減少需要的 gateway 數量 |
| **Enhanced QoS** | ISL 容量大，可以平行傳輸、即時共享資料，分散單一 feeder link 的負擔；也能把不均勻的流量分流出去 |
| **Favorable propagation（傳播環境好）** | 真空中沒有雨衰、對流層衰減；直接走 ISL 比「下地面再上去」路徑短 |

### 雷射 ISL（LISL）vs RF ISL（Table I）
| 特性 | LISL | RF-based |
|---|---|---|
| 頻段 | 光波（論文寫 THz） | Ka、毫米波 |
| 頻譜 | 不需執照 | 需執照 |
| 速率 | 快（Gbps 級，預期到 10 Gbps、未來 Tbps） | 較低 |
| 天線 | 小 | 大 |
| 指向性 | 光束集中 | 光束擴散 |
| 能源效率 | 高 | 低 |
| 安全性 | 高 | 低 |
| 干擾 | 無 | 嚴重 |
| 太陽輻射 | 容易受影響 | 不受影響 |
| **建鏈時間** | **長** | 短 |

最後一列最重要：**雷射 ISL 建鏈要做 PAT（Pointing, Acquisition, Tracking），實驗上要幾十秒**（§IV-4）。這會直接影響路由和拓樸設計，見 §3.5。

### 同軌 ISL vs 跨軌 ISL（§I，Fig. 1）
- **Intra-orbit（同軌）**：同一個軌道面上前後兩顆，距離**幾乎固定**，所以**比較穩定**。
- **Inter-orbit（跨軌）**：相鄰軌道面的衛星，距離和角度一直變，比較不穩定；兩顆衛星**反向運行**（counter-rotating seam）或**經過極區**時最容易斷。
- Starlink 和 Telesat Lightspeed 每顆衛星都規劃 **4 條 LISL**。
- 這和 StepClimb 筆記裡的「同軌 ISL 長度固定、跨軌 ISL 隨緯度變」是同一件事，見 [[Shortest_Path_in_LEO_Satellite_Constellation_Networks_An_Explicit_Analytic_Approach]]。

### +Grid（Fig. 2(a)）
每顆衛星維持 **4 條準永久 ISL**：2 條同軌（前、後）＋ 2 條跨軌（左、右），像 Iridium。穩定，但彈性和效能受限。在極區和 seam 會斷，需要 ISL handover。

## 1. Problem／研究目標
- **問題**：地面站比衛星少很多（成本高、地理限制），feeder link 是瓶頸。例如衛星的遙測資料可能在短暫的過頂時間內來不及全部下傳。
- **目標**：這篇不解一個具體問題，而是**整理 ISL 在大型 LEO 星座中的願景和挑戰**，並用一個案例展示效益。

### 有沒有 problem formulation？
**只有案例研究有**（§III-C-3）：最小化平均延遲，決策變數是**鏈路指派（衛星透過 ISL 跟誰配對）**和**每類檔案的下載比例**，非快取檔案再多一個頻寬分配因子。但**正文沒有寫出式子**，細節在他們另一篇 [13]（arXiv 2405.18919）。

## 2. Key Idea
1. ISL 讓星座從「依賴地面的轉發器」變成「自主的太空網路」。
2. 衛星軌道是**確定性的**，所以很多效能指標（hop 數、覆蓋、容量、斷線機率）可以用**解析方法**算，不用大量模擬。
3. ISL 帶來新的設計維度：**鏈路排程**（跟誰連）變成和時間、頻率、功率並列的資源。

## 3. 內容整理
### 3.1 ISL 的應用（§II，Fig. 1）
| 類別 | 應用 | 為什麼需要 ISL |
|---|---|---|
| **沒有 ISL 做不到** | 無所不在的即時接入 | 極地、海洋沒有地面站；最近的衛星透過 ISL 轉給看得到 gateway 的衛星 |
| | 快速災後通訊恢復 | 地面設施毀損時，不用人工部署 UAV／高空平台 |
| | 多衛星協同任務 | 磁層研究、天文觀測需要衛星之間即時交換資料 |
| **有 ISL 會更好** | 現代 GNSS | ISL 可做次毫米級測距，改善時間同步和定軌 |
| | 飛機上網（IFC） | 飛機沿途的衛星不一定看得到 gateway；也可以從其他衛星的快取下載 |
| | 雙向任務卸載 | 衛星把資料／計算卸載到地面；或把地面任務卸載到衛星，再透過 ISL 從忙的衛星轉給閒的 |

### 3.2 效能分析（§III-A）：和 StepClimb 最相關的一節
#### 1) ISL 路徑分析
- 兩個相距很遠的使用者（例如倫敦和新加坡）可以透過多跳 ISL 連起來，但路徑太長傳播延遲就受不了。所以 **SDP（最短距離路徑）** 和 **MHP（最少跳數路徑）** 是兩個重要指標。
- **[5] Chen et al., TVT 2021**：用**理論分析**（而不是網路模擬）估算傾斜軌道星座中兩個地面使用者之間的 ISL hop 數。以 Starlink phase I-B 為例，**從不同起點衛星出發，路徑可以差到 45 hops**。所以「地面使用者要連哪顆衛星」很重要。
- **[6] Chen et al., TAES 2022**：給出一個判別式，判斷 SDP 是不是落在 MHP 集合裡。結論：**傾角低於 68°，或 phasing offset 夠大**的星座，**所有 SDP 都屬於 MHP**。
  - → 內容和 StepClimb 筆記裡的 ref [20]（「SDP ⊂ MHP 是本文的地基」）一致，應該是同一篇（我手上沒有 StepClimb 的參考文獻表，未逐字核對）。這篇的第三作者 Qian Chen（港大）和 StepClimb 的 Quan Chen 是不同人；[5][6] 的作者名單（Q. Chen、L. Yang、X. Chen）看起來和 StepClimb 是同一個團隊。
- 作者指出的缺口：**SDP 和 MHP 都沒有考慮鏈路品質和鏈路能維持多久**。既然同軌 ISL 比跨軌穩定，值得研究「**足夠穩定＋hop 數少**」的路由模式。
- 一般性觀點：LEO 衛星的移動由確定性軌道決定，所以覆蓋、容量、誤碼率、斷線機率等都可以利用拓樸規律做解析分析。

#### 2) Gateway 佈建
- Gateway 太少 → 接入差；太多 → 成本高、受地理限制。
- [7] 做過 gateway 位置最佳化，但**沒考慮 ISL**。
- ISL 和 gateway 數量是**互相取捨**：ISL 越多需要的 gateway 越少，反之亦然。需要一個長期成本模型同時考慮 gateway 建置成本和 ISL 運作成本，並考慮干擾、頻寬、每顆衛星的 ISL 數量限制。

#### 3) 星座設計
- 傳統星座設計常忽略 ISL。一般來說**極軌**比傾斜軌道好（ISL 比較不會斷斷續續），但對 LEO 不一定理想 [8]。
- 要注意傾角和軌道面間距，達到「最少衛星數覆蓋全球」，同時**避免 ISL 在特定區域壅塞**。

### 3.3 路由與負載平衡（§III-B，Fig. 2）
兩件事要一起做：
- **ISL handover**：衛星之間可見性改變時，ISL 斷掉和重建。
- **Routing**：找一條可靠的端到端路徑。

#### 1) 從靜態路由到動態、混合路由
| 類型 | 做法 | 問題 | 本專案對應 |
|---|---|---|---|
| **Static（snapshot）** | 把軌道週期切成時槽，每個時槽內拓樸不變，跑 Dijkstra 或基因／蟻群演算法 | 假設 ISL 只有開／關，忽略鏈路屬性；**無法處理突發斷線** | OPSPF 靜態模組、StableRoute SR_G |
| 圖模型＋樹 [9] | 用 coordinate graph 描述拓樸，建 binary tree 找最少跳數路徑 | 仍是靜態 | |
| **Dynamic** | 定期收集鏈路埠、負載等狀態並 flood 給全網，維護最新路由表 | **全網 flooding 收斂時間長、開銷大** | 傳統 OSPF |
| **Hybrid（Fig. 2(b)）** | 多層結構：**MEO/GEO 當路由管理者**，GS 當中繼協助 | 需要多層星座 | |
| **ML** [10] | 預測拓樸變化，快速收斂、穩健 | | |

作者特別提到：拓樸可預測，所以 handover 和路由可以**事先排程**。例子：兩顆衛星**反向運行**或**穿越極區**時，跨軌 ISL 最脆弱、最容易 handover。→ 這就是 OPSPF 用軌道預測處理的「可預測斷線」。

#### 2) 負載平衡與 QoS
- 使用者分布不均 → 最短路（距離或 hop 數）可能壅塞，需要新的負載平衡策略。
- 流量類型不同（延遲敏感 vs 吞吐量導向），路由要同時考慮延遲、抖動、容量、hop 數、排隊延遲。
- 這和 survey（2310.07646）§4.3 的結論一樣：依流量類型分配路徑。

### 3.4 資源分配（§III-C）
#### 1) 鏈路排程（Link Scheduling）：和 DoTD／LAMP 最相關
- **+Grid 的限制**：穩定，但系統設計彈性和網路效能都受限。
- **動態 ISL 架構**：衛星可以和通訊範圍內**任何**衛星建立臨時 ISL。動態、多路徑 ISL 對低延遲內容傳送很重要。
- **核心問題**：怎麼有成本效益地做**鏈路指派（link assignment／satellite association）**。三個難點：
  1. **大量二元變數**（每一對衛星連或不連），而且每顆衛星能建的 ISL 數量有上限，形成額外限制。可用 penalty-based 鬆弛，或 **Hungarian algorithm、many-to-many matching**（兩群衛星、一跳傳輸時）。
  2. **穩定性和能源**：要考慮鏈路距離、衛星角速度、**建鏈時間**。鏈路距離同時影響斷線機率和功耗，所以「傳輸距離（效能）」和「ISL 運作成本」之間有**根本取捨**。而且**跨軌 ISL 應該比同軌 ISL 少選**，避免斷斷續續。
  3. **干擾**：ISL 很多時在太空中會互相干擾（RF 頻段尤其嚴重），需要雷射多工和波束管理。

#### 2) 任務卸載／下載
- 鏈路排程和資源分配要**聯合最佳化**（例如 [11]：ISL 間卸載＋地面站下載一起排程）。
- 變數耦合讓問題更難；branch and bound 規模太大算不動；分群（clustering）可降複雜度但只能得次佳解。
- GNSS 的特殊限制 [12]：**測距限制**（短時間內要切換 ISL 到足夠多的不同衛星）和**流量守恆**。

### 3.5 案例研究：飛機上網（§III-C-3，Fig. 3、Fig. 4）
**情境**：太空和地面網路合作，把乘客要的內容送到飛機上。
- **快取檔案**：衛星上已經有，可立即取得。
- **非快取檔案**：只能從地面站取得。
- ISL 用來在衛星之間分享內容。

**最佳化問題**：最小化平均延遲；變數是鏈路指派（衛星間用 ISL 怎麼配對）＋每類檔案的下載比例（非快取再加頻寬分配因子）。先分析最佳傳輸延遲的性質，消去下載比例變數，縮小問題規模。

**參數**：
| 項目 | 值 |
|---|---|
| 星座 | 120 顆，6 軌道面 × 20 顆，1,000 km，傾角 53° |
| 地面站 | 5 個 |
| 飛機 | Airbus A320，每個時槽最多一個檔案請求 |
| 檔案大小（封包數） | 均勻分布 [50,100]、[500,1000]、[1000,3000]、[10,1000]；每封包 1,080 bits |
| 發射功率 | 衛星 5 W、地面站 10 W |
| 頻率 | 衛星→飛機 15 GHz、地面→飛機 18 GHz、地面→衛星 30 GHz，各 100 MHz |
| ISL | 雷射，197 THz |
| 天線增益 | 衛星 40 dB、飛機 30 dB、地面站 52 dB |

**比較對象**：非快取用「平均分配頻寬」、快取用「貪婪衛星配對」當 baseline；另畫「不限 ISL 數（全連通）」當上限。

**結果（Fig. 4，橫軸：每顆衛星最多幾條 ISL，1～6；縱軸：平均延遲）**：
- 快取和非快取檔案的平均延遲都隨允許的 ISL 數**大幅下降**（雷射 ISL 容量大）。
- 但**不是一直降**：會收斂到全連通的值，因為看得到、又有快取該檔案的衛星數量有限。
- 結論：「多建 ISL 的成本」和「帶來的效能」之間的取捨是開放問題。
- （Fig. 4 的確切數值從 PDF 抽不出來，只能看趨勢。）

### 3.6 開放問題（§IV）
1. **SAGIN**（太空–空中–地面整合網路）：需要新標準和協定管理異質性。
2. **網路管理**：SDN／NFV。例子：Amazon Kuiper（2023）用 SDN 管理波束排程和資源分配。
3. **RIS 輔助 ISL**：在衛星上裝可重構智慧表面，補償高頻路徑損耗、降低單條 ISL 功耗。
4. **雷射 ISL 專用技術**：PAT 要求嚴格；要處理都卜勒效應；**建鏈時間要幾十秒**，所以**為 RF ISL 設計的路由不能直接用在雷射 ISL 上，可能造成部分拓樸孤立** [15]。
5. **AI**：深度學習選模型困難，泛化能力和計算效率是問題。

## 4. Novelty／貢獻
- **不是新方法**。主要是把 ISL 的好處、應用、挑戰整理成一張清楚的地圖，加上一個小規模案例。
- 比較有特色的地方：
  - 把「**SDP/MHP 的解析分析**」放進系統設計的脈絡（作者群自己的工作 [5][6][13]）。
  - 明確講出「**同軌比跨軌穩定 → 路由和鏈路指派都應該偏好同軌**」。
  - 提出 ISL 數量 vs 效能的**飽和**現象。

## 5. Limitations／質疑點
1. **深度很淺**：每個挑戰只有一兩段，引用只有 15 篇，而且大部分是作者群自己的工作（[5][6][13]）。
2. **案例研究沒有式子**：最佳化問題、演算法都在 [13]，這篇只給參數和一張圖。Fig. 4 的數值也看不清，無法重現。
3. **案例的星座很小**：120 顆、6 個軌道面，跟標題的 "mega-constellation" 有落差；也沒說星座是 Walker Star 還是 Delta（傾角 53° 推測是 Delta）。
4. **Table I 的頻段寫 "Terahertz (THz)"**：雷射 ISL 一般是光波段（例如 1550 nm ≈ 193 THz，案例用 197 THz），技術上是幾百 THz，但通訊領域說「THz 頻段」通常指 0.1–10 THz。這樣寫容易混淆。
5. **Table I 寫 LISL「干擾：無」**，但 §III-C-1 又說 ISL 多了會有嚴重干擾、需要雷射多工。兩處語氣不一致（後者主要指 RF）。
6. **「Starlink 2027 年中要發射 42,000 顆」**：論文的說法，沒有引用來源。42,000 是 SpaceX 申請的總數，時程我沒有查證。
7. **「同軌比跨軌穩定 → 少選跨軌」太簡化**：StepClimb 顯示最短距離路徑反而要一口氣在高緯度做完跨軌；只用同軌的話很多目的地根本到不了。真正的取捨要看延遲和穩定性的權重，這篇沒有量化。
8. **「建鏈要幾十秒」的影響沒有展開**：這點對 DoTD／LAMP 這種「動態／臨時 ISL」很關鍵（臨時鏈路如果只能用幾分鐘，扣掉幾十秒的建鏈時間可能就不划算），但文章只在開放問題提一句。

## 6. 與其他筆記的關聯
### 和 StepClimb 的關係
- §III-A 引用的 [5][6] 應該是 StepClimb 的前身（同團隊）：[6] 證明「傾角 < 68° 或 phasing offset 大時，SDP ⊂ MHP」，就是 StepClimb 只在 MHP 區域裡找路的理論基礎。
- [5] 的「起點不同可差 45 hops」對應 StepClimb 的「路徑形狀由起點相位 u0 決定」。
- 這篇提出的缺口「SDP／MHP 沒考慮鏈路穩定性」，正好是 StepClimb 的 limitation（StepClimb 假設高緯度 ISL 一直在，還偏好在那裡跨軌）。

### 和 OPSPF 的關係
- Fig. 2(a) 畫的「極區 ISL 斷線、handover、seam area」就是 OPSPF 處理的情境。
- 文中「拓樸可預測 → handover 和路由可以事先排程」＋「動態路由 flood 全網開銷大」＝ OPSPF 的動機。
- OPSPF 的做法是本文分類裡的「static＋dynamic 混合」，但不是 Fig. 2(b) 那種靠 MEO/GEO 管理的混合，而是**每顆衛星自己預測**。

### 和 StableRoute 的關係
- 本文主張找「**足夠穩定＋hop 數少**」的路由模式；StableRoute 正是「hop 數最少的前提下，盡量不換路」（SR_K 甚至願意多走 1–2 跳換穩定）。
- 差別：本文的「穩定」是**鏈路穩定**（偏好同軌 ISL）；StableRoute 的「穩定」是**路徑不變**（減少換路次數）。兩者可以結合：在 next-hop set 裡優先選同軌鄰居。

### 和 DoTD／LAMP 的關係（拓樸設計）
- §III-C-1 的「動態 ISL、鏈路指派、大量二元變數、每顆衛星 ISL 數上限、Hungarian／matching」就是 DoTD 和 LAMP 在解的問題。
  - **LAMP**：保留 3 條固定骨幹（2 同軌＋1 跨軌），第 4 條拿來建臨時 ISL，用貪婪法選。符合本文「同軌優先、跨軌少用」的建議。見 [[LAMP_Low-Latency_Dynamic_Topology_for_LEO_Satellite_Constellations]]。
  - **DoTD**：時間相依的拓樸最佳化。見 [[Time-_Dependent_Network_Topology_Optimization_for_LEO_Satellite_Constellations]]。
- 本文提醒的「**雷射建鏈要幾十秒**」是評估 DoTD／LAMP 臨時鏈路是否划算時應該加進去的成本。

### 和 Starlink Self-Driving（MobiCom 2023）的關係
- 本文說 Starlink 每顆衛星規劃 4 條 LISL；Self-Driving 那篇討論軌道機動對 ISL 拓樸的影響。兩者合起來：ISL 不只會因為「可預測的軌道運動」斷，還會因為「不可預測的避撞機動」斷。見 [[A_Networking_Perspective_on_Starlinks_Self-Driving_LEO_Mega-Constellation]]。

### 和 LEO survey（2310.07646）的關係
- 兩篇都是概覽型。Survey 偏**網路層和標準化**（OSPF、BGP、SDN、3GPP、IETF），本文偏**通訊和最佳化**（鏈路指派、資源分配、案例）。
- 路由分類一致（snapshot/static vs dynamic，加上混合）。
- 都提到 MEO/GEO 當路由管理者的多層架構；survey 認為這條路線已經冷下來（商用星座都是單層），本文仍把它當作解法之一。
- 見 [[LEO_Satellite_Networking_Relaunched_Survey_and_Current_Research_Challenges]]。

### 小對照表

| | LEO survey（2310.07646） | ISL vision（2406.05078） |
|---|---|---|
| 類型 | 網路層 survey，21 頁（含參考文獻）、182 篇引用 | Magazine 願景文，7 頁、15 篇引用 |
| 重點 | 路由協定、SDN、ML、3GPP／IETF | ISL 應用、效能分析、鏈路指派、資源分配 |
| 有沒有實驗 | 沒有 | 有一個小案例（IFC，120 顆衛星） |
| 提到 OPSPF／StableRoute | **有**，直接引用 | 沒有，但描述的問題相同 |
| 和 StepClimb 的連結 | 間接（MHP 矩形、起終點選擇） | **直接**（引用同團隊的 SDP⊂MHP 論文） |
| 和 DoTD／LAMP 的連結 | motif 拓樸設計 [18] | 動態 ISL 鏈路指派、雷射建鏈時間 |
