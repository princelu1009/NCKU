---
title: "StableRoute: When Dijkstra's Algorithm Meets Topology-Varying Satellite Networks"
authors: [Tian Pan, Guohao Ruan, Qiang Fu, Zhengjie Luo, Junkai Huang, Xingshuang Luo, Tao Huang]
venue: IEEE INFOCOM 2025
doi: 10.1109/INFOCOM55648.2025.11044485
tags: [LEO, satellite-routing, ISL, Dijkstra, route-stability]
---

## Introduction
* In LEO networks, inter-orbit ISLs go down as satellites approach orbit intersection points and come back up afterwards, while intra-orbit ISLs stay up. Routes are therefore recomputed constantly.
* Most routing protocols use Dijkstra, which is stateless: when several shortest paths tie, it keeps the first one it finds. Many next-hop changes are therefore unnecessary, and each one reorders packets, invalidates the TCP congestion window and triggers retransmissions.
* StableRoute makes Dijkstra remember its previous choice and keep it whenever possible, in three variants:
    * **SR_L** keeps the current next hop as long as it is still on a shortest path.
    * **SR_K** also keeps it if the current path is at most k hops longer than the shortest.
    * **SR_G** uses the predictable orbits to plan a whole cycle of next-hop choices with the fewest changes.
* In emulation, all three cut route updates by roughly 43–46% on a 40×40 constellation compared with plain Dijkstra.

## System Model
* Each satellite has 4 ISLs: 2 intra-orbit and 2 inter-orbit. All ISL weights are 1, so path cost = hop count.
* A route update happens for one of three reasons:
    * **S1**: an ISL on the current path goes down (unavoidable).
    * **S2**: Dijkstra's default tie-breaking picks a different equal-cost path (avoidable, targeted by SR_L / SR_G).
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
* Constellations: mostly N×N Walker Star (4×4 to 40×40, OneWeb-like, 87.9°), plus one 20×20 Walker Delta (Starlink-like, 53°).
* Baseline: plain Dijkstra only. OPSPF and similar schemes are not compared, since they also use Dijkstra internally. SR_K uses k ≤ 2.

## Main Results
* **Causes of updates** (Walker Star): S1 28.24–52.93% (avg 42.69%); S2 20.27–50.85% (avg 31.86%); S3 with a 1-hop-shorter path 14.63–21.08% (avg 18.03%); S3 with a 2-hop-shorter path 4.12–11.65% (avg 7.41%). Paths more than 2 hops shorter are rare, so k ≥ 3 is unnecessary.
* **Update reduction vs Dijkstra**, all events:

| Constellation | SR_L | SR_K | SR_G |
|---------------|------|------|------|
| 4×4 Walker Star | 9.86% | 14.25% | 11.60% |
| 40×40 Walker Star | 43.40% | 44.97% | 45.86% |
| 20×20 Walker Delta | 34.24% | 40.30% | 46.93% |

* **ISL-down events only**: SR_L and SR_G clearly beat SR_K, and SR_G is slightly better than SR_L. The gain is not always positive, because avoiding an update at link-down can cost one at link-up.
* **ISL-up events only**: SR_K reduces updates by well over 80%; SR_L / SR_G by 38.57–65.66%.
* SR_L ≈ SR_G in Walker Star, but SR_G pulls clearly ahead in Walker Delta.
* **TCP case studies** (single flow): Dijkstra's switches cause 1240 retransmissions when moving to a congested path, 2271 when moving to a higher-bandwidth path (reordering), and 457 when moving to a 1-hop-shorter path. SR_L / SR_G avoid the first two; SR_K avoids the third.

## Limitations
* Stated by the authors:
    * SR_L and SR_K are only locally optimal: avoiding an update now can cause more later.
    * SR_K trades path length for stability and does worse on link-down events.
    * SR_G needs predictable snapshots; for arbitrary changes it must be combined with SR_L.
    * The design stays within Dijkstra-based routing.
* My additional concerns:
    * There is no dedicated limitations section.
    * The evaluation counts route updates only, with no end-to-end latency or throughput.
    * The TCP results are single-flow examples rather than workloads.
    * With all weights set to 1, the many hop-count ties make S2 look larger than it would be under latency weights.

## 與其他筆記的關聯
* 與 [[Stable_Hierarchical_Routing_for_Operational_LEO_Networks]]（SHORT）目標相同，都是要減少路由更新，但做法不同：SHORT 用座標不變量取代 Dijkstra；StableRoute 保留 Dijkstra，只改 tie-breaking 和「容忍稍長路徑」。
* [[Time-_Dependent_Network_Topology_Optimization_for_LEO_Satellite_Constellations]]（DoTD）在拓樸層減少換線，StableRoute 在路由層減少換路，兩者可以疊加。
* [[Minimum-hop Constellation Design for LEO Satellite Networks]] 的 slanted grid 會改變 equal-cost path 的數量，進而影響 S2 的比例。
