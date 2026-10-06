---
title: "Achieving Resilient and Performance-Guaranteed Routing in Space-Terrestrial Integrated Networks (STARCURE)"
authors: [Zeqi Lai, Hewu Li, Yikun Wang, Qian Wu, Yangtao Deng, Jun Liu, Yuanjie Li, Jianping Wu]
venue: IEEE INFOCOM 2023
DOI: 10.1109/INFOCOM53939.2023.10229104
tags: [LEO, STIN, satellite-routing, resilient-routing, failure-recovery, traffic-engineering]
---

## Citation
Z. Lai, H. Li, Y. Wang, Q. Wu, Y. Deng, J. Liu, Y. Li, J. Wu, "Achieving Resilient and Performance-Guaranteed Routing in Space-Terrestrial Integrated Networks," IEEE INFOCOM 2023.（清華 Institute for Network Sciences and Cyberspace、Zhongguancun Laboratory）

## 一句話摘要
衛星網路故障很多：有可預測的（GSL handover 造成的斷線，每幾十秒一次），也有突發的（碎片撞擊、太陽風暴）。Reactive 方法（OSPF）一直在收斂；proactive 方法要為「所有可能故障組合」預算備用路由，算不完也存不下。STARCURE 的關鍵想法是 **TSM：把「拓樸變化」轉換成「流量變化」**，讓拓樸在邏輯上固定不變，再用混合路由處理：可預測故障用 LP 事先算好最佳路由；突發故障先用 **location-guided protection routing（LGPR）** 在本地繞過，同時重新計算，算完再切回去。在 Starlink 1584 顆、Kuiper 的模擬中，可達性接近 100%，吞吐量最多提升 197%。

## 1. Problem & Motivation
- **STIN（Space-Terrestrial Integrated Network）**：LEO 衛星（以 ISL/GSL 組成太空骨幹）＋地面站／終端。
- **為什麼容易故障**：
  - LEO dynamics：GSL 頻繁斷開、重連，Starlink 的 space-ground link churn 平均只有幾十秒。
  - 太空環境：碎片碰撞（Kessler syndrome、2009 Iridium–Cosmos 相撞）、地磁暴（一次毀掉 40 顆 Starlink）。
  - 小衛星較脆弱：Starlink 到 2022/12 發射超過 3300 顆，其中 353 顆（11%）已衰退或離軌。
- **兩類故障**：
  - **Predictable failure**：由 LEO 運動造成，常見、頻繁、短暫。
  - **Unexpected failure**：碰撞、輻射等造成，少見但可能是永久性的。
- **既有方法的不足**：
  - **Reactive（OSPF、IS-IS）**：每次故障都要全網收斂；故障太頻繁時，可達性只剩約 55–60%。
  - **Proactive（FCP、R3、Keep-forwarding、Plinko、SlickPackets、OPSPF 等）**：要為所有故障情境預算備用路由。情境數為 $\sum_{m=1}^{M}\sum_{p=1}^{N}\binom{|L_m|}{p}$（M 個 snapshot × 最多 N 條突發斷線）。Starlink 的 $|L_m|>3000$，拓樸每幾十秒就變一次，CPU 和 memory 都扛不住。

## 2. Problem Formulation（P1：RSRP）
- 圖 $G=(V,L)$，$V=S\cup U$（P 顆衛星＋Q 個地面站）。雷射 ISL 是點對點；無線 GSL 則是一顆衛星同時連多個地面站，共享容量。
- 故障事件 $\pi^k_t: L_t \to L_{t+1}$ 讓拓樸隨時間變化，形成 $G^{all}=\{G_t\}$。Node failure 視為其所有鏈路失效。
- 流量需求 $f_{ab}=\{st, et, d\}$；二元變數 $\gamma_{ab}(i,j,t)$ 表示流量 $f_{ab}$ 在時槽 t 是否經過鏈路 (i,j)。
- 限制：(1) flow conservation；(2) 鏈路容量；(5) 延遲 $\le D_{ab}=\alpha D^{sp}_{ab}$（α ≥ 1 倍最短路徑延遲）；(6) 每條鏈路的負載 ≤ MLU。
- 目標：**min MLU**（maximum link utilization），留出餘裕吸收突發流量和故障。
- 每次故障都要在新的 $G_t$ 上快速解一次 ILP，這正是困難所在。

## 3. Method
### 3.1 TSM（Topology-Stabilizing Model）：把拓樸變化轉成流量變化
兩個 surjection：graph $\phi: G^T \to \hat G$，traffic $\omega: F \to \hat F$。

**穩定的邏輯拓樸 $\hat G$（Alg. 1）**
- 每顆衛星 $s_i$ 在 $\hat G$ 裡有一個鏡像，另外再配一個 **virtual terrestrial node $vs_i$**，代表「此刻連到 $s_i$ 的所有地面站」。所以 $\hat G$ 共有 2P 個節點。
- 邏輯鏈路 $(s_i, vs_i)$ 的容量等於 $s_i$ 的 GSL 容量，而且**永遠存在**（沒有地面站連上時，就只是沒有流量）。
- ISL 照實體連線方式複製（同軌道前後 2 條、相鄰軌道左右 2 條，+Grid）。
- 為什麼穩定：Walker Delta 同一個 shell 內的衛星高度、速度都相同，相對位置固定，所以 ISL 拓樸不變；GSL 的變化則被 $vs_i$ 吸收掉了。

**把故障轉成流量（Alg. 2）**
- **Predictable（GSL handover）**：一條長時間的流量 $f_{ab}$，切成多段**依時間接續的短流量**，每段的端點是「當時服務 a、b 的那兩顆衛星的 $vs$」。
- **Unexpected（鏈路故障）**：在故障鏈路 (i,j) 上加一條 **burst flow** $f^{burst}=\{t_s, t_e, b_{ij}\}$，流量等於整條鏈路的容量，延遲要求設為該鏈路的延遲，強迫它走這條鏈路。結果是這條鏈路被「塞滿」，其他流量自然會避開。

**白話範例（論文 Fig. 3）**：3 顆衛星 S1、S2、S3，2 個地面站 GS1、GS2，流量從 GS1 到 GS2，時間 [t1, t3)。
- 時槽 1：GS1–S1、GS2–S2；時槽 2 發生 handover，變成 GS1–S2、GS2–S3；同時 S1–S2 的 ISL 突然故障。
- TSM 的處理：流量切成 $f_{v1,v2}=\{t1,t2,D\}$ 和 $f_{v2,v3}=\{t2,t3,D\}$，另外在 S1–S2 加上 $\{t2,t3,\text{Cap}(S1,S2)\}$ 的 burst flow。
- 拓樸 $\hat G$ 從頭到尾都沒變，只有流量矩陣在變。

**轉換後的問題（P2）**：在固定的 $\hat G$ 上，給定隨時間變化的 $\hat F$，求滿足 (1)(2)(5)(6) 的動態路由。這變成一個 dynamic routing scheduling 問題，不必再對無數種拓樸預算。

剩下兩個問題：$\hat F$ 變得非常大，直接用 LP 求解很慢；而且突發故障造成的 burst flow 無法事先知道。

### 3.2 Adaptive Hybrid Routing
將 $\hat F = \hat F_{pr} + \hat F_{up}$ 分成可預測和突發兩部分。

**Basic routing（處理可預測故障，離線計算、上線時配置）**：用 Gurobi 當 constraint optimizer，加上兩個加速方法：
- **Path filtering**：估計流量 $f_{ab}$ 若經過鏈路 (i,j) 的路徑長度 $p^{ij}_{ab}$ = great-circle(a,i) + |(i,j)| + great-circle(j,b)。若 $p^{ij}_{ab} \ge D_{ab}$，就不考慮這條鏈路，藉此刪掉大量 $\gamma$ 變數（離 a→b 地面投影太遠的衛星不會被用到）。
- **Merging demands**：同一時槽、同一組 src–dst 的需求合併成一筆，減少限制式數量。

**Protection routing：LGPR（Alg. 3，處理突發故障）**
- 每顆衛星用 (orbit index p, intra-orbit index q) 標示位置；同一 shell 內的相對位置固定。
- 收到封包時，在可用的鄰居中選**離目的地最近**的那個轉送（本地決策，不需要收斂）。
- **Loop avoidance**：如果某顆衛星只剩一條可用鏈路（dead end），就通知鄰居暫時把它視為不可用，避免封包在死路上來回。
- 用 IPv6 Hop-by-Hop header 裡的 **1-bit protection flag** 標示封包要走 basic 還是 protection routing。

**運作流程**：平時走 basic routing → 發生突發故障 → 受影響的節點立刻切到 LGPR 在本地繞過，**同時**通知 basic routing 把 burst flow 加進 $\hat F$ 重新計算 → 新的路由算好後切回 basic routing。

## 4. Implementation & Evaluation Setup
- **Prototype**：Linux；basic routing 用 Gurobi 計算，再透過 Linux `route` 寫入 data plane；LGPR 用 Geopy 算距離；protection flag 放在 IPv6 HBH header。
- **Hardware-in-the-loop testbed**：用 Mininet 容器模擬每顆衛星和地面站（3 台 Dell R730，Ubuntu 20.04），軌道資料取自 CelesTrak＋STK；另外用一台曾經上過太空的 **Raspberry Pi 4** 量測實際 CPU 成本。
- **星座**：Starlink 第一 shell（1584 顆、72 個 orbit、550 km）＋SpaceX 地面站；Kuiper＋AWS 地面站。鏈路容量參考 FCC filings 和先前研究。
- **流量**：依城市人口乘積產生（ICARUS 的方法），涵蓋地面站服務範圍內的大城市。
- **Baselines**：OSPF、FCP（failure-carrying packets）、SCRR（source-controlled，結合 Handley 的方法與 SlickPackets）、OrbitCast（純地理位置路由、無收斂）。
- **故障情境**：只有 PF（可預測故障），或加上 10%／15%／30% 鏈路的 UF（突發故障）。

## 5. Main Results
**Routing restoration time（Table I）**

| 情境 | OSPF | FCP | SCRR | OrbitCast | STARCURE |
|---|---|---|---|---|---|
| Single PF | 7.4 s | 1.4 s | 1.8 s | 1.3 s | **0.6 s** |
| 10% UF | 35.2 s | 17.4 s | 6.4 s | 1.2 s | **1.1 s** |
| 15% UF | 79.1 s | 21.2 s | 11.7 s | 1.2 s | 1.2 s |
| 30% UF | 153.8 s | 29.4 s | 16.2 s | **1.2 s** | 1.3 s |

- **Reachability**：OSPF 平均約 55%；FCP、SCRR 約 80–90%（故障資訊放在封包 header 裡，不用全網收斂，但仍要一直線上重算）；OrbitCast 在故障少時可達 100%，故障率高時因 dead end 產生 loop 而下降；**STARCURE 接近 100%**（數字只畫在圖上）。
- **Latency**：故障越多，所有方法的延遲都越高。FCP 會繞路（只依據不完整的拓樸圖反覆重算）；STARCURE 與 OrbitCast 接近最佳延遲。
- **Throughput**：把流量放大到 MLU = 100% 時的總吞吐量，其他四種方法約 100–200 Gbps（都走最短路徑，容易擠在同一條路上）；STARCURE **最多提升 197%**，因為它以 min MLU 為目標分散流量。
- **CPU（Raspberry Pi 4）**：OrbitCast 最高（每個封包都要算一次）；STARCURE 的負擔「可接受」（只有圖）。SCRR 的計算集中在 source controller，所以沒有畫。

## 6. 這篇算是全新的方法嗎？
- **新的部分**：
  - **TSM 是核心貢獻**：把「拓樸不確定性」轉成「流量不確定性」。這樣就能借用成熟的 traffic engineering（min-MLU LP），不必為每一種故障組合各算一份路由表。
  - 以 min MLU 加延遲上限作為目標，同時顧及可用性和效能（resilient＋performance-guaranteed），而不是只求「到得了」。
- **沿用的部分**：
  - Hybrid（basic＋protection）的架構概念和 OPSPF 類似：可預測的事先算好，突發的在本地處理。
  - LGPR 其實就是 geographic greedy forwarding，和同團隊的 OrbitCast 很接近，只是多了 dead-end 通知。
  - Path filtering 和 demand merging 是常見的 LP 加速技巧。

## 7. Limitations / 質疑點
**我的質疑（inference）**
- **「拓樸穩定」的前提**：TSM 假設 ISL 是固定的 +Grid（同 shell、相對位置不變），所以只有 GSL 會變動。像 OPSPF 那樣的極區 ISL 斷線、跨 shell ISL、seam，或 DoTD／LAMP 這類動態 ISL 拓樸，都會破壞這個前提，論文沒有討論。
- **Burst flow 建模**：用「塞滿容量的假流量」代表故障，在 min-MLU 下確實能讓其他流量避開，但這條鏈路的 MLU 永遠是 100%，目標函數要排除這些鏈路才合理，論文沒有說明如何處理。Node failure 則需要對所有相連鏈路各加一條 burst flow。
- **需要流量矩陣**：basic routing 依賴預估的 $F$；實際流量偏離預估時表現如何，沒有評估。
- **重算時間**：突發故障發生後，Gurobi 重新計算要多久、在大規模故障下是否來得及，論文沒有給數字。在算完之前，全靠 LGPR。
- **LGPR 的 loop avoidance 只處理「只剩一條鏈路」的 dead end**。更大範圍的故障區（例如太陽風暴一次毀掉多顆相鄰衛星）仍可能形成 loop。之後 SHORT 的 detour 機制才更系統地處理這個問題。
- 「close-to-100% reachability」、CPU 使用量和延遲都只畫在圖上，沒有具體數字；評估只有單一 shell。

## 8. 與其他筆記的關聯
- [[OPSPF_Orbit_Prediction_Shortest_Path_First_Routing_for_Resilient_LEO_Satellite_Networks]]：本文引用 OPSPF（ICC 2019，ref [14]），批評它「只適用 polar orbit」。兩者都是「可預測的事先處理＋突發的另外處理」；差別在於 OPSPF 對突發故障用 OSPF flooding 再全網重算，STARCURE 則先本地繞過（LGPR），再以 min MLU 重新規劃。
- [[Stable_Hierarchical_Routing_for_Operational_LEO_Networks]]：同一個清華團隊（Lai、Hewu Li、Yuanjie Li）。SHORT（2024）的 orbital-geodetic 座標和 detour 可以看作 LGPR 的升級版，而且 SHORT 也一樣用「地面站當 backbone」。
- [[Your Mega-Constellations Can Be Slim]]：同團隊。該文以 edge-disjoint paths 定義生存性，本文以 reachability 和故障後的效能來衡量；兩者可以互補（衛星要幾顆 vs 故障時路由怎麼救）。
- [[StableRoute_When_Dijkstras_Algorithm_Meets_Topology-Varying_Satellite_Networks]]：StableRoute 處理「沒必要的路由更新」，STARCURE 處理「故障後的恢復」，都在追求路由穩定。
- [[Time-_Dependent_Network_Topology_Optimization_for_LEO_Satellite_Constellations]]、[[LAMP_Low-Latency_Dynamic_Topology_for_LEO_Satellite_Constellations]]：如果 ISL 拓樸改為動態設計，TSM「ISL 固定」的前提就不成立，需要延伸。
