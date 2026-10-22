---
title: "Stable Hierarchical Routing for Operational LEO Networks"
authors: [Yuanjie Li, Lixin Liu, Hewu Li, Wei Liu, Yimei Chen, Jianping Wu, Qian Wu, Jun Liu, Zeqi Lai]
venue: ACM MobiCom 2024
doi: 10.1145/3636534.3649362
tags: [LEO, satellite-routing, ISL, hierarchical-routing, geographic-routing]
---

## 一句話總結
SHORT（Stable Hierarchical Orbital Routing Technique）不去追著衛星跑，而是以「地面上的地理位置不會動」為錨點來做階層式路由。
真實 LEO 網路有三層動態：衛星對地面（GSL 每 1.46–3.98 s 斷一次）、同一個 shell 內（ISL 因 maneuver、故障、部分部署而斷）、不同 shell 之間（相對運動非線性）。把整個網路當成一個平面路由域（flat routing）時，每一次斷線都要全網重新收斂／重算／重新探索；靠軌道預測（TS-SDN、Starlink 預排程）又會遇到 table 爆炸，而且 maneuver 和故障本來就預測不了。SHORT 把網路切成三層：Tier 1 地面網路（GS 當 shell 之間的 backbone）、Tier 2 orbital shell、Tier 3 orbit；再為每個 shell 定義 orbital-geodetic 座標 $(\alpha,\gamma)$，讓衛星的 sub-point 在這個座標裡**線性**移動、同 shell 兩顆衛星的座標差**不隨時間變**（Proposition 1）。地面節點的位址綁在地理 cell 上而不是綁在服務它的衛星上，所以 GSL 換手不必更新路由。實驗用 StarryNet 重播真實 TLE：相對 flat proactive routing 少 9–31,816× 的路由表更新，可用率約 100%，路徑只比最佳多約 3.48 ms／3 hops。代價是沒有流量／壅塞模型，而且只跑 1 小時。

## Introduction
* **Problem:** LEO mega-constellation 正從 1-hop bent-pipe 走向以 ISL 做全球路由，但拓樸變動在三個層次交錯（地面–衛星、shell 內、shell 間），再加上 maneuver 和隨機故障，比多數衛星路由論文假設的還難預測。Flat 路由（不論 proactive、reactive 或 predictive）在 mega-constellation 規模下會失去 availability、efficiency 或 resiliency。傳統階層式路由需要穩定的 routing domain 邊界，而既有的階層式衛星路由 [70, 71] 要靠 GEO/MEO backbone，營運中的 LEO 網路沒有。論文的問題是：**如何在高動態的營運 LEO 網路中建出穩定、高效、可靠的 routing hierarchy？**
* **Target:** 減少路由更新（routing／switch table entry updates、全域 control message），同時維持 availability（不因重收斂停擺）、efficiency（路徑長度接近最佳 flat routing）與 resiliency（對 maneuver、故障、部分部署）。沒有寫成最佳化問題（沒有 objective／constraint），是系統設計＋實驗驗證。
* 論文分兩部分：§3 用 SSN TLE 資料＋Starlink 終端實測量化三層動態；§4–5 提出 SHORT 並實作成 control-plane overlay；§6 評估。
* 作者：Tsinghua University、Zhongguancun Laboratory（通訊作者 Hewu Li）。首頁列 9 位作者；但頁首（running header）多了 W. Zhao，共 10 位。
* 核心想法：**地球上的地理位置不受衛星移動影響**。路由到地面節點時看它的地理位置，任何覆蓋它的衛星都可以送達，路由就和快速變動的 serving satellite 解耦。
* 和傳統 geographic routing 不同：SHORT 是多 shell 的 3D geographic routing。3D geographic routing 已被證明不可能永遠保證可達 [74]，SHORT 把它階層式拆成可解的 2D geographic routing 來繞過這個不可能性。

## Background knowledge
### Orbital shell 是什麼
**Orbital shell（軌道殼層）**＝一群**高度相同、傾角相同**的衛星，像一層包住地球的薄殼。一個 shell 由很多條平均分佈的軌道（orbital plane）組成，每條軌道上有等間距的衛星。例如 Starlink shell 2：72 條軌道 × 22 顆 = 1584 顆，全部在 540 km、53.2°。

![[orbital_shells.png]]

* **(a) 一個 shell**：很多條軌道，高度、傾角都一樣。同一個 shell 裡的衛星一起移動，相對運動小，所以 shell 內的 ISL 相對穩定。SHORT 因此把每個 shell 當成一個 routing domain。
* **(b) Starlink 疊了好幾個 shell**：傾角不同（53.2°、70°、97.6°）。真實高度只差 540–570 km，圖裡故意把間距拉大才看得出來。
* **(c) 為什麼要好幾個 shell**：衛星的 sub-point 最高只到約「傾角」那個緯度（傾角 > 90° 的逆行軌道是 180° − 傾角，97.6° → ±82.4°）。53° 的 shell 涵蓋台北、紐約這些人口多的地方；70° 和 97.6° 的 shell 負責高緯度，衛星也比較少。南極只有 shell 4 涵蓋。
* 不同傾角的 shell 彼此相對運動大，所以 shell 之間的 ISL 一直斷（§3.3）。這就是 SHORT 改用 GS 當 shell 之間橋樑的原因：一個 GS 同時看得到好幾個 shell 的衛星，Starlink 任何時刻都至少有 20 個共同 GS。

### 層級：constellation → shell → orbit → satellite
順序不是 shell → constellation，而是由大到小：

| 層級 | 是什麼 | 例子 |
|---|---|---|
| **Constellation（星座）** | 一家營運商的整個衛星系統 | Starlink、Iridium、Telesat |
| **Orbital shell** | 星座裡高度、傾角相同的一群衛星 | Starlink shell 2（540 km、53.2°） |
| **Orbit／orbital plane（軌道面）** | shell 裡的一條軌道 | shell 2 的 72 條之一 |
| **Satellite** | 軌道上的一顆衛星 | 每條軌道 22 顆 |

對應到 SHORT 的三層 hierarchy（Fig. 11）：**Tier 1 = 地面網路（GS、終端）**，**Tier 2 = orbital shell**，**Tier 3 = orbit**。Constellation 本身不是一個 tier，它就是整個網路；衛星是 Tier 3 裡的節點。所以 SHORT 是在「地面 → shell → orbit」這三層上分 routing domain。

### Walker Star 還是 Walker Delta？
論文**沒有**用 Walker Star／Delta 這兩個詞（全文搜尋為 0），下面是依 Fig. 2a 的傾角自己判斷的（inference）：
* **Walker Delta**（傾斜軌道，軌道面分散在整個 360° 經度，交會點在中緯度）：Starlink shell 1、2（53°、53.2°）、shell 3（70°），以及 Telesat shell 2（50.88°）。
* **Walker Star**（接近極軌道，軌道面只分散在 180°，所有軌道在極區交會，有 seam）：Iridium（86.4°，6 條軌道）是典型例子；Telesat shell 1（98.98°）也接近這種。
* **Starlink shell 4（97.6°）**：接近極軌道，但 2023 年只部署了 6 條軌道（partial deployment），還不是完整的 Walker 構型，論文也沒說軌道面怎麼分佈，無法確定。
* 對 SHORT 來說這個分類不重要：orbital-geodetic 座標 $(\alpha,\gamma)$ 對任何傾角 $i$ 都成立，每個 shell 用自己的座標系。
* 對照 [[StableRoute_When_Dijkstras_Algorithm_Meets_Topology-Varying_Satellite_Networks]]：StableRoute 主要用 Walker Star（N×N），只測一組 Walker Delta；SHORT 直接用真實星座，以 Walker Delta 型的 Starlink 為主。

### 真實 LEO 的三層動態（§3）
**資料與假設**：U.S. Space Surveillance Network（SSN）的 TLE（space-track.org，2019/5–2023/8，49,650,187 筆，31,880 個物體）；Starlink GS 分佈 [53]，每個 GS 連每個 shell 中最近的衛星；紐約一台 Starlink Gen2 碟（Ku-band、bent-pipe、無 ISL）；網格拓樸每顆 4–5 條 ISL（2 intra-orbit、2 inter-orbit、1 條選配 inter-shell）；採保守的換鏈策略（鄰居離開可視範圍才斷），所以算出的 churn 頻率是**下界**。

#### 1. Space-terrestrial（§3.1）
* Sub-point 隨時間是非線性、不重複的：
$$\sin\varphi_t=\sin i\,\sin\!\left(\tfrac{2\pi}{T_S}t-\gamma_0\right)\qquad(1)$$
$$\tan\!\left(\lambda_t+\tfrac{2\pi}{T_E}t-\alpha_0\right)=\cos i\,\tan\!\left(\tfrac{2\pi}{T_S}t-\gamma_0\right)\qquad(2)$$
$\varphi_t,\lambda_t$ 為緯度／經度，$i$ 傾角，$T_E\approx 24$ h，$T_S$ 為軌道週期，$\alpha_0$ 為 $t=0$ 時的 right ascension，$\gamma_0$ 為 $t=0$ 時軌道內的相位角。
* 全域 GSL churn 平均每 **1.46–3.98 s** 一次（Fig. 5a，所有星座）；Introduction 單講 Starlink 是 1.46–2.61 s。
* GSL 品質也在變：Ku/Ka band 受遮蔽、大氣、天氣、Doppler 影響；Starlink app 顯示「每 57 s 中斷一次」。
* 每個終端同時看得到 8–16 顆 Starlink 衛星（Fig. 4c），有備援，但這些備援隨時間變，很難直接利用。
* Handoff 不夠：handoff 處理的是「最後一跳」的使用者移動；GSL churn 發生在基礎設施內部（衛星↔GS），handoff 保住連線，但穩不住拓樸和路由。

#### 2. Intra-orbital-shell（§3.2）
* Starlink shell 2／3 的 ISL churn 約每 **208.7 s／970.0 s** 一次；部分部署的 shell 4 inter-orbit ISL 每 **11.4 s** 一次，總 ISL churn 是 shell 2 的 **18.3×**。
* **Maneuver**：避撞時抬升／降低約 1 km 高度，與鄰居距離累積拉開，可能斷 ISL 或改變 intra-orbit 鄰居關係。Shell 2 平均每天 **48** 次（最多 259 次）maneuver 造成的 ISL churn。Maneuver 難預測：SSN 觀測有誤差，且有些營運商不公開軌道資訊。
* **隨機故障**：到 2023/5，Starlink 約每 **13 顆就有 1 顆**故障（棄置、地磁暴、飛控、硬體等）。
* **部分部署**：截至 2023/8 shell 3、4 未完成，但已經在服務（例如南極只有 shell 4 覆蓋）。

#### 3. Inter-orbital-shell（§3.3）
* 不同 shell 的高度、傾角不同，相對運動非線性、累積。若 Starlink 啟用 inter-shell ISL，其 churn 會**與 GSL churn 相當甚至更頻繁**（Fig. 10，是推估）。
* Inter-shell ISL 目前在營運網路中**幾乎沒有**：Starlink 2016 FCC filing 規劃每顆 5 條 ISL（含 1 條 inter-shell），2018 filing 已移除。
* Inter-shell ISL 是 nice-to-have：只有當來源在對方 shell 覆蓋不到的高緯度區才必要，人口少、少見。沒有它，shell 之間仍可經 GS 間接連接；Starlink 的 shell 之間任何時刻都至少有 20 個共同 GS（Fig. 5b）。

### 既有路由為什麼不夠（§3.4）
* **Flat proactive**（distance-vector／link-state、label switching、SDN）：每次 churn 都觸發全網路由交換與重收斂，收斂前可能 loop／black hole。Loon 用全域 Time-to-Enact 同步，但被最慢節點拖到分鐘等級。
* **Flat reactive**（AODV、DSR）：on-demand flooding。但多數 GSL churn 發生在連外的 GS 上，幾乎一定牽動 active flow，flooding 會耗盡網路；DSR 的 cache 也很快過期。
* **Flat predictive**（TS-SDN、OSPF+、Starlink 預排程）：預測拓樸、切 snapshot、預裝路由表。兩個問題：(1) 三層動態交錯，計算量爆炸（Fig. 3b）；(2) maneuver、故障、部分部署預測不準。
* **Hierarchical**：需要穩定的 routing domain 邊界；既有方案需要 GEO/MEO backbone。
* Starlink 現在主要還是 bent-pipe，所以還沒遇到這些問題；但它正逐步啟用 ISL、減少對 GS 的依賴，到時就會遇到。

## System Model
* **三層 backbone**（Fig. 11）：
    * Tier 1：地面網路與終端；GS 經 GSL 橋接不同 shell，也是連外部網路的 gateway。
    * Tier 2：每個 orbital shell 是一個 routing domain。
    * Tier 3：shell 內的各條 orbit。
    * 另有 **opportunistic links**（inter-shell ISL）只當 inter-domain 的捷徑，不屬於 backbone。
* **拓樸**：+Grid，每顆衛星 2 條 intra-orbit ISL＋2 條 inter-orbit ISL；inter-shell ISL 預設關閉、可選。
* **位置資訊**：地面終端和衛星都有 GPS；營運商透過 TT&C 介面定期把衛星位置推給 GS 和其他衛星（避撞用），所以每顆衛星知道鄰居位置。
* **Proposition 1 的前提**：圓軌道、同 shell、無 maneuver。

## Method
### Orbital-geodetic coordinate（§4.1 Step 1）
* 一般座標（經緯度、H3、S2、Geohash）都不管衛星移動和地球自轉，從 Eq. (1)–(2) 可以看出 sub-point 在這些座標裡是非線性的。
* SHORT 為每個傾角 $i$ 的 shell 定義座標 $(\alpha,\gamma)$：$\alpha$ 是 right ascension；$\gamma$ 是沿著傾角 $i$ 的大圓，從經度 $\alpha$ 的赤道點量到該位置的角距離；原點是赤道與本初子午線的交點（Fig. 13）。在這個座標裡，衛星 sub-point **線性**移動：
$$\alpha_t=\alpha_0-\tfrac{2\pi}{T_E}t,\qquad \gamma_t=\gamma_0+\tfrac{2\pi}{T_S}t\qquad(3)$$
* 換回經緯度（footnote 4）：$\sin\varphi_t=\sin i\,\sin(\gamma_t-2\gamma_0)$，$\tan(\lambda_t-\alpha_t)=\cos i\,\tan(\gamma_t-2\gamma_0)$。代入 Eq. (3) 就回到 Eq. (1)–(2)。
* **Proposition 1**（Stable orbital-geodetic routing）：同 shell、無 maneuver 的任兩顆衛星 $S,D$，
$$\Delta\alpha^{S,D}_t\equiv\alpha^S_0-\alpha^D_0,\qquad \Delta\gamma^{S,D}_t\equiv\gamma^S_0-\gamma^D_0$$
不隨時間變。證明就是把 Eq. (3) 代入兩顆衛星相減。意思是：走一條 intra／inter-orbit ISL 在這個座標裡永遠是固定的（非歐氏）距離。實際的歐氏距離（也就是 ISL 延遲）仍會變，SHORT 只是把它遮掉，換取穩定（近似最佳）的路由。
* **對 maneuver 的韌性**：避撞 maneuver 高度變化很小（Starlink 約 2%），對座標差的影響可追蹤；鄰居的座標差在 maneuver 時是線性變化。

### Hierarchical geographic address（§4.1 Step 2，Fig. 14）
`network prefix | orbital shell ID | col(α) | row(γ) | terrestrial node ID`
* prefix：連外部網路用（選配）；shell ID：inter-shell 路由（§4.3）；cell（col, row）：intra-shell 路由（§4.2）；node ID：cell 內唯一識別。
* 位址和 serving satellite 解耦，只有節點移到別的 cell（cell 很大，少見）或換 serving shell（營運商控制，可避免）時才會變。
* 原型中嵌入 IPv6 位址。

### Intra-orbital-shell routing（§4.2）
**選項 A：ISL churn-resilient geographic routing**（Fig. 15，給部分部署或 maneuver 多的 shell）
* 每顆衛星記錄到所有鄰居的 $(\Delta\alpha,\Delta\gamma)$（無 maneuver 時不變）。
* **Greedy mode**：先走 inter-orbit ISL 減少到目的地的 $\Delta\alpha$（因為大多數 churn 發生在 orbit 之間，有機會就先用）；沒有的話走 intra-orbit ISL 減少 $\Delta\gamma$。
* **Detour mode**：若沒有任何鄰居更接近目的地（local minimum，來自部分 ISL 或間歇 GSL），就依**逆時針**順序遞迴地改送給鄰居，直到下一跳可達且更接近目的地，再回到 greedy。
* 當某顆衛星覆蓋目標 cell 且能送達使用者時結束；最壞會走遍所有覆蓋該終端的衛星。本質上是每顆衛星做逆時針 DFS，只要還有路就保證可達。

**選項 B：強化既有 flat routing**（給完整、穩定的 shell）
* **Terrestrial-satellite mapping**：來源選最接近目的地的目標衛星，
$$\text{dst\_satellite}=\arg\min_{d\in\text{shell}}\sqrt{(\Delta\alpha^{S,d}_t-\Delta\alpha^{S,D}_t)^2+(\Delta\gamma^{S,d}_t-\Delta\gamma^{S,D}_t)^2}$$
其中 $S$ 是來源衛星；由 Proposition 1，$\Delta\alpha^{S,d}_t,\Delta\gamma^{S,d}_t$ 是常數，計算很便宜。再沿用既有的 inter-satellite routing 送到它。
* **加速 failover**：衛星或鏈路故障時，不等 legacy routing 收斂，直接用同樣的程序改選一顆**仍可達、且覆蓋同一個地面目的地**的備援衛星；與 legacy 的復原平行進行，論文說不會產生 loop 或 black hole。
* 也可配合其他 policy（shortest path、load balancing、QoS）；例如沿實體較短的路徑走，可做到隱性的 geographic load balancing。

### Inter-orbital-shell routing（§4.3）
1. **Backbone：經 GS**。來源 $S$ 找一個同時被兩個 shell 覆蓋的 GS，把 GS 的地理位址封裝進 header，用 §4.2 送到 GS；GS 解封裝後在目的 shell 內用 §4.2 送到 $D$。最佳化時選
$$\text{ground\_station}=\arg\min_{gs\in\text{shell}_S\cap\text{shell}_D}\{C_S(S\to gs)+C_D(gs\to D)\}$$
$C_S, C_D$ 為兩個 shell 各自 intra-shell 路由的成本。GS 也可以執行 inter-shell policy（類似 BGP border gateway）。
2. **Opportunistic en-route shortcut**：途中每顆衛星檢查有沒有通往目的 shell 的 inter-shell ISL，有就不去 GS、直接轉過去。Loop-free，因為 backbone 不用 inter-shell ISL，目的 shell 也不會把封包送回來。
* **相容 bent-pipe shell**（例如 Starlink shell 1）：兩邊同時覆蓋一個 GS 就由 GS 中繼；否則經 GS 進入有 ISL 的 shell（如 shell 2）繞過去，再由另一個 GS 送到目的 bent-pipe 衛星。

### Putting everything together（§4.4，Fig. 12）
* **Case 1：目的 prefix 屬於本星座** → 依 §4.1–4.3 在衛星上分散式路由，不必繞到遠端 GS（今天的 LEO 網路總是經 GS，成為 throughput／latency／signaling 瓶頸）。
* **Case 2：目的在外部網路** → 送到最近的 GS（border gateway）。每顆衛星用 orbital-geodetic 座標記下所有 GS 的位置，執行時用 Eq. (3) 找最近的 GS，再用 §4.2 送過去。
* **Security**（論文有一小段）：位址 spoofing／hijacking 可用 distance-bounding 驗證位置；位置隱私靠 cell 夠大（數十到數千 km²）來模糊；DDoS 靠多路徑 geographic load balancing 緩解。只是定性討論，沒有實驗。
* 論文也說座標與近乎無狀態的 geographic routing 適用於 MEO/GEO，可以整合進同一個 hierarchy。

### Implementation（§5，Fig. 16）
* SHORT 是 **control-plane overlay**：改 control-plane 邏輯，沿用既有 data-plane 轉發；可疊在 OLSR、AODV、MPLS、SDN 等 L2/L3 方案上，用 OTA 軟體更新部署。
* 每個 shell 可以保留自己的 L2/L3 路由；中間衛星找不到目標衛星（因 churn 或故障）時才交給 SHORT 重選目標並改寫 header。
* **原型**：約 13K 行 C，客製化 Linux kernel＋Quagga v0.99.23。封包先查 L3 FIB，命中就照常轉發；沒命中就交給 SHORT control plane 解析座標、跑 Fig. 12 流程、產生 FIB，之後同一目的地的封包直接查 FIB。

## Evaluation Setup
* **平台**：StarryNet [91] emulation（同團隊的 NSDI'23 框架），以商用硬體模擬每顆衛星，重播真實 TLE 的位置與速度（含 maneuver）。
* **星座**（Fig. 2a）：

| 星座 | Shell | 衛星數 | 軌道數 | 高度 (km) | 傾角 | ISL |
|---|---|---|---|---|---|---|
| Starlink | 1 | 1584 | 72 | 550 | 53° | 無 |
| Starlink | 2 | 1584 | 72 | 540 | 53.2° | 有 |
| Starlink | 3 | 720 | 36 | 570 | 70° | 有 |
| Starlink | 4 | 348 | 6 | 560 | 97.6° | 有 |
| Iridium | 1 | 66 | 6 | 781 | 86.4° | 有 |
| Telesat | 1 | 78 | 6 | 1015 | 98.98° | 有 |
| Telesat | 2 | 220 | 20 | 1325 | 50.88° | 有 |

* **拓樸**：grid，每顆 4 條 1 Gbps ISL（2 intra、2 inter-orbit，連同 shell 最近鄰居）；inter-shell ISL 預設關閉，需要時在最近的衛星之間 opportunistic 開啟。
* **GS**：Starlink 真實 GS 分佈 [53]。
* **流量**：隨機 500 組地面 S-D pair，跑 1 小時。沒有流量負載或壅塞模型。
* **Baselines**：
    * Flat proactive：OLSR（distributed link-state）、TS-SDN（temporospatial SDN 預測式路由 [19, 20, 93]）。
    * Flat reactive：AODV。
    * Flat geographic：LBP [13]（同團隊、orbit-direction-aware，每個 shell 各自跑）。
* **指標**：(a) stability（路由更新、control message）；(b) availability（因重收斂／重探索／重設定造成的停擺）；(c) efficiency（延遲、hop 數、計算成本）；(d) resiliency（maneuver、故障、部分部署）。

## Main Results
### 整體效果（§6.1）
* **Stability**（Fig. 17）：Flat proactive 的 routing／switch table entry 更新比 SHORT 多 **31,816× / 9× / 323×**（Starlink／Iridium／Telesat），全域 control message 多 **31,908× / 12× / 339×**。AODV 仍多 **11× / 21× / 46×**。TS-SDN 的拓樸更新與重算多 **3,477×**（即使已合併 snapshot）。
* **Availability**（Fig. 18）：SHORT 約 **100%**。原生 link-state（OLSR）服務可用時間只有約 **8%**（重收斂太慢）；TS-SDN 靠預先設定提高可用率，代價是上面的計算量；AODV 也接近 100%，但 route re-discovery 讓 active flow 延遲 **71.91 / 103.15 / 118.76 ms**。
* **Efficiency**（Fig. 19）：Iridium、Telesat 路徑長度與最佳 flat routing 相同；Starlink 多 **3.48 ms（3 hops）**。§6.1 寫「at most」，Introduction 寫「on average」，論文前後不一致。SHORT 有 80% 的時間比 LBP 短（LBP 在缺 inter-orbit ISL 的 shell 3/4 繞比較多路）。
* **Resiliency**（Fig. 20）：隨機 ISL 故障與部分 ISL 下仍保證可達，路徑長度隨故障數平滑增加；4–5 條故障時略降（繞路剛好經過離地面節點更近的衛星）。圖上 x 軸到約 43 條失效時約 150 hops、約 500 ms（從圖讀出，論文沒寫數字，也沒寫是哪個星座）。

### Micro benchmark（§6.2）
| 項目 | 結果 |
|---|---|
| 座標穩定性（Fig. 21） | 真實軌道下鄰居的 $(\Delta\alpha,\Delta\gamma)$ 變動 ≤ 0.05°；經緯度差變動 4.02°／6.26°；H3 距離變動 6.90 個相鄰 hexagon |
| Cell index（Fig. 22） | 8-bit 足以讓 cell 小於每顆衛星的覆蓋；24-bit 可定址 42,000 顆的完整 Starlink |
| 目標衛星選擇（Fig. 23a） | 與實體最近的衛星最多差 387.6 km、GSL 延遲多 1.94 ms；比經緯度快 1.5×、比 H3 快 78.9×（每封包省 0.55 ms／83.5 ms） |
| GS 選擇（Fig. 23b） | 比最佳 GS 平均多 1.23 hops、3.33 ms |
| Opportunistic inter-shell ISL（Fig. 24） | 從只走 GS 到 ISL 全開：延遲 52.32 → 34.78 ms，hops 12.69 → 6.98 |
| Data plane（Fig. 25） | 1 Gbps ISL 接近飽和、與 link-state 相當；CPU < 1%；記憶體約 1.3 MB（1344 kB）；新目的地第一個封包 0.048–0.058 ms；FIB lookup 0.006–0.008 ms；table 約 2 MB |

* Fig. 25 表格把 0.006–0.008 ms 標為「FIB lookup time」，正文卻說這是「cell 轉成 $(\alpha,\gamma)$ 座標」的時間，兩處說法不一致。

## Limitations
* Stated by the authors:
    * Proposition 1 只在理想軌道（無 maneuver）成立；真實軌道靠持續飛控才讓座標差保持穩定（≤ 0.05°）。
    * Inter-shell 經 GS 一定會繞路，要靠 opportunistic ISL 彌補。
    * Security 只有定性分析：spoofing、hijacking、位置隱私、DDoS。
* My additional concerns:
    * Proposition 1 假設圓軌道；J2 攝動與 drag 漂移只以「實測 ≤ 0.05°」帶過，沒有長期分析。
    * 只跑 1 小時、500 組 S-D，沒有流量負載或壅塞模型；geographic routing 沿最短地理路徑走，熱點容易壅塞，而 load balancing 只是「有潛力」。
    * 海洋、極區可能沒有兩個 shell 共同覆蓋的 GS，繞路大；GS 同時是 inter-shell 的瓶頸和信任點。
    * Detour 最壞約 150 hops、500 ms（Fig. 20），對即時應用很難接受。
    * 「~100% availability」只算路由層面的停擺，沒有計入 GSL 遮蔽和天候（§3.1 自己量到每 57 s 一次中斷）。
    * 最大的倍數（31,816×）是跟 OLSR 這種極端 baseline 比；沒有和 Starlink 實際的預排程系統比較（也無法取得）。
    * Efficiency 的 3.48 ms／3 hops 是「最多」還是「平均」，論文前後寫法不同。
    * 選目標衛星用 $(\alpha,\gamma)$ 的歐氏距離，最多差 387.6 km；這是近似，不是真正最近的衛星。
    * Baseline LBP 和 StarryNet 都是同一個團隊的作品。

## 與其他筆記的關聯
* 與 [[StableRoute_When_Dijkstras_Algorithm_Meets_Topology-Varying_Satellite_Networks]]（StableRoute）目標相同，都是要減少路由更新，但做法不同：SHORT 用座標不變量和階層取代 flat 的最短路徑計算；StableRoute 保留 Dijkstra，只改 tie-breaking 和「容忍稍長路徑」。兩者的衡量也不同：SHORT 看路由表更新數、control message、availability；StableRoute 看換路次數和 TCP 重傳。
* 與 [[Time-_Dependent_Network_Topology_Optimization_for_LEO_Satellite_Constellations]]（DoTD）對照：DoTD 用歷史分數穩定**拓樸**，SHORT 用座標不變量穩定**路由**。
* 與 [[OPSPF_Orbit_Prediction_Shortest_Path_First_Routing_for_Resilient_LEO_Satellite_Networks]]（OPSPF）：OPSPF 屬於 SHORT 所說的 flat predictive routing（預測拓樸＋本地重跑 Dijkstra），正是 SHORT 認為在 maneuver／故障下不夠的那一類。
* 與 [[On_distributed_geographic-based_packet_routing_for_LEO_satellite_networks]]（Henderson，SHORT 引用為 [12]）：早期的 LEO geographic routing，針對 Iridium 式極軌道；SHORT 認為這類設計靠近 90° 傾角的簡單動態，難以推廣到傾角 < 90° 的 mega-constellation。
* 與 [[Starlink_SelfDriving_Walkthrough]]（同團隊 MobiCom'23 [23]）：SHORT 的 maneuver 偵測方法和「Starlink 會自主 maneuver」的觀察來自這篇。
* 與 ISL topology design 的關聯：
    * 支持 +Grid 作基線：inter-orbit ISL 對應 $\Delta\alpha$、intra-orbit ISL 對應 $\Delta\gamma$。
    * Inter-shell ISL 雖然 churn 頻繁，當 shortcut 可降延遲約 33%（52.32 → 34.78 ms）、hops 約 45%（12.69 → 6.98）。
    * 部分部署讓 churn 增加 18.3×，拓樸設計需要考慮過渡期拓樸的穩定性。
    * 可以把 routing stability 納入拓樸目標，偏好在 $(\alpha,\gamma)$ 中距離固定的鄰居。
