---
title: "Improving the Snapshot Routing Performance through Reassigning the Inter-Satellite Links"
authors: [Zhu Tang, Zhenqian Feng, Wei Han, Wanrong Yu, Baokang Zhao, Chunqing Wu]
venue: "IEEE INFOCOM 2015 Workshops（Student Workshop, Poster Session），pp. 97–98"
doi: 10.1109/INFCOMW.2015.7179361
affiliation: "國防科技大學（NUDT）計算機學院，長沙"
source: "Improving_the_snapshot_routing_performance_through_reassigning_the_inter-satellite_links.pdf"
tags: [LEO, satellite-routing, snapshot-routing, ISL-assignment, topology-design, polar-orbit, Iridium, phased-array, StableRoute-ref]
---

## Citation
Z. Tang, Z. Feng, W. Han, W. Yu, B. Zhao, C. Wu, "Improving the Snapshot Routing Performance through Reassigning the Inter-Satellite Links," *IEEE INFOCOM Workshops (Student Workshop poster)*, 2015, pp. 97–98. DOI: 10.1109/INFCOMW.2015.7179361.

> 這是一篇 **2 頁的學生 poster**，沒有演算法虛擬碼、沒有模擬器說明、很多名詞沒定義。下面凡是「論文沒寫、是我推的」都會標 **（推論）**；第 8 節集中列出無法驗證的地方。

## 一句話總結
快照路由（snapshot routing）把一個軌道週期切成很多時間片，每片裡拓樸當作不變、預先算好路由表。切法有兩種，各有缺點：**natural partition**（ISL 一斷/一連就開新快照）ISL 利用率高，但快照又多又長短不一；**equal partition**（固定時間切，Iridium 用 2.5 分鐘）簡單穩定，但切點跟真實 ISL 變化對不上，只好保守地少用鏈路。作者認為兩者的取捨來自「**ISL 怎麼接是固定的**」，於是提出：**每次切換快照時，順便重新指派跨軌 ISL**（靠相位陣列天線可以改指向別顆衛星），讓 ISL 的斷/連事件剛好對齊快照邊界。在 Iridium（6 軌 × 11 顆）上：快照數固定 **22 個、每個 274.31 s**，且**不受極區邊界緯度影響**；快照數是 natural 的一半（44）；比 Iridium 的 150 s 等分**長 82.87%**；跨軌 ISL 利用率比 equal2 高 **8.68%**（比 natural 高 3.71%）；「延遲變短的路徑比例」相對 natural / equal1 / equal2 平均是 47.67% / 52.61% / **54.57%**（注意：這是**路徑比例**，不是延遲降了 54.57%，見 §4）。

**English:** A two-page INFOCOM 2015 student poster on snapshot routing for polar-orbit LEO networks. Snapshot routing slices the orbital period into intervals with a fixed topology. Natural partitioning (a new snapshot at every ISL on/off event) uses links well but produces many unequal snapshots; equal partitioning (Iridium uses 2.5 min) is simple but wastes links. The authors argue the trade-off comes from fixed ISL assignment, and propose reassigning inter-plane ISLs at each snapshot switch (enabled by phased-array antennas). Oblique "zig-zag" inter-plane links are assigned first, horizontal ones fill a single idle row near the pole, and links that would break soon are not assigned at all, so every snapshot is triggered by exactly one break or set-up event. For Iridium (6×11) this gives 22 equal snapshots of 274.31 s regardless of the polar-border latitude (natural: 44), 82.87% longer snapshots than the 150 s equal partition, 8.68% higher inter-plane ISL utilization than it, and a reported 47.67–54.57% share of paths with reduced end-to-end delay.

## 0. 背景知識
### 這篇用到的名詞
| 名詞 | 意思 |
| --- | --- |
| **Snapshot routing（快照路由）** | 把軌道週期切成時間片；每片拓樸不變，地面預先算好每片的路由表上傳，衛星按時間表換表。ref [1] Fischer et al., TMC 2013 |
| **Natural partition** | Gounder 1999（ref [2]）：只要有 ISL 建立或中斷（超出視線 LOS 或相對角速度太大）就開一個新快照 |
| **Equal partition** | Werner 1997（ref [3]）：固定時間間隔切；Iridium 設 **2.5 min = 150 s**（ref [4]） |
| **Inter-plane ISL（跨軌 ISL）** | 不同軌道面衛星之間的鏈路；在極區附近因為衛星靠太近、天線追蹤角速度太大而關掉 |
| **Polar border L_pa** | 極區邊界緯度。超過這個緯度跨軌 ISL 關閉（論文測 60°、65°、68°、70°、75°） |
| **S_{i,j}** | 第 i 個軌道面的第 j 顆衛星 |
| **LS（line of satellites）** | 一「排」衛星：各軌道面同一位置的衛星連成一列（Fig. 1 的藍色虛線）（推論：論文沒正式定義） |
| **LS_npa0** | 剛離開極區、進入非極區的那一排；重新指派從這排開始往另一極推 |
| **LS_last** | 快要進入極區的最後一排；不給它斜向跨軌 ISL，因為很快就會斷 |
| **NLS_npa** | 非極區內的排數 |
| **Oblique / horizontal inter-plane ISL** | 斜向（黑色，連到相鄰軌道面「錯開一格」的衛星，形成鋸齒）／水平（綠色，連到相鄰軌道面同一排的衛星） |
| **Phased-array antenna** | 電子掃描天線，不用機械轉向就能改指向，讓「換一顆鄰居連」變得可行（ref [5]） |
| **δ / N_ISL / S** | Table I 的欄位：快照長度（s）／跨軌 ISL 數量／一個週期內的快照數。帶上橫線的是模擬值，沒橫線的是理論值 |

### 傳統 +Grid 和本文的差別
* 一般的 +Grid（StableRoute、OPSPF、STARVERI 筆記裡都是這個）：每顆衛星**固定**連同軌前後 + 左右相鄰軌**同一排**的衛星。誰跟誰連一輩子不變，只是在極區「開/關」。
* 本文：跨軌 ISL **不固定**。每次快照切換時重新決定「我跟隔壁軌道的哪一顆連」。Fig. 1 裡大部分跨軌 ISL 是**斜的**（S_{1,1}–S_{2,2} 這種錯一格），整張圖看起來是鋸齒（zig-zag）。

## 1. Problem
* 兩種切法的優點分不開（§I）：
    * **Natural**：跟著真實 ISL 開關走 → ISL 利用率高；但快照數多（Iridium 44 個）、長短差很多（L_pa=65° 時最短只有 **10.11 s**、最長 261.91 s）。
    * **Equal**：長度一樣 → 路由演算法簡單穩定；但切點和 ISL 實際開關不對齊，一條 ISL 只要在某片裡任何時刻不可用，整片都不能用它 → 利用率低（推論：論文沒明講這個機制，但這是 equal 利用率較低的合理原因）。
* 作者的診斷：取捨的根源是**現有 ISL 指派是固定的（inflexible）**。
* 研究問題：「**能不能透過重新指派 ISL 來優化快照分布？怎麼做？**」

## 2. Solution：ISL reassignment（§II，Fig. 1）
### 2.1 做法
1. 在任一時刻 t，從 **LS_npa0**（南北兩半球各自剛離開極區的那一排）開始，往對面極區的邊界依序指派跨軌 ISL。
2. **先指派斜向**跨軌 ISL（黑色斜線）。
3. 如果靠近極區還剩**單獨一排沒配到**（idle line），再補**水平**跨軌 ISL（綠色），見 Fig. 1(c)(d)。
4. 衛星越過極區邊界 L_pa、彼此靠近時，跨軌 ISL 關掉（橘色叉叉），避免超過天線追蹤速度。

### 2.2 為什麼快照會等長
* 規則：**每個快照邊界只由「一個」ISL 中斷事件或「一個」ISL 建立事件觸發**，所以每個快照長度不變。
* **中斷事件**有 4 種重新指派方式（Fig. 1(a)–(d)），依兩個奇偶性分類：(i) 非極區排數 NLS_npa 的奇偶；(ii) LS_npa0 上軌道面編號的奇偶。
* **建立事件**有 8 種：前 4 種同上，多出來的 4 種是為了不讓中斷事件打亂切分 → **LS_last 不給斜向 ISL**，因為它馬上要進極區。
* 論文**沒有**給這 12 種情況的具體規則或虛擬碼，只有 Fig. 1 四張圖。

### 2.3 我對「22 個快照」的理解（推論）
* Iridium 每個軌道面 11 顆，每隔 T/11 就有一排衛星越過某個緯度。
* Natural：每排在一個週期內會碰到 4 次邊界（進北極區、出北極區、進南極區、出南極區），所以 4 × 11 = **44** 個事件 = 44 個快照（推論，和 Table I 的 44 吻合）。
* Reassign：把事件配對（一次中斷 + 一次建立合併處理），每 T/22 切一次 → **22** 個快照。
* 驗算：274.31 s × 22 = 6034.8 s；150 s × 40.23 = 6034.5 s。所以 Iridium 週期約 **6035 s（≈100.6 min）**，**274.31 s = T/22**，**equal2 的 40.23 = T/150**（不是整數，就只是週期除以 150）。這兩個數字都是我自己驗算的，論文沒寫週期。

## 3. Evaluation（§III）
### 3.1 設定
* 星座：**Iridium 6 × 11**（66 顆）。L_pa = 60°、65°、68°、70°、75°。
* 比較：natural、**equal1（274.31 s，等於 reassign 的長度）**、**equal2（150 s，Iridium 實際用的）**。
* 模擬工具、流量模型、延遲怎麼算、「利用率」怎麼定義：**全部沒寫**。

### 3.2 Table I 重點（理論值 / 模擬值一致）
| L_pa | Reassign N_ISL | Natural δ max / min (s) | Natural N_ISL max / min |
| --- | --- | --- | --- |
| 60° | 34 | 179.90 / 92.10 | 30 / 35 |
| 65° | 34 | 261.91 / **10.11** | 35 / 30 |
| 68° | 40 | 182.16 / 89.88 | 35 / 40 |
| 70° | 40 | 158.29 / 113.75 | 40 / 35 |
| 75° | 44 | 215.75 / 56.33 | 40 / 45 |
* Reassign：δ 永遠 274.31 s、S 永遠 22。Natural：S 永遠 44、δ 長短不一。Equal1/2：S = 22 / 40.23。
* Natural 的 N_ISL「max / min」欄位裡 max 有時比 min 小（例如 60° 是 30 / 35），推測欄位意思是「最長快照 / 最短快照時的 ISL 數」而不是最大/最小值（推論）。Equal 欄位的括號值（如 30(25)）論文沒解釋。

### 3.3 四個結果
1. **快照數少且固定**：22 = natural 44 的一半；是 equal2（40.23）的 54.59%（我驗算 22/40.23 = 54.69%，論文數字差 0.1 個百分點）。快照少 → 換路由表的次數少。
2. **快照長且固定**：和 L_pa 無關；平均是 natural 的 2 倍；比 equal2 的 150 s 長 **82.87%**（= 274.31/150 − 1，數字對得上）。
3. **跨軌 ISL 利用率較高**（Fig. 2）：比 equal1 平均高 11.34%、比 equal2 高 8.68%、比 natural 高 3.71%。但 **L_pa = 65° 時比 natural 低**，作者承認「固定等長快照會錯失拿到最多 ISL 的機會」。另外 Fig. 2 只畫了 natural、reassign、equal1 三條，**沒有 equal2 的曲線**，8.68% 無法從圖驗證。
4. **端到端延遲**（Fig. 3，L_pa = 68°，0–6000 s）：y 軸是「**ratio of delay-reduced paths**」，曲線在 0.3–0.8 之間跳動。內文寫「averagely reduces 47.67%, 52.61% and 54.57% end to end path delay」，摘要寫「reduce the delay of 54.57% end to end paths」。對照圖的 y 軸，**正確讀法應是「約 48–55% 的路徑延遲變短」，不是延遲本身降了 50%**；而且反過來說，**另外約一半的路徑延遲沒變短甚至變長**，論文沒報告。

### 3.4 實例：Iridium 在 L_pa = 68° 時到底發生什麼事 ⭐
> 這一節是**我自己做的**，不是論文裡的。用真實的 Iridium 參數寫了一個小模擬（[[snapshot_isl_example/iridium_natural_partition.py]]），重現論文 Table I 的 natural 欄位，再用它說明 reassign 在做什麼。

#### 模型（全部是 Iridium 的公開參數，加上一些簡化）
* 6 個軌道面 × 11 顆，傾角 86.4°，週期 T ≈ 6034.5 s（從 Table I 反推，見 §2.3）。
* 同一軌道面衛星間隔 360°/11 = 32.7°；相鄰軌道面錯開半格（16.4°），這是 Iridium 的相位設計。
* 固定 +Grid：衛星 (p, k) 連 (p+1, k)。第 6 面和第 1 面之間是反向旋轉的 seam，不連 → 跨軌 ISL 最多 **5 × 11 = 55 條**。
* 一條跨軌 ISL 只有在**兩端衛星都在 |緯度| < 68°** 時才開。
* 簡化：地球不自轉、圓軌道、不考慮天線角速度的細節。

#### 圖 A：星座長什麼樣子、ISL 怎麼斷
![constellation](snapshot_isl_example/fig3_constellation_break.png)
* 縱軸是衛星在軌道上的位置 u（90° = 北極正上方、270° = 南極），粉紅帶是極區（|緯度| > 68°）。每一直排是一個軌道面。
* 因為相鄰軌道面錯開半格，固定 +Grid 的跨軌 ISL 看起來本來就是斜的。
* 左圖 t = 42 s 有 40 條；10 秒後（右圖）有一排衛星碰到極區邊界，**5 條 ISL 同時斷**（橘色虛線，南北兩邊都有）。這一刻就是 natural partition 的一個快照邊界。

#### 結果 1：natural partition 的快照「一長一短」交替
在 L_pa = 68° 下，一個週期內 ISL 開關事件的時間（只列前幾個）：

| 時間 (s) | 事件 | 開著的跨軌 ISL |
| --- | --- | --- |
| 47.4 | 一排衛星進入極區 → **5 條 ISL 斷** | 40 → 35 |
| 226.9 | 另一排衛星離開極區 → **5 條 ISL 接回** | 35 → 40 |
| 321.7 | 下一排進極區 → 5 條斷 | 40 → 35 |
| 501.2 | 下一排出極區 → 5 條接回 | 35 → 40 |
| … | 一直重複 | |

* 快照長度就是 **179.5 s、94.8 s、179.5 s、94.8 s……** 交替，一個週期 **44 個**。
* 和論文 Table I 對照（natural, 68°）：論文 δ max / min = **182.16 / 89.88 s**，N_ISL = **35 / 40**，S = **44**。我的簡化模型算出 179.5 / 94.8 s、35 / 40 條、44 個，很接近。其他緯度也對得上：

| L_pa | 我的模型 δ max / min (s) | 論文 δ max / min (s) | 我的 N_ISL | 論文 N_ISL |
| --- | --- | --- | --- | --- |
| 60° | 176.3 / 98.0 | 179.90 / 92.10 | 30 / 35 | 30 / 35 |
| 65° | 267.3 / **7.1** | 261.91 / **10.11** | 30 / 35 | 35 / 30 |
| 68° | 179.5 / 94.8 | 182.16 / 89.88 | 35 / 40 | 35 / 40 |
| 70° | 162.9 / 111.4 | 158.29 / 113.75 | 35 / 40 | 40 / 35 |
| 75° | 214.3 / 60.1 | 215.75 / 56.33 | 40 / 45 | 40 / 45 |

（N_ISL 的順序有時相反，是因為論文欄位是「最長 / 最短那個快照裡的 ISL 數」，不是最大 / 最小值；數量本身都一致。剩下幾秒的差異應該來自我沒算地球自轉、天線限制等細節。）

#### 圖 B：時間軸上，natural vs reassign
![timeline](snapshot_isl_example/fig1_timeline_68deg.png)
* 上圖（我的模擬）：ISL 數在 40 和 35 之間跳，每跳一次就換一張路由表，快照是 **180 s、95 s、180 s、95 s……**。
* 下圖（論文 Table I 的數值）：reassign 每 274.31 s 才切一次，ISL 數固定 40。上圖每一對「180 + 95」剛好對應下圖的一格。

#### 結果 2：關鍵觀察：一長加一短剛好等於 274.31 s
* 179.5 + 94.8 = **274.3 s = T/22**。每個緯度都一樣：98.0 + 176.3、7.1 + 267.3、111.4 + 162.9……加起來都是 274.3 s。
* 原因：相鄰軌道面錯開半格，所以**每隔 T/22 就有一排衛星（在某個軌道面）越過極區邊界**。每個 T/22 週期裡都有「一次斷 + 一次接」，只是這兩個事件的相對時間由 L_pa 決定，所以 natural 的快照長短會隨緯度變。
* 65° 最誇張：斷和接只差 **7 s**，等於路由表要為了一個 7 秒的快照換一次。

#### 圖 C：每個緯度都一樣，以及利用率
![durations](snapshot_isl_example/fig2_durations_util.png)
* 左圖：藍色（natural 長快照）+ 淺藍（短快照）疊起來的高度，在每個緯度都剛好等於紅色（reassign 的 274.31 s）。65° 的淺藍只有 **7 s**。灰色虛線是 Iridium 實際用的 150 s 等分。
* 右圖：紅線是論文 Table I 的 N_ISL ÷ 55，藍線是我模擬的 natural 時間平均。65° 時紅線在藍線下面，就是論文說「65° 輸給 natural」的地方。可以和論文 Fig. 2 對照。
* 這三張圖由 [[snapshot_isl_example/plot_figs.py]] 產生；reassign 的部分只用了論文的數字，**沒有模擬論文的 12 種重指派規則**。

#### 結果 3：reassign 做的事就是把「一斷一接」合併成一次
* Reassign 在每個 T/22 邊界一次處理完：剛離開極區的那排（LS_npa0）立刻配上新的斜向 ISL，快要進極區的那排（LS_last）乾脆不給斜向 ISL。
* 這樣每個 274.31 s 內 ISL 都不會變 → **一個週期 22 個等長快照**，和 L_pa 無關。這就是 Table I 裡 δ_reassign 永遠是 274.31、S 永遠是 22 的由來。
* 在 68° 時，論文的 reassign 一直保持 **40 條**跨軌 ISL；natural 則在 40 和 35 之間跳。

#### 結果 4：順便猜出「利用率」的定義
論文沒定義 Fig. 2 的 utility ratio。如果用「開著的跨軌 ISL ÷ 55」：
* Reassign：34/55 = 0.62（60°、65°）、40/55 = 0.73（68°、70°）、44/55 = 0.80（75°）→ **和 Fig. 2 紅線完全吻合**。
* Natural（我的模型，時間平均）：68° 時 0.668，Fig. 2 藍線約 0.66。
* 65° 時 natural 有 267 s 是 35 條、只有 7 s 是 30 條，平均 ≈ 0.63，**高於** reassign 的 0.62 → 這就是論文說「65° 時 reassign 輸給 natural」的原因。
* 所以利用率 ≈ 可用跨軌 ISL 數 / 55，這是**推論**，但數字對得很準。

#### 用一句話說這個例子
在 Iridium 上，固定接法讓「一排進極區」和「另一排出極區」在每 274 秒內各發生一次，但時間錯開，於是快照被切成一長一短；本文改變 ISL 的接法，讓這兩件事在同一個時刻一起處理，快照就變成固定 274 秒一個。

## 4. 值得讀的段落
1. **§I 最後一段**：「取捨的根源是固定 ISL 指派」這個觀點，是全文唯一的核心 insight，短但值得記。
2. **Fig. 1**：對著圖看斜向 ISL 的鋸齒、LS_npa0 和 LS_last 的位置、橘色叉叉的中斷 ISL。四張子圖就是四種中斷情況。
3. **Table I**：natural 的 δ min 只有 10.11 s，很直觀地說明 natural 切法為什麼不穩。
4. **Fig. 3**：看 y 軸定義，學會分辨「路徑比例」和「延遲降幅」。

## 5. 和專案裡其他 routing 筆記的關係
* **[[StableRoute_When_Dijkstras_Algorithm_Meets_Topology-Varying_Satellite_Networks]]（StableRoute，INFOCOM 2025）**：
    * 同一個目標「**穩定**」，下手的層不同：本文在**鏈路層 / 拓樸層**減少拓樸變化次數（快照 44 → 22）；StableRoute 在**路由層**，拓樸變化照樣發生，只是讓 Dijkstra 不做沒必要的 next-hop 切換。
    * StableRoute 的 **SR_G** 也把一個軌道週期切成 m 個 snapshot、在分層圖上找換路最少的序列。本文正好提供一個**讓 m 變小、每片等長**的方法，兩者可以疊加（推論）。
    * StableRoute 假設固定的 +Grid（Iridium 式接法）；本文直接改掉「固定接法」這個前提。
    * 協調者提到本文是 StableRoute 的參考文獻 [18]；專案裡的 StableRoute 筆記沒有列參考文獻清單，**我無法驗證這個編號**。
* **[[OPSPF_Orbit_Prediction_Shortest_Path_First_Routing_for_Resilient_LEO_Satellite_Networks]]（OPSPF，ICC 2019）**：
    * OPSPF 在 related work 批評 snapshot-based 方法兩點：(i) 要存大量路由表、或頻繁跟地面互動上傳；(ii) 處理不了突發故障。本文是 snapshot-based，所以這兩點都適用。
    * 本文把快照數減半，**部分回應了 (i)**（表少一半）；但 (ii) 完全沒碰，論文也沒討論故障。
    * 極區模型相同：都是「超過某緯度就關跨軌 ISL」。OPSPF 用 70°、LEO-48（6 × 8 Walker Star）；本文掃 60°–75°、Iridium 6 × 11。
    * OPSPF 的做法是「每顆衛星用軌道預測即時算拓樸，不存快照」；本文是「地面預先切好快照」。兩者都利用了極區斷線的**可預測性**，只是一個在線上算、一個離線切。
* **[[Time-_Dependent_Network_Topology_Optimization_for_LEO_Satellite_Constellations]]（DoTD）**、**[[Network_topology_design_at_27000_km_hour]]（Motif）**：都屬於「**ISL 該怎麼接**」這條線。Motif 用重複的連線圖樣、DoTD 隨時間優化拓樸；本文是更早（2015）、更簡單的版本：用斜向 + 水平 ISL 的規則讓快照等長。差別是本文的目標是**快照分布**，不是延遲或吞吐量。
* **[[OSPF_in_Low_Earth_Orbit_Stability_overhead_and_routing_dynamics]]**：那篇量的是 OSPF 在 LEO 每隔多久有 SPF 事件；本文是在**事件發生之前**就把事件數量減少。
* **[[A_Disruption_Tolerant_Distributed_Routing_Algorithm_in_LEO_Satellite_Networks]]（DTDR）**：同樣是極軌 + 極區斷線的場景；DTDR 是分散式路由，本文是集中式快照，可以對照兩種處理同一個可預期斷線的思路。
* **[[Enhancing_LEO_Mega-Constellations_with_Inter-Satellite_Links_Vision_and_Challenges]]（ISL vision）**：相位陣列天線讓 ISL 可以換對象，是本文成立的硬體前提；vision 那篇對 ISL 硬體能力有較新的討論。

## 6. 讀後評論
* **優點**：
    * 觀點有意思：把「ISL 怎麼接」從固定條件變成可調變數，讓 natural 和 equal 的優點可以兼得。
    * 結果乾淨：快照長度和 L_pa 無關，這點對實作很友善。
    * 理論和模擬一致（Table I 兩組數字完全相同）。
* **限制 / 質疑點**：
    * **2 頁 poster**：12 種重新指派規則只給名稱和圖，無法重現；沒有模擬器、流量、延遲模型、利用率定義。
    * **Fig. 3 的表述容易誤讀**：「54.57%」是延遲變短的路徑比例，不是延遲降幅；另外一半路徑的情況沒說。
    * **換鄰居的代價沒算**：重新指派 ISL 要重新對準、捕獲、建鏈（acquisition / pointing），即使是相位陣列也需要時間；論文完全沒算鏈路建立延遲或建鏈期間的中斷。
    * **只測 Iridium**：Iridium 有 **反向旋轉的接縫（seam）**（第 1 和第 6 軌道面方向相反、通常不建跨軌 ISL），論文和 Fig. 1 都沒提到 seam 怎麼處理。傾斜軌道（Walker Delta，例如 Starlink 53°）沒有「極區」這種整齊的斷線模式，方法能不能延伸不清楚。
    * **斜向 ISL 比較長**：連到錯一格的衛星，鏈路距離比水平連線長，單跳傳播延遲會變大；論文沒有分析這個代價（推論）。
    * **L_pa = 65° 時利用率輸給 natural**，作者承認但沒深入。
    * 只看拓樸層的快照數，沒有和實際路由效能（掉包、收斂、換路次數）直接連結。
* **對研究的啟示**：如果研究方向是 OSPF / OPSPF 這類 link-state 路由，這篇提供一個不同的槓桿：**不是讓路由協定更會處理拓樸變化，而是直接設計拓樸，讓變化變少、變規律**。可以想成「OPSPF + 本文的 ISL 規則」：拓樸變化事件減半，OPSPF 的靜態模組要處理的變化也跟著減少（推論，沒有論文驗證過）。

## 7. 參考文獻（論文本身的 5 篇）
1. D. Fischer et al., "Predictable Mobile Routing for Spacecraft Networks," IEEE TMC, 2013.（snapshot routing）
2. V. V. Gounder et al., "Routing in LEO-based satellite networks," 1999.（natural partition）
3. M. Werner, "A dynamic routing concept for ATM-based satellite personal communication networks," IEEE JSAC, 1997.（equal partition）
4. J. V. Evans, "Satellite systems for personal communications," Proc. IEEE, 1998.（Iridium 2.5 min）
5. W. G. Cowley, H. E. Green, "Phased-array antennas for inter-satellite links," AusCTW, 2010.

## 8. 無法驗證 / 待確認
* 本文是否為 StableRoute 的參考文獻 [18]：專案筆記沒有 StableRoute 的參考文獻清單，未核對。
* 12 種重新指派規則的細節：論文只給 Fig. 1，無法重現。
* Table I 中 natural 的 N_ISL「max/min」：§3.4 的模擬支持「最長 / 最短快照裡的 ISL 數」這個讀法；equal 欄位的括號值仍然不知道是什麼。
* 「跨軌 ISL 利用率」的定義：論文沒寫；§3.4 推論為「可用跨軌 ISL ÷ 55」，和 Fig. 2 的 reassign 曲線吻合。
* §3.4 的模擬是簡化模型（不含地球自轉、天線角速度），只重現了 natural partition；reassign 的 12 種規則沒有實作。
* 比 equal2 高 8.68% 的利用率：Fig. 2 沒有 equal2 曲線，無法從圖核對。
* 22/40.23 論文寫 54.59%，我算是 54.69%，差異來源不明（可能是筆誤）。
* Iridium 週期 ≈ 6035 s、274.31 s = T/22 是我從表中數字反推的，論文沒寫。
* Iridium seam 的處理：論文沒提。
