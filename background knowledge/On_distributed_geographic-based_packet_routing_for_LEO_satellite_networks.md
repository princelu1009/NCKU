---
title: "On Distributed, Geographic-Based Packet Routing for LEO Satellite Networks"
authors: [Thomas R. Henderson, Randy H. Katz]
venue: "IEEE GLOBECOM 2000, vol. 2, pp. 1119–1123"
survey_ref: "LEO survey 2310.07646 [70]（§4.7 Geographic Routing）"
tags: [LEO, satellite-routing, geographic-routing, greedy-forwarding, polar-orbit, Walker-Star, seam, background]
---

## Citation
T. R. Henderson, R. H. Katz, "On Distributed, Geographic-Based Packet Routing for LEO Satellite Networks," *Proc. IEEE GLOBECOM 2000*, vol. 2, pp. 1119–1123, 2000.（UC Berkeley EECS；Henderson 的博士論文 [3] 有更完整的方法和結果。）

## 一句話總結
這是一篇 5 頁的會議短文，拿來當**背景知識**很好用。當時有人假設：極軌星座的拓樸是半規則的網格，所以只要把地理位置放進位址，每顆衛星把封包交給**「離目的地剩餘距離最短」的鄰居**（greedy geographic forwarding），就能得到接近最佳的路徑，不需要任何路由協定。作者在 ns 上用 Teledesic（288 顆）和寬頻版 Iridium（66 顆，假設有跨 seam 的 ISL）實際測試。結果分兩半：**延遲上大致成立**，和全域最小延遲最短路比，平均差不到 10 ms（但最壞多到約 60 ms）；**穩健性上不成立**，在三個地方會走進死路（dead-end）：**目的地附近**、**極區**、**counter-rotating seam**。前兩個作者用「局部 link state＋特殊極區範圍」補起來了，第三個（seam 造成的拓樸撕裂）怎麼補都補不完。作者的結論是：對極軌星座，純地理轉送要嘛會失敗，要嘛會變得很複雜，**沒有大家以為的那麼吸引人**。

## 0. 背景知識（讀這篇需要的）
### Greedy geographic forwarding 是什麼
* 每顆衛星知道自己和鄰居的位置。封包要送去目的地時，算「自己到目的地」和「每個鄰居到目的地」的距離，交給**讓距離減少最多**的鄰居。
* 好處：不用交換路由資訊、不用存路由表，計算量 O(鄰居數)。
* 經典弱點（地面 ad hoc 網路也一樣）：**local minimum**。走到某顆衛星，所有鄰居都比自己離目的地遠，就卡住了（dead-end）。
* 本文用的距離是**大圓距離（great-circle distance）**，從衛星目前服務的 cell 中心算到目的地 cell 中心。⚠ Survey 寫成「Euclidean distance」，和原文不同。

### Virtual node（虛擬節點）和地理位址
* Shacham（1988）[4] 最早提出用地理位置當位址；Mauger & Rosenberg [5] 提出 **virtual node**：把地表切成固定的 cell，每個 cell 有一個邏輯位址，當下飛過那個 cell 的衛星就「扮演」那個邏輯節點（Fig. 2）。
* 如果 cell 和衛星一對一，可以定一張完全靜態的邏輯網路，根本不用動態路由。但作者指出這會降低可用度：(1) footprint 邊緣的終端可能被鄰近衛星服務得更好；(2) **sun outage**（衛星和太陽同一視線時無法通訊）需要換衛星；(3) 使用者分布不均，鄰近衛星可能需要把 spot beam 打到熱點。一旦一對一被打破，就需要某種動態路由，地理轉送就是候選之一。
* 同時期的類似工作：Hashimoto & Sarikaya [6]（提了協定但沒驗證）、**Ekici, Akyildiz, Bender [7]（DRA，INFOCOM 2000）**：兩階段，先挑「能讓 hop 數少 1」的候選鄰居，再從裡面挑最能減少剩餘距離的。DRA 後來也被 DTDR 拿來當 baseline。

### 為什麼極軌星座的「網格」不規則
* 星座：極軌（Iridium 86.4°、Teledesic 84.7°）、每顆衛星連 4 個（Iridium）或 8 個（Teledesic）鄰居。
* **極區**：高緯度時 inter-plane ISL 要關掉（Fig. 1）。
* **Counter-rotating seam**：在兩個經度帶上，相鄰兩個軌道面運動方向相反，跨 seam 的鏈路很難維持（Fig. 1 標出「causes rapid ISL handoff」）。
* **Footprint 重疊**：一個終端可能同時看到好幾顆衛星，「目的地由哪顆衛星服務」不一定是幾何上最近的那顆。

## 1. Problem
* **要驗證的假設**：在規則網格的 LEO 星座上，一連串「局部最佳」（往距離減少最多的鄰居送）的決定，會得到接近全域最佳的路徑。
* **兩個指標**：
    * **Robustness**：能不能避免 dead-end（不能就掉包）。
    * **Delay degradation**：和全域最小傳播延遲最短路比，多了多少延遲。
* **有沒有正式 formulation**：沒有。這是一篇「實作＋模擬後回報困難」的論文，沒有最佳化問題、沒有定理。它的價值在於**把失敗模式講清楚**。

## 2. Method：基本規則＋三個補丁
### 2.1 基本 greedy 規則（§III）
1. 衛星收到一個不是自己服務的目的地的封包。
2. 算自己 cell 中心到目的地 cell 中心的大圓距離，再算每個鄰居到目的地的距離。
3. 有鄰居比自己近 → 送給最近的那個；**沒有 → 丟掉封包**。

測試方法：每次在地球上隨機挑兩點，送兩個封包，一個用全域最短路（以傳播延遲為權重，永遠不會掉），一個用分散式協定，比延遲並記錄失敗。每次試驗 10,000 組隨機點（窮舉算不完，但隨機試驗已足以暴露問題）。

### 2.2 補丁一：目的地附近 → Locally scoped shortest path（§III-A）
* **問題**（Fig. 3）：S（接在衛星 1）要送到 D（由衛星 6 服務）。地理轉送一路送到衛星 4，但 4 的所有鄰居都比它離 D 遠，卡住。原因是 footprint 重疊：D 在地理上離 4 很近，但實際服務它的是 6。如果允許「往變遠的方向送」，就可能形成迴圈；在封包頭記錄走過的節點可以避免回頭，但還是不能保證最後找到正確的出口衛星。
* **解法：混合式（hybrid）**。每顆衛星跑一個 link state 協定，但 LSP **只 flood 到 routing radius（路由半徑）內**，半徑要大到涵蓋「所有可能服務目的地的衛星」。所以每顆衛星都有一張以自己為中心的小地圖。**遠的時候用地理轉送，靠近目的地時用最短路走完最後幾跳**。Fig. 3 的例子半徑是 2 跳。
* **迴圈風險**：每顆衛星看到的子圖不同，可能出問題：(1) 半徑不同的節點會留著過期資訊；(2) 封包進了目的地的半徑後又被送出去；(3) 用不同 metric 會有迴圈（一般動態路由的通病）。
* **作者的處理**：每顆衛星在 LSP 裡宣告自己的半徑；最短路演算法改成建出**完整路徑**，並檢查「自己是否在路徑上所有節點的半徑內」；衛星縮小半徑時，要主動把舊 LSP 從原本半徑邊緣的節點清掉。
* 開銷：要實作 shortest-path 協定，但 LSP 只傳到小範圍，處理和記憶體開銷大幅減少。

### 2.3 補丁二：極區 → 特殊 polar routing zone（§III-B）
* **問題**：高緯度 inter-plane ISL 關掉，要送到「跨軌鏈路已關」的衛星，必須先送到同一軌道上**較低緯度**的衛星再往上走。所以地理轉送可能在**離目的地好幾跳的地方**就走錯。
* 直接把半徑加大到能涵蓋整個極區需要 5–6 跳，會大量溢出到低緯度，而路由狀態大約隨跳數**平方**成長，太貴。
* **解法**：把衛星分三種狀態（Fig. 4 是從北極上方看 Iridium 86.4°）：
    * **POLAR**：至少一條 inter-plane ISL 關掉的衛星。
    * **POLARBORDER**：和 POLAR 相鄰的衛星。
    * **LOWLATITUDE**：其他。
    * 每顆衛星看自己同軌鄰居的狀態就知道自己是哪一種。
* 極區內的衛星把 LSP 傳遍**整個極區**（另外加上一般的半徑）。往極區目的地的封包，只要碰到任何一顆極區衛星，就改用最短路。離開極區時把這些額外狀態清掉，並通知其他極區衛星。
* **Tunnel**：從極區往外送時，用兩顆 POLARBORDER 衛星的位置來算方向，而不是直接用鄰居的位置。
* 代價：高緯衛星要存更多狀態、訊息也多一點；但極區流量本來就很少，可以抵銷。

### 2.4 補丁三（失敗）：Counter-rotating seam（§III-C）
* 跨 seam 可以建 ISL，但捕獲和同步比一般跨軌 ISL 難很多，而且網格在這裡**變形**：
    * Teledesic 設計裡每顆衛星**只有 1 條**跨 seam ISL（另一條用來在換手前先捕獲下一顆），鏈路很少。
    * 更嚴重的是：傾角不是 90° 時，seam 兩側反向的軌道面會在**比其他軌道面低很多的緯度**交叉，所以跨 seam ISL 必須在**相對低的緯度**就關掉，造成網格**撕裂（tear）**。
    * 交叉緯度公式：**φ = arctan( sin(s/2) · tan(i) )**，i = 傾角、s = 軌道面間距。Teledesic（i = 84.7°）約 **54°**。
    * 我自己代入驗算：Teledesic 288 顆 = 12 面 × 24 顆，s = 15° → φ ≈ 54.6°，和論文一致；若 Iridium seam 間距約 22°、i = 86.4° → φ ≈ 72°（Iridium seam 間距是我用常見參數推的，論文沒給）。
* 作者試了好幾種分散式的「繞過撕裂」tunnel 方法，**都找不到夠簡單的**；就算建了 tunnel，總還是找得到會撞到 dead-end 的例子。而且只要 seam 把兩端隔開，失敗就會**持續（可能好幾小時）**，不是偶發。
* 作者也指出：衛星故障造成的其他拓樸撕裂應該也會有類似 dead-end，但沒研究。
* 這個問題「不論用不用地理轉送都存在」，是傾角偏離 90° 的極軌星座本身的缺點；但完全 90° 的軌道發射成本太高。

## 3. Evaluation（§IV，Fig. 6）
* Teledesic，10,000 組隨機終端，同一組點跑兩次（hybrid 協定 vs 全域最小延遲最短路），逐點相減，按終端距離每 1000 km 分一組取平均。
* 重點看 **> 5000 km** 的點（這些封包至少要走一段地理轉送才會進入最短路半徑）。
* 結果（從圖讀）：
    * **平均差 < 10 ms**（大多在約 2–8 ms，誤差棒 = 1 個標準差）。作者認為這對使用者不算明顯。
    * **最大差可到約 60 ms**（圖上大約 15–57 ms，距離越遠越大），來自少數離群值：發生在**極區附近**，地理轉送先把封包帶到「距離上」離目的地很近、但「拓樸上」很遠的地方，之後只好繞回服務目的地的那個軌道面。
* 本文**沒有給 dead-end 的發生頻率**，也沒有 Iridium 的數字圖；穩健性的結論主要是定性描述。
* ⚠ Survey 寫「the paper bounds the delay difference by 10 ms」，其實 10 ms 是**平均**，不是上界；最壞情況約 60 ms。

## 4. Novelty／貢獻
* 不是提出新演算法，而是**用模擬檢驗一個流行的假設，並系統性地列出它在哪裡失敗**。
* 提出的 hybrid（遠處地理轉送＋近處 scoped link state）和 POLAR/POLARBORDER/LOWLATITUDE 分區是實際可用的補丁。
* 給出 seam 交叉緯度的公式，說明「近極軌但不是 90°」的星座在 seam 附近會有低緯度的拓樸撕裂。

## 5. Limitations
* 作者自己承認的：
    * seam 問題沒有解決；沒有考慮節點故障。
    * 結論只針對極軌星座。
* 我額外看到的：
    * **只有 5 頁**：協定細節、半徑值、模擬參數（高度、ISL 數量、cell 大小）都在 Henderson 的博士論文 [3] 和 [8]，本文無法單獨重現。
    * **沒有量化穩健性**：dead-end 發生率、在哪些位置、持續多久都沒有數字。
    * **只看傳播延遲**：沒有流量、壅塞、佇列，所以「延遲差 < 10 ms」是在空網路下的結果。
    * **只有 Teledesic 的延遲圖**；Iridium 的結果沒有呈現。
    * **沒有和 DRA [7] 比較**，雖然它們的規則非常接近。
    * **時代限制**：2000 年的 Teledesic、Iridium 設計都是近極軌；今天的主流（Starlink 53°）是 Walker Delta，沒有 seam 和極區，這篇的三個失敗點中有兩個不存在（推論）。

## 6. 與其他筆記的關聯
### 和 survey §4.7 Geographic Routing（[[LEO_Satellite_Networking_Relaunched_Survey_and_Current_Research_Challenges]]）
* Survey 把本文列為 [70]，說在一般情況下接近最佳路徑（「bounds the delay difference by 10 ms」），但在 counter-rotating seam、極區、靠近目的地時會失效。
* 讀完原文的修正：(1) 用的是**大圓距離**，不是 Euclidean；(2) 10 ms 是**平均**，最壞約 60 ms；(3) 前兩個失效點作者其實**已經補起來**（scoped link state、極區分區），真正補不了的只有 seam。
* Survey 的 "What next?" 說地理路由的好處是極簡單，但目的地（地面）相對星座在移動、上升段和下降段拓樸不同，要能快速算出鄰居位置才實用。本文的經驗是：簡單只在網格規則的地方成立。

### 和 OPSPF（[[OPSPF_Orbit_Prediction_Shortest_Path_First_Routing_for_Resilient_LEO_Satellite_Networks]]）
* **同一種星座、同兩個麻煩**：OPSPF 也是極軌＋seam，也要處理極區 inter-plane ISL 關掉。
* **處理方式相反**：
    * 本文：不交換全網資訊，靠地理位置貪婪轉送；極區的不規則用「極區內 flooding」補。
    * OPSPF：每顆衛星用軌道參數**預測**全網拓樸、先移除極區鏈路，再跑全網 Dijkstra；只有非預期故障才 flood LSU。
* 本文的結論（極區和 seam 讓局部決策走進死路，要加很多複雜度）可以看成 OPSPF 選擇「每顆衛星都算全網最短路」的動機之一：有了全網拓樸預測，極區和 seam 只是「預知哪些邊不存在」，Dijkstra 自然會繞開，不會有 dead-end（推論）。
* 但 OPSPF 也沒有特別處理 seam（OPSPF 筆記 §7 有提），而本文的公式顯示：對 86.4°–84.7° 的星座，跨 seam 鏈路在 54°–72° 就要關，比 OPSPF 假設的 ±70° 極區邊界**更低**（Teledesic 情況）。如果要在 OPSPF 的靜態模組裡正確預測拓樸，seam 鏈路應該用這個交叉緯度規則另外處理（推論）。

### 和 StepClimb（[[Shortest_Path_in_LEO_Satellite_Constellation_Networks_An_Explicit_Analytic_Approach]]）
* 兩者都想「不跑 Dijkstra、直接用幾何算路」，但方向不同：
    * 本文：**逐跳貪婪**，每一步只看鄰居，沒有全域保證，所以會走進 local minimum。
    * StepClimb：**一次算出整條路**，先限制在最少 hop 的 MHP 矩形裡，再用解析式決定在哪個緯度換軌，保證是（近似）最短距離。
* 本文的延遲離群值（極區附近，「距離近但拓樸遠」）正好說明：在 LEO 網格上，**地理距離減少 ≠ hop 數或路徑長度減少**。StepClimb 的做法是先保證 hop 數最少（不會繞遠），再在等 hop 的路徑裡挑公里數最短，避開了這個陷阱。
* 但兩者都要面對「高緯 ISL 斷不斷」的問題：StepClimb 假設高緯 ISL 不斷（StepClimb 筆記 Limitations），本文則證明在極軌星座上 ISL 一斷，局部規則就會出錯。

### 其他
* [[A_Distributed_Survivable_Routing_Algorithm_for_Mega-Constellations_With_Inclined_Orbits]]（Qi 2020）和 [[A_Disruption_Tolerant_Distributed_Routing_Algorithm_in_LEO_Satellite_Networks]]（DTDR 2022）：本文的「LSP 只 flood 到 routing radius 內」是後來 Qi 的 f_ttl 和 DTDR 的 n 跳 flooding 的前身。Qi 用 Walker Delta（沒有 seam、沒有極區）剛好避開了本文的兩個失敗點；DTDR 則假設極區 ISL 不斷、seam 不建鏈，所以也繞過了本文的問題。
* DRA（Ekici et al. [7]）：本文說它是「兩階段：先挑讓 hop 少 1 的鄰居，再挑最能減少距離的」，比純地理轉送多了「hop 數一定變少」的保證，所以不會像本文那樣在目的地附近卡住（推論）。
