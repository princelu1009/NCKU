---
title: "Your Mega-Constellations Can Be Slim: A Cost-Effective Approach for Constructing Survivable and Performant LEO Satellite Networks"
authors: [Zeqi Lai, Yibo Wang, Hewu Li, Qian Wu, Qi Zhang, Yunan Hou, Jun Liu, Yuanjie Li]
venue: IEEE INFOCOM 2024
pages: 521–530
doi: 10.1109/INFOCOM52122.2024.10621083
code: https://github.com/SpaceNetLab/MegaReduce
tags: [LEO, constellation-design, survivability, mega-constellation, cost]
---

## Introduction
* **Trend**: companies such as SpaceX and Amazon are deploying LEO mega-constellations of thousands of satellites. Since most of these LEO satellite networks (LSNs) are still being built out, how many satellites are actually needed is a timely question.
* **Trade-off**: more satellites improve survivability and performance, but raise cost and governance problems such as conjunctions and space debris.
* **Gaps in prior work**:
    * Constellation design ignores survivability.
    * Survivable network design assumes a static network.
    * Multi-tier designs with GEO satellites suffer from higher delay and limited capacity.
* **Contribution**:
    * The SPLD problem: the fewest satellites that still meet survivability, capacity and delay requirements, for a single operator without GEO.
    * MegaReduce, which the authors claim finds a near-optimal answer in polynomial time; neither claim is proven.
    * Evaluation on real constellation data (Starlink, Kuiper).

## Problem Formulation
* The formulation in Section III defines what a feasible design is. MegaReduce does not solve it directly; instead it guesses a constellation size, checks it against these constraints, and uses binary search (Shrink / Expand) to converge on the smallest feasible size.
* Model:
    * G_t = (V, E_t), with V = S ∪ C: satellites plus ground cells (H3 hexagons).
    * I: visibility; e: active link; x(i) ∈ {0,1}: whether satellite i is kept.
    * Each cell is served by one beam, and ISLs are assumed not to be the bottleneck.
    * Demands are (src, dst, size), and r_ij edge-disjoint paths are required in every time slot.
* ILP: min Σ x(i) subject to
    * (1) I ≥ e: a link exists only if the endpoints are visible to each other.
    * (2) x(i)·x(j) ≥ e: a link exists only between kept satellites.
    * (3) Σ e ≤ N_ISL per satellite.
    * (4)(5) per-cell uplink/downlink capacity; (6)(7) per-satellite Cap_max for uplink/downlink.
    * (8) Cut constraint: for every nonempty 𝒱 ⊂ V, the edges crossing the cut σ(𝒱) number at least max r_pq. This is what guarantees r edge-disjoint paths.
* Delay extension: a layered directed graph with L_d + 1 layers, L_d = ⌈λ·L^sp⌉ (λ ≥ 1 times the shortest-path hop count), with binary flows ω and constraints for (9) flow conservation, (10) no local loops, (11) Σ_l ω ≤ x(i), and (12) capacity.
* Hardness: with r = 1 and a single slot, the problem reduces to Steiner Tree, so it is NP-hard (stated as a remark, not a theorem). The ILP is intractable at hundreds of satellites, which motivates the heuristic.

## Requirement Driven LS Optimization
* MegaReduce consists of three algorithms that together implement a guess-and-check search, starting from the operator's own constellation.
* **Algorithm 1 (main loop)**
    * Sets a search range [N_min, N_max] for the constellation size. N_max = |V| is the original size. N_min comes from GetSurvivableBound: r disjoint paths require each cell to see at least max_j r_ij satellites during service hours.
    * The constellation is a Walker shell [Inc, O, M, H]. While the iteration count is at most I_limit:
        * if the candidate is feasible, set N_max ← O·M, store it and call Shrink;
        * otherwise set N_min ← O·M and call Expand.
    * Returns the smallest feasible constellation stored.
* **Algorithm 2 (feasibility checker)**, run for every time slot:
    * Computes the available capacity from the active links.
    * For each demand, builds the delay-constrained layered graph (GraphTransform, L = λ·L^SP) and runs max-flow to count edge-disjoint paths within the hop limit.
    * The candidate is infeasible if fewer than r_src,dst paths exist or if the demand exceeds the remaining uplink/downlink capacity at the source or destination cell.
    * Demands are processed greedily, one after another, so the order can matter.
* **Algorithm 3 (next candidate)**, binary search:
    * Expand targets ⌊(cur + N_max)/2⌋, adding orbits if O ≤ M and satellites per orbit otherwise.
    * Shrink targets ⌊(cur − N_min)/2⌋ as printed (presumably a typo for the midpoint (cur + N_min)/2), removing satellites per orbit if O ≤ M and orbits otherwise.
    * Both rules push O and M toward each other, since the maximum hop count ⌈(O + M)/2⌉ is smallest when O = M.

## Evaluation Setup
* Tools: an extended StarPerf, Gurobi and SkyField. Constellation parameters come from FCC filings; ground stations from starlink.sx.
* Links: ISL 20 Gbps, GSL 4 Gbps shared, N_ISL = 4 (following ICARUS). One regression period is simulated.
* Traffic: Starlink availability map plus a population model.
* Initial constellations: Starlink phase 1 (4408 satellites, 5 shells, 540–570 km) and Kuiper (3236).
* Sweeps: r_min 2–6; per-cell capacity 20–40 Mbps; λ ∈ {1.35, 1.5, 2, 2.5, 3}; altitude; inclination.
* Resilience baseline: UltraDense (Deng et al., TWC 2021) at 680 / 830 / 940 satellites. Failures are a solar storm (0–20 satellites destroyed) and random loss (0–100%); the metric is reachability.

## Main Results
* At r_min = 6, MegaReduce needs **20.05% fewer** satellites than Starlink and **21.88% fewer** than Kuiper.
* The required count rises with r_min, with per-cell capacity and with smaller λ (tighter delay). It falls with altitude (at the cost of more delay) and rises with inclination (most cells lie within ±70°).
* At the same satellite count, reachability is higher than UltraDense (shown in figures only).
* Case studies use the mapping req = F⁻¹(N_sat): 1500 satellites → r_min 5, 1600 → 6, so 1550 → 5.
    * Starlink's r_min went from 0 to 5 between Dec 2019 and Jun 2023. With the same launch counts, MegaReduce reaches higher survivability earlier.
    * Decay: Starlink loses about 2.6% of its satellites per year (Space-Track, 3 years). Projections at 3% and 5% show that in-orbit adjustment keeps survivability higher.

## Limitations
* ISL capacity is ignored and each cell gets a single beam.
* Only uniform Walker shells are searched, by tuning O and M.
* The binary search is a heuristic with no optimality or complexity proof, despite the "polynomial time, near-optimal" claim.
* Simulation only. Instant switching to backup paths is assumed.
* The resilience and case-study gains appear only in figures.
* N_min, I_limit and the chosen O/M are not reported. The Fig. 8 legend has the typo "MegaRudce".

## 與其他筆記的關聯
* 與 [[Stable_Hierarchical_Routing_for_Operational_LEO_Networks]]（SHORT）同為清華 Hewu Li、Yuanjie Li、Zeqi Lai 團隊的工作：本文決定「要幾顆衛星」，SHORT 則在既有星座上做穩定路由。
* 本文假設 +Grid、每顆 4 條 ISL，且 ISL 不是瓶頸。拓樸層的改進（[[Minimum-hop Constellation Design for LEO Satellite Networks]]、[[LAMP_Low-Latency_Dynamic_Topology_for_LEO_Satellite_Constellations]]、[[Time-_Dependent_Network_Topology_Optimization_for_LEO_Satellite_Constellations]]）能縮短 hop 數，理論上可放寬 λ 限制，讓所需衛星數更少。
* 「edge-disjoint paths」是本文的生存性定義，可作為 survivable LEO network 研究的基準指標。
