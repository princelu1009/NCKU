---
title: "A Distributed Survivable Routing Algorithm for Mega-Constellations With Inclined Orbits"
authors: [Xiaoxin Qi, Bing Zhang, Zhiliang Qiu]
venue: "IEEE Access, vol. 8, pp. 219199–219213, 2020"
doi: 10.1109/ACCESS.2020.3041346
survey_ref: "LEO survey 2310.07646 [123]（§4.6 Disruption tolerant networking & Reliability）"
tags: [LEO, satellite-routing, ISL, survivable-routing, fault-tolerance, Walker-Delta, inclined-orbit, distributed-routing, mega-constellation]
---

## Citation
X. Qi, B. Zhang, Z. Qiu, "A Distributed Survivable Routing Algorithm for Mega-Constellations With Inclined Orbits," *IEEE Access*, vol. 8, pp. 219199–219213, 2020. DOI: 10.1109/ACCESS.2020.3041346（Xidian University，西安電子科技大學 ISN 國家重點實驗室）

## 一句話總結
這篇是給 **inclined（傾斜軌道，Walker Delta）** 大型星座設計的**分散式、可存活（survivable）路由**。作者先指出傾斜星座有一個特別的拓樸性質：inter-plane ISL 連一圈回來時，衛星編號會**偏移 F**（phase factor），所以整張網不是單純的環面（torus）而是**螺旋（spiral）**。在這個前提下，每顆衛星不跑 Dijkstra，而是用 **X-Y routing**（先橫再直或先直再橫）以 O(1) 算出到每個目的地的 **1–2 條 primary path** 和 **2 條 secondary path**。鏈路壞掉時，兩端衛星只把 link state 封包 **flood 到 f_ttl 跳內（restricted flooding）**，收到的衛星更新每條路徑上的「壞鏈路數」N_block，然後在**還沒碰到斷點之前就先繞（pre-detour）**。另外還有 **partial-record loop avoidance**（只有在走了非 primary 方向之後才開始把經過的節點寫進封包頭）和 **vector-based next hop selection**（把各種條件編成 0/1 向量，用字典序比大小選下一跳）。在 Starlink 第一期（32×50、53°、1150 km）模擬：20% 鏈路故障時，f_ttl = 5 的延遲比「只看本地」的演算法少 15.5%，flooding 封包數只有全網 link state 演算法的約 1/50（f_ttl = 10 時約 1/10）。

## 0. 背景知識（讀這篇需要的）
### Polar（Walker Star）vs Inclined（Walker Delta）
| | Polar / Walker Star | Inclined / Walker Delta |
|---|---|---|
| 傾角 | 接近 90° | 明顯小於 90°（Starlink 第一層 53°） |
| 軌道平面分布 | 只鋪滿 180°（所以有 **seam**，兩側反向） | 鋪滿 360°（**2π-constellation**），沒有 seam |
| 極區 | 跨軌 ISL 進極區會關（天線追不上） | 衛星根本到不了高緯，**ISL 可以一直開** |
| 拓樸 | 隨時間變 | 選好 ISL 後可以**永久不變**（permanent topology） |

* 作者的動機：前人的分散式路由（DRA [8]、LSP [9]、SDRA [10]）幾乎都是為 polar 星座設計的，傾斜星座「看起來比較簡單（拓樸不變）」所以被忽略，但它其實有自己的拓樸特性要處理。
* 和 [[OPSPF_Orbit_Prediction_Shortest_Path_First_Routing_for_Resilient_LEO_Satellite_Networks]] 的關係：本文 Related Work 直接把 **OPSPF 列為 [6]**，當作「link state 演算法用在 LEO」的例子，並指出這類演算法在大星座上計算和通訊開銷太大，而且「只有一條鏈路變化時，沒必要 flood 給全網、讓每顆衛星都重算路由表」。

### 符號
* N 條軌道、每條 M 顆，(n, m) = 第 n 條軌道的第 m 顆。
* F ∈ {0, …, N−1}：phase factor。
* 相鄰軌道同編號衛星的相位差 φ1 = 2πF/(MN)；同軌相鄰衛星 φ2 = 2π/M。
* 每顆衛星 4 條 ISL：2 條 intra-plane（長度、方向固定）、2 條 inter-plane（長度方向會變）。

## 1. Problem
* **Problem**：mega-constellation（上千顆）需要一個 (a) **計算量小**、(b) **訊令開銷小**、(c) **鏈路故障時延遲不要變差太多**的路由演算法。
    * Dijkstra/link state：有全域視野、路徑最佳，但每次變動都要全網 flooding、每顆都重算 O(M²N²)，大星座撐不住。
    * 純本地繞道（如 DRA）：不用訊令，但因為沒有全域視野，常常走到斷點前才繞，多出很多延遲（Fig. 7 紅線）。
* **Target**：在「全網 flooding」和「純本地」之間找折衷：讓故障資訊只傳到附近，讓附近的衛星**提早**繞。
* **有沒有正式 formulation**：沒有。沒有 objective function 或最佳化問題，是一組啟發式機制＋模擬驗證；唯一的定理是拓樸性質（Theorem 1、2）和 loop-free 證明（Theorem 3，附錄）。

## 2. 拓樸特性：螺旋（§III-B）
* **Theorem 1**：如果 (n, m) 連到 (n+1, m)（n = 0…N−2），那麼最後一條軌道 (N−1, m) 會連到 **(0, (m+F) mod M)**。
    * 白話：沿著 inter-plane ISL 往東一直走，繞地球一圈回到第 0 條軌道時，編號**錯開 F 格**。原因是每跨一條軌道累積 φ1 的相位差，N 條累積 N·φ1 = 2πF/M = F·φ2，剛好是 F 顆衛星的間距。
    * 所以 inter-plane ISL 會形成**螺旋**（Fig. 1 的黃線是 Starlink 上 32 條連續跨軌 ISL）。
    * **閉合螺旋的數量 = gcd(M, F)**。Fig. 2：M = 6、N = 8、F = 3 → 3 條螺旋。Starlink 模擬參數 M = 50、F = 5 → 5 條（我自己算的）。
* **Theorem 2**：實際上跨軌 ISL 不一定連「同編號」，可以連編號差 δ 的衛星，例如 (0,0)–(1,48)（δ = −2），換取比較好的南北向連通（Fig. 3）。把衛星**重新編號** m′ = (m − nδ) mod M 後，又變回「同編號相連」的形式，只是最後一圈的偏移變成 F + δN。Starlink 例子：δ = −2 → 偏移 5 − 64 ≡ 41 (mod 50)，所以 (31, 0) 連到 (0, 41)。
* **為什麼重要**：在 bi-directional **Manhattan Street Network（MSN）** 裡，只要比較兩顆衛星的軌道號和編號就能算出最短 X-Y 路徑；但傾斜星座有這個偏移，同編號的兩顆衛星（例如 (0,0) 和 (19,0)）往左走和往右走的 Y 分量**不一樣**（Fig. 5），算路時要把偏移考慮進去。
* 作者也提到，[17][18] 選特定參數（例如 F = 0 或「8 字形」）可以讓傾斜星座直接變成 MSN，但實際星座的參數還要考慮覆蓋和防碰撞，不一定能這樣選，所以本文處理一般情況。

## 3. Method
整個演算法三部分：**(1) 路徑預算（Algorithm 1）→ (2) 鏈路狀態更新（Algorithm 2）→ (3) 封包轉送（Algorithm 3）**。

### 3.1 Basic X-Y routing：primary 和 secondary（§IV-A）
* 目標是**最少 hop 數**（作者認為端到端延遲主要是傳播延遲，所以以 hop 數為準）。X 分量 = 走 inter-plane（左/右/0），Y 分量 = 走 intra-plane（上/下/0）。
* 定義：
    * **Primary direction**：讓封包離目的地更近的方向。**Primary path**：下一跳是 primary direction 的路徑。
    * **Secondary direction / path**：不會更近的方向，當備援用。
* 怎麼算（Algorithm 1）：
    1. 同軌道：只有 1 條 primary path（沿軌道）。
    2. 不同軌道：分別算往左 P_l、往右 P_r 的 X-Y 路徑長度（**要考慮螺旋偏移**），挑短的。
        * Y 分量是 0 → 1 條 primary path。
        * X、Y 都不是 0 → 先橫再直、先直再橫兩條等長，所以有 **2 條 primary path、2 個 primary direction**。
    3. Secondary：1 條 primary 時，取和 primary **垂直的兩個方向**；2 條 primary 時，取**剩下的兩個方向**（Fig. 6）。
* 每顆衛星**不存整條路徑**，只要「知道路徑長怎樣」（X、Y 分量），需要時用編號算就好。
* 複雜度：每個目的地 O(1)，全部 O(MN)；Dijkstra（linked list 實作）每顆衛星 O(M²N²)。

### 3.2 Failure recovery（§IV-B）
**(a) Restricted flooding**
* 鏈路壞掉（從下層通知或 Hello 偵測到），兩端衛星各發一個 link state 封包，帶 **TTL = f_ttl**，每轉一次減 1，所以只傳到 **f_ttl 跳內**。用 sequence number 抑制 flooding storm。
* 每顆衛星對「每個目的地的每條 primary/secondary 路徑」維護一個計數器 **N_block = 這條路徑上壞掉的鏈路數**。收到「壞了」就 +1，收到「恢復了」就 −1。
* 判斷「這條鏈路在不在某條路徑上」也是用編號直接算，不需要存路徑。
* 衛星整顆壞掉 = 它的 4 條鏈路都壞掉，當作特例。

**(b) Pre-detour**
* 有了 N_block，衛星可以在**還沒走到斷點**之前就換路（Fig. 7：s 知道 1–2 斷了，一開始就往下走綠線，而不是走到 1 才繞紅線）。
* 選擇規則（「give priority to」，最後還要經過 3.4 的向量比較）：
    * **1 條 primary**：primary 的 N_block = 0 就走 primary；N_block > 0 就優先選沒有壞鏈路的 secondary；三條都有壞的，還是選 primary。primary 方向的直接鏈路斷了，就選 secondary，最後才選反方向。
    * **2 條 primary**：先在兩條 primary 裡挑沒壞的；兩個 primary 方向的直接鏈路都斷了，才看 secondary。

### 3.3 Partial-record loop avoidance（§IV-C）
* 分散式路由常見問題：每顆衛星各自決定，封包可能繞回原處（Fig. 8）。常用的兩招都不夠：「不送回上一跳」擋不住較大的環；「TTL」只是讓封包最後被丟掉，不是跳出環。
* 前人 TLR [22] 的 **full-record**：每一跳都把自己寫進封包頭，下一跳避開已走過的節點，走投無路就**回溯**到記錄中自己第一次出現位置的前一個節點。缺點是**每個封包都要帶整串 ID**，即使網路很健康。
* 本文 **partial-record**：
    * 封包頭有 nNodeTraversed（記錄了幾個）和 nodeList。
    * 一開始不記。**只有當某顆衛星選了非 primary 方向**，才開始記；從此之後每一跳都記（「一旦開始就一直記」）。直覺：走了非 primary 方向，表示正在繞路，比較可能形成環。
    * **Backtracking node**：s 從 p 收到封包時，看自己在 nodeList 第一次出現的位置。位置是 0 或根本沒被記 → p 是回溯點；否則 → 位置前一個節點是回溯點。所有可用的下一跳都走過了，就送回回溯點。
* 附錄 Theorem 3：只要目的地可達，就**不會有無窮迴圈**（分兩種：兩節點來回、多跳環，逐一討論）。

### 3.4 Vector-based next hop selection（§IV-D）
* 條件很多而且會互相糾纏，所以對每個可用方向算一個 **0/1 評估向量**，用**字典序（lexicographic）**比較，最大的就是下一跳。
* **1 條 primary**：(not_traversed, backtracking_node, **no_blockage, primary_direction**, secondary_direction)
* **2 條 primary**：(not_traversed, backtracking_node, **primary_direction, no_blockage**)
* 兩種情況 no_blockage 和 primary_direction 的順序**相反**，理由（Fig. 9）：
    * 1 條 primary（同一直線上）：primary 路徑有壞鏈路的話，最後一定要多繞，hop 數本來就會變多，乾脆走 secondary。
    * 2 條 primary：primary 路徑上有壞鏈路，不一定會增加 hop 數（可以在矩形內換條路），所以還是優先 primary。
* 例：(1,0,0,0) > (0,1,1,1)。代表「沒走過」比其他條件都重要。

### 3.5 一個小例子（照 Fig. 7 走一遍）
1. 鏈路 1–2 斷了，節點 1、2 各發 link state 封包，f_ttl = 5，s 在 5 跳內所以收到。
2. s 到 d 有兩條 primary：「先右再下」經過 1–2 → N_block = 1；「先下再右」沒經過 → N_block = 0。
3. s 選「先下」，封包從頭到尾沒碰到斷點，因為走的還是 primary 方向，不會觸發記錄，封包頭不用多帶 ID。
4. 如果 s 不在 5 跳內（沒收到通知），封包會照原路走到節點 1 才發現，只能本地繞道，這就是 local distributed algorithm 的行為（相當於 f_ttl = 0）。

## 4. Evaluation Setup（§V-A）
| 項目 | 值 |
|---|---|
| 星座 | Starlink 第一期：32 軌 × 50 顆 = 1600 顆，1150 km，**53°**，F = 5（依 Handley [13]） |
| ISL / UDL 速率 | 100 Mbps / 100 Mbps |
| 佇列 | 100 packets；封包 1000 bytes |
| 模擬器 | **論文沒寫** |
| Baselines | (1) **Link state algorithm**：故障就全網 flooding，收到後等 2 s 收集更多變化再跑 Dijkstra（作者自己實作的 OSPF 類）；(2) **Local distributed algorithm**：一樣的 primary/secondary，但只看自己的鏈路，碰到才繞 = f_ttl = 0 |

## 5. Main Results
* **實驗 1：延遲 vs 故障比例（Fig. 11）**。衛星上隨機產生封包、隨機目的地，平均 1 packet/s（故意很低，只看路徑長度），跑 2 小時；每個故障情境都保證圖仍連通。
    * 0% 故障約 75–76 ms；20% 故障時：local ≈ 108 ms、本文（f_ttl = 2/5/10）≈ 92–94 ms、link state ≈ 79 ms（數字是我從圖讀的）。
    * f_ttl = 5 比 local 少 **15.5%**（作者數字）。
    * f_ttl 從 2 → 5 略好；**f_ttl = 10 在高故障時反而變差**：網路壞太多，所有 primary/secondary 都有壞鏈路，資訊多反而造成多餘繞道。
* **實驗 1：訊令開銷（Fig. 12，log 軸）**。20% 故障時，本文的 link state 封包數約為 link state 演算法的 **1/10（f_ttl = 10）、1/50（f_ttl = 5）**；f_ttl = 2 再少一個數量級。
* **實驗 2：兩個地面終端的會話（Fig. 13、Table 2）**。A（66.1°W, 10.1°S）→ B（71.8°E, 14.9°N），100 packets/s，2 分鐘；只在兩者接入衛星圍成的矩形內隨機產生故障和恢復（間隔和持續時間都是平均 5 s 的指數分布）。
    * 本文延遲曲線和 link state 大致一樣；local 在開頭有一段明顯偏高（約 76 vs 70 ms）。
    * Table 2 封包數：local 0、f_ttl = 2 → 4,425、f_ttl = 5 → 43,034、link state → **1,953,597**。
* **實驗 3：secondary path 的狀態有沒有用（Fig. 14）**。proposal-2 = 不維護 secondary 的 N_block（只有 primary 不能用時才走 secondary）。故障少時差不多；20% 時 proposal-2 約 97–100 ms，本文約 92 ms。
* **實驗 4：partial vs full record（Fig. 15、16）**。延遲幾乎一樣；額外封包頭 bytes：full-record 約 41–50 B（每個 ID 2 B），partial-record 0% 故障時是 0、20% 時約 24 B，大約一半。
* **實驗 5：吞吐量和掉包（Fig. 17–20）**。100 個地面終端（60°S–60°N），一開始就 10% ISL 故障，30 s。三種演算法吞吐量差不多；兩個情境裡沒有哪個一直贏。作者承認：都不是為負載平衡設計的，結果看流量分布。

## 6. Novelty／貢獻
* **新的部分**：
    * 把傾斜星座的**螺旋偏移**形式化（Theorem 1、2），並把它用進 X-Y 路由。這是本文最「傾斜星座專屬」的貢獻。
    * Restricted flooding + **路徑級**的 N_block 計數 + pre-detour 的組合。
    * Partial-record（只在繞路後才開始記）。
    * 向量字典序把多個條件合成一個決策，而且依 primary 數量調整優先順序。
* **不新的部分**：X-Y / Manhattan 路由（[20]）、TTL 限制範圍的 flooding、full-record 回溯（TLR [22]）、主/備方向（DRA [8] 就有）都是既有概念，本文是組合起來並針對傾斜星座調整。

## 7. Limitations
* 作者自己承認的：
    * 只為傾斜星座設計；polar 和混合星座（拓樸更間歇）留待未來。
    * 沒有負載平衡；吞吐量表現取決於流量分布。
* 我額外看到的：
    * **以 hop 數為準，不是距離**：X-Y 路徑是最少 hop，但在矩形裡所有 X-Y 組合 hop 數都一樣，公里數不一樣。作者只取「先橫後直」和「先直後橫」兩條，沒有挑公里數最短的那條。這正是 StepClimb 要解的 SDP vs MHP 問題（見 §8）。
    * **f_ttl 怎麼選沒有方法**：只試了 2/5/10，而且 10 在高故障時反而變差，沒有提供依星座大小或故障率選 f_ttl 的準則。
    * **故障模型很粗**：隨機挑 x% 的 ISL 讓它壞；20% 的 ISL 同時壞掉在現實不太可能。沒有區域性（例如太陽干擾、某一區域集中故障）或相關故障。
    * **模擬器、重複次數、信賴區間都沒寫**，程式碼沒公開。
    * **Baseline 只有兩個自己實作的極端**（全網 link state、純本地），沒有和 DRA、OPSPF、TLR 等已發表方法直接比。
    * **小錯**：Algorithm 3 第 7 行寫 Dm = **arg min**(assess_vec)，但正文說選**最大**的向量，應該是 arg max；結論說「f_ttl = 5 時封包數只有 link state 的 1/10」，但 §V-B 和 Fig. 12 是 **f_ttl = 10 → 1/10、f_ttl = 5 → 1/50**，結論寫錯了。
    * **Link state 那邊的 2 s hold**：baseline 收到 LSA 後等 2 s 才重算，這段時間封包可能還走舊路；延遲是不是包含這部分、掉包怎麼算，論文沒說清楚。
    * 路徑預算依賴「拓樸固定」，只要 ISL 配對規則改變（例如 Theorem 2 的 δ 換了、或多殼層互連），所有衛星都要同步換規則。

## 8. 與其他筆記的關聯
### 和 OPSPF（[[OPSPF_Orbit_Prediction_Shortest_Path_First_Routing_for_Resilient_LEO_Satellite_Networks]]）
* **補 OPSPF 的缺口**：OPSPF 筆記 §7 寫過，OPSPF 只對 **Walker Star（近極軌）** 有用，它的最大賣點（可預測的極區斷線 0 封包）在 Walker Delta（例如 Starlink 53°）上幾乎用不到，因為衛星根本到不了 ±70°。本文剛好是 **Walker Delta 專用**：傾斜星座沒有極區斷線，拓樸可以永久不變，所以「可預測部分」完全用編號規則處理，連 OPSPF 那種每秒推算位置、跑 Dijkstra 都不用。
* **兩者都是「可預測靠預算、不可預測靠訊令」**，但做法不同：

| | OPSPF | 本文（Qi 2020） |
|---|---|---|
| 星座 | Walker Star（polar），LEO-48 | Walker Delta（53°），Starlink 1600 顆 |
| 可預測部分 | 每秒用 GPS＋軌道參數推算拓樸，移除極區鏈路，跑 Dijkstra | 拓樸固定，用編號算 X-Y 主/備路徑，O(1)/目的地 |
| 不可預測故障 | OSPF Hello 偵測，**LSU 全網 flooding** | 偵測後只 flood 到 **f_ttl 跳內** |
| 故障後怎麼走 | 全網重跑 Dijkstra（最佳） | 依 N_block **提早繞道**（不一定最佳） |
| 路徑標準 | 最短距離（Dijkstra 權重） | 最少 hop |
| 開銷 | 每顆每次 O(V log V) 以上 | O(MN) 更新、O(1) 選下一跳 |

* 本文的 link state baseline 本質上就是「OPSPF 的動態模組」那一類（全網 flooding＋Dijkstra），Table 2 顯示它的訊令是 f_ttl = 5 的約 45 倍。所以如果把 OPSPF 搬到 Walker Delta 大星座，它省掉的「極區斷線」部分沒用，剩下的動態模組正好是本文要改進的對象（推論）。
* 反過來，OPSPF 的極區斷線規則本文沒處理：如果把本文用在 polar 星座，primary/secondary 的 X 分量在極區會不可用，需要像 OPSPF 那樣先把可預測的斷線排除（作者也把 polar 列為未來工作）。

### 和 survey 的容錯章節（[[LEO_Satellite_Networking_Relaunched_Survey_and_Current_Research_Challenges]]）
* Survey §4.6「Disruption tolerant networking & Reliability」把本文列為 **[123]**，描述是「傾斜軌道的拓樸和極軌不同，但可預測性讓每顆衛星能決定到每個目的地的多條 primary 和 secondary path；故障時加上有限範圍 flooding 和預算繞道」。和原文一致。
* 同一節還有 DODR [80]（類 AODV＋本地修復）和 **DTDR [84]**（見 [[A_Disruption_Tolerant_Distributed_Routing_Algorithm_in_LEO_Satellite_Networks]]）。本文和 DTDR 機制非常像（都是「主/備方向＋只 flood 到 n 跳＋提早繞道＋loop avoidance」），差別在：本文是 Walker Delta、2 個 secondary、以 hop 數為準；DTDR 是 polar、1 個 alternate、會看緯度挑先橫還是先直。
* Survey 的 "What next?" 認為「收斂夠快的路由設計」是最好的方向。本文的思路其實是另一條：**不追求收斂**，讓遠處的衛星根本不用知道故障，只要附近的提早繞開就好。
* 專案裡的 survey 深讀筆記（project files 的 LEO_Survey_2310.07646_Network_Layer_Deep_Dive.md §2）已經推論「[123] 剛好補 OPSPF 只對 Walker Star 有效的缺口」，讀完原文確認這個推論成立。

### 和 StepClimb（[[Shortest_Path_in_LEO_Satellite_Constellation_Networks_An_Explicit_Analytic_Approach]]）
* 兩者都利用網格規律、不跑 Dijkstra，也都把所有最少 hop 路徑限制在起終點圍成的矩形（StepClimb 叫 MHP 區域）裡。
* 差別：本文只取矩形的兩條「L 形」邊（先橫後直、先直後橫），以 hop 數為準；StepClimb 在矩形裡用解析法找**公里數最短**的那條（依緯度決定在哪裡換軌）。StepClimb 的結果可以直接當本文 primary path 的升級版（推論），但 StepClimb 假設高緯 ISL 不斷，這在傾斜星座裡反而成立，所以兩者比較相容。

### 其他
* [[StableRoute_When_Dijkstras_Algorithm_Meets_Topology-Varying_Satellite_Networks]]：本文的拓樸固定，primary path 不會因為 snapshot 切換而換，天生沒有 StableRoute 處理的「等長路徑 tie-breaking 造成換路」問題；但 pre-detour 會依 N_block 換路，故障頻繁時路徑也會一直換。
* [[A Load Balancing Routing Strategy for LEO Satellite Network]]：本文吞吐量實驗承認沒有負載平衡；SIDA/SSLB 是在同樣的 +Grid 等長路徑上做分流，兩者可以互補（推論）。
