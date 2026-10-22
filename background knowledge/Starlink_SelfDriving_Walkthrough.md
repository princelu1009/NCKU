# A Networking Perspective on Starlink's Self-Driving LEO Mega-Constellation — Walkthrough

**Authors:** Yuanjie Li, Hewu Li, et al. (Tsinghua University, Zhongguancun Laboratory)
**Venue:** ACM MobiCom 2023, Madrid · DOI: [10.1145/3570361.3592519](https://doi.org/10.1145/3570361.3592519)
**Type:** Measurement study (public SSA data) + what-if network simulation

---

## TL;DR

Starlink satellites run a proprietary onboard "autonomous driving" system that maneuvers them automatically. The authors reverse-engineer its behavior from public orbit data (TLEs) and conjunction reports, and ask how those maneuvers affect a future ISL-based Starlink network. The answer is mixed: continuous orbit maintenance **helps** networking a lot, but collision-avoidance and inter-shell maneuvers are **networking-agnostic** and cause unnecessary ISL breaks, longer delays and topology churn. They propose networking/maneuver co-design, e.g. a *reactive* ISL update policy (6.85× fewer topology updates, 1.69× fewer link failures, about 0.34 ms extra delay).

---

## 1. Problem

- LEO is crowded (about 27k tracked debris pieces; 3,500–60,000 conjunctions/month). Satellites must keep correcting orbits and dodging objects.
- Classical maneuver planning is manual and ad hoc. Starlink instead uses onboard self-driving: the satellite reports GPS trajectory → Starlink + US 18th SPCS compute conjunctions → the satellite schedules its own maneuver (trigger: collision probability Pc ≥ 1e-5, 10× stricter than NASA's 1e-4).
- Nobody outside SpaceX knows if this system is good for the **constellation** and the **network** (not just each satellite's safety).

Three questions: (Q1) per-satellite safety, (Q2) constellation-level distribution, (Q3) network stability/performance.

Five "myths" they set out to test (all turn out false):

| Myth | Verdict |
|---|---|
| M1: LEO orbits are naturally stable and predictable | ✗ Drag causes chaotic, *heterogeneous* decay |
| M2: Good constellation design avoids intra-Starlink collisions | ✗ 22.8% of high-risk conjunctions are Starlink-on-Starlink |
| M3: All collision-avoidance maneuvers are necessary | ✗ 56.8% happen below Starlink's own threshold |
| M4: Non-launch maneuvers are short and negligible | ✗ Inter-shell maneuvers last 10+ days (max 43.97 days) |
| M5: Maneuvers barely affect ISLs/topology/routing | ✗ Frequent ISL breaks and topology updates |

## 2. Method

**Data**
- 41.2M TLEs, 2019/05–2022/07, from space-track.org (8,487 satellites, 15,061 debris).
- 9.35M conjunction reports, 2022/04–2022/08, from celestrak.org.

**Maneuver detection (3 steps)**
1. **Predict:** propagate TLE_n with SGP4 to get the expected TLE_{n+1} if there were no maneuver.
2. **Detect:** sliding window of 5 TLEs; if the predicted semi-major axis differs from the window average by **> 1 km**, label it a maneuver.
3. **Root cause:** match maneuvers to conjunction reports for the same satellite within a **12 h window** (Starlink does a single in-track burn about 12 h before TCA), plus public incident reports (OneWeb, Aeolus, China's complaint).

**Network impact model**
- +Grid topology: 4 ISLs per satellite (2 intra-orbit, 2 inter-orbit), neighbors inferred by grouping on inclination, semi-major axis and RAAN, then nearest distance.
- Compare real trajectories vs SGP4 "no-maintenance" counterfactuals; shortest-path routing over 1,000 random flows with 10 Gbps ISLs.
- Max ISL visibility distance ≈ 4,782 km; laser ISL fails if point-ahead angle > ½ beam divergence (70 µrad assumed).

## 3. Key results

### §5 Continuous orbit maintenance (good for networking)
- Starlink continuously reboosts; real orbits track an ideal Kepler orbit. Only 6% of other satellites at 550 ± 30 km do regular maintenance (Iridium and OneWeb also maintain more often).
- Without maintenance, decay is heterogeneous, so nearest-neighbor changes grow from tens/day (2022-04) to hundreds/day (2022-07).
- With maintenance, about 90% of ISLs stay connected over 3 months. Without it, most ISLs eventually break, and survivors gain about 2 ms (intra) / 3 ms (inter) delay.
- Maintenance gives **1.2× per-flow throughput** and better load balance.

### §6 Collision avoidance (safe but over-cautious)
- Three cases: Starlink-on-debris, on 3rd-party satellite (maneuverable like OneWeb, or not like Cosmos), and Starlink-on-Starlink.
- Maneuvers raise orbit via burns in R and T directions. Median duration about 120 h, median extent about 5 km (max 21 km).
- Effective: miss distance up by about **28.4 km** on average, Pc down 1–3 orders of magnitude.
- **Cooperative avoidance:** in Starlink-on-Starlink events, the satellite with more empty space around it moves, in more than 98% of cases. 90% of these conjunctions are under 2.5 min long.
- **Unnecessary:** 56.8% of maneuvers had Pc averaging 2.77e-6 (below 1e-5) yet moved about 2.8 km. Likely cause: conservative covariance estimates.
- **High-risk:** some maneuvers create new near-misses (e.g., STARLINK-2180 came within 3 km of another Starlink after dodging a 3rd-party satellite).
- These can push the laser point-ahead angle past ½ beam divergence, breaking ISLs.

### §7 Inter-shell maneuvers (worst for networking)
- Shells (2022-07): 83.57% at 550 km/53°, 14.29% at 540 km/53.2°, 2.14% at 570 km/70°.
- 81 satellites from 46 orbits did protracted shell migrations in 2022/04–07. Over 99.95% lasted more than 10 days, max 43.97 days, mostly in May–June 2022.
- Higher orbit means slower angular velocity, so neighbor distance grows steadily until it exceeds the visibility limit, and the ISL breaks.
- Studied 1,441 satellites in the 550 km shell under {4-ISL, n-ISL} × {proactive, reactive} rewiring:
  - Proactive rewiring touches about 90% of satellites; reactive cuts that to about 50%.
  - **4-ISL:** reactive beats proactive with **6.85× fewer topology updates** and **1.69× fewer failures**, at +0.34 ms average delay.
  - **n-ISL:** proactive gives lowest delay/failures at 7.66× more topology updates (argued acceptable with SDN).
- Higher layers suffer: more SDN recomputation, or OSPF/BGP reconvergence windows with no delivery.

## 4. Suggestions (§8)

- **Maneuver-aware networking:** keep the strict 1e-5 threshold, but have the network know about upcoming maneuvers. Use reactive ISL updates for today's 4-ISL satellites and proactive ones if more ISL terminals arrive. Predict ISL churn and prefetch; jointly schedule link realignment with maneuvers via ground SDN (like Google Loon).
- **Network-aware maneuvering:** cut unnecessary avoidances (better covariance, less over-caution), reschedule launches/deorbits to avoid conjunctions, batch inter-shell moves to reduce topology updates, and avoid the high-risk maneuvers by burning earlier with smaller altitude changes.

## 5. Weak spots / critique

- **Detection floor:** the 1 km semi-major-axis threshold and TLE cadence (3–35 h) miss small or short maneuvers, so counts are lower bounds and durations are coarse. The authors admit this.
- **"Unnecessary" relies on public data.** It is judged from public conjunction reports, but Starlink's internal covariance and ephemeris are more precise. Some "unnecessary" maneuvers may have been justified by data the authors can't see (or were for other missions altogether).
- **Root cause is correlation only.** A maneuver with no matching conjunction is ambiguous. Inter-shell moves are inferred, not confirmed by SpaceX.
- **Hypothetical network.** Starlink ran bent-pipe in 2022. The ISL topology (+Grid, 4 ISLs, nearest-neighbor) is assumed, not measured, and real Starlink laser topology is unknown. Network impacts are therefore what-if.
- **Simplified link model.** ISL failure uses a distance limit and a generic 70 µrad beam divergence. There is no acquisition/tracking time, no pointing-mechanism limits and no real laser terminal specs.
- **Light traffic model:** 1,000 random satellite-to-satellite flows with shortest path. No ground-station or user demand, no congestion control.
- **Short windows:** conjunction data covers only 4 months, and the network analysis covers 2022/04–07, a period with unusual shell migrations (May–June). Generality to today's much larger constellation is unclear.
- **Co-design suggestions are mostly qualitative.** Only the reactive/proactive ISL policy is quantified. "Reduce unnecessary maneuvers" has no proposed safe threshold or risk analysis.
- Minor issues: "M4" is mislabeled in §6.2 (should be M2), and "M5" in §6.3 (should be M3).

## 6. Takeaways for my research

- Real LEO topology is **not** the ideal Walker shell that most routing/simulation papers assume. Maneuvers inject drift, link breaks and churn.
- A useful pattern: **SGP4 counterfactual vs real TLE** to isolate the effect of a behavior.
- Open idea: maneuver-aware routing/topology control that knows the maneuver schedule ahead of time.
- Dataset + code: released by the authors (ref [10] in the paper).

## Related notes

- [[DoTD_Walkthrough]]
