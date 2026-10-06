---
title: "Stable Hierarchical Routing for Operational LEO Networks"
authors: [Yuanjie Li, Lixin Liu, Hewu Li, Wei Liu, Yimei Chen, Jianping Wu, Qian Wu, Jun Liu, Zeqi Lai]
venue: ACM MobiCom 2024
tags: [LEO, satellite-routing, ISL, hierarchical-routing, geographic-routing]
---

# Stable Hierarchical Routing for Operational LEO Networks (SHORT)

## Citation
- Authors: Yuanjie Li et al.（Tsinghua University、Zhongguancun Laboratory）
- ACM MobiCom '24, pp. 296–311. DOI: 10.1145/3636534.3649362

## 1. Problem & Motivation
- LEO mega-constellation 正從 bent-pipe 走向以 ISL 做全球路由，但 topology 變動劇烈，flat routing 不穩。
- 以 SSN TLE 資料（2019/5–2023/8）＋ Starlink 終端實測量化三層 dynamics：
  1. Space-terrestrial：GSL churn 每 1.46–3.98 s 一次。
  2. Intra-shell：shell 2/3 ISL churn 約 208.7 s / 970.0 s；部分部署的 shell 4 inter-orbit ISL 每 11.4 s（多 18.3×）。Maneuver 平均 48 次 ISL churn/天；約每 13 顆有 1 顆故障。
  3. Inter-shell：相對運動非線性，ISL churn 比 GSL 更頻繁。
- 既有方案不足：flat proactive（OLSR、SDN）每次 churn 全域 re-convergence；reactive（AODV）flooding；predictive（TS-SDN、Starlink 預排程）table 爆炸且 maneuver/failure 難預測；傳統 hierarchical 需要 GEO/MEO backbone。
- 核心問題：如何在高動態 LEO 中建構穩定的 routing hierarchy？

## 2. Key Idea
以 Earth-centric geographic 為不變量。三層 hierarchy：Tier 1 terrestrial（GS 作 shell 間 backbone）→ Tier 2 orbital shell → Tier 3 orbit。Stable backbone ＋ opportunistic inter-shell ISL；可作為 control-plane overlay 疊在 OLSR/AODV/MPLS/SDN 上。

## 3. Method
### 3.1 Orbital-geodetic coordinate
Sub-point 原本非線性：
$$\sin\varphi_t=\sin i\,\sin(\tfrac{2\pi}{T_S}t-\gamma_0)$$
SHORT 對傾角 $i$ 的 shell 定義 $(\alpha,\gamma)$，使其線性：
$$\alpha_t=\alpha_0-\tfrac{2\pi}{T_E}t,\qquad \gamma_t=\gamma_0+\tfrac{2\pi}{T_S}t$$
Proposition 1：同 shell 無 maneuver 時，$\Delta\alpha,\Delta\gamma$ 時不變。
Address：`prefix | shell ID | cell col(α) | cell row(γ) | node ID`，與 serving satellite 解耦，可嵌入 IPv6。

### 3.2 Intra-shell routing
- Greedy：先用 inter-orbit ISL 減 $\Delta\alpha$，再用 intra-orbit ISL 減 $\Delta\gamma$。
- Detour：local minimum 時 counter-clockwise 遞迴 reroute（DFS），有路必達。
- Legacy 增強：dst_sat = argmin 與目標 $(\Delta\alpha,\Delta\gamma)$ 的歐氏距離；失效時改選同 cell 的 backup。

### 3.3 Inter-shell routing
1. 經 GS：$\text{gs}=\arg\min\{C_S(S\to gs)+C_D(gs\to D)\}$，封裝後由 GS 轉入目的 shell。
2. Opportunistic shortcut：途中若有通往目的 shell 的 ISL 就直接走，無 loop；相容 bent-pipe shell。

## 4. Evaluation
StarryNet emulation 重播真實 TLE；Starlink、Iridium、Telesat；+Grid、4 條 1 Gbps ISL；500 組 S-D、1 小時。Baselines：OLSR、TS-SDN、AODV、LBP。Prototype 約 13K 行 C（Linux kernel＋Quagga）。

## 5. Main Results
- Stability：routing table updates 比 OLSR 少 31,816×（Starlink）/9×/323×；TS-SDN re-computation 多 3,477×。
- Availability：約 100%；OLSR ≤8%；AODV re-discovery 72–119 ms。
- Efficiency：Iridium/Telesat 與最佳相同；Starlink 最多多 3.48 ms / 3 hops。
- Resiliency：40+ ISL 失效仍可達。
- 座標變動 ≤0.05°（lat/lon 4.02°/6.26°）；計算比 H3 快 78.9×；24-bit cell 支援 42,000 顆。
- GS selection 比 optimal 多 1.23 hops / 3.33 ms。
- Opportunistic ISL 0%→100%：延遲 52.32→34.78 ms，hops 12.69→6.98。
- Data plane：CPU <1%、記憶體約 1.3 MB、FIB lookup 約 0.007 ms。

## 6. 質疑點
- Proposition 1 假設 circular orbit、無 maneuver；J2 與 drag 漂移只以「≤0.05°」帶過。
- 只跑 1 小時、無 traffic load/congestion 模型；geographic routing 易在熱點壅塞。
- 海洋/極區可能沒有共同 GS，繞路大；GS 成為 bottleneck 與信任點。
- Detour 最壞約 150 hops、500 ms。
- 「~100% availability」未計入 GSL 遮蔽與天候。
- 主要對比 OLSR 這種極端 baseline，倍數有誇大之嫌；未與 Starlink 實際 SDN 比較。
- Security（位置 spoofing）討論很少。

## 7. 與 ISL Topology Design 的關聯
- 支持 +Grid 作基線：inter-orbit ISL 對應 $\Delta\alpha$、intra-orbit 對應 $\Delta\gamma$。
- Inter-shell ISL 雖 churn 頻繁，作為 shortcut 可降延遲約 33%、hops 約 45%。
- Partial deployment 使 churn 增 18.3×，設計需考慮過渡拓撲的穩定性。
- 可把 routing stability 納入 topology 目標，偏好 $(\alpha,\gamma)$ 中距離固定的鄰居。
- 與 [[Time-_Dependent_Network_Topology_Optimization_for_LEO_Satellite_Constellations]]（DoTD）對照：DoTD 用歷史分數穩定拓撲，SHORT 用座標不變量穩定路由。
