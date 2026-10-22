---
title: "StableRoute: When Dijkstra's Algorithm Meets Topology-Varying Satellite Networks"
authors: [Tian Pan, Guohao Ruan, Qiang Fu, Zhengjie Luo, Junkai Huang, Xingshuang Luo, Tao Huang]
venue: IEEE INFOCOM 2025
doi: 10.1109/INFOCOM55648.2025.11044485
tags: [LEO, satellite-routing, ISL, Dijkstra, route-stability]
---

## 一句話總結
StableRoute 沒有換掉 Dijkstra，只是讓它在不必換路的時候就不換。
低軌衛星網路裡，不同軌道之間的連線會因為衛星移動一直斷掉又接回來，所以每顆衛星要不停重新計算「封包下一步要傳給誰」（下一跳）。問題是一般用的 Dijkstra 演算法沒有記憶：如果有好幾條路一樣短，它只會挑計算時第一個找到的那條，不管上次選的是哪條。所以就算原本的路還能用、也還是最短，它也可能突然換到另一條一樣短的路。這種沒必要的換路會讓封包順序亂掉，TCP 誤以為掉封包，就開始重傳、降速。StableRoute 的想法很簡單，就是讓 Dijkstra「念舊」：先把所有一樣短的下一跳都記下來，如果原本那個還在裡面就繼續用它，不要換。這是 SR_L。SR_K 更寬鬆一點，原本的路就算比最短路多 1～2 跳也照用，等於用多走一兩跳來換取穩定。SR_G 則是利用衛星軌道可以事先預測，直接把一整圈軌道週期的路線一次規劃好，讓換路次數最少。實驗結果是，在大型星座（40×40 Walker Star）上，跟原本的 Dijkstra 相比，換路次數大約少了 43～46%；小星座的改善少很多（4×4 只有約 10～14%）。你筆記裡提到的缺點是：論文只算了「換路次數」，沒有量延遲或吞吐量；TCP 的例子只有單一連線；而且每條連線的成本都設成 1，會讓「一樣短的路」比實際情況多，可能讓問題看起來比真的嚴重。

## Introduction
* **Problem:** In LEO networks, routes have to be recomputed all the time, and plain Dijkstra makes this worse. When several paths are equally short it keeps the first one it finds and ignores what it chose last time, so it often switches the next hop even when the current path still works and is still shortest. Each unnecessary switch reorders packets, which makes TCP assume loss, retransmit and slow down. The paper asks how to cut these avoidable route updates while staying within Dijkstra-based routing.
* **Target:** The only objective is to minimize the **number** of next-hop route updates (path switches), not the time a switch takes. Path length is kept shortest by SR_L / SR_G and deliberately traded for stability by SR_K (1–2 extra hops). Latency, throughput and load balancing are neither optimized nor measured. Better TCP performance is the intended end effect, but it is only shown with three single-flow examples, so the paper directly demonstrates fewer switches and assumes the TCP benefit follows. There is no formal problem formulation (no objective function or constraints); the paper motivates the three scenarios with examples and gives one algorithm per perspective.
* In LEO networks, inter-orbit ISLs go down as satellites approach orbit intersection points and come back up afterwards, while intra-orbit ISLs stay up. Routes are therefore recomputed constantly.
* Most routing protocols use Dijkstra, which is stateless: when several shortest paths tie, it keeps the first one it finds. Many next-hop changes are therefore unnecessary, and each one reorders packets, invalidates the TCP congestion window and triggers retransmissions.
* StableRoute makes Dijkstra remember its previous choice and keep it whenever possible, in three variants:
    * **SR_L** keeps the current next hop as long as it is still on a shortest path.
    * **SR_K** also keeps it if the current path is at most k hops longer than the shortest.
    * **SR_G** uses the predictable orbits to plan a whole cycle of next-hop choices with the fewest changes.
* In emulation, all three cut route updates by roughly 43–46% on a 40×40 constellation compared with plain Dijkstra.

## 背景：星座類型與模擬工具
### Walker Star 和 Walker Delta（§II-A）
* **傾角（inclination）**：軌道平面和赤道的夾角。0° 是沿著赤道繞，90° 是正好經過南北極（polar orbit，極軌道）。
* **Walker 星座**：一種「平均擺放」衛星的標準設計，所有軌道高度、傾角都一樣，每條軌道上的衛星等間距。論文的 **N×N** 就是 N 條軌道、每條 N 顆。
* **Walker Star**（論文例子：OneWeb，87.9°）：
    * 傾角接近 90°，各軌道平面只分散在 180° 經度內。
    * 比喻：像**地球儀上的經線**，所有軌道在**極區交會**。
    * 有一條 **seam（縫）**：第一條和最後一條軌道並排，但一條往北、一條往南飛，通常不連 ISL。
    * 本文主要實驗（4×4～40×40）都用這種，OPSPF 的 LEO-48 也是這一類。
* **Walker Delta**（論文例子：Starlink，53°）：
    * 傾角較小，各軌道平面分散在完整 360° 經度。
    * 比喻：像**毛線球上斜斜纏繞的毛線**，軌道在**中緯度交會**。
    * 本文只測一組 20×20（Fig. 12）。
* **為什麼要分兩種**：ISL 是在「靠近軌道交會點」時斷（Fig. 1），交會點位置不同，斷線的地點和模式就不同。論文說 Walker Star 結構比較簡單，所以 SR_L（只看局部）和 SR_G（看全局）表現接近；Walker Delta 結構較複雜，SR_G 的全局優勢才明顯（46.93% vs SR_L 34.24%）。

### STK、SPP、SNG（實驗用的工具鏈，§V-A）
* **STK（Systems Tool Kit）**：AGI（現屬 Ansys）出的商用衛星軌道模擬軟體，輸入軌道參數就算出**每顆衛星在每個時間點的位置**和 ISL 資訊。比喻：它是「天文台」，只告訴你衛星在哪，不管網路。
* **SPP（Satellite Position Parser）**：作者自己寫的程式，讀 STK 的輸出，整理成「什麼時候哪條 ISL 存在」。
* **SNG（Satellite Network Generator）**：作者基於 **Mininet** 寫的工具。Mininet 是開源工具，能在一台 Linux 電腦上建出很多虛擬節點和鏈路，跑**真實的協定程式碼**。SNG 用它建出虛擬衛星網路，並依照 SPP 的資料開關鏈路。
* **流程**：STK 算軌道 → SPP 轉成鏈路開關時間表 → SNG 在 Mininet 上建網路、跑路由。OPSPF 用的是同樣的 STK，但網路部分用 Exata。

### 衛星位置怎麼預測：TLE、GPS、SGP4、STK
StableRoute 和 OPSPF 都說「衛星軌跡可預測」。實際上靠的是下面四樣東西，分成三種角色：**量現在的位置、記錄資料、往未來推**。

| | 是什麼 | 角色 | 比喻 |
|---|---|---|---|
| **GPS** | 衛星上的接收器，量出自己**現在**在哪 | 量測（在衛星上） | 看手機地圖知道「我現在在這」 |
| **TLE**（Two-Line Element set） | 美國太空軍用地面雷達追蹤後，定期發布的兩行軌道資料（CelesTrak、Space-Track 可免費下載） | 資料（某一刻的軌道） | 一張寫著「幾點、在哪條路、跑多快」的紙條 |
| **SGP4** | 一套軌道推算公式／程式 | 往未來推 | 計算機：拿紙條算出 1 小時後在哪 |
| **STK**（Systems Tool Kit） | AGI（現屬 Ansys）的商用軟體 | 全包的工具箱 | 一整套導航 App |

```
量現在的位置               往未來推                  網路要的東西
地面雷達 ──→ TLE ──┐
                   ├──→ SGP4（或其他模型）──→ 未來位置 ──→ ISL 開關 ──→ 拓樸 ──→ 路由
衛星上的 GPS ──────┘
          ╰─────────────── STK 把這一段全包了 ───────────────╯
```

* **TLE 的重要欄位**：衛星編號、**epoch**（這筆資料是哪一刻）、傾角、RAAN（軌道平面的方向，Walker 星座各軌道靠它錯開）、離心率（接近 0 就是圓軌道）、每天繞幾圈、大氣阻力項 B*。
* **TLE 和 SGP4 是一對**：TLE 是專門給 SGP4 用的格式，配別的模型反而會不準。
* **GPS 和 TLE 都是「現在」的起點**，差在誰量的：
    * TLE：地面從外面看，人人拿得到，但只有幾公里的精度，而且一兩天才更新一次。離 epoch 越久越不準；衛星變軌後舊的就不能用。
    * GPS：衛星自己量，精確到公尺級、隨時都有，但只有衛星自己（或營運商）知道。
* **STK 是工具箱**：裡面有 SGP4，也有更精確的模型；可以吃 TLE，也可以自己設定星座參數。還會算「兩顆衛星看不看得到彼此」，也就是 ISL 能不能開。
* **做路由研究需要知道的程度**：位置計算交給現成工具；要關心的是**預測什麼時候不可信**（TLE 太舊、變軌、故障），以及**誰來算**（地面先算好上傳，還是衛星自己算）。

**各論文用哪一種**
| 論文 | 位置從哪來 | 誰算 |
|---|---|---|
| StableRoute | STK 自己設定理想 Walker 星座，**沒用 TLE 和 GPS**；論文沒寫 STK 用哪個模型 | 真實系統沒交代 |
| OPSPF | 每顆衛星用 GPS 知道自己在哪，再用軌道幾何推算別人；實驗也用 STK | 每顆衛星自己 |
| DoTD | 真實 Starlink TLE，所以星座不整齊 | 預先算好約 10 分鐘的拓樸 |
| Starlink self-driving | 用 SGP4 從前一筆 TLE 推出「沒變軌的話應該在哪」，再和下一筆 TLE 比，對不上就是變軌 | （量測論文） |

**想深入**：Vallado et al.〈Revisiting Spacetrack Report #3〉（AIAA 2006，SGP4 的標準參考，[免費](https://celestrak.org/publications/AIAA/2006-6753/)），只讀前半講 SGP4 在算什麼的部分就夠。

## 背景：ISL 拓樸（+Grid 和同一層的其他設計）
**這一層在做什麼**：先決定「每顆衛星的雷射 ISL 要接到哪幾顆衛星」，接好的線就是路由看到的圖。StableRoute、OPSPF、OSPF 都是**這張圖上面**的路由層；這一節講的是**下面那層**：圖本身怎麼長。比喻：拓樸設計是「修哪幾條路」，路由是「在修好的路上選怎麼走」。

### +Grid（本文和 OPSPF 用的拓樸）
* **定義**：每顆衛星 4 條 ISL：
    * **2 條同軌（intra-plane）**：連同一條軌道上的前一顆和後一顆。
    * **2 條跨軌（inter-plane）**：連左右相鄰軌道上、編號相同的那顆。
* **長相**：像棋盤，每格往上下左右各一條線（所以叫「+」）。同軌方向首尾相接成環；Walker Delta 的跨軌方向也繞一圈，整張圖像甜甜圈表面（torus）。
```
   軌道1  軌道2  軌道3
    o------o------o      o = 衛星
    |      |      |      | = 同軌 ISL（一直都在）
    o------o------o      - = 跨軌 ISL（高緯度或軌道交會處會斷）
    |      |      |
    o------o------o
```
* **Walker Star 的 seam**：第一條和最後一條軌道並排但反向飛，相對速度太大，中間不接線，所以棋盤只有左右兩邊的邊界，不是完整的甜甜圈。
* **為什麼會斷**：同軌 ISL 前後兩顆一起飛，幾乎不會斷。跨軌 ISL 在軌道交會處（Walker Star 在極區、Walker Delta 在中緯度）左右鄰居會換邊、天線轉不過去，只好暫時關掉（Fig. 1，OPSPF 用 >70° 關線）。**StableRoute 處理的「拓樸變化」主要就是這些跨軌 ISL 的開和關**。
* **為什麼大家都用**：Iridium 就是這樣接；規則簡單、每顆衛星接法一樣；相連的兩顆相對速度小，線可以撐很久。
* **和本文的關係**：
    * **很多等長路**：棋盤上從 A 到 B，「先往右再往上」和「先往上再往右」跳數一樣。這就是 **Scenario 2（換到另一條一樣短的路）占 31.9%** 的原因，SR_L 的 next-hop set 才常常有東西可選。
    * **鄰居最多 4 個**：next-hop set 最多 4 個成員，SR_G 每步最多 4 種選擇，所以樸素窮舉是 **4^m**。
    * **我的推論（論文沒說）**：如果換成下面那些拓樸，等長路會變少，SR_L 能「不換路」的機會可能變小；論文只測 +Grid，這點沒有驗證。
* **缺點**：只接最近的鄰居，遠距離要走很多跳。Motif 論文的數字：+Grid 平均 stretch（延遲比）1.25 已經很低，但平均跳數 10.57 很高；DoTD 論文的紐約→舊金山例子 +Grid 要 22 跳，DoTD 5 跳。

### 同一層的其他拓樸
都在回答同一個問題：每顆衛星有限的雷射頭（通常 4 個）要接誰。

| 拓樸 | 每顆幾條 | 接誰 | 會不會換線 | 一句話 |
|---|---|---|---|---|
| **只有同軌（ring）** | 2 | 同軌前後 | 不換 | 最穩但每條軌道各自孤立，要靠地面站轉；只能當比較的底線 |
| **+Grid** | 4 | 同軌前後 + 左右軌道同編號 | 只在交會處關跨軌線 | 本文用的，見上 |
| **3 條 backbone（LAMP 的骨幹）** | 3 | 同軌前後 + 1 條跨軌，東西交錯 | 不換 | 比 +Grid 少 1/4 的線，延遲 +18.4%，省下的雷射頭拿去接臨時線 |
| **xGrid** | 4 | 跨軌改接斜對角那顆（我的理解，未核對原文） | 和 +Grid 類似 | DoTD 程式裡的 `CROSS_GRID` baseline |
| **Motif**（CoNEXT 2019） | 4 | 窮舉「一顆衛星往外接哪 2 顆」的局部樣式，全星座套同一個 | 不換 | +Grid 只是 1,029 個 motif 裡的一個；最好的 motif 的 Φ1 比 +Grid 少約 45% |
| **Multi-motif** | 4 | 依緯度分 3 區，每區用不同 motif | 跨區時換，約每 12 分鐘 | 比單一 motif 再好約 7% |
| **LAMP**（ICC 2025） | 3 固定 + 1 臨時 | 第 4 個雷射頭臨時接剛好飛很近的遠軌道衛星 | 臨時線會換 | 延遲比 +Grid 少 18.5% |
| **DoTD**（INFOCOM 2025） | 4 | 每個時段每顆衛星重選分數最高的 4 顆 | 每個時段都可能換 | 用真實 TLE（907 顆 Starlink），論文自稱最佳但其實是 greedy |

* **一條光譜**：從「完全不換線、規則固定」（ring、+Grid、motif）到「一直換、針對當下最佳化」（LAMP、DoTD）。越往右延遲越低，但**拓樸變化越多**，路由層的負擔越大。
* **為什麼這和 StableRoute 有關**：StableRoute 的目標正是「拓樸變了，路盡量不要換」。拓樸設計越激進（DoTD、LAMP），StableRoute 這種方法越有用武之地，但論文只測了 +Grid 這種最平靜的拓樸。這是我的推論，可以當成延伸方向。
* **不是同一層的**：Slim（INFOCOM 2024）決定的是「星座要幾顆衛星」，比拓樸再下面一層；Starlink self-driving（MobiCom 2023）講衛星變軌造成的拓樸變化，是拓樸變化的**來源**之一。

### 延伸閱讀（你已經有筆記）
* Motif：[[Network_topology_design_at_27000_km_hour]]
* LAMP
* DoTD：[[Time-_Dependent_Network_Topology_Optimization_for_LEO_Satellite_Constellations]]
* Slim：[[Your Mega-Constellations Can Be Slim]]

## 背景：Snapshot 路由（取自 [18] 的 Introduction）
StableRoute 的 Related Work 把 [15–18] 歸成「snapshot 路由」，SR_G 也是在一串 snapshot 上做事。這一節整理 [18]（Tang et al., INFOCOM Workshops 2015）Introduction 講的基本概念。

### Snapshot 路由是什麼
* **想法**：衛星的拓樸雖然一直變，但**可以預測**。所以把一個軌道週期切成很多時間片（snapshot），每一片裡當作拓樸不變（哪些 ISL 開著、每條延遲多少都固定）。
* **做法**：每一片預先算好一張路由表，衛星照時間表到點就換表。
* **比喻**：像公車的「平日時刻表、假日時刻表」。路況會變，但變化是事先知道的，所以按時間切換就好，不用每次塞車才重新規劃。

### 兩種切法
| | Natural partition（Gounder 1999，StableRoute [17]） | Equal partition（Werner 1997，StableRoute [16]） |
|---|---|---|
| 怎麼切 | 只要有一條 ISL 建立或中斷，就開一個新 snapshot | 每隔固定時間切一次。Iridium 用 2.5 分鐘（150 s） |
| 好處 | 跟著真實的 ISL 開關走，**ISL 利用率高** | snapshot 長度都一樣，路由演算法**簡單、穩定** |
| 壞處 | snapshot 數量多、長短不一（Iridium 一個週期 44 個；極區邊界 65° 時最短只有約 10 s） | 切點和 ISL 實際開關對不上，用到的 ISL 比較少 |

* **兩個優點分不開**：[18] 認為原因是 ISL 的指派方式固定不變（每顆衛星永遠連同樣的鄰居，只在極區開關）。
* **[18] 的解法**：每次切換路由表時，順便把跨軌 ISL **重新配對**（要用 phased-array antenna 才能快速換對象），讓「一斷一接」合併成一次切換。結果 Iridium 一個週期是 22 個等長的 snapshot（每個 274.31 s），和極區邊界設在幾度無關。詳細見 [[Improving_the_snapshot_routing_performance_through_reassigning_the_inter-satellite_links]]。

### 和 StableRoute 的關係
* **SR_G 的 snapshot 是 natural 切法**：每次 ISL 斷或連就是一個新 snapshot（見下面 System Model）。
* **StableRoute 對 snapshot 路由的批評**（§VI）：它們只在乎效能，沒處理路由穩定。但在同一個 snapshot 裡路由本來就不會變，換路只發生在 snapshot 切換時。所以 snapshot 越少、越長，換路機會越少。這是我的推論，論文沒這樣比。
* **兩種「減少換路」的方向**：
    * [18] 從**拓樸層**下手：改 ISL 怎麼接，讓 snapshot 變少（44 → 22）。
    * StableRoute 從**路由層**下手：拓樸照樣變，但每次變的時候盡量選和原本一樣的下一跳。
    * 兩者理論上可以疊在一起用（推論）。

## 背景：TCP 為什麼怕換路
* **TCP** 是網路上最常用的傳輸協定（網頁、檔案下載都靠它），負責讓資料完整、照順序送到：資料切成很多編號的小封包，接收端每收到一個就回報「收到第幾號了」。
* 如果後送的封包比前面的先到，接收端會一直重複回報「還在等第 N 號」（duplicate ACK）。傳送端看到重複回報，會以為封包掉了，於是**重傳**（Fast Retransmit），並把傳送速度**降下來**。
* **Congestion window（壅塞視窗，cwnd）** 是 TCP 用來控制一次能送多少資料的設定，是依照目前這條路的狀況慢慢調出來的。換到新路後，舊的 cwnd 不一定適用：新路比較塞就會送太多而掉封包，新路比較寬就會用不滿頻寬。
* 所以換路會傷 TCP：新路比舊路快時，後送的封包先到，順序亂掉，TCP 誤以為掉封包。封包其實沒丟，卻白白重傳又降速。這就是 StableRoute 要盡量減少換路的原因。

### 論文的 TCP 實驗（§V-D，Fig. 13–15）
* **實驗做法**：在模擬器裡架一個只有 s1～s6 幾顆衛星的小網路，從 s1 到 s6 開一條 TCP 連線持續傳資料，傳到一半故意讓某條連線斷掉或接回來，逼路由重算，再比較 Dijkstra（會換路）和 StableRoute（不換路）的重傳次數。
* **圖怎麼看**：橫軸是時間（秒），縱軸是接收端照順序收到的資料量（sequence number，byte）。線越陡 = 傳得越快；線變平 = 卡住；變成一大片鋸齒 = 封包亂序、一直重傳。藍色實線是 Dijkstra，紅色虛線是 StableRoute。
* **Fig. 13：換到比較塞的路（重傳 1240 次）**：2.3 秒前兩條線重疊；Dijkstra 換路後藍線幾乎變平，4.5～5 秒的小鋸齒是重傳。紅線（SR_L/G）留在原路，一路往上。這次換路讓 Dijkstra 的傳輸幾乎停擺。
* **Fig. 14：換到比較寬的路（重傳 2271 次）**：1.1～2.7 秒那一大片藍色是嚴重亂序（新路的封包先到、舊路的封包還在後面）。2.7 秒後亂序結束，藍線快速爬升；紅線（SR_L/G）平穩但很慢，因為它留在比較塞的舊路上。到 3.5 秒時 Dijkstra 收到約 3.4×10⁷ B，StableRoute 只有約 0.4×10⁷ B。
* **Fig. 15：換到短 1 跳的路（重傳 457 次）**：橫軸只放大 0.35～0.55 秒。約 0.42 秒 Dijkstra 換路後出現約 0.05 秒的鋸齒，之後恢復，藍線略落後紅線（SR_K）。影響只是一個短暫卡頓。
* **我的質疑：時間範圍可能是挑過的**。三張圖都只畫換路前後很短的一段時間，看不到長期結果。Fig. 14 在畫出的範圍內就已經看到 Dijkstra 後來反超；Fig. 15 只畫了 0.2 秒，換到更短的路長期來說延遲更低，Dijkstra 很可能在更長的時間後追上甚至超越 SR_K。這是推論，論文沒有給更長時間的數據可以驗證。

## System Model
* Each satellite has 4 ISLs: 2 intra-orbit and 2 inter-orbit. All ISL weights are 1, so path cost = hop count.
* **拓樸怎麼變（§II-A、Fig. 1）**：
    * 沒有固定的 70° 門檻；衛星**靠近軌道交會點**時，pointing／acquisition／tracking 跟不上，inter-orbit ISL 斷掉。Walker Star 的交會點在極區，Walker Delta 在中緯度。
    * 一個和 OPSPF 不同的細節：靠近交會點時 s1.b–s2.a 斷，**離開後接上的是 s1.a–s2.b**，也就是換了配對對象，不是原本那條接回來。
    * 實際哪條什麼時候斷／連，由 STK 產生的資料決定（經 SPP 解析）；論文沒給高度和門檻角度。
    * 一個軌道週期被切成 **m 個 snapshot**，每次 ISL 斷或連就是一個事件。SR_L／SR_K 只看當下這張，SR_G 一次看整個週期。
* A route update happens for one of three reasons:
    * **S1**: an ISL on the current path goes down. The current update cannot be avoided, but with a global view of the predictable orbits the number of such updates can be reduced (targeted by SR_G).
    * **S2**: Dijkstra's default tie-breaking picks a different equal-cost path (avoidable, targeted by SR_L).
    * **S3**: a shorter path appears (avoidable if a slightly longer path is acceptable, targeted by SR_K).

## Method
### Next-hop set (Alg. 1)
Dijkstra modified to keep **all** equal-cost next hops toward each destination v. When relaxing edge (u, v) with d[v] ≥ d[u] + w(u, v):
* if strictly greater, update d[v] and reset set[v] = ∅;
* then set[v] = {v} if u is the source, otherwise set[v] = set[v] ∪ set[u].

### SR_L (Local)
If the current next hop H[v] ∈ set[v], keep it; otherwise pick one at random from set[v]. Uses only local state, so it handles arbitrary (unpredicted) changes.

### SR_K (K-Short, Alg. 2)
If H[v] ∉ set[v], compute l = CountLength(G, H[v], w), the cost of the current path obtained by following H (infinite if the path is broken). Keep H[v] if l − k ≤ d[v]; otherwise pick at random from set[v]. SR_L is SR_K with k = 0.
* Proposition: if the current path satisfies the K-Short condition, the path SR_K chooses is exactly the current path. The proof goes by induction along the path, using $d_{v'_i v'_n} - w'_i = d_{v'_{i+1} v'_n}$ and $d^s_{v'_i v'_n} \le d^s_{v'_{i+1} v'_n} + w'_i$.

### SR_G (Global, Alg. 3)
* Compute the next-hop set in each of the m snapshots of one orbital cycle.
* Find_Sequence builds a layered graph with columns 1 … m+1 (column m+1 is snapshot 1 again). An edge costs 0 if the next hop stays the same and 1 if it changes.
* The shortest path from a vertex in column 1 to the same vertex in column m+1 gives the next-hop sequence with the fewest updates per cycle. Naive enumeration would cost 4^m.

### Complexity
Next-hop set O(n²) (set size ≤ 4); CountLength O(n) over n − 1 targets; SR_K overall O(n²).

## Evaluation Setup
* Pipeline: STK → Satellite Position Parser → Satellite Network Generator, emulated on Mininet with a real protocol stack (Xeon Gold 5220, Ubuntu 18.04).
* Constellations: mostly N×N Walker Star (4×4 to 40×40), plus one 20×20 Walker Delta. The paper names OneWeb (87.9°) and Starlink (53°) only as examples of each type and does not state the inclinations used in the emulation.
* Baseline: plain Dijkstra only. OPSPF and similar schemes are not compared, since they also use Dijkstra internally. SR_K uses k ≤ 2.

## Main Results
* **Causes of updates** (Walker Star): S1 28.24–52.93% (avg 42.69%); S2 20.27–50.85% (avg 31.86%); S3 with a 1-hop-shorter path 14.63–21.08% (avg 18.03%); S3 with a 2-hop-shorter path 4.12–11.65% (avg 7.41%). Paths more than 2 hops shorter are rare, so k ≥ 3 is unnecessary.
* **Update reduction vs Dijkstra**, all events:

| Constellation | SR_L | SR_K | SR_G |
|---------------|------|------|------|
| 4×4 Walker Star | 9.86% | 14.25% | 11.60% |
| 40×40 Walker Star | 43.40% | 44.97% | 45.86% |
| 20×20 Walker Delta | 34.24% | 40.30% | 46.93% |

* **ISL-down events only**: SR_L and SR_G clearly beat SR_K, and SR_G is slightly better than SR_L. The gain is not always positive: at link-up the schemes may skip a new shortest path and stay on the current one, but that current path may break soon afterwards, so the update saved at link-up shows up later as a link-down update (credit goes to link-up, penalty to link-down).
* **ISL-up events only**: SR_K reduces updates by well over 80%; SR_L / SR_G by 38.57–65.66%.
* SR_L ≈ SR_G in Walker Star, but SR_G pulls clearly ahead in Walker Delta.
* **TCP case studies** (single flow): Dijkstra's switches cause 1240 retransmissions when moving to a congested path, 2271 when moving to a higher-bandwidth path (reordering), and 457 when moving to a 1-hop-shorter path. SR_L / SR_G avoid the first two; SR_K avoids the third.

## 重現實驗（簡化版，2026-10）
自己寫 Python 模擬重做「換路次數」實驗（Fig. 8–11），程式和圖在 project files 的 `notes/stableroute/sim/`。
* **假設**（論文沒給的部分自己定）：N×N Walker Star、傾角 87.9°、高度 1200 km；任一端 |緯度| > 70° 就斷 inter-orbit ISL，出來後**同一條**接回；seam 兩側不連；權重全為 1；Dijkstra 平手時選編號最小的前驅。
* **結果**（相對 Dijkstra 減少的換路次數）：

| | SR_L | SR_K (k=2) | SR_G |
|---|---|---|---|
| 論文 4×4 | 9.86% | 14.25% | 11.60% |
| 重現 4×4 | 38% | 53% | 41% |
| 論文 40×40 | 43.40% | 44.97% | 45.86% |
| 重現 36×36 | 53% | 54% | 57% |
| 重現 40×40 | 33% | 34% | 38% |

* **和論文一致**：
    * 除了最小的星座，排名都是 SR_G > SR_K ≥ SR_L。
    * ISL-down 事件時 SR_L／SR_K 常比 Dijkstra 換得**更多**（負的 gain），SR_G 則大幅減少（約 −46～−75%）。
    * ISL-up 事件時 SR_K 幾乎不換（減少 85～100%）。
    * Scenario 比例 S1 約 42–50%、S2 約 22–37%，接近論文的 42.7%／31.9%。
* **和論文不同**：
    * 數字整體偏高，小星座差最多。
    * 曲線會跳：N = 10–13、20、22、28、40 的改善特別低，是衛星同時越過 70° 的時間點排列不同造成的。
    * 權重全為 1 時網格是二分圖，路徑長度差只會是偶數，所以 S3「短 1 hop」幾乎是 0、全落在「短 2 hop」；論文 S3 k=1 平均 18%。
    * 可能原因：論文用 STK 實際資料，而且離開交會點後**換配對對象**；我的模擬是同一條鏈路關掉再接回。

## Limitations
* Stated by the authors:
    * SR_L and SR_K are only locally optimal: avoiding an update now can cause more later.
    * SR_K trades path length for stability and does worse on link-down events.
    * SR_G needs predictable snapshots; for arbitrary changes it must be combined with SR_L.
    * StableRoute is currently built on Dijkstra, but the authors say the stateful next-hop idea can work with other algorithms that compute the next-hop set.
* My additional concerns:
    * There is no dedicated limitations section.
    * Only Dijkstra-based StableRoute is evaluated; the claim that the idea carries over to other algorithms is not tested.
    * The evaluation counts route updates only, with no end-to-end latency or throughput.
    * The TCP results are single-flow examples rather than workloads.
    * Fig. 14 cuts both ways: Dijkstra's switch to the higher-bandwidth path causes heavy reordering (2271 retransmissions) until about 2.7 s, but afterwards it climbs much faster, and by 3.5 s it has delivered roughly 3.4×10⁷ B versus about 0.4×10⁷ B for SR_L/G, which stays on the congested path. Avoiding the switch avoids reordering but gives up the better path; the paper reports only retransmissions and does not discuss this.
    * With all weights set to 1, the many hop-count ties make S2 look larger than it would be under latency weights.
    * **Perfect predictability is assumed, never tested.** The constellations are idealized Walker Star/Delta shells generated in STK (§V-A), not real TLE data, with no orbit-prediction error, maneuvers or failures. The paper does not say which orbit model STK used.
    * **Where SR_G's snapshots come from is not specified.** The paper never says who computes the cycle's snapshots in a real system (ground or onboard), from what data (TLE, GPS), or how often they are refreshed. Real Starlink satellites maneuver on their own (see the Starlink self-driving paper), which breaks the "repeats every cycle" assumption; SR_G would then fall back on SR_L.

## 與其他筆記的關聯
* 與 [[OPSPF_Orbit_Prediction_Shortest_Path_First_Routing_for_Resilient_LEO_Satellite_Networks]]（OPSPF）直接相關：同一個團隊（Tian Pan、Tao Huang，BUPT），StableRoute 引用 OPSPF 為 [5]，可以看成它的後續。
    * **OPSPF 解決「什麼時候重算」**：用軌道預測拓樸，每 1 秒在本地重跑 Dijkstra，突發故障再用 OSPF 補。但每次重算都是無狀態的 Dijkstra，不管路徑穩不穩定。
    * **StableRoute 解決「重算後選哪條」**：論文 §II-B 明講 OPSPF 這類 calculation-based 方法仍會因換路造成封包亂序，所以 StableRoute 改的是 Dijkstra 本身，不是重算的時機。
    * 兩者可以疊加（inference）：把 OPSPF static module 裡的 SPF 換成 SR_L/SR_K 即可。分工也很像：SR_G 對應 OPSPF 的「可預測變化用預測處理」，SR_L 對應「突發變化用本地資訊處理」。
    * 也因為 OPSPF 底層同樣是 Dijkstra，StableRoute 的實驗只拿 Dijkstra 當 baseline，沒有另外和 OPSPF 比。
    * **接點**：OPSPF Fig. 2 最後一格 *Shortest path first algorithm* 就是普通 Dijkstra，StableRoute 等於把這一格換掉。
    * **玩具例子**（四顆衛星 A–B／C–D 為 inter-plane，A 要送到 D，一開始走 A→C→D）：

| 時間 | 事件 | OPSPF（普通 Dijkstra） | 換成 StableRoute |
|---|---|---|---|
| 10 s | C–D 壞了 | 必須改走 A→B→D | 一樣必須換（S1） |
| 20 s | C–D 修好 | 兩條一樣短，可能換回 A→C→D | SR_L：現在這條仍是最短，不換（S2） |
| 30 s | A 進極區，A–B 斷 | 若 20 s 沒換回就得換 | SR_G 事先知道 A–B 會斷，20 s 時就選撐得比較久的路 |

* 同樣的變化，OPSPF 可能換路 2–3 次，StableRoute 盡量壓到 1 次。
* 兩者衡量的東西也不同：OPSPF 看**控制封包數和收斂時間**（地圖更新得省不省、快不快），StableRoute 看**換路次數和 TCP 重傳**（路線穩不穩）。
* 與 [[Stable_Hierarchical_Routing_for_Operational_LEO_Networks]]（SHORT）目標相同，都是要減少路由更新，但做法不同：SHORT 用座標不變量取代 Dijkstra；StableRoute 保留 Dijkstra，只改 tie-breaking 和「容忍稍長路徑」。
* [[Time-_Dependent_Network_Topology_Optimization_for_LEO_Satellite_Constellations]]（DoTD）在拓樸層減少換線，StableRoute 在路由層減少換路，兩者可以疊加。
* [[Minimum-hop Constellation Design for LEO Satellite Networks]] 的 slanted grid 會改變 equal-cost path 的數量，進而影響 S2 的比例。
