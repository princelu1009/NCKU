---
title: Minimum-hop Constellation Design for LEO Satellite Networks
tags: [LEO, ISL, topology-design, ASPL]
---

## Introduction
* LEO satellites connect to each other with laser ISLs. Each satellite carries only 3 or 4 terminals, but because the lasers can be re-pointed, the operator chooses which satellites each one links to. This paper asks how to make that choice so that a packet crosses as few satellites as possible on average, measured by the average shortest path length (ASPL), since fewer hops means lower latency.
* Method: first derive the best ASPL any topology could achieve, by bounding how many satellites can be reached at each hop distance and assuming every layer is completely filled; then look for wirings that reach that bound.
* Symmetric case (every satellite uses the same connection pattern): the bound grows like √N. The common mesh grid (+Grid) falls short because its straight links wrap around the constellation and land on satellites already reached. The fix keeps the in-orbit links and slants the cross-orbit links by an offset of about √(2n_s/n_o) − 1, which meets the bound exactly for certain constellation shapes and comes very close for others.
* General case (satellites may use different patterns): the bound improves to log N. The exact optimum is hard to find, but random wiring refined by simulated annealing gets close, even when links are restricted to nearby satellites, as long as the constellation is dense enough.

## System Model
* Notation: n_o orbits with n_s satellites each, N = n_s · n_o satellites in total; each satellite has k ISLs (k = 3 or 4); ℓ is hop distance and D the diameter.
* Bound technique: if at most f(ℓ) satellites can sit at exactly ℓ hops, then N ≤ 1 + Σ_{ℓ=1}^{D} f(ℓ), and ASPL is minimized when every layer is full. Each case below differs only in f(ℓ).

| Case | f(ℓ) | ASPL lower bound | Construction that meets it |
|------|------|------------------|----------------------------|
| Symmetric, k = 4 | 4ℓ | ≈ 0.47√N | Slanted grid, offset √(2n_s/n_o) − 1 |
| Symmetric, k = 3 | 3ℓ | ≈ 0.54√N | Exact only for n_s/n_o = 6; offset search otherwise |
| Non-symmetric, k = 4 | 4·3^{ℓ−1} | Θ(log N) | Simulated annealing; PRGS under a range limit |

## Results by Case
* Symmetric, 4 links per satellite

    With one shared pattern, at most 4ℓ satellites are reachable at exactly ℓ hops, so N ≤ 1 + 2D(D + 1) and the best possible ASPL is about (2/3)√(N/2) ≈ 0.47√N. The mesh grid misses this once the constellation is moderately large (n_s + n_o ≥ 10): its wrap-around links revisit satellites already reached, leaving layers unfilled and the diameter larger than necessary. The bound is met exactly in two cases:
    1. n_s and n_o co-prime: relabel the grid as a single ring of N satellites and use a known optimal ring (circulant) wiring.
    2. n_s/n_o = m²/2: keep the in-orbit links and slant the cross-orbit links by an offset of √(2n_s/n_o) − 1 = m − 1.

    The slanted grid is the practical choice: it keeps the stable in-orbit links and needs no longer links than the mesh grid. For other shapes, rounding the same offset stays close to the bound in simulation, and the authors conjecture this always holds.

* Symmetric, 3 links per satellite

    Three links save one terminal per satellite, but a single shared pattern produces links in ± pairs, so the degree is naturally even. The authors therefore redefine symmetry: satellites are split into two checkerboard groups that use mirrored patterns. Under this definition at most 3ℓ satellites are reachable at ℓ hops, giving a best possible ASPL of about 0.54√N, roughly 15% more hops than with 4 links (0.54 / 0.47 ≈ 1.15). The standard honeycomb wiring misses this bound when n_s + n_o ≥ 16, for the same wrap-around reason as the mesh grid.

    There is one exact construction, for n_s/n_o = 6, using jumps of 1 and 3 positions within the orbit and 5 positions into the next orbit. For other shapes the authors keep the in-orbit links and search for the best cross-orbit offset, which gets close to the bound in simulation. This case rests more on simulation than proof: the exact construction covers a single shape, its proof is omitted, and it changes the in-orbit links.

* Non-symmetric, any 4-regular wiring

    If satellites may use different patterns, each hop can reach three times as many new satellites as the last (4, 12, 36, 108, …), the Moore bound, so the best possible ASPL drops to the order of log N, far below the √N of the symmetric cases. The difficulty is that graphs meeting this bound are not guaranteed to exist and are hard to find, and random graphs that come close use links spanning the whole constellation, which lasers cannot do.

    The authors start from a random 4-regular graph, refine it with simulated annealing, and show it tracks the bound. To respect link range they restrict the same search to nearby pairs, which still tracks the bound, and they introduce the cell-based PRGS procedure to prove that log N scaling is achievable under a range limit when the constellation is dense enough. The cost is that the resulting topology has no regular structure, which makes routing and management harder, and the guarantee only applies to dense constellations.

## 與其他筆記的關聯
* 本文只看 ASPL（hop 數），不考慮容量與鏈路變動；[[Time-_Dependent_Network_Topology_Optimization_for_LEO_Satellite_Constellations]]（DoTD）則把延遲、容量和換線次數一起納入，可作為補充。
* Slanted grid 保留 intra-orbit ISL、只改 inter-orbit 的偏移量，仍保有 [[Stable_Hierarchical_Routing_for_Operational_LEO_Networks]]（SHORT）所依賴的 $(\Delta\alpha,\Delta\gamma)$ 規則結構；非對稱的 random 拓樸則會失去這個結構。
