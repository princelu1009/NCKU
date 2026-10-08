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
### 2.1 符號表
| 符號 | 意義 | 例子（台北 → LA 故事） |
|---|---|---|
| $S=\{s_1,\dots,s_P\}$ | 所有衛星，共 P 顆 | Starlink 第一 shell，P = 1584 |
| $U=\{u_1,\dots,u_Q\}$ | 所有地面站，共 Q 個 | 台北站、LA 站…… |
| $V=S\cup U$ | 網路裡所有節點 | |
| $(i,j)$ | 一條鏈路（ISL 或 GSL），雙向 | 台北–A（GSL）、A–B（ISL） |
| $L$、$L_t$ | 所有可用鏈路；$L_t$ 是時槽 t 可用的鏈路 | |
| $T=\{t_1,t_2,\dots\}$ | 時間切成的時槽 | |
| $\pi^k_t: L_t\to L_{t+1}$ | 第 k 個故障事件：把時槽 t 的可用鏈路集合變成 t+1 的 | t = 30 s 換手、t = 45 s M 被毀 |
| $G_t=(V,L_t)$ | 時槽 t 的網路圖；$G^{all}=\{G_t\}$ 是所有時槽的圖 | |
| $f_{ab}=\{st_{f_{ab}}, et_{f_{ab}}, d_{f_{ab}}\}$ | 從 a 到 b 的流量需求：開始時槽、結束時槽、頻寬需求 | 台北 → LA 影片，0–90 s，需要 d |
| $F$ | 所有流量需求的集合（流量矩陣） | |
| $b_{ij}$ | 鏈路 (i,j) 在 i→j 方向的容量 | |
| $l_{ij}$ | 鏈路 (i,j) 的延遲 | |
| $\gamma_{ab}(i,j,t)\in\{0,1\}$ | **決策變數**：流量 $f_{ab}$ 在時槽 t 是否經過鏈路 (i,j) | |
| $r^t_{ab}=\{\gamma_{ab}(i,j,t)\}$ | $f_{ab}$ 在時槽 t 的路徑 | |
| $R_t$ | 時槽 t 所有流量的路徑集合（routing scheme） | |
| $D^{sp}_{ab}$ | a 到 b 最短路徑的延遲 | |
| $\alpha\ge 1$ | 允許比最短路徑慢多少倍 | α = 1.5 |
| $D_{ab}=\alpha D^{sp}_{ab}$ | $f_{ab}$ 的延遲上限 | |
| MLU | 所有鏈路中最高的使用率（maximum link utilization） | |

### 2.2 限制式與目標
- **(1) Flow conservation**：對每個節點 v，
  $\sum_w \gamma_{ab}(v,w,t) - \sum_w \gamma_{ab}(w,v,t) = 1$（v = a）、$-1$（v = b）、0（其他）。
  意思：從 a 出去一份流量、在 b 收到一份，中間節點進多少就出多少，所以 a 到 b 一定有一條連通的路。
- **(2) Capacity**：$\sum_{a,b}\gamma_{ab}(i,j,t)\cdot d_{f_{ab}} \le b_{ij}$。
  意思：所有經過 (i,j) 的流量（$\gamma=1$ 的那些 $d$ 相加），不能超過這條鏈路的容量 $b_{ij}$。
- **(3) MLU 定義**：$\text{MLU}=\max_{t,(i,j)} \dfrac{\sum_{a,b} d_{f_{ab}}\,\gamma_{ab}(i,j,t)}{b_{ij}}$。
  意思：每條鏈路「實際負載 ÷ 容量」= 使用率，MLU 是其中最大的那個。
- **(5) Latency**：$\sum_{(i,j)\in L_t}\gamma_{ab}(i,j,t)\cdot l_{ij} \le D_{ab}$。
  意思：$f_{ab}$ 走過的每條鏈路延遲 $l_{ij}$ 加總（= 路徑延遲），不能超過上限 $D_{ab}=\alpha D^{sp}_{ab}$。
- **(6) MLU 上限**：每條鏈路的負載 $\sum_{a,b}\gamma_{ab}(i,j,t)\cdot d_{f_{ab}} \le \text{MLU}\cdot b_{ij}$（把 (3) 寫成限制式）。
- **(4) 目標**：$\min \text{MLU}$，讓最忙的鏈路盡量閒，留餘裕吸收突發流量和故障。
- **困難所在**：$\gamma$ 是 0/1，所以這是 ILP；而每個故障事件 $\pi^k_t$ 都會產生新的 $G_t$，必須在新圖上**快速**重解一次。

## 3. Method
### 3.1 TSM（Topology-Stabilizing Model）：把拓樸變化轉成流量變化
TSM 由兩個 surjection（多對一映射）組成：
- **Graph surjection** $\phi: G^T \to \hat G$：把隨時間變化的圖序列 $G^T=\{G_t\}$ 對應到**一張固定的邏輯圖** $\hat G=(\hat V,\hat L)$。
- **Traffic surjection** $\omega: F \to \hat F$：把原本的流量矩陣 $F$ 轉成定義在 $\hat G$ 上的新流量矩陣 $\hat F$。

#### (a) 建立固定的邏輯圖 $\hat G$（Alg. 1）
| 新符號 | 意義 | 例子 |
|---|---|---|
| $\hat V$ | 邏輯圖的節點：每顆衛星 $s_i$ 的鏡像＋它的虛擬地面節點 $vs_i$，共 2P 個 | A, vs_A, B, vs_B, …, Z, vs_Z |
| $vs_i$ | **virtual terrestrial node**：「此刻連到 $s_i$ 的所有地面站」的集合；沒人連時為 ∅ | 0–30 s 的 vs_A = {台北站} |
| $(s_i, vs_i)$ | 邏輯 GSL，容量 = $s_i$ 的 GSL 容量，**永遠存在** | |
| $\hat L$ | 邏輯鏈路：所有 $(s_i,vs_i)$＋照實體連線複製的 ISL（+Grid，每顆 4 條） | |

步驟：
1. 對每顆衛星 $s\in S$：建立 $vs$，把 $s$ 和 $vs$ 加進 $\hat V$。
2. 建立鏈路 $(s, vs)$，容量設為 $s$ 的 GSL 容量，加進 $\hat L$。
3. 把 $G^T$ 中所有 ISL 複製到 $\hat L$。

為什麼 $\hat G$ 不會變：
- **ISL 部分**：Walker Delta 同一 shell 內的衛星高度、速度相同，相對位置固定，所以 +Grid 的 ISL 不變。
- **GSL 部分**：換手只會改變「$vs_i$ 實際代表哪些地面站」，但邏輯鏈路 $(s_i, vs_i)$ 本身一直存在。

#### (b) 把故障轉成流量（Alg. 2）
| 新符號 | 意義 |
|---|---|
| $\hat F=\omega(F)$ | 轉換後、定義在 $\hat G$ 上的流量矩陣 |
| find$(\hat G, a, t)$ | 在時槽 t，原本的端點 a 對應到 $\hat G$ 的哪個 $vs$（即「a 此刻連在哪顆衛星下」） |
| $f_{sd}$ | 轉換後的一段短流量，從 $vs$ 端點 s 到 d，只持續一個時槽 |
| $[t_s, t_e)$ | 突發故障的開始、結束時槽 |
| $f^{burst}=\{t_s,t_e,b_{ij}\}$ | 代表故障的假流量：在 $[t_s,t_e)$ 期間，需求 = 該鏈路容量 $b_{ij}$ |

步驟：
1. **可預測故障（GSL 換手）**：對每個 $f_{ab}\in F$、它持續期間的每個時槽 t（從 $st_{f_{ab}}$ 到 $et_{f_{ab}}$）：
   - $s$ = find$(\hat G, a, t)$，$d$ = find$(\hat G, b, t)$；
   - 建立 $f_{sd}=\{t, t+1, d_{f_{ab}}\}$ 加進 $\hat F$。
   - 結果：一條長流量被切成**依時間接續的多段短流量**，每段的端點是「當下服務 a、b 的衛星的 $vs$」。
2. **突發故障**：對每個在 $[t_s,t_e)$ 故障的鏈路 (i,j)：
   - 建立 $f^{burst}=\{t_s,t_e,b_{ij}\}$ 加進 $\hat F$，並把它的延遲上限設為 $l_{ij}$，強迫它只能走 (i,j)。
   - 結果：(i,j) 被假流量**塞滿**，在 (2) 容量限制下，其他流量自然會避開。

#### (c) 論文 Fig. 3 的小例子
3 顆衛星 S1、S2、S3，地面站 GS1、GS2；需求 $f_{GS1,GS2}=\{t_1,t_3,D\}$。
- 時槽 1 $[t_1,t_2)$：GS1–S1、GS2–S2。
- 時槽 2 $[t_2,t_3)$：換手成 GS1–S2、GS2–S3；同時 ISL S1–S2 突然故障。
- TSM 轉換後：
  - $f_{v1,v2}=\{t_1,t_2,D\}$、$f_{v2,v3}=\{t_2,t_3,D\}$（可預測換手 → 兩段短流量）；
  - $f_{S1,S2}=\{t_2,t_3,\text{Cap}(S1,S2)\}$，其中 $\text{Cap}(S1,S2)=b_{S1,S2}$（突發故障 → burst flow）。
- $\hat G$ 從頭到尾不變，只有 $\hat F$ 在變。

**圖解：P1 vs P2（G 與 F 的比較）**

<svg viewBox="0 0 720 600" xmlns="http://www.w3.org/2000/svg" font-family="system-ui, sans-serif" style="max-width:100%"><text x="20" y="28" text-anchor="start" font-size="15" fill="currentColor" font-weight="600">P1：圖 G 會變，流量 F 很單純</text><rect x="15" y="40" width="200" height="200" rx="10" fill="none" stroke="#888" stroke-opacity="0.35"/><text x="25" y="60" text-anchor="start" font-size="12" fill="currentColor" font-weight="400">時槽 1（t1–t2）的 G</text><line x1="55" y1="100" x2="115" y2="100" stroke="#888" stroke-width="2"/><line x1="115" y1="100" x2="175" y2="100" stroke="#888" stroke-width="2"/><line x1="55" y1="100" x2="55" y2="190" stroke="#888" stroke-width="2"/><line x1="115" y1="100" x2="115" y2="190" stroke="#888" stroke-width="2"/><circle cx="55" cy="100" r="16" fill="#5b8def"/><text x="55" y="104" text-anchor="middle" fill="#fff" font-size="11" font-weight="600">S1</text><circle cx="115" cy="100" r="16" fill="#5b8def"/><text x="115" y="104" text-anchor="middle" fill="#fff" font-size="11" font-weight="600">S2</text><circle cx="175" cy="100" r="16" fill="#5b8def"/><text x="175" y="104" text-anchor="middle" fill="#fff" font-size="11" font-weight="600">S3</text><rect x="37" y="190" width="36" height="22" rx="4" fill="#3fae7a"/><text x="55" y="205" text-anchor="middle" fill="#fff" font-size="11" font-weight="600">GS1</text><rect x="97" y="190" width="36" height="22" rx="4" fill="#3fae7a"/><text x="115" y="205" text-anchor="middle" fill="#fff" font-size="11" font-weight="600">GS2</text><rect x="230" y="40" width="200" height="200" rx="10" fill="none" stroke="#888" stroke-opacity="0.35"/><text x="240" y="60" text-anchor="start" font-size="12" fill="currentColor" font-weight="400">時槽 2（t2–t3）的 G</text><line x1="270" y1="100" x2="330" y2="100" stroke="#e5484d" stroke-width="2.5" stroke-dasharray="6 4"/><text x="300" y="90" text-anchor="middle" font-size="13" fill="#e5484d" font-weight="700">✕</text><line x1="330" y1="100" x2="390" y2="100" stroke="#888" stroke-width="2"/><line x1="330" y1="100" x2="330" y2="190" stroke="#888" stroke-width="2"/><line x1="390" y1="100" x2="390" y2="190" stroke="#888" stroke-width="2"/><circle cx="270" cy="100" r="16" fill="#5b8def"/><text x="270" y="104" text-anchor="middle" fill="#fff" font-size="11" font-weight="600">S1</text><circle cx="330" cy="100" r="16" fill="#5b8def"/><text x="330" y="104" text-anchor="middle" fill="#fff" font-size="11" font-weight="600">S2</text><circle cx="390" cy="100" r="16" fill="#5b8def"/><text x="390" y="104" text-anchor="middle" fill="#fff" font-size="11" font-weight="600">S3</text><rect x="312" y="190" width="36" height="22" rx="4" fill="#3fae7a"/><text x="330" y="205" text-anchor="middle" fill="#fff" font-size="11" font-weight="600">GS1</text><rect x="372" y="190" width="36" height="22" rx="4" fill="#3fae7a"/><text x="390" y="205" text-anchor="middle" fill="#fff" font-size="11" font-weight="600">GS2</text><text x="330" y="232" text-anchor="middle" font-size="12" fill="currentColor" font-weight="400">換手＋S1–S2 斷線</text><rect x="445" y="40" width="260" height="200" rx="10" fill="none" stroke="#888" stroke-opacity="0.35"/><text x="455" y="60" text-anchor="start" font-size="12" fill="currentColor" font-weight="400">流量 F（一筆需求）</text><line x1="475" y1="170" x2="685" y2="170" stroke="#888" stroke-width="2"/><text x="475" y="188" text-anchor="middle" font-size="12" fill="currentColor" font-weight="400">t1</text><text x="580" y="188" text-anchor="middle" font-size="12" fill="currentColor" font-weight="400">t2</text><text x="685" y="188" text-anchor="middle" font-size="12" fill="currentColor" font-weight="400">t3</text><rect x="475" y="115" width="210" height="28" rx="5" fill="#5b8def"/><text x="580.0" y="133" text-anchor="middle" font-size="11" fill="#fff" font-weight="600">GS1 → GS2，頻寬 D</text><text x="575" y="222" text-anchor="middle" font-size="12" fill="currentColor" font-weight="400">只有 1 列，但要在 2 張不同的圖上解</text><text x="20" y="300" text-anchor="start" font-size="15" fill="currentColor" font-weight="600">P2：圖 Ĝ 固定，所有變化都搬進流量 F̂</text><rect x="15" y="312" width="300" height="270" rx="10" fill="none" stroke="#888" stroke-opacity="0.35"/><text x="25" y="332" text-anchor="start" font-size="12" fill="currentColor" font-weight="400">Ĝ：每個時槽都一樣</text><line x1="75" y1="380" x2="165" y2="380" stroke="#888" stroke-width="2"/><line x1="165" y1="380" x2="255" y2="380" stroke="#888" stroke-width="2"/><line x1="75" y1="380" x2="75" y2="480" stroke="#888" stroke-width="2"/><line x1="165" y1="380" x2="165" y2="480" stroke="#888" stroke-width="2"/><line x1="255" y1="380" x2="255" y2="480" stroke="#888" stroke-width="2"/><circle cx="75" cy="380" r="16" fill="#5b8def"/><text x="75" y="384" text-anchor="middle" fill="#fff" font-size="11" font-weight="600">S1</text><circle cx="165" cy="380" r="16" fill="#5b8def"/><text x="165" y="384" text-anchor="middle" fill="#fff" font-size="11" font-weight="600">S2</text><circle cx="255" cy="380" r="16" fill="#5b8def"/><text x="255" y="384" text-anchor="middle" fill="#fff" font-size="11" font-weight="600">S3</text><circle cx="75" cy="490" r="15" fill="#3fae7a"/><text x="75" y="494" text-anchor="middle" fill="#fff" font-size="11" font-weight="600">v1</text><circle cx="165" cy="490" r="15" fill="#3fae7a"/><text x="165" y="494" text-anchor="middle" fill="#fff" font-size="11" font-weight="600">v2</text><circle cx="255" cy="490" r="15" fill="#3fae7a"/><text x="255" y="494" text-anchor="middle" fill="#fff" font-size="11" font-weight="600">v3</text><text x="165" y="535" text-anchor="middle" font-size="12" fill="currentColor" font-weight="400">每顆衛星都有自己的虛擬地面節點 v</text><text x="165" y="553" text-anchor="middle" font-size="12" fill="currentColor" font-weight="400">S1–S2 仍畫在圖上：什麼都沒刪</text><rect x="330" y="312" width="375" height="270" rx="10" fill="none" stroke="#888" stroke-opacity="0.35"/><text x="340" y="332" text-anchor="start" font-size="12" fill="currentColor" font-weight="400">流量 F̂（3 列）</text><line x1="400" y1="510" x2="680" y2="510" stroke="#888" stroke-width="2"/><text x="400" y="528" text-anchor="middle" font-size="12" fill="currentColor" font-weight="400">t1</text><text x="540" y="528" text-anchor="middle" font-size="12" fill="currentColor" font-weight="400">t2</text><text x="680" y="528" text-anchor="middle" font-size="12" fill="currentColor" font-weight="400">t3</text><line x1="540" y1="350" x2="540" y2="510" stroke="#888" stroke-dasharray="3 3" stroke-opacity="0.5"/><rect x="400" y="360" width="140" height="28" rx="5" fill="#5b8def"/><text x="470.0" y="378" text-anchor="middle" font-size="11" fill="#fff" font-weight="600">v1 → v2，D</text><rect x="540" y="405" width="140" height="28" rx="5" fill="#5b8def"/><text x="610.0" y="423" text-anchor="middle" font-size="11" fill="#fff" font-weight="600">v2 → v3，D</text><rect x="540" y="450" width="140" height="28" rx="5" fill="#e5484d"/><text x="610.0" y="468" text-anchor="middle" font-size="11" fill="#fff" font-weight="600">S1 → S2，Cap(S1,S2)</text><text x="340" y="378" text-anchor="start" font-size="12" fill="currentColor" font-weight="400">換手</text><text x="340" y="423" text-anchor="start" font-size="12" fill="currentColor" font-weight="400">換手</text><text x="340" y="468" text-anchor="start" font-size="12" fill="#e5484d" font-weight="400">突發</text><text x="517" y="558" text-anchor="middle" font-size="12" fill="currentColor" font-weight="400">藍 = 真實需求按時槽切段，紅 = 塞滿斷線鏈路的假流量</text></svg>

- **上半（P1）**：流量只有**一列**（GS1 → GS2，整段時間），但時槽 1 和時槽 2 的**圖不一樣**：地面站換到別的衛星底下，S1–S2 也斷了。等於要在兩張不同的地圖上各解一次。
- **下半（P2）**：**一張地圖** $\hat G$ 用到底（S1–S3＋v1–v3，S1–S2 仍在圖上）；改成**流量表變成三列**：兩段藍色換手流量在 t2 交棒，加上一段紅色假流量在故障期間塞滿 S1–S2。
- 一句話：P1 =「流量少、地圖多」；P2 =「地圖一張、流量多」。

#### (d) 轉換後的問題（P2）
給定固定的 $\hat G$ 和隨時間變化的 $\hat F$，求一組動態路由 $R_T$，滿足 (1)(2)(5)(6)，並最小化 MLU。不必再對無數種拓樸各算一次。

剩下兩個實務問題：
1. $\hat F$ 變得非常大（每條流量被切成很多段），直接用 LP／ILP 求解很慢。
2. $\hat F$ 裡的 burst flow 來自突發故障，無法事先知道。

→ 3.2 的 hybrid routing 就是為了解決這兩點。

### 3.2 Adaptive Hybrid Routing
把 $\hat F$ 拆成兩部分：$\hat F = \hat F_{pr} + \hat F_{up}$。
- $\hat F_{pr}$（predictable）：換手造成的短流量，**事先知道** → 交給 basic routing。
- $\hat F_{up}$（unexpected）：突發故障的 burst flow，**事先不知道** → 先交給 protection routing 應急，再讓 basic routing 重算。

#### (a) Basic routing（處理 $\hat F_{pr}$：離線計算、上線時按表配置）
用 Gurobi 當 constraint optimizer 解 P2，並用兩個方法縮小問題：

**先備：最短路徑延遲 $D^{sp}_{ab}$ 怎麼來？**（筆記推論：論文只定義 $D^{sp}_{ab}$ 是「a 到 b 最短路徑的延遲」，沒寫怎麼算；以下是標準做法）
- 衛星軌道事先已知，所以每個時槽每顆衛星的位置都知道。
- 每條鏈路的延遲 $l_{ij}$ = (i,j) 距離 ÷ 光速，例如 1,000 km 的 ISL ≈ 3.3 ms。
- 在 $\hat G$ 上把每條鏈路的權重設成 $l_{ij}$，跑 **Dijkstra**（例如 vs_A → vs_Z），得到的最小總延遲就是 $D^{sp}_{ab}$，也就是「整個網路只給這條流量用」時的延遲。
- 例：Dijkstra 算出台北 → LA = 40 ms，α = 1.5 → 上限 $D_{ab}$ = 60 ms。
- 這不會讓問題變難：Dijkstra 在約 3,000 個節點上只要幾毫秒。真正難的是「**所有流量一起**選路、不爆量、MLU 最小」那個 ILP，$D^{sp}$ 只是事先算好、拿來設定每條流量延遲預算的參考值。

**Path filtering（刪掉不可能用到的 $\gamma$ 變數）**
| 符號 | 意義 |
|---|---|
| $p^{ij}_{ab}$ | 估計「$f_{ab}$ 如果經過鏈路 (i,j)」的路徑長度 |
| great-circle$(a,i)$ | a 到衛星 i 的地面投影的大圓距離 |
| $\lvert(i,j)\rvert$ | 鏈路 (i,j) 的長度 |

- $p^{ij}_{ab}$ = great-circle$(a,i)$ + $\lvert(i,j)\rvert$ + great-circle$(j,b)$。
- 若 $p^{ij}_{ab}\ge D_{ab}$（已經超過延遲上限），這條鏈路不可能出現在 $f_{ab}$ 的合法路徑上，就直接把 $\gamma_{ab}(i,j,t)$ 刪掉（固定為 0）。
- 直覺：離 a→b 地面連線太遠的衛星，不會被拿來載這條流量。

例：台北 → LA（粗略數字；延遲和距離成正比，這裡直接用 km）。最短路徑 ≈ 11,000 km，α = 1.5 → 上限 ≈ 16,500 km。

| 鏈路 (i,j) 位於 | 台北 → i | 鏈路 | j → LA | $p^{ij}_{ab}$ | 保留？ |
|---|---|---|---|---|---|
| 太平洋中央 | 5,000 | 1,000 | 5,000 | 11,000 | ✅ |
| 阿拉斯加 | 7,000 | 1,000 | 3,500 | 11,500 | ✅ |
| 澳洲 | 7,300 | 1,000 | 12,000 | 20,300 | ❌ |
| 歐洲 | 9,500 | 1,000 | 9,000 | 19,500 | ❌ |

- 只有台北–LA 航線周圍一個「橢圓」裡的鏈路會留下，全球約 6,000 條鏈路大部分都被刪掉（對這條流量而言）。
- **為什麼安全**：$p^{ij}_{ab}$ 是**樂觀估計**（用地面直線距離，忽略上下 550 km 和沿 +Grid 走的鋸齒），真正經過該鏈路的路徑只會更長。連 p 都超過上限，真實路徑一定也超過，所以不會刪掉任何合法的解。

**Merging demands（減少限制式）**
- 同一時槽、同一組 src–dst 的兩筆需求 $f^1_{ab}$、$f^2_{ab}$ 合併成一筆，頻寬 = $d_{f^1_{ab}}+d_{f^2_{ab}}$。
- **總流量不變**，只是**列數變少**：例如 vs_A → vs_Z 兩列 10 Mbps、5 Mbps → 一列 15 Mbps。
- **為什麼有用**：$\hat F$ 每一列都帶一組 $\gamma$（每條鏈路一個）和一組 flow conservation 限制式（每個節點一條）。約 6,000 條 ISL 時，2 列 ≈ 12,000 個 $\gamma$，1 列 ≈ 6,000 個。台北 → LA 可能有上百個使用者，合併後只剩一列。
- TSM 之後 src、dst 是 **vs 節點**而不是城市：同一時槽台北和高雄若都在 A 底下、都送往 LA（在 Z 底下），兩者都變成 vs_A → vs_Z，也會被合併（筆記推論，由 $vs_i$ 的定義而來）。
- **小代價**：合併後的流量必須**走同一條路徑**，不能再把 10 走一邊、5 走另一邊。同一對端點的流量本來通常就會走同一條路，所以代價不大（筆記推論）。

> **合併 vs 過濾**：合併 → 少**列**（流量數）；過濾 → 每列少 **$\gamma$**（每條流量要考慮的鏈路數）。兩者一起讓 P2 小到 Gurobi 解得動。

#### (b) Protection routing：LGPR（處理 $\hat F_{up}$，Alg. 3）
| 符號 | 意義 |
|---|---|
| $(p,q)$ | 衛星的位置索引：第 p 個 orbit 的第 q 顆；同一 shell 內相對位置固定 |
| $pkt_{dst}$ | 要送往 dst 的封包 |
| protection flag | IPv6 Hop-by-Hop header 中的 1 bit；true = 走 LGPR，false = 走 basic routing |
| in_link | 封包進來的那條鏈路 |
| available_links | 這顆衛星目前可用的鏈路 |

步驟（每收到一個封包）：
1. 把 $pkt_{dst}$ 的 protection flag 設為 true。
2. 若 available_links 扣掉 in_link 之後是空的（只剩進來那條路，dead end）→ 執行 loop avoidance，回傳 NULL（不轉送）。
3. 否則，在相鄰節點中選 distance(n, dst) 最小的 n 當 out_link 轉送。

**Loop avoidance**：只剩一條可用鏈路的衛星（dead end）會通知鄰居，鄰居就暫時不往它送，避免封包在死路上來回。

#### (c) 運作流程
1. 平時：protection flag = false，照 basic routing 預先算好的表轉送；換手時按時間表切換。
2. 突發故障發生：受影響的節點立刻把 flag 設為 true，用 LGPR 在本地繞過。
3. **同時**：把 $f^{burst}$ 加進 $\hat F$，basic routing 重新解 P2。
4. 新的路由算好後：切回 basic routing（flag = false）。

### 3.3 白話版：GPS 導航比喻
1. **假裝地圖永遠不變（TSM）**：每顆衛星配一個「虛擬地面站」，代表「此刻地面上連到我的人」，所以地面站怎麼換手，這條連線都一直在。某條鏈路突然斷了，就當作「這條路被假流量塞滿」，別人自然不會走。
2. **事先規劃最佳路線（basic routing）**：在這張固定地圖上，用最佳化求解器算出「不爆量、不太慢、最忙的鏈路盡量閒」的路徑；換手可預測，所以能事先算好、按時切換。
3. **突發故障時先繞路、再重新規劃（LGPR）**：撞到斷線的衛星立刻把封包送給「離目的地最近」的鄰居；同時重新計算，算完再切回最佳路線。

> **比喻**：就像開車用導航。平常照事先規劃好的路線走（步驟 2）；前方突然封路，你先轉進一條「往目的地方向比較近」的路（步驟 3，LGPR），同時導航在背景重新計算最佳路線，算好之後再照新路線走。步驟 1 的技巧，是把「封路」改記成「那條路塞滿了」，導航就永遠不用重畫地圖。

### 3.4 哪些鏈路會斷？
- **可預測故障 = GSL（地面↔衛星）**：衛星飛過頭頂，地面站必須換手到下一顆衛星（每幾十秒一次），這部分由 virtual terrestrial node 吸收。
- **ISL（衛星↔衛星）在本文被假設為穩定**：Walker Delta 同一 shell 內相鄰衛星的相對位置固定，所以 +Grid 的 4 條 ISL 不會規律地斷開（這和 OPSPF 不同，OPSPF 的 polar orbit 在極區會規律地斷 inter-plane ISL）。
- **突發故障 = 任何鏈路都可能**（ISL 或 GSL，node failure 則是它的所有鏈路一起失效）。論文的例子（Fig. 3 的 S1–S2、Fig. 4 的 S12–S22）都是 ISL 突然故障，LGPR 也是在太空中沿著 ISL 繞路。

### 3.5 完整故事：台北 → LA 看影片
> 以論文的設定（Starlink 第一 shell，1584 顆）為背景，時間細節為說明用的假設。

**情境**：台北的使用者看 LA 伺服器上的影片。路徑是：台北地面站 → 衛星 A → 跨太平洋的多條 ISL → 衛星 Z → LA 地面站。

**問題 1：可預測的換手太頻繁**
- t = 0 s 台北站連 A；t = 30 s A 飛走，換成 B；t = 60 s 換成 C。LA 那邊也每幾十秒換一次。
- 地面站本身沒有移動，是「服務它的衛星」一直在換。對網路來說，「台北在哪裡」指的是「台北連在哪顆衛星底下」：30 s 前是「A 底下」，30 s 後變成「B 底下」。
- **用 OSPF**：換手後，其他衛星的 routing table 還寫著「要到台北，送給 A」，封包被送到 A，但 A 已經不連台北了，封包就被丟掉。要等 A 偵測到斷線、flood 全網、大家重算之後才恢復，每次約 7.4 s（Table I）。全網一直有人在換手，網路永遠在收斂，可達性只有約 55%。
  - 比喻：你站在路邊靠計程車收包裹。計程車 A 開過來，你從它那裡拿包裹；A 開走後換計程車 B。你沒有動，但寄件人還把包裹交給 A，就送不到你手上。
- **用 proactive / snapshot**：換手可預測，可以每個 snapshot 先算一張表；但若也要防突發故障，得把「每個 snapshot × 可能斷的鏈路組合」全部預算，3000 多條鏈路根本算不完。

**問題 2：突發故障**
- t = 45 s，地磁暴打壞太平洋上空的衛星 M（論文引用真實事件：一次毀掉 40 顆 Starlink），而路徑剛好經過 M。
- OSPF 又要收斂幾十秒（10–30% 故障時 35–150 s）；snapshot 若沒預算「M 壞掉」這種情況，就派不上用場。

**STARCURE 的解法（依時間順序）**

出發前（離線準備）：
1. **固定地圖（TSM）**：每顆衛星配一個虛擬地面站 vs_A、vs_B、vs_C、vs_Z……地圖永遠是「1584 顆衛星＋1584 個虛擬地面站」，不會變。
2. **把換手變成時間表**：由軌道事先知道台北 0–30 s 在 A 底下、30–60 s 在 B 底下、60–90 s 在 C 底下，LA 一直在 Z 底下。所以「台北 → LA 一條影片流」被切成：
   - 0–30 s：vs_A → vs_Z
   - 30–60 s：vs_B → vs_Z
   - 60–90 s：vs_C → vs_Z
3. **事先算好路線**：在固定地圖上用 Gurobi 算出每個時槽的路徑（不爆量、不太慢、最忙的鏈路盡量閒），存成時間表。

運作中：

4. **t = 30 s 換手 A → B**：衛星照時間表切到 30–60 s 的路線，台北的封包改從 B 上去，LA → 台北的封包也改送到 B。**不用偵測、不用 flood，也不會送錯**（論文為 0.6 s）。
5. **t = 45 s 衛星 M 被毀**：
   - M 前一顆衛星發現往 M 的鏈路斷了，立刻把封包的 protection flag 設為 true，轉給**離 LA 最近**的鄰居，繞過 M（約 1 s）。
   - 因為平常流量就已經分散，接手繞路流量的鏈路還有餘裕，不會塞車。
   - 同時，把「M 的鏈路被假流量塞滿」加進模型，重新計算 45 s 之後的路線；算好後切回新的最佳路線（自然會避開 M）。
6. **t = 60 s 換手 B → C**：和第 4 步一樣，按時切換（時間表已經更新成避開 M 的版本）。

**結果**：整個過程中台北使用者的影片幾乎不會卡，可達性接近 100%，恢復後的吞吐量比其他方法高最多 197%。可預測的換手事先準備、按時切換；突發故障先緊急繞路、再重新規劃。

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
