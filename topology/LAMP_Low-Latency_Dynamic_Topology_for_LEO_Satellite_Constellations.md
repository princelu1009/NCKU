---
title: "LAMP: Low-Latency Dynamic Topology for LEO Satellite Constellations"
authors: [Robert Esswein, Quincy Bayer, Mai Abdelhakim, Robert Cunningham, Samuel Mergendahl, Jon Ruffley]
venue: IEEE ICC 2025
pages: 4714–4719
doi: 10.1109/ICC52391.2025.11162074
tags: [LEO, ISL, topology-design, dynamic-topology, propagation-delay]
---

## Introduction
* LEO constellations traditionally use a static grid (+Grid): each satellite keeps fixed links to its two neighbours in the same orbital plane (OP) and to one satellite in each adjacent OP. Because OPs cross, two satellites that are physically close often sit in OPs far apart in the grid, so traffic between them takes a long multi-hop detour and propagation delay rises.
* The existing dynamic approach, DTLS, adds temporary links on top of a backbone. It picks them by lowest average link length, which does not necessarily lower latency, and every satellite slews its laser at the same time-slice boundary.
* LAMP uses three of each satellite's four laser transceivers as a fixed backbone that guarantees connectivity. The fourth carries temporary links chosen offline by a greedy algorithm.
    * Each round adds the candidate link with the largest weight. The weight is the sum, over the link's usable period, of the current shortest-path length minus the direct distance between the two satellites.
    * This ties link selection directly to the objective, mean propagation delay, and links may switch at any time rather than all at once.
* The authors also model laser slew time under relative satellite motion, so the cost of switching is included in the evaluation.
* Walker-Delta simulations at 550 km and 53° with 100–256 satellites:
    * LAMP lowers mean propagation delay by 18.5% compared with the static grid and by 5.61% compared with DTLS.
    * LAMP varies least across configurations (std 1.75 ms, against 2.60 ms for DTLS and 6.46 ms for the static grid).

## System Model
* Constellation: Walker-Delta i:t/p/f, with phasing f chosen to maximise the minimum passing distance. Each satellite has 4 laser transceivers.
* Link types:
    * **Intra-OP**: constant length.
    * **Inter-OP**: between adjacent OPs; the length varies.
    * **Temporary**: between non-adjacent OPs, only while the two satellites are close.
* Notation: E Earth radius, a altitude, v ≈ 80 km (highest altitude of water vapour, which must not block the laser), L the set of possible temporary links, L_topo ⊂ L the chosen ones, T timesteps per orbit, t satellites, Δ shortest-path length (km), δ straight-line distance, ω slew rate.
* Related work:
    * Motifs (Bhattacherjee & Singla 2019) uses only long-lived links.
    * DTLS (Zhu et al., Electronics 2023) uses a backbone plus temporary links in time slices. The links of slice 1 are chosen by lowest average link length and reused for the other slices by symmetry.

## Method
### A. Backbone
Keep all intra-OP links plus **one** permanent inter-OP link per satellite, alternating east and west along the OP. This removes 1/4 of the grid links but stays connected, and frees the 4th transceiver for temporary links.

### B. Objective and greedy selection
* Objective, assuming shortest-path routing:
$$\min_{L_{topo}} \sum_{\tau} \sum_{s_1,s_2} \frac{\Delta(s_1,s_2)}{T(t^2-t)} \;\;\Longleftrightarrow\;\; \min_{L_{topo}} \sum_{\tau}\sum_{s_1,s_2}\Delta(s_1,s_2)$$
* Visibility, where the line of sight must stay above altitude v:
$$\delta(s_1,s_2) \le 2\sqrt{(E+a)^2-(E+v)^2}$$
* A temporary link is a tuple l = (u, d, s1, s2), where u and d are its first and last usable timesteps. Its weight is
$$w_l = \sum_{\tau=l.u}^{l.d} \big[\Delta(l.s_1,l.s_2) - \delta(l.s_1,l.s_2)\big]$$
  and the window excludes slew time.
* Greedy loop:
    1. Find all satellite pairs that satisfy the visibility condition at each timestep; these form L.
    2. Compute w_l for every l in L.
    3. Add the largest-weight link l₀.
    4. Trim the start and end of remaining links that share a satellite with l₀.
    5. Recompute weights and repeat until no candidate is left.
* Claim, given as a "simple proof" rather than a numbered theorem: every added link lowers the mean delay, and w_l is a **lower bound** on the decrease of the objective. For s_x one hop from l.s1, before the link is added Δ(s_x, l.s2) ≥ Δ(l.s1, l.s2) − δ(l.s1, s_x), and after it is added Δ(s_x, l.s2) = δ(s_x, l.s1) + δ(l.s1, l.s2). Nearby pairs therefore benefit too.

### C. Slew model
* Constant slew rate ω; a link is unusable while its terminal slews.
* Orientation is fixed to the OP, so intra-OP links never slew, and the static inter-OP link is assumed to stay connected.
* Local frame: Z points toward the next satellite in the OP, and X = cross(s1→s0, s1→s2).
* The slew angle is first estimated from ∠s_x s1 s_y, then refined at each timestep.

## Evaluation Setup
* 550 km altitude, 53° inclination, orbital period 5737 s; 10–16 OPs × 10–16 satellites per OP (step 2).
* Timestep 1.9999 s; slew 1°/s; each temporary link adds 2 s of setup and 2 s of cleanup.
* Compared: static grid, backbone only (3 links per satellite), DTLS, LAMP.
* Metric: mean propagation delay over all satellite pairs over one orbit.

## Main Results
| Comparison | Change in mean delay |
|------------|----------------------|
| LAMP vs static grid | −18.5% |
| LAMP vs DTLS | −5.61% |
| Backbone only vs static grid | +18.4% |

* Gains grow with more OPs and fewer satellites per OP. The static grid's worst case (t = 160, p = 16) is 75.1 ms, against 49.4 ms for LAMP (−34.2%).
* Std across configurations: LAMP 1.75 ms, DTLS 2.60 ms, static 6.46 ms.
* The paper's explanation for beating DTLS is that DTLS slews every satellite at once, leaving only the backbone for a while, whereas LAMP switches links asynchronously.

## Discussion and Limitations
* Stated by the authors:
    * Phased deployment requires recomputing the schedule.
    * Multi-layer constellations could be handled with the LCM of the orbital periods and infinite weights for impossible links. This is proposed, not evaluated; the authors argue DTLS would fail there.
    * Links are computed offline, which suits predictive routing such as OPSPF.
    * Future work: multi-layer constellations, feeding the predictions into routing, and ground-to-ground paths, which are not evaluated.
* Implied by the paper:
    * The greedy heuristic has no optimality guarantee.
    * The slew model is simplified.
    * Only ≤ 256 satellites at a single 550 km / 53° shell are tested.
    * Only propagation delay is measured, with no traffic, congestion or throughput.
    * No runtime is reported.
    * The backbone alone is 18.4% worse than the grid, so the gain depends on the temporary links being available.

## 與其他筆記的關聯
* 與 [[Time-_Dependent_Network_Topology_Optimization_for_LEO_Satellite_Constellations]]（DoTD）同為動態拓樸：DoTD 每個時槽重新選 4 條鏈路，並用歷史分數壓低換線；LAMP 固定 3 條 backbone，只動第 4 條，並明確建模 slew 成本。
* [[Minimum-hop Constellation Design for LEO Satellite Networks]] 以 hop 數為目標，LAMP 以傳播延遲（km）為目標；兩者都指出 +Grid 的跨 OP 繞路問題。
* LAMP 的鏈路離線排程適合預測式路由；換線時的路由穩定性可參考 [[StableRoute_When_Dijkstras_Algorithm_Meets_Topology-Varying_Satellite_Networks]] 與 [[Stable_Hierarchical_Routing_for_Operational_LEO_Networks]]。
