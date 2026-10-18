---
title: "OSPF in Low Earth Orbit: Stability, overhead, and routing dynamics"
authors: [Josemaría Malgosa-Sanahuja, Juanjo Alíns Delgado, Sergio Machado Sánchez, Javier Ozón Górriz, Jorge Mata-Díaz]
venue: "Computer Networks 287 (2026) 112555（Special issue MOD-PERF-W-ADHOC-UBI-NET）"
doi: 10.1016/j.comnet.2026.112555
received: "2026-03-14；accepted 2026-07-05；online 2026-07-11；open access CC BY"
tags: [LEO, satellite-routing, OSPF, link-state, ECMP, Walker-Star, polar-orbit, Teledesic, emulation, FRRouting, RFC9717, Nautilus-effect]
---

## Citation
J. Malgosa-Sanahuja, J. Alíns Delgado, S. Machado Sánchez, J. Ozón Górriz, J. Mata-Díaz, "OSPF in Low Earth Orbit: Stability, overhead, and routing dynamics," *Computer Networks*, vol. 287, 112555, 2026. DOI: 10.1016/j.comnet.2026.112555（Universidad Politécnica de Cartagena、Universitat Politècnica de Catalunya）

## 一句話總結
這篇**不提出新的路由協定**，而是回答一個問題：**不改協定本身的 OSPF，放到 LEO 星座上到底能不能用？** 作者自己做了一個模擬平台：用改過的 Hypatia（叫 **HypatiaSeam**）算軌道和 ISL/GSL 的開關，再把每秒的拓樸變化套到 **LXD/Incus 容器**裡，每顆衛星都是一台跑真的 **FRRouting OSPF** 的 Linux。拿 **Teledesic 12 × 24 = 288 顆、Walker-Star（有 seam）、極區 ±60° 以上關掉跨軌 ISL** 跑兩小時。結論是 OSPF **比想像中好**：(1) 一條 ISL 斷掉只影響附近一小群衛星（同一列和上下兩列）；(2) +Grid 拓樸裡兩點之間有大量 **hop 數一樣的最短路（ECMP）**，斷線時直接切到備用 next hop，**< 30～60 ms** 就恢復，不用等 SPF 重算；(3) 訊令每顆衛星 **< 2 kbps**，相對 ISL 容量可以忽略；(4) 兩次「重算 SPF」的事件平均隔約 **23 s**，不會疊在一起造成不穩定。唯一的毛病他們叫 **Nautilus effect**：衛星**即將**進入極區時，經過它的那條最短路會突然失效、短暫掉包。解法是照 **RFC 9717** 的精神，**在進入極區前把那條跨軌 link 的 cost 從 1 調成 2**，讓流量提早改走另一條路。

**English:** This is an evaluation paper, not a new protocol. It asks whether unmodified OSPF works in a LEO constellation. The authors build an emulator (HypatiaSeam for orbits and link schedules, plus LXD/Incus containers running FRRouting OSPF on every satellite) and run a 288-satellite Teledesic Walker-Star constellation for two hours. Link failures stay local, ECMP backup next hops recover traffic in under 30–60 ms, signalling is under 2 kbps per node, and full SPF events are about 23 s apart. The one problem, the "Nautilus effect", is packet loss when a satellite on one of two equal-cost paths enters the polar zone. Raising that link's cost before the polar entry (as RFC 9717 recommends) removes it.

## 0. 背景知識
### Walker-Star 的 seam 和極區（§3，Fig. 1–2）
* **Walker-Star（極軌）**：所有軌道面都經過兩極；相鄰軌道的衛星在極區附近相對距離、指向角變化很快，所以**跨軌 ISL 在緯度 ±60° 以上關掉**（本文設定，Table 1）。注意 [[OPSPF_Orbit_Prediction_Shortest_Path_First_Routing_for_Resilient_LEO_Satellite_Networks]] 用的是 70°。
* **Seam**：Walker-Star 的 RAAN 只鋪 180°，第一個和最後一個軌道面方向相反，**中間不能建跨軌 ISL**。所以拓樸是「有一條縫的網格」，不是環面（torus）。
* **Walker-Delta**：傾斜軌道、RAAN 鋪 360°，跨軌 ISL 幾何關係大致固定、比較穩定（[[A_Distributed_Survivable_Routing_Algorithm_for_Mega-Constellations_With_Inclined_Orbits]] 處理的就是這種）。

### ECMP（Equal-Cost Multi-Path）
* OSPF 本來就支援：若到同一目的地有好幾條**總 cost 一樣**的路，路由表可以同時放好幾個 next hop。
* +Grid 拓樸、所有 ISL cost = 1 時，兩點相隔 N 列 × N 行的最短路數量是 **C(2N, N) ≈ 2^{2N}/√(πN)**（Fig. 4，lattice path）。作者的關鍵觀察：**不必存全部路徑**，因為大約一半的最短路第一步往水平走、另一半往垂直走，所以只要兩個 next hop（一個「預設」、一個「備用」）就能概括全部。只有起點和終點**同一列或同一行**時才沒有備用。

### RFC 9717（背景）
* IETF 2025 年的「A Routing Architecture for Satellite Networks」（Tony Li）。建議沿用 IETF 的 link-state 協定（偏好 **IS-IS**，因為 area 定義彈性較大），再加上「**已排程的鏈路變化**」的知識；對**可預期要斷的 link，提前調高 metric**，讓流量在斷之前就改道。本文的 Nautilus 解法就是這個建議的實證。

## 1. Problem
* **Problem**（§1、§2、§4）：
    * 文獻對 OSPF 在 LEO 的結論互相矛盾：有人說稍微改就能用，有人說要整個重寫。作者認為原因是很多評估**直接用地面網路的預設參數**、模擬平台過度簡化、指標選得不對（只看 end-to-end delay、loss、hop count，不看路由本身的指標）。
    * 作者一開始的**假設是 OSPF 不行**（§4），理由有三：
        1. OSPF 經驗上單一 area 大約 50～60 台 router，LEO 星座動輒上百上千。
        2. 拓樸變化太頻繁：本文量到**平均每 3 s 一次變化**，最常見的間隔是 **2 s**（P ≈ 0.435，Fig. 3），比一般 OSPF 收斂計時器還短。
        3. 最短路數量隨距離指數成長（C(2N, N)），擔心 SPF 計算爆掉。
* **Target**：在**真實 protocol stack + 真實軌道**的環境下，量 OSPF 的**收斂時間、訊令量、路由表變動次數、SPF 間隔**，並找出要怎麼調才能用。
* **有沒有正式 formulation**：沒有。純實驗研究，唯一的公式是快照時間 t_k = t_0 + kΔt（Eq. 1）。

## 2. 主要發現（§5–§7）
### 2.1 斷一條 ISL 會影響誰（§5.1，Fig. 5）
實驗：Teledesic 12 × 24，把 **衛星 1–13 和 2–13 之間的 link 斷掉一分鐘再接回來**。受影響的只有三組：

| 組 | 是誰 | link 斷掉時 | 一分鐘後 link 恢復時 |
|---|---|---|---|
| **Set A**（紅） | link 兩端那 2 顆 | < 1 ms 偵測到，**< 30 ms** 換到 ECMP 備用路 | 介面 < 30 ms 起來，但要**約 10 s** 才算出新路由，之後 < 30 ms 生效 |
| **Set B**（灰） | 同一緯度帶（同一「列」）的跨軌鄰居 | 沒有自己的 link 斷，**< 60 ms** 更新 | 約 10 s 算完，約 2 ms 生效 |
| **Set C**（白） | 上下兩列（同軌鄰居） | 只刪掉那條 entry | 一分鐘後加回 |

* 重點：**大多數路由本來就有 ECMP 備用 next hop，所以斷線不需要「密集的 SPF 重算」**，使用者幾乎感覺不到。重算後的路由表仍維持「預設＋備用」結構。
* 影響範圍因為軌道傾角和相位偏移，不會剛好落在一條緯線上，但仍是局部的一小群。
* **我們的推測**：恢復時那「約 10 s」很可能主要是 OSPF 重新建 adjacency（Hello、DD 交換）的時間，而不是 Dijkstra 本身慢；論文沒拆開說明，也**沒有列出 Hello/Dead interval、SPF throttle 等計時器的設定值**，雖然摘要說是「properly configured OSPF」。

### 2.2 訊令量（§5.2，Fig. 6）
* Hello 是固定背景流量，排除不算；只看 link 變化引起的 DD 和 LS 封包。
* **斷線**：持續約 **10 s**，鄰近節點峰值 **< 2 kbps**，其他節點 < 0.75 kbps（在極區的只有兩個介面，< 0.25 kbps）。
* **恢復**：持續約 **20 s**（大量重複 LSA ＋ 要交換 DD），link 兩端約 2 kbps。
* 對照 ISL 容量（RF 約 1 Gbps、光學 10～100 Gbps 甚至 400 Gbps），訊令只佔 **0.0002%**（1 Gbps）到 **0.000002%**（100 Gbps）。

### 2.3 路由表變動次數（§5.3，Fig. 7）
* 斷線那一分鐘內，**1–13 更新路由表 37,719 次、2–13 更新 3,186 次**（原文用歐式寫法「37.719」）。原因：1–13 到軌道面 2～12 的大部分路由都走這條 link；2–13 只有到軌道面 1 才用它。
* 附近節點 < 2000 次，其餘 < 60 次。0.75 kbps 的流量大多是重複 LSA，不會真的改路由。

### 2.4 SPF 之間的空檔（§5.4）
* 相鄰軌道的衛星不會同時進極區（相位錯開），所以極區斷線在時間上也是分散的。
* 實測兩次「密集 SPF 計算」之間通常隔 **約 23 s**，對現代處理器綽綽有餘 → 不會疊加成不穩定。
* ⚠️ 論文說這代表 OSPF 在 **Walker-Star 和 Walker-Delta 都穩**，但實驗只跑了 Walker-Star（Teledesic），Walker-Delta 部分是推論。另外 §5.4 寫「24 × 12」，Fig. 5 和 Table 1 是 12 個軌道面 × 每面 24 顆，只是寫法順序不同。

### 2.5 End-to-end（§6，Fig. 8）
* 流量用 D-ITG 產生。
* **衛星 2–14 → 11–10**（9 水平 + 4 垂直 hop）：150 s 內有 25 次南北極區斷線/恢復，延遲完全平穩（圖上約 135 ms）。
* **地面終端 ↔ 地面站**（6 水平 + 4 垂直 = 10 hop，約 100 ms），跑 7200 s：
    * 一端換到同軌下一顆衛星 → hop 少 1，延遲小降（約 95 ms）；另一端也換之後回到原值。
    * 約 3600 s 時因地球自轉，一端換到相鄰軌道面的衛星 → hop 少 2，降到約 83 ms，50 s 內另一端也換，恢復原狀。
    * 共送 **73,596 個 UDP 封包、平均 9.97 pkt/s、掉包率 0.07%**。
    * 中途隨機斷幾條 ISL，延遲也不受影響（ECMP）。

### 2.6 Nautilus effect（§7，Fig. 9）⭐
* 情境：A 送到 B，有兩條等長最短路：**route 1：A–I1–I2–B**、**route 2：A–I1–I3–B**。
* 衛星往北飛，**I3 先進入極區**，I3–B 的跨軌 link 關掉 → route 2 斷了，但在 OSPF 收斂前 I1 仍可能把流量丟給 I3 → **短暫掉包**。
* 因為多數地面站在較高緯度、靠近極區，這種情況**比想像中常見**。
* 論文沒有解釋 Nautilus 這個名字的由來。
* **解法**：照 RFC 9717 **主動調 metric**。所有 link 平常 cost = 1；**衛星接近極點時，把它連到同軌前一顆衛星的那條 link 的 cost 調成 2**，離開極區再調回 1。套到 Fig. 9：I3 快進極區時，**I1–I3（同軌）** 的 cost 變 2，route 2 總 cost 變 4、route 1 仍是 3，所以在 I3–B（跨軌）真的斷掉之前，OSPF 已經只用 route 1。注意調的是**同軌** link，斷的是**跨軌** link，兩者不是同一條。作者說模擬結果顯示這能**消除 Nautilus 造成的掉包**，但**沒有給數字或圖**。

## 3. 模擬平台（§8，Fig. 10–11）
這部分是本文的另一個貢獻，想自己做實驗時值得細讀。
* **兩段解耦**：
    1. **HypatiaSeam**（改自 Hypatia 的 satgenpy）：產生每個快照 k 的 `satellite_network_k.gml`（衛星位置、有效 ISL、距離）和 `satellites_in_range_k.txt`（每個地面站看得到的衛星）。新增：epoch 日期、TLE 產生、地球自轉、GSL 仰角/距離限制、**RAAN 180°（有 seam 的 Star）或 360°（Delta）**、相位設定、**依緯度啟用跨軌 ISL**、每個地面站的天線數上限和 **hard handover**（選最近的衛星，超過距離門檻才換）。
    2. **Topology Builder → LXD/Incus 模擬**：距離換成單向傳播延遲，輸出 `time_k.json`、`initialGroundStations.json`、`topology.json`。每顆衛星、地面站、終端都是獨立 Linux 容器，衛星和地面站跑 **FRR/OSPF**；**每條 ISL/GSL 一座 Layer-2 Linux bridge**，延遲用 `tc/netem` 設。
* **執行期**：APScheduler 以 **1:1 時間比例**套用四種事件：`linkup`、`linkdown`、`linkmodify`（改延遲）、`linkGS`（地面站換衛星）。容器和 FRR 不重啟，OSPF 狀態連續。
* **參數（Table 1）**：12 軌道面 × 24 顆、RAAN 180°、傾角 98.2°、高度 1315 km、phase factor 1、GSL 最小仰角 10°、跨軌 ISL 只在 −60° < lat < 60°；Δt = 1 s；跑 120 min；4 個地面站（Canary Islands、UK、New York、Mexico）。
* **沒有模擬**：頻寬限制、指向誤差、隨機掉包、jitter。只有連通性和傳播延遲。
* Hypatia 也能輸出預算好的最短路（給 SDN 集中式用），本文沒用。

## 4. 值得讀的段落
| 段落 | 為什麼值得讀 |
|---|---|
| **§4.1 + Fig. 3** | LEO 拓樸變化頻率的實測分佈（最常 2 s，平均 3 s 一次） |
| **§4.2 + Fig. 4** | +Grid 上最短路數 C(2N,N)，以及「只需預設＋備用兩個 next hop」的觀察 |
| **§5.1 + Fig. 5** | 斷一條 link 只影響三組節點；ECMP 讓斷線恢復 < 30 ms |
| **§5.4** | SPF 事件間隔約 23 s，解釋為何不會不穩定 |
| **§7 + Fig. 9** ⭐ | Nautilus effect 和「提前調 cost」的解法，是本文唯一的設計建議 |
| **§8 + Fig. 10–11** | 可重現的模擬架構（Hypatia + 容器 + FRR），想自己做實驗時參考 |
| §2 | 很好的 OSPF-in-LEO 相關研究清單（ASER、OPSPF、SATNET-OSPF、MPLF、IFAR、SER…） |

## 5. 和專案裡其他 routing 筆記的關係
* **[[OPSPF_Orbit_Prediction_Shortest_Path_First_Routing_for_Resilient_LEO_Satellite_Networks]]**（本文 ref [16]）：
    * **同一個問題**：極軌星座極區跨軌 ISL 規律性斷線，OSPF 不知道這是可預期的。
    * **不同的解法強度**：OPSPF 用軌道預測**直接把要斷的 link 從 LSDB 拿掉**，每秒自己跑 Dijkstra；本文只是**提前把 cost 從 1 調成 2**，link 還在、仍可當備用，其餘照用原版 OSPF。可以把本文的 Nautilus 解法看成「OPSPF 的輕量版」，只處理 ECMP 平手被打破的那一刻。
    * **對 OPSPF 動機的挑戰**：OPSPF 的痛點是 OSPF 要等 **Hello 逾時**才發現斷線。本文的模擬裡介面一斷 **< 1 ms 就偵測到**，加上 ECMP 備用路，所以斷線本身幾乎不痛；真正的掉包只剩 Nautilus 那種情況。⚠️ 這差異可能來自模擬方式（bridge 拆掉 → 介面立刻 down），真實雷射 ISL 不一定這麼快偵測，值得記住。
    * 極區門檻：OPSPF 70°、本文 60°。
* **[[ASER_Scalable_Distributed_Routing_Protocol_for_LEO_Satellite_Networks]]**（本文 ref [7]）：ASER 在 NS2 上認為大星座下 OSPF 收斂慢、開銷大，所以要分 area。本文在 288 顆、真 FRR 的環境下認為**單一 area 的 OSPF 就夠用**，並在 future work 說要試多個 OSPF area 來進一步減少 Nautilus 和 flooding。兩篇規模（ASER 288～1296 顆）和平台不同，結論不能直接相比。
* **[[StableRoute_When_Dijkstras_Algorithm_Meets_Topology-Varying_Satellite_Networks]]** 和 **[[SIDA_note]]**：三篇都在處理 +Grid 上「很多條一樣短的路」。StableRoute 把它看成**沒必要換路**的來源；SIDA 拿它來**做負載平衡**；本文則把它當成**免費的備用路**（ECMP），是 OSPF 能撐住的主因。Nautilus effect 正好是「平手被打破」那一刻出問題。
* **[[Shortest_Path_in_LEO_Satellite_Constellation_Networks_An_Explicit_Analytic_Approach]]**（StepClimb）：同樣利用 Walker 網格的規律性；StepClimb 用公式取代 Dijkstra，本文則說明 Dijkstra 本身在 288 顆時並不是瓶頸（SPF 間隔 23 s）。
* **[[On_distributed_geographic-based_packet_routing_for_LEO_satellite_networks]]**（Henderson）：**同樣用 Teledesic 288 顆**當測試星座，可以對照：Henderson 試「不跑路由協定、只用地理位置貪婪轉送」，本文試「直接跑標準 link-state 協定」。
* **[[A_Disruption_Tolerant_Distributed_Routing_Algorithm_in_LEO_Satellite_Networks]]**（DTDR）：DTDR 假設極區 ISL 不斷；本文和 OPSPF、ASER 都假設會斷。
* **[[A_Distributed_Survivable_Routing_Algorithm_for_Mega-Constellations_With_Inclined_Orbits]]**（Qi 2020）：處理 Walker-Delta；本文聲稱結論也適用 Walker-Delta，但沒實驗。
* **[[The_Nearest_Is_Not_The_Fastest]]**：本文 §6 的 Fig. 8(b) 剛好示範了「地面站換到哪顆衛星會改變 hop 數」（換到相鄰軌道少 2 hop），和 FROG 的論點一致，但本文用的是最近衛星的 hard handover，沒有最佳化入口/出口。
* **[[LEO_Satellite_Networking_Relaunched_Survey_and_Current_Research_Challenges]]**：本文 ref [3]，相關工作的分類（分散式 vs 集中式 snapshot）沿用這篇。

## 6. 讀後評論
* **優點**：用真實 OSPF 實作（FRR）和真實軌道，而不是抽象模型；把「ECMP 備用 next hop」講清楚，解釋了為什麼 OSPF 在 +Grid 上比預期好；模擬架構拆得很乾淨，換協定就能重用。
* **限制**：
    * 只有一個星座（288 顆、Walker-Star），沒有 Starlink 規模，也沒跑 Walker-Delta。
    * **沒有和任何其他協定比較**（OPSPF、ASER 等都只在相關研究提到）。
    * 沒列 OSPF 計時器設定，難以重現「properly configured」。
    * Nautilus 解法只有文字描述，沒有掉包數字或前後對照圖。
    * 模擬不含頻寬、指向誤差、隨機掉包；資料「confidential」，不公開。
* **對研究的啟示**：如果要和 OPSPF 類方法比，這篇提供了一個更強的 baseline：「**原版 OSPF + ECMP + 進極區前調 cost**」。新方法至少要贏過它，而不是只贏過預設參數的 OSPF。
