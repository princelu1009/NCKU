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

## 與其他筆記的關聯
* 與 [[OPSPF_Orbit_Prediction_Shortest_Path_First_Routing_for_Resilient_LEO_Satellite_Networks]]（OPSPF）直接相關：同一個團隊（Tian Pan、Tao Huang，BUPT），StableRoute 引用 OPSPF 為 [5]，可以看成它的後續。
    * **OPSPF 解決「什麼時候重算」**：用軌道預測拓樸，每 1 秒在本地重跑 Dijkstra，突發故障再用 OSPF 補。但每次重算都是無狀態的 Dijkstra，不管路徑穩不穩定。
    * **StableRoute 解決「重算後選哪條」**：論文 §II-B 明講 OPSPF 這類 calculation-based 方法仍會因換路造成封包亂序，所以 StableRoute 改的是 Dijkstra 本身，不是重算的時機。
    * 兩者可以疊加（inference）：把 OPSPF static module 裡的 SPF 換成 SR_L/SR_K 即可。分工也很像：SR_G 對應 OPSPF 的「可預測變化用預測處理」，SR_L 對應「突發變化用本地資訊處理」。
    * 也因為 OPSPF 底層同樣是 Dijkstra，StableRoute 的實驗只拿 Dijkstra 當 baseline，沒有另外和 OPSPF 比。
* 與 [[Stable_Hierarchical_Routing_for_Operational_LEO_Networks]]（SHORT）目標相同，都是要減少路由更新，但做法不同：SHORT 用座標不變量取代 Dijkstra；StableRoute 保留 Dijkstra，只改 tie-breaking 和「容忍稍長路徑」。
* [[Time-_Dependent_Network_Topology_Optimization_for_LEO_Satellite_Constellations]]（DoTD）在拓樸層減少換線，StableRoute 在路由層減少換路，兩者可以疊加。
* [[Minimum-hop Constellation Design for LEO Satellite Networks]] 的 slanted grid 會改變 equal-cost path 的數量，進而影響 S2 的比例。
