---
title: "Lightweight Route Flooding via Flooding Topology Pruning for LEO Satellite Networks"
authors: [Guohao Ruan, Tian Pan, Chengcheng Lu, Zhengjie Luo, Houtian Wang, Jiao Zhang, Yushi Shen, Tao Huang, Yunjie Liu]
venue: "IEEE ICC 2022, pp. 1149–1154"
doi: 10.1109/ICC45855.2022.9838278
affiliation: "北京郵電大學（BUPT）、紫金山實驗室、中國空間技術研究院、NovNet（通訊作者 Tian Pan，也是 OPSPF 的第一作者）"
tags: [LEO, satellite-routing, OSPF, link-state, flooding, flooding-topology, LSU, LSAck, route-convergence, Walker-Star, polar-orbit, +Grid, search-tree, branch-and-bound, Docker]
---

## Citation
G. Ruan, T. Pan, C. Lu, Z. Luo, H. Wang, J. Zhang, Y. Shen, T. Huang, Y. Liu, "Lightweight Route Flooding via Flooding Topology Pruning for LEO Satellite Networks," *IEEE International Conference on Communications (ICC)*, 2022, pp. 1149–1154. DOI: 10.1109/ICC45855.2022.9838278

## 一句話總結
這篇是 **OPSPF 團隊（BUPT，Tian Pan）的後續**。OPSPF 已經把「可預期的」極區斷線從 OSPF 泛洪裡拿掉了，但**不可預期的故障（突發 link failure / recovery）還是要全網泛洪 LSU**。在 +Grid 這種很密的網格上，OSPF 每台 router 會把 LSU 往「除了進來那個介面以外的所有介面」送，同一個 LSA 會被大量重複複製。作者的做法是：**不改 OSPF，只把「拿來泛洪的拓樸」換成原拓樸的一個稀疏子圖（flooding topology）**。子圖的規則很簡單：**保留全部同軌 ISL（每軌一個環），再在每對相鄰軌道之間、seam 左右兩半各只留 1 條跨軌 ISL**。剩下的問題是「留哪一條」：作者把所有選法排成一棵搜尋樹，用 Algorithm 1 估算從軌道 1 走到軌道 M 的最遠距離（flooding diameter），用 Algorithm 2 做帶剪枝的 DFS（超過門檻 d(i) 就回溯）找出直徑最小的解。在 Docker 模擬的 **LEO-96（8 軌 × 12 顆）** 上：ISL 從 155 條減到 110 條，單一 ISL 斷線時 **LSU 418 → 246（−41.15%）、LSAck 平均 291 → 197（−32.3%），合計泛洪流量 −37.52%，收斂時間 1.4894 s → 1.3401 s（−10.03%）**；星座越大改善越多（12 × 14 時 LSU −45.19%、收斂 −21.65%）。

**English:** This is a follow-up to OPSPF by the same BUPT group. OPSPF removes predictable polar link changes from OSPF flooding, but unexpected link failures and recoveries still trigger network-wide LSU flooding, which is highly redundant on a dense +Grid mesh. The paper keeps OSPF unchanged and floods only on a sparse sub-topology: all intra-orbit ISLs (one ring per orbit) plus exactly one inter-orbit ISL per adjacent orbit pair in each half of the constellation (either side of the seam). Every node then has degree at least two, so no single failure disconnects the flooding topology, unlike a minimum spanning tree. Which inter-orbit links to keep is chosen by a depth-first search over a selection tree, with a per-step distance bound d(i) to prune branches whose orbit-1-to-orbit-M distance is already too large. On a Docker emulation of LEO-96 (8 × 12), one ISL failure produces 41% fewer LSUs, 32% fewer LSAcks (38% less flooded traffic overall) and 10% faster convergence than OSPF, and the gains grow with constellation size.

## 0. 背景知識
### 這篇用到的名詞
* **Flooding（泛洪）**：OSPF 用來同步 LSDB 的機制。某條 link 狀態改變 → 端點 router 產生新的 **LSA**，包在 **LSU**（Link State Update）裡送出；收到的 router **往除了進來那個介面以外的所有介面轉送**，並回 **LSAck**（可以好幾個 LSA 合成一個 LSAck，所以 LSAck 數量不固定，§IV-B）。所有 router 的 LSDB 一樣時，泛洪結束 = 收斂。細節見 [[OSPF_RFC2328_Key_Sections]]。
* **Regular vs irregular topology change**（§I）：
    * **regular**：衛星週期運動造成的，可以預測（例如進極區關掉跨軌 ISL）。snapshot 法、OPSPF 處理的是這種，**不需要泛洪**。
    * **irregular**：突發的 link failure 和之後的 recovery，無法預測，**還是要全網泛洪**。本文只處理這種。
* **Flooding topology（泛洪拓樸）**：只拿來傳 LSU/LSAck 的子圖。它必須和原拓樸**對每個節點有一樣的可達性**（連通、涵蓋所有衛星），但不需要承載使用者流量；被剪掉的 link「對 load balancing 有用、對泛洪是多餘的」（§I）。
* **Flooding diameter**（§III-A）：泛洪拓樸上最遠兩點之間的最短距離（hop 數）。直徑越大，LSU 傳到最遠那顆所需的 hop 越多 → 收斂越慢。剪 link 會讓直徑變大，這就是取捨。
* **MST（最小生成樹）為什麼不行**：MST 是邊最少的連通子圖，但**任何一條邊斷掉就分成兩半**，泛洪無法傳到另一半。LEO 偏偏就是 failure-prone，所以要留一點冗餘。

### 星座和 seam（§II-A，Fig. 1–2）
* M × N 極軌星座（Walker-Star 類），每顆 4 個介面：2 條同軌（intra-orbit）、2 條跨軌（inter-orbit）。
* **Seam** 把星座分成兩半，兩半的衛星運動方向相反（Fig. 2：左半往北、右半往南）。注意這裡的「左半 / 右半」是**每條軌道環的上升段和下降段**，不是不同的軌道。
* 進極區的衛星**關掉跨軌 ISL**（天線追不上），同軌 ISL 一直連著。所以 Fig. 6 的實體拓樸 ISL 數比完整 +Grid 少（例：8 × 12 完整網格應有 96 條同軌 + 7 × 12 = 84 條跨軌，Fig. 6 只有 155 條 → 跨軌實際只有 59 條，差的 25 條應該是極區關掉的；這是我們從 Fig. 6 數字推算的，論文沒寫）。
* 這和 [[OPSPF_Orbit_Prediction_Shortest_Path_First_Routing_for_Resilient_LEO_Satellite_Networks]] 用的是同一種星座模型，只是 OPSPF 是 LEO-48（6 × 8），這篇主打 LEO-96（8 × 12）。

### 和 IETF 的「Dynamic Flooding」的關係（我們補充的背景，論文沒提）
* 資料中心那種很密的 Clos/fat-tree 也有一樣的問題，IETF LSR 工作組因此做了 **Dynamic Flooding**（後來成為 RFC 9667，2024）：所有 link 照樣建 adjacency、照樣進 LSDB 給 SPF 用，但 **LSA 只沿著一個指定的稀疏 flooding topology 傳**。
* 本文的想法本質上就是這個概念套到 LEO，但它宣稱「**不改 OSPF**」，論文沒說怎麼做到（見 §6 的質疑）。

## 1. Problem
* **Problem**（§I、§II-C）：
    * OPSPF 在偵測到突發故障時，會從 static mode 切到 dynamic mode，**照 OSPF 的方式全網泛洪**。在 full-mesh（作者用詞，實際是 +Grid）星座上，泛洪沒有約束，有兩個代價：
        1. **ISL 頻寬**：ISL 本來就貴、已經有使用者流量；多出來的路由訊息會排隊甚至掉包，反過來拖慢收斂。
        2. **控制平面處理**：每顆衛星收到一堆重複 LSU 全都要送上 control plane 處理；衛星 CPU 弱，最糟情況泛洪速率超過處理速率 → 丟 LSU → 收斂更慢。
    * 星座越大越密，重複的 LSU 越多。
* **Target**：在**不改 OSPF 協定**的前提下，找一個稀疏的 flooding topology，同時滿足：(a) 連通所有節點；(b) 夠穩健（單點故障不會斷開）；(c) flooding diameter 盡量小（收斂不要變慢）。
* **有沒有正式 formulation**：**沒有**。沒有目標函數、沒有限制式，只有四條設計原則（§III-A）+ 兩段 pseudocode。「最佳」的意思是「在原則限制下 flooding diameter 最小」，而直徑本身是用 Algorithm 1 的**估計式**算的，不是真正的圖直徑。

## 2. Solution：四條設計原則（§III-A）⭐
1. Flooding topology 要**連通原拓樸的所有節點**。
2. **每個節點度數 ≥ 2**，作者說這樣就「沒有單點故障」。
3. 同軌 ISL 比跨軌穩定 → **全部同軌 ISL 都保留**。每軌是一個環，剛好給每顆衛星度數 2。
4. 為了在冗餘和直徑之間取捨，**每條軌道和相鄰兩條軌道各連 2 條跨軌 ISL**（實際操作：每對相鄰軌道之間，seam 左半 1 條、右半 1 條，§III-B）。

**直覺**：每條軌道是一個環；相鄰的兩個環用 2 條 link 接起來（一條在上升段、一條在下降段）。整個 flooding topology 就像「一串用雙鏈條扣起來的環」。

* 邊數：M·N（同軌）+ 2(M−1)（跨軌）。用這個公式對 Fig. 6 的「Flooding Topology」欄**完全吻合**：48 = 6×8 → 48+10 = 58；72 = 6×12 → 82；96 = 8×12 → 110；120 = 10×12 → 138；144 = 12×12 → 166；168 = 12×14 → 190。（星座組合是我們從數字反推的，Fig. 6 只標總衛星數；6×8、8×12、12×12、12×14 則有在內文出現。）
* **單一 link 故障為什麼不會斷開**：同軌 link 斷 → 那個環變成一條路徑，仍連通；跨軌 link 斷 → 那兩條軌道之間還有另一半的那一條。**兩條同一對軌道之間的跨軌 link 同時斷**，或兩個環各斷一條在不巧的位置，就會分裂（§III-D 的 multi-point failure）。
* ⚠️ 原則 2「度數 ≥ 2 → 沒有單點故障」**在一般圖上不成立**（度數 ≥ 2 不保證 2-edge-connected，例如兩個環只用一條邊相連）。這裡成立是因為原則 3+4 的特殊結構，不是因為度數。

## 3. 方法細節（§III-B）
### 3.1 搜尋樹（Fig. 3）
* 記號：**n_{i,1}** = 軌道 i 和 i+1 之間、左半（上升段）**目前可用**的跨軌 ISL 數；**n_{i,2}** = 右半的。
* 根節點 = 只有同軌 ISL 的拓樸。
* 第 1 層：從 n_{1,1} 條選 1 條、從 n_{1,2} 條選 1 條 → n_{1,1} × n_{1,2} 個子節點。第 2 層對軌道 2–3 做同樣的事，依此類推。
* 葉節點 = 一個完整的 flooding topology。總數 **∏_{i=1}^{M−1} (n_{i,1} × n_{i,2})**，指數成長。
* Fig. 3 畫的是深度 2 的片段，紅色線是被選中的跨軌 ISL。

### 3.2 Algorithm 1：估計軌道間距離
輸入：已選的跨軌 link 集合 L、軌道 i、軌道周長 P（= 每軌衛星數 N，以 hop 計）。L_{i,1}、L_{i,2} 可以理解成「軌道 i 上被選中的左/右 link 所在衛星的位置編號」。

| 情況 | 公式 | 意思 |
|---|---|---|
| i = 1（起點軌道） | D = \|L_{1,1} − L_{1,2}\|；SD₁ = SD₂ = ⌊max{D, P−D}/2⌋ + 1 | 軌道 1 是環，離兩條出口 link 最遠的衛星在**較長那段弧的中點**，走到出口要 ⌊max/2⌋ hop，再 +1 跨過去 |
| 1 < i < M−1（中間軌道） | SD₁ = \|L_{i,1} − L_{i−1,1}\| + 1（SD₂ 同理用右半） | 從「進來的 link 端點」沿軌道走到「出去的 link 端點」，再 +1 跨到下一軌 |
| i = M−1（最後一段） | 中間軌道的公式 + ⌊max{D, P−D}/2⌋（D 用軌道 M 上兩條 link 的距離） | 跨到軌道 M 之後，還要走到軌道 M 上最遠那顆 |

* 把 SD(1) + SD(2) + … + SD(M−1) 加起來 = 「從軌道 1 最遠那顆走到軌道 M 最遠那顆」的 hop 數，作者把它當 flooding diameter。左半、右半各算一個（distance₁、distance₂）。

**小例子（我們自己算的）**：N = P = 12，軌道 1 上左 link 在位置 2、右 link 在位置 8 → D = 6，max{6, 6}/2 = 3 → SD(1) = 4。如果右 link 換到位置 3 → D = 1，max{1, 11}/2 = 5.5 → ⌊5.5⌋ + 1 = 6。所以**兩條 link 在環上分得越開，起點那一軌的最遠距離越小**。

### 3.3 Algorithm 2：帶剪枝的 DFS
* 從根開始 DFS。每走到一個子節點：把它的兩條 link 加進 L，呼叫 Algorithm 1，累加 distance₁、distance₂。
* **剪枝**：若 distance₁ > d(i) 或 distance₂ > d(i)，扣回這一步的距離並**回溯**（不展開子樹）。d(i) 是「從軌道 1 到軌道 i 可容忍的最大距離」的估計函數。
* 走到葉節點 → 輸出一個候選 flooding topology。

**Case study（§III-C，Fig. 4）**：8 × 12 找到 **12000 個直徑最小的候選**，Fig. 4 畫其中一個，**比原拓樸少 45 條跨軌 ISL**（155 − 110 = 45，和 Fig. 6 一致）。

⚠️ 讀 pseudocode 要注意的地方（我們自己看出來的）：
* **d(i) 從頭到尾沒定義**。剪枝效果、能不能保證「找到的是最小直徑」完全取決於 d(i)。如果 d(i) 設得比最佳解寬鬆，就是正確的 branch-and-bound；設太緊會剪掉最佳解。論文沒說怎麼設，也沒說 12000 個候選是怎麼確認「最小」的。
* **回溯時狀態沒有還原**：只有剪枝那條路徑（line 12）會把距離扣回去；正常遞迴返回後 distance₁/₂ 和 L 沒有撤銷，而且 line 20 的 `i++` 在 for 迴圈裡對每個兄弟節點都加一次。照字面實作會出錯，應該理解成「子節點用 i+1、返回時撤銷」。
* **「直徑」只是沿著軌道 1 → M 那條鏈的估計**，不是真正的圖直徑：沒考慮兩個端點都在中間軌道的情況，也沒考慮從左半經過環繞到右半的路徑；左右兩半是分開算的。所以「最小直徑」是對這個代理指標而言。
* 摘要說 LEO-96 的 ISL 少 **28.1%**，但 Fig. 6 的數字是 45/155 = **29.0%**（45/160 才是 28.1%）。12 × 14 的 87/277 = 31.4% 則和內文一致。小差異，可能是不同版本的拓樸數字。

### 3.4 隨時間變化與多點故障（§III-D）
* **regular change**：實體拓樸週期性變化，flooding topology 也要跟著換。做法：**預先算好每個週期拓樸對應的 flooding topology，存在每顆衛星上，定時切換**（和 snapshot 法同一個思路，見 [[Improving_the_snapshot_routing_performance_through_reassigning_the_inter-satellite_links]]）。
* **irregular change**：單點故障不會斷開。多點故障（很少見）若把 flooding topology 切成兩半，就要偵測出來，**改用備用 flooding topology，或直接退回 OSPF 原本的泛洪**。怎麼偵測、誰決定切換，論文沒寫。

## 4. Evaluation（§IV）
### 4.1 設定
* M × N 個 **Docker container** 當衛星，**veth-pair** 當 ISL；把 veth 斷開模擬突發故障，**Wireshark** 抓 LSU/LSAck。
* 硬體：Intel Xeon Gold 5520、Ubuntu 20.04.1。
* **論文沒寫**：用哪套 OSPF 實作（Quagga / FRR / BIRD？）、Hello/Dead interval、LSA pacing 等計時器、有沒有和 OPSPF 結合（看起來是直接和原版 OSPF 比）、以及沒有真實軌道或 ISL 延遲（veth 的延遲接近 0）。
* 指標：搜尋演算法時間（有/無剪枝）、LSU/LSAck 數量、收斂時間、不同規模的趨勢。比較對象**只有 OSPF**。

### 4.2 結果
| 圖 | 內容 | 數字 |
|---|---|---|
| Fig. 5 | 搜尋時間（ms），原始 vs 剪枝 | 48 顆：1.69 vs 1.54；72：116.18 vs 16.5；96：9567.74 vs 145.3；120：814462.04 vs 2058.62；144（12×12）：57,332,721 vs 4,797,158（**−91.63%**） |
| Fig. 6 | ISL 數，實體 vs 泛洪 | 73→58、115→82、155→110、195→138、235→166、277→190（12×14 少 87 條，**31.4%**） |
| Fig. 7 | LSU 數（8×12） | OSPF **418**，Lightweight **246**（**−41.15%**） |
| Fig. 8 | LSAck 數 vs 斷的是哪條 link（8×12，約 110 條 link 逐一測） | OSPF 280–350（平均 **291**），Lightweight 150–230（平均 **197**）（**−32.3%**） |
| Fig. 9 | 收斂時間 vs link（8×12） | OSPF 1.4–1.6 s（平均 **1.4894 s**），Lightweight 1.2–1.4 s（平均 **1.3401 s**）（**−10.03%**） |
| Fig. 10 | LSU vs 規模 | 6×8：−30.93%；12×14：−45.19%（OSPF 約 770 vs 約 420） |
| Fig. 11 | LSAck vs 規模 | 6×8：−31.34%；12×14：−29.93% |
| Fig. 12 | 收斂時間 vs 規模 | 6×8：−4.29%；12×14：−21.65%（約 3.4 s vs 約 2.7 s） |

**驗算（我們自己算的）**：
* 37.52% 是把 LSU 和 LSAck 加總：(418 + 291) = 709 → (246 + 197) = 443，(709 − 443)/709 = **37.52%** ✓。
* 41.15% = 172/418 ✓；32.3% = 94/291 ✓；10.03% ≈ (1.4894 − 1.3401)/1.4894 = 10.02% ✓。

**觀察**：
* **LSU 少了 41%，收斂只快 10%**。合理的解釋是收斂時間主要被「最遠那顆要幾 hop」和每跳的處理/計時器延遲決定，而剪 link 會讓直徑略增，抵掉一部分好處。論文沒有報告 flooding topology 和原拓樸的實際直徑，這是我們的推測。
* **Fig. 5 的剪枝效果不是單調的**：96 顆省 98.5%、120 顆省 99.7%，到 144 顆反而只省 91.63%，和「越大越明顯」的說法不太一致。更重要的是 **12 × 12 剪枝後仍要約 4,797 s（80 分鐘）**，未剪枝約 15.9 小時，成長是指數的；Starlink 一個 shell 72 × 22 根本不可能這樣搜（見 §6）。
* LSAck 的減少比例（約 30%）比 LSU 小，且隨規模**沒有變大**（12×14 反而 29.93% < 6×8 的 31.34%）；摘要說「改善會隨規模放大」主要是 LSU 和收斂時間。

## 5. 值得讀的段落
1. **§III-A 四條原則**：整篇的核心，一頁就講完。先想通「每對軌道左右各 1 條」為什麼剛好給 2-edge-connected。
2. **Algorithm 1 + 內文說明**：環上「最遠點在長弧中點」那個 ⌊max{D, P−D}/2⌋，建議自己用 N = 12 畫一次（上面 §3.2 的小例子）。
3. **§III-D Discussion**：預先算好 flooding topology 隨週期切換、多點故障退回 OSPF。這段很短，但它決定了這個方法在真實系統要怎麼部署。
4. **Fig. 6–9**：數字之間互相對得起來（ISL 數用公式可驗、37.52% 可驗），可以當作讀 IEEE 短文時「自己驗算」的練習。

## 6. 和專案裡其他 routing 筆記的關係
* **[[OPSPF_Orbit_Prediction_Shortest_Path_First_Routing_for_Resilient_LEO_Satellite_Networks]]（本文 ref [8]，同一位通訊作者 Tian Pan）**：
    * **直接互補**：OPSPF 處理**可預期**的極區斷線（0 封包、0 收斂時間）；本文處理 OPSPF 的 dynamic mode 剩下來的**突發故障**，讓那部分泛洪變輕。兩者合起來才是完整的「OSPF 改良版」。
    * OPSPF 把突發故障的控制封包減少約 57%，其中一部分來自**拿掉 LSAck**；本文則保留 LSAck、改從「少走幾條 link」下手。兩種省法理論上可以疊加。
    * 數字不能直接比：OPSPF 用 Exata + STK、LEO-48，OSPF 收斂約 543 ms；本文用 Docker、LEO-96，OSPF 收斂約 1.49 s（6×8 約 1.1 s）。平台和計時器不同。
* **[[StableRoute_When_Dijkstras_Algorithm_Meets_Topology-Varying_Satellite_Networks]]（本文 ref [4] 是同團隊 Luo et al. 的 ICDCS 2021 poster「A refined Dijkstra's algorithm with stable route generation...」，應是 StableRoute 的短版）**：本文 regular/irregular 拓樸變化的分類就是引自那篇。StableRoute 處理「SPF 算出來的 next hop 不要亂換」，本文處理「LSA 怎麼傳比較省」，是 link-state 路由的不同環節。
* **[[Improving_the_snapshot_routing_performance_through_reassigning_the_inter-satellite_links]]（本文 ref [7]）**：snapshot 法處理 regular change；本文 §III-D「預先算好每個週期的 flooding topology、存在衛星上定時切換」其實就是**把 snapshot 的想法用在泛洪拓樸上**。
* **[[OSPF_in_Low_Earth_Orbit_Stability_overhead_and_routing_dynamics]]**：
    * 那篇的結論是原版 OSPF 的訊令每顆 **< 2 kbps**，相對 ISL 容量可以忽略；本文的動機卻是「泛洪會讓貴的 ISL 擁塞、讓 CPU 過載」。兩篇對「泛洪 overhead 嚴不嚴重」的看法相反。
    * 差別在指標：那篇看頻寬（kbps），本文看封包數和控制平面處理量。若 ISL 是 Gbps 級的光學鏈路，本文的頻寬論點會變弱，但**控制平面處理**的論點仍然成立（每個 LSU 都要上 CPU）。
    * 那篇也觀察到斷線影響是局部的、大部分路由靠 ECMP 備用 next hop 就恢復，SPF 重算不急；這意味著收斂時間省 10% 對使用者的實際影響可能有限。
* **[[OSPF_RFC2328_Key_Sections]]**：讀本文前先看 RFC 2328 的 flooding procedure（§13）就夠，本文 §II-B 的介紹很簡略。
* **[[ASER_Scalable_Distributed_Routing_Protocol_for_LEO_Satellite_Networks]]**、**[[A_Disruption_Tolerant_Distributed_Routing_Algorithm_in_LEO_Satellite_Networks]]（DTDR）**、**[[A_Distributed_Survivable_Routing_Algorithm_for_Mega-Constellations_With_Inclined_Orbits]]（Qi 2020）**：都在降低分散式路由的控制開銷或提升抗故障能力。ASER 是用分區/分層讓更新不要傳全網；本文是不分區、但讓全網泛洪走比較少的 link。這兩種想法可以互補（區內用稀疏 flooding topology）。
* **[[Your Mega-Constellations Can Be Slim]]**：那篇從**實體拓樸**層面問「需要多少 ISL 才夠存活又夠快」；本文是在**控制平面**層面問同一個問題（泛洪需要多少 link）。概念很像，都在保證連通 + 冗餘的前提下把邊剪掉。
* **[[Shortest_Path_in_LEO_Satellite_Constellation_Networks_An_Explicit_Analytic_Approach]]（StepClimb）**：同樣利用 +Grid 的座標規律。Algorithm 1 的軌道間距離估計（沿軌走 \|Δ位置\| + 1 跨軌）和 StepClimb 對網格最短路的分析是同一類幾何計算。
* STARVERI（ICNP 2024，筆記在 project files 的 notes/routing/，PC 上還沒有）：兩篇都是在 +Grid 上「刻意只用部分路徑」，但目的不同（STARVERI 為了可驗證、本文為了省泛洪）。

## 7. 讀後評論
* **優點**：
    * 問題很實際：OPSPF 之後剩下的突發故障泛洪，確實是 OSPF 類方案的下一個瓶頸。
    * 解法簡單、好部署：「每軌環 + 相鄰軌左右各 1 條」很容易理解，而且邊數公式和 Fig. 6 完全對得上。
    * 比 MST 多想了一步（穩健性），也有考慮直徑對收斂的影響。
    * 有在真實的 OSPF 實作（container）上量 LSU/LSAck，而不是純數值模擬。
* **限制 / 質疑點**：
    * **「不改 OSPF」怎麼做到？論文沒說。** 標準 OSPF 沒辦法「在某條 link 上維持 adjacency 但不在上面泛洪」。最可能的實作是**只在 flooding topology 的介面上開 OSPF**，但這樣被剪掉的 link 就不會出現在 LSDB 裡，**SPF 只能在稀疏拓樸上算路**，使用者流量會繞遠、也失去 ECMP。這和作者說「被剪掉的 link 留給 load balancing」矛盾。要兩者兼得需要像 IETF Dynamic Flooding（RFC 9667）那樣修改協定。論文完全沒有量**資料平面的路徑長度或延遲**。
    * **規模**：搜尋是指數的，12 × 12 剪枝後仍要約 80 分鐘；且每個週期拓樸都要搜一次。對 Starlink 規模（一個 shell 上千顆）不可行。其實對這種規則網格，應該可以直接用幾何推出最佳選法（例如左右 link 在環上相距 N/2、相鄰軌道的 link 位置對齊），根本不用搜尋；12000 個等價最佳解也暗示解空間高度對稱。
    * **d(i) 沒定義**，pseudocode 的狀態還原也有問題，無法照論文重現演算法。
    * **「直徑」是代理指標**，只算軌道 1 → M 的鏈，沒驗證和真正圖直徑的關係。
    * **比較基準只有原版 OSPF**：沒和 OPSPF 結合後的效果比、沒和 MST + 備援或其他稀疏化方法比，也沒和「拿掉 LSAck」這種做法比。
    * **只測單一 link 斷線**。多點故障、link recovery（恢復時要交換 DD，訊息量更大）都沒測；退回 OSPF 的偵測機制沒有實作。
    * **flooding topology 切換的同步問題**：每顆衛星要在同一時刻換 flooding topology，否則切換瞬間兩邊用的子圖不一致，可能出現 LSU 傳不到的空窗。論文沒討論。
    * **Docker + veth 沒有傳播延遲**（LEO ISL 單跳約數 ms～十幾 ms），所以收斂時間的絕對值和相對改善在真實星座可能不同；直徑變大的代價在有傳播延遲時會更明顯。
    * 只考慮 Walker-Star（有 seam）；Walker-Delta 沒有 seam，「左半/右半」的定義要改。
* **對研究的啟示**：如果要延伸 OPSPF 這條線，本文提供了一個很清楚的切入點：**泛洪拓樸可以和路由拓樸分開設計**。幾個可以做的方向：(1) 用 +Grid 的幾何**直接構造**最佳 flooding topology，取代指數搜尋，並證明直徑上界；(2) 照 Dynamic Flooding 的方式做到「所有 link 進 LSDB、只在子圖泛洪」，並量資料平面的影響；(3) 把 OPSPF 的軌道預測拿來選 flooding link：**優先選短時間內不會進極區、不會斷的跨軌 ISL**，讓 flooding topology 本身更少切換；(4) 在有真實延遲的平台（例如 StarryNet）重測收斂時間。
