---
title: "Enabling Low-latency-capable Satellite-Ground Topology for Emerging LEO Satellite Networks"
authors: [Yaoying Zhang, Qian Wu, Zeqi Lai, Hewu Li]
venue: "IEEE INFOCOM 2022, pp. 1329–1338"
doi: 10.1109/INFOCOM48880.2022.9796886
survey_ref: "LEO survey 2310.07646 [179]（網路層：地面接入 + 路由穩定）"
tags: [LEO, ground-station, GSL, access-satellite, satellite-ground-topology, handover, LSGI, CSGI, Starlink, Kuiper, +Grid, OSPF]
---

## Citation
Y. Zhang, Q. Wu, Z. Lai, H. Li, "Enabling Low-latency-capable Satellite-Ground Topology for Emerging LEO Satellite Networks," *Proc. IEEE INFOCOM 2022*, pp. 1329–1338. DOI: 10.1109/INFOCOM48880.2022.9796886（清華大學 Institute for Network Sciences and Cyberspace；通訊作者 Zeqi Lai，也是 StarPerf 模擬器的作者）

## 一句話總結
過去的拓樸論文（例如 motif）都在設計**衛星和衛星之間**怎麼連（ISL），地面站（GS）要接哪顆衛星則用老方法決定：接**剩餘服務時間最長**的（LRST）、接**最近的直到它消失**（NSD）、或**永遠接最近的**（NSH）。作者指出，在 Starlink、Kuiper 這種大星座裡，一個 GS 同時看得到二、三十顆衛星，而且衛星會參與多跳路由，所以「接哪顆」會影響**整條端到端路徑**：同一對 GS 的單向延遲會在 **40 ms 到 120 ms** 之間跳動，換接入衛星時路由也跟著震盪。他們把問題寫成 **LSGI**（在「拓樸變動間隔 ≥ θ_T」的限制下，最小化 GS 對之間「整段時間內的最大跳數」），再提出 **CSGI**：(1) 起始時刻，把每個 GS 看得到的衛星依**往南飛／往北飛**分成兩群，每群用中心點代表，從兩群中挑一群讓所有 GS 對的總跳數最小；(2) 之後**不換**，直到某顆接入衛星飛出範圍；要換時，**所有 GS 一起往同一個 (Δx, Δy) 方向換**，讓 GS 對之間的跳數保持不變。在 Starlink 第一殼層和 Kuiper 第一殼層、兩組真實 GS 資料上，CSGI 比三種舊方法少約 **35% 的平均最大跳數、19% 的延遲、70% 的 jitter**，封包送達率也最高（OSPF Hello = 0.2 s 時 Starlink 92.91%、Kuiper 95.97%）。

## 0. 背景知識
### 端到端路徑的三段
GS-to-GS 路徑 = **上行 GSL** + **星座內 ISL 路徑** + **下行 GSL**。舊的接入方法只看第一段或最後一段（一跳），不管中間那段。本文的 ISTN（Integrated Satellite and Terrestrial Network）情境是：資料中心或使用者的流量先到 GS，經過 LEO 星座繞過大半個地球，再從另一個 GS 下來（例如 AWS Ground Station、SpaceX 和 Google 資料中心的合作）。

### 三種舊的接入方法（§III-A）
| 縮寫 | 規則 | 優點 | 缺點 |
|---|---|---|---|
| **LRST** | 接「剩餘可見時間最長」的衛星，直到它離開 | 單一 GS 換手最少 | 只顧自己，不管對端 |
| **NSD** | 接最近的，直到它離開才換 | 起始時訊號最好 | 換手比 LRST 多 |
| **NSH** | 永遠接最近的，一有更近的就馬上換 | 一跳的距離最短 | 換手最頻繁 |

這三種都來自早期 Iridium／Globalstar 時代的換手文獻，當時一個 GS 只看得到一兩顆衛星，而且衛星多半是 bent-pipe（只轉發、不參與路由），所以「接哪顆」只影響一跳。

### 為什麼大星座情況不同（§III-B）
1. **GSL 一直在變**：衛星高速移動，接入關係幾秒到幾分鐘就要換。
2. **看得到很多顆**：中低緯度的 GS 能同時看到幾十顆，設計空間變大。
3. **衛星參與多跳路由**：換接入衛星會改變整條路徑，不只是一跳。

## 1. Motivation：量化舊方法的問題（§III-C）
* **設定**：StarPerf 模擬 Starlink 第一殼層（1,584 顆）和 Kuiper 第一殼層（1,156 顆），ISL 用 **+Grid**，路由用**最少跳數**最短路；GS 取自 AWS Ground Station；模擬一個軌道週期約 6,000 s。
* **觀察一：GSL 震盪很大**（Fig. 2）。單一 GS（Ireland）的換手間隔從幾秒到約 300 s；看**兩個** GS（Oregon + Sydney）時，任一端換手都算一次，所以間隔更短。LRST 間隔最長。Kuiper 比 Starlink 換得少，因為 Kuiper 高度較高、最小仰角較低，單顆衛星覆蓋範圍較大。
* **觀察二：換手太頻繁會讓封包送不到**（Fig. 3）。在 OSPF 上量送達率：換手間隔約 25 s 時送達率**不到 54%**；Hello interval 越短（0.2 s vs 5 s），收斂越快，送達率越高。
* **觀察三：延遲大幅跳動**（Fig. 4、5）。三種方法下，GS 對的單向延遲都在約 40–120 ms 之間跳。Fig. 5 的例子：Sydney 換一次接入衛星，Oregon–Sydney 的最短路從 **49.92 ms 變成 113.51 ms**（同一段內文寫 49.93，差 0.01 是原文不一致）。
* **Takeaways**：(i) LRST 只從**單一** GS 的角度延長換手間隔；(ii) 兩端的接入衛星**一起**決定延遲和 jitter；(iii) 現有方法都沒有同時考慮路由穩定和低延遲。

## 2. Problem：LSGI（§IV）
### 模型
* 時間切成 1 s 一格，T = {1, …, T}；每一格是一張無向圖 G^t = (V, E^t)，V = 衛星 V_S ∪ 地面站 V_G。
* 星座 N 個軌道面 × 每面 M 顆，衛星 s_i 的座標是 (i_p, i_q)，i = i_p·M + i_q。
* **可見矩陣** L：ℓ_ij^t = 1 表示 s_i 和 g_j 在 t 時可建 GSL（只看最小仰角；作者假設 GS 天線夠好，不管仰角對訊號的影響）。
* **決策矩陣** L̃：ℓ̃_ij^t = 1 表示真的建了 GSL。ISL 固定（+Grid），只有 GSL 隨時間變。
* **跳數矩陣** H^t = Dijkstra(L̃^t)，h_jj'^t = g_j 到 g_j' 的最少跳數。
* **需求矩陣** D：d_jj' = 1 表示這對 GS 需要經過衛星網路通訊（不是每對都需要，短距離走地面光纖就好）。
* **拓樸變動**（Def. 5）：g_j 或 g_j' 任一端的 GSL 有接上或斷開，就算這對 GS 的拓樸變了一次。Algorithm 1（ATCI）算出這對的**平均拓樸變動間隔** ΔT_jj'。

### 最佳化問題
* **目標 (1)**：min Σ d_jj' · max_t h_jj'^t ／ Σ d_jj'。也就是所有有需求的 GS 對，取**整段時間內最差的跳數**再平均。用跳數近似延遲。
* **限制**：(2) 跳數由 Dijkstra 算；(3) 只能接看得到的衛星；(4) **每個 GS 剛好接 1 顆**（Σ_i ℓ̃_ij^t = 1）；(5) 每對有需求的 GS，ΔT_jj' ≥ θ_T。
* **為什麼用「最大跳數」**：同時壓低平均延遲和 jitter（最大值低，跳動就小）。
* **複雜度**：每個 GS 有 ψ 個選擇（ψ 可能 > 20），一個時刻就有 ψ^{N_G} 種拓樸；整段時間不能拆成各時刻獨立解，所以是 ψ^{N_G·T}。

## 3. Method：CSGI（§V）
### 3.1 拆解
* **拆 GS**：把沒有通訊需求的 GS 分到不同的子集合（用 Disjoint Set Union），各子集合獨立解（式 7–10）。
* **拆時間**：先解起始時刻 t₀ 的最佳拓樸（Algorithm 2），之後用「轉移」演算法（Algorithm 3）維持。理由（式 11）：整段時間的最佳值 opt* ≥ 單一時刻的最佳值；如果之後每個時刻都能讓 h^t − h^{t₀} < ξ，則 CSGI 的結果 opt⁺ − opt* < ξ。
* D 改變時，重新跑 Algorithm 2。

### 3.2 單一時刻：TCTP（Algorithm 2）
兩個觀察：
* **Prop. 1**：一對 GS 的拓樸變動間隔不會超過兩端接入衛星的剩餘服務時間。所以先把剩餘時間 < θ 的衛星排除（θ 由 θ_T 推出）。
* **Prop. 2**：同軌相鄰或相鄰軌道相鄰的衛星，到任何衛星的跳數只差 1。所以看得到的衛星可以**群組化**。

步驟：
1. 把 GS 看得到的衛星畫在 N × M 的格子上（Fig. 6 左）。以 Kuiper 的 Sydney 為例，可見衛星明顯分成**兩群**：一群往南飛（descending）、一群往北飛（ascending）。它們在地圖上靠在一起，在 +Grid 格子上卻離很遠。
2. 每群取**中心點** μ⁰_j、μ¹_j（中心點不一定是真的衛星），每個 GS 的候選從幾十顆縮成 **2 個**。
3. 對所有 GS 選 σ_j ∈ {0, 1}，最小化 Tr(D · Λᵀ)，其中 λ_jj' = DMC(μ_j^{σ_j}, μ_j'^{σ_j'})，DMC 是格子上的距離（約等於跳數）。作者說這可以化成 **Facility Location Problem**（NP-hard）；GS 只有幾十個時可以直接窮舉（2^{N_G}），更多時用現成的近似演算法。
4. 每個 GS 在選定的那一群裡，挑「到其他有需求 GS 的中心點加權距離最小」的那顆衛星。

### 3.3 時間上的轉移：AT（Algorithm 3）
1. **能不換就不換**：只要每個 GS 原本接的衛星還看得到，拓樸維持不變（h^t − h^{t₀} = 0）。
2. 有任一顆離開範圍時，**所有 GS 一起換**（這是「Coordinated」的意思）：
    * **Prop. 3**：如果 g_j 的接入衛星從 s(x₁, y₁) 換到 s(x₁+Δx, y₁+Δy)，g_j' 也從 s(x₂, y₂) 換到 s(x₂+Δx, y₂+Δy)，兩者之間的跳數不變。
    * 每個 GS 列出所有合格候選相對目前接入衛星的位移集合 ΔXY_j；取**在所有 GS 中出現最多次**的 (Δx, Δy)；有這個位移的 GS 就照它換，沒有的就挑曼哈頓距離最接近的位移。
* 直覺：衛星都往同一個方向移動，GS 們就一起往同一個方向「平移」接入點，整張圖上 GS 之間的相對位置不變，所以路徑長度不變，jitter 也小。

## 4. Evaluation（§VI）
### 設定
* 星座：Starlink 第一殼層、Kuiper 第一殼層；ISL 用 +Grid；最少跳數路由；StarPerf 模擬 6,000 s。
* **Dataset A**：Amazon 目前的 8 個 GS（去掉覆蓋不好的 Stockholm），全部兩兩有需求，共 **28 對**，多半跨洲跨洋。
* **Dataset B**：從 Starlink 已部署／規劃的 GS 中選 **31 個**（兩兩距離 > 500 km），隨機挑 **126 對**距離 > 3,000 km 的 GS 對（引用 Handley 2018：超過 3,000 km 時 LEO 可能比光纖快）。
* θ_T：Starlink **150**、Kuiper **250**（原文寫「ms」，但換手間隔是以秒計，應該是 **s** 的筆誤；推論）。
* 送達率：在 **mininet** 上跑 OSPF，Hello interval 設 **0.2 s 和 5 s**（0.2 s 實際上太頻繁，作者用它模擬 OPSPF 這類預測式 OSPF）。
* 四種演算法都用 Python 實作，數值模擬。

### 結果
* **拓樸變動間隔**（Fig. 7）：LRST 和 CSGI 都比 NSD、NSH 長；CSGI 在約 60% 的情況比 LRST 長。平均拓樸變動**次數**比 LRST 少 **15%（Starlink）／33%（Kuiper）**。
* **送達率**（Fig. 8）：Hello = 5 s 時 NSD、NSH 都 < 40%；Hello = 0.2 s 時仍 < 90%。CSGI 在所有測試中都**高於 LRST**；Hello = 0.2 s 時 Kuiper **95.97%**、Starlink **92.91%**。Kuiper 普遍比 Starlink 高（覆蓋較大）。
* **跳數**（Table I）：

| 指標 | 星座 | NSH | NSD | LRST | CSGI |
|---|---|---|---|---|---|
| 平均最大跳數（A） | Starlink | 38.57 | 38.86 | 37.75 | **22.86** |
| | Kuiper | 25.79 | 25.21 | 26.04 | **16.43** |
| 平均跳數（A） | Starlink | 26.47 | 26.74 | 25.11 | **20.67** |
| | Kuiper | 19.24 | 19.26 | 18.74 | **15.52** |
| 平均最大跳數（B） | Starlink | 39.21 | 39.41 | 34.49 | **22.44** |
| | Kuiper | 26.11 | 25.86 | 26.74 | **17.45** |
| 平均跳數（B） | Starlink | 26.41 | 26.28 | 27.06 | **19.58** |
| | Kuiper | 20.15 | 20.05 | 20.13 | **16.25** |

* 平均最大跳數比其他三種少約 **35%**；單向延遲平均少約 **19%**；jitter（同一對的最大延遲 − 最小延遲）只有其他方法的約 **30%**（即少約 70%）。
* 我用 Table I 驗算：Starlink-A 的平均最大跳數 22.86 vs LRST 37.75，少 39%；Kuiper-B 17.45 vs LRST 26.74，少 35%。「約 35%」大致對得上。
* CSGI 的「最大跳數 − 平均跳數」只有約 1–2 跳（Starlink-A：22.86 − 20.67 = 2.19），其他方法約 12 跳（LRST：37.75 − 25.11 = 12.64）。這就是 jitter 變小的原因。
* **Fig. 9 例子**：Bahrain–Ohio 用 LRST 時延遲在 < 50 ms 到 125 ms 之間跳，用 CSGI 穩定在約 50 ms，平均延遲約是 LRST 的一半。

## 5. Novelty／貢獻
1. **第一篇**專門研究大星座**衛星–地面拓樸**（GS 接哪顆衛星）對整個網路效能影響的論文（作者自稱）。過去換手文獻只看一跳的指標（換手次數、阻斷率）。
2. 把「接入選擇」從**各 GS 自己決定**改成**所有 GS 協調**：起始時一起選、之後一起平移。
3. 用 +Grid 格子的平移不變性（Prop. 3），把「整段時間都要好」的難題化成「起始時刻選好 + 之後保持相對位置」。
4. 往南／往北兩群的觀察，把每個 GS 的候選從幾十顆降到 2 個。

## 6. Limitations
1. **每個 GS 只接 1 顆衛星**（限制 4）。真實 GS 有多個天線（Starlink gateway 有多個碟）。多天線時可以 make-before-break，換手不必斷線（推論）。
2. **延遲用跳數代替**：目標和 Table I 都是跳數。Prop. 3 保證的是**跳數**不變，不是公里數不變。同樣跳數下，路徑經過高緯度或低緯度，實際距離會不同（StepClimb 筆記裡的「同 hop 數不同公里數」）。Fig. 9 的 ms 結果有一部分補上了這點，但沒有用公里數當目標。
3. **Prop. 3 依賴格子座標的平移不變性**。我寫了一個小程式在 +Grid 上 BFS 驗證（72×22、34×34，隨機 300 對、平移 |Δ| ≤ 2）：當跨軌連線在最後一個軌道面接回第一個時**沒有相位偏移**（F = 0），跳數**完全不變**；如果 Walker Delta 的相位因子 F ≠ 0（例如 72×22、F = 11），用論文那種「p、q 各自取模」的平移，約 3–6% 的 GS 對跳數會改變，因為跨過 p = N−1 → 0 那條縫時 q 也要跟著偏 F。論文沒講 Starlink／Kuiper 用的 F 值。這是演算法實作細節，修正平移公式就可以解決（推論）。
4. **衛星群組化只在 +Grid 成立**：往南／往北兩群在 +Grid 格子上距離遠，是 +Grid 的特性。如果 ISL 換成 motif、DoTD 或 LAMP 那種有斜線或臨時連線的拓樸，Prop. 2、Prop. 3 都要重新檢查（推論）。
5. **只考慮傳播延遲**，不考慮流量、壅塞、GS 容量。所有 GS 都往同一顆或相鄰幾顆衛星擠時，可能造成負載集中（推論）。
6. **需求矩陣 D 是 0/1**，沒有流量權重。Dataset A 是「全部兩兩都有需求」，GS 數量少（8 個），2^{N_G} 窮舉很容易。GS 變多（真實 Starlink 有一百多個 gateway）時，Facility Location 的近似解品質沒有測。
7. **θ_T 的單位筆誤**（ms vs s），以及 Algorithm 2 的文字敘述把兩群都寫成「north to south」、Prop. 3 的 y 座標寫成「y₂ + Δx」，都是排版錯誤，讀的時候要對照演算法本體。
8. **集中式**：需要知道所有 GS 的可見集合、統一決定位移。標題說「distributed ground stations」是指 GS 分散在各地，不是分散式演算法。
9. 沒有公開程式碼；送達率實驗的 mininet 設定細節（GS 數、流量）描述很少。

## 7. 與其他筆記的關聯
### 和 motif（[[Network_topology_design_at_27000_km_hour]]）
* 本文的 related work 直接點名 motif（[13]）：motif 設計了 ISL 拓樸，但「假設 GS 只要在衛星範圍內就能連」，沒處理 GSL。兩篇剛好是同一個問題的兩半：**motif 設計衛星之間，本文設計衛星和地面之間**。
* 本文的實驗 ISL 用的就是 +Grid（motif 論文的基準線），沒有在 motif 拓樸上測。

### 和 LAMP（[[LAMP_Low-Latency_Dynamic_Topology_for_LEO_Satellite_Constellations]]）、DoTD（[[Time-_Dependent_Network_Topology_Optimization_for_LEO_Satellite_Constellations]]）
* 三篇都在處理「拓樸隨時間變，要低延遲又要少變動」的取捨：LAMP、DoTD 調**ISL**（加臨時連線、限制換線次數），本文調**GSL**（限制拓樸變動間隔 ≥ θ_T）。
* 三篇都把「拓樸變動」當成成本：DoTD 限制 ISL 重建次數、LAMP 有 slew cost、本文有 θ_T。差別是本文直接用 OSPF 的**送達率**證明變動的代價，比另外兩篇具體。
* 理論上兩者可以疊加：ISL 用 LAMP／DoTD，GSL 用 CSGI 的協調想法。但 CSGI 的 Prop. 2、3 假設 +Grid，疊加時要改（推論）。

### 和 Minimum-hop constellation design（[[Minimum-hop Constellation Design for LEO Satellite Networks]]）
* 同樣用跳數當主要指標。那篇從**星座參數**（軌道數、每軌顆數）降低跳數，本文在星座固定的前提下從**接入選擇**降低跳數。

### 和 StepClimb（[[Shortest_Path_in_LEO_Satellite_Constellation_Networks_An_Explicit_Analytic_Approach]]）
* StepClimb 說 +Grid 上兩顆衛星的最少跳數 = 軌道差 + 軌內差（MHP 矩形）。本文的 DMC「格子距離」就是同一件事。
* StepClimb 也區分 ascending／descending 衛星：來源和目的一個往北一個往南時路徑很長。本文 Fig. 6 的「兩群」就是這個現象出現在 GS 端：GS 選錯方向的那一群，就會多出十幾跳（Table I 的最大跳數差 12–15 跳，推論是主要來自這裡）。

### 和 OSPF 相關筆記
* [[OSPF_RFC2328_Key_Sections]]：本文的送達率實驗跑的是標準 OSPF，用 HELLO_INTERVAL 控制偵測速度。Hello 越短，斷線發現越快，送達率越高，但控制封包越多。
* [[OPSPF_Orbit_Prediction_Shortest_Path_First_Routing_for_Resilient_LEO_Satellite_Networks]]：本文把 Hello = 0.2 s 當成「預測式 OSPF」的替代（引用 OPSPF 為 [39]）。OPSPF 預測的是**極區 ISL** 斷線；GSL 換手也一樣可以預測（衛星軌道已知），所以 CSGI 的換手時刻其實可以事先通知路由協定（推論）。
* [[OSPF_in_Low_Earth_Orbit_Stability_overhead_and_routing_dynamics]]：那篇的模擬平台用 **hard handover**（選最近的衛星，超過距離門檻才換），大概就是本文的 NSD。那篇發現 OSPF 對 ISL 斷線恢復很快（ECMP），本文則顯示 **GSL** 換手對送達率的影響大得多：因為 GS 只接 1 顆衛星，GSL 斷了沒有備用路徑（推論）。

### 和 StableRoute（[[StableRoute_When_Dijkstras_Algorithm_Meets_Topology-Varying_Satellite_Networks]]）
* 兩篇都在追求**路由穩定**：StableRoute 在 ISL 變動時讓 Dijkstra 不要亂換路，本文在 GSL 變動時讓整條路徑長度不變。StableRoute 只看星座內，本文只看接入端，兩者互補。
* Survey 深讀筆記也建議把本文（[179] LSGI）和 StableRoute、DoTD 的 link churn 放在一起讀。

### 和 snapshot ISL 重新指派（[[Improving_the_snapshot_routing_performance_through_reassigning_the_inter-satellite_links]]）
* 那篇調整 ISL 指派讓快照數變少、快照時間變長；本文的 θ_T 和「能不換就不換」是同一個想法用在 GSL 上：**讓拓樸維持不變的時間拉長**，路由表就不用常常重算。

### 和 survey（[[LEO_Satellite_Networking_Relaunched_Survey_and_Current_Research_Challenges]]）
* Survey 把本文列為 **[179]**，描述為「把地面段和太空段一起設計，最小化最大延遲，同時保持路由穩定」。和原文一致。

### 其他（PC 筆記庫裡還沒有的）
* **The Nearest Is Not The Fastest**（FROG，VTC2022-Fall；筆記在專案資料夾 notes/nearest_not_fastest/，PC 上還沒有）：幾乎是同一個想法的獨立版本。FROG 讓每個 GS 從最近 k 顆裡、兩端一起挑 k² 種組合中路徑最短的；本文則多考慮了**時間**（θ_T、協調平移）和**多對 GS 一起最佳化**。兩篇都指出「最近的衛星不一定讓整條路最短」，原因都是 +Grid 上往南和往北的衛星圖上距離很遠。
* 你 PDF 資料夾裡的 *Efficient_Satellite-Ground_Interconnection_Design_for_Low-Orbit_Mega-Constellation_Topology* 是同一個題目（survey 深讀筆記這樣判斷），可能是本文的延伸版，還沒讀。
