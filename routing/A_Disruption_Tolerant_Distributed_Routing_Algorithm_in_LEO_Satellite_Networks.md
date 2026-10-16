---
title: "A Disruption Tolerant Distributed Routing Algorithm in LEO Satellite Networks"
authors: [Jifeng Jin, Feng Tian, Zijian Yang, Hao Di, Guotong Li]
venue: "Applied Sciences (MDPI), vol. 12, no. 8, 3802, 2022"
doi: 10.3390/app12083802
survey_ref: "LEO survey 2310.07646 [84]（§4.6 Disruption tolerant networking & Reliability）"
tags: [LEO, satellite-routing, ISL, DTDR, fault-tolerance, distributed-routing, Walker-Star, polar-orbit, load-balancing]
---

## Citation
J. Jin, F. Tian, Z. Yang, H. Di, G. Li, "A Disruption Tolerant Distributed Routing Algorithm in LEO Satellite Networks," *Applied Sciences*, vol. 12, no. 8, 3802, 2022. DOI: 10.3390/app12083802（中科院微小衛星創新研究院、國科大、上海元心衛星）

## 一句話總結
**DTDR** 要解的問題是：雷射 ISL 不穩定（通道突變、衛星工作模式切換、設備故障），鏈路一斷，衛星上的 buffer 又很小，如果沒有馬上能用的備援路徑，就會大量掉包。做法是每顆衛星維護**三張路由表**：(1) **main path**：依緯度從「橫向（換軌）」或「縱向（同軌）」兩個方向中挑最短的那個；(2) **alternate path**：兩個方向中剩下的那個，直接相連的鏈路斷了就**立刻**換過去；(3) **detour path**：斷線的那兩顆衛星只把故障資訊 flood 到 **n 跳內**，收到的衛星進入 **failure area state（FAS）**，凡是 main path 會經過那條壞鏈路的目的地，就**提早**改走 alternate 方向，同時把流量分散到故障區周圍。作者用 3×3 網格分析，建議**高緯 n = 2、低緯 n = 3**。在 OPNET 上模擬 Walker 128/8/3、1050 km、89° 的極軌星座，和 DRA、OPSPF、DODR 比：DTDR 的**掉包率在所有故障率和負載下都最低**，延遲和 OPSPF 差不多，但比 DODR 略高。Survey 把它列為 [84]，描述成「只廣播壞掉的鏈路資訊」。

## 0. 背景知識
### 這篇的星座：極軌（Walker Star）＋seam
* 每顆衛星 4 個雷射介面（連鄰居）＋2 個介面接地面站和 space base station。雷射終端會告訴路由器「這條鏈路故障了」。
* 有兩條 **seam**：seam 兩側的軌道反向運動，**不建鏈**（和 OPSPF、StableRoute、SIDA 的模型一樣）。
* **跨極區時 inter-plane ISL 不斷**（§2，Fig. 2）：在西半球，B 用介面 1 連 A 的介面 3；衛星飛過極區進入東半球時**姿態不變**，只是介面 1 和 3 的實際方位互換，鏈路**一直保持**。所以從路由角度看，拓樸只有 inter-plane ISL **長度**在變，連線關係不變。
    * ⚠ 這和 OPSPF 的假設**正好相反**：OPSPF 的核心就是「進入 ±70° 極區時跨軌 ISL 會因天線追不上而斷」。

### ISL 長度（§2，Eq. 1–5）
* 參數：軌道半徑 R、N 條軌道、每條 M 顆、相鄰軌道衛星相位差 φ、相鄰軌道升交點赤經差 RI。
* Intra-plane：固定，L_v = √2·R·√(1 − cos(360°/M))。
* Inter-plane：用等腰梯形 AA′BB′ 算，L_h = √2·R·√(1 − cos φ + (1 − cos RI)·cos(latA)·cos(latB))。
* 白話：**緯度越高，cos 越小，跨軌 ISL 越短**（軌道在極區收攏）。
* 我用論文的模擬參數驗算（R = 6371 + 1050 km、M = 16、N = 8、F = 3 → φ ≈ 8.44°、RI = 22.5°）：L_v ≈ 2896 km；L_h 在赤道 ≈ 3080 km、30° ≈ 2623 km、45° ≈ 2173 km、60° ≈ 1653 km、75° ≈ 1200 km。
* ⚠ 論文 Algorithm 1 的證明寫「L_h increases with increasing latitude」，和自己的公式相反；但結論「在高緯度做橫向跳比較好」只有在 L_h **隨緯度變短**時才對，§4.1 也寫 latV1 > latV7 所以 L_h3 > L_h1。所以那句是筆誤，應該是 decreases。

## 1. Problem
* **Problem**：LEO 雷射 ISL 會**非預期地斷**（不是 OPSPF 那種可預測的極區斷線）。現有方法的問題：
    * Snapshot / virtual topology（FSA [8]）：處理不了突發故障。
    * On-demand（LAOR [9]、DODR [13]）：故障時要重新找路，**沒有立即可用的路徑**，容易掉包。
    * 分散式多路徑（ELB [10]）：只能局部平衡，容易形成迴圈和連鎖壅塞。
    * Link state（包括 **OPSPF [15]**）：能看到全網，但故障時要**全網廣播**，同步時間長，同步完成前容易形成迴圈，大星座收斂慢。
    * 衛星 buffer 小（雷射頻寬大，buffer 只夠 ISL 切換時存一點點封包），不能靠排隊撐過去。
* **Target**：鏈路一斷就有**立即可用、可靠**的備援路徑，降低**掉包率**；故障區附近順便**分散負載**；訊令開銷有上限。
* **有沒有正式 formulation**：沒有。是啟發式演算法＋一個 3×3 的小分析＋模擬。作者也明說「沒有給全域或局部負載平衡的明確策略」，負載平衡只是繞道的副作用。

## 2. Method
### 2.1 全貌（§3.1）
四個部分：
1. **全域拓樸生成**（週期性）
2. **Main path direction**（主路徑方向）
3. **Alternate path direction**（備援方向）
4. **Failure area routing**（故障資訊傳遞＋繞道路由表）

轉送時：平常查 main；直接相連的鏈路壞了 → 查 alternate；收到故障通知 → 查 detour。三張表的優先順序見 2.6。

### 2.2 全域拓樸生成（§3.2）
* 借用 virtual topology 的想法：把一個軌道週期切成 **K 個等長時間區間**。每進入新區間，衛星更新自己的緯度 latC，重算路由。
* 因為連線關係不變，只有 L_h 隨緯度變，所以「拓樸變化」= 「緯度變化」。

### 2.3 Main path direction（§3.3，Algorithm 1）
* 每顆衛星用（軌道號, 相位號）標記。兩顆衛星之間的軌道差和相位差**永遠不變**，所以要走幾次橫向跳（horizontal hop，跨軌）、幾次縱向跳（vertical hop，同軌）是固定的（Fig. 3：從 ⟨2,6⟩ 到 ⟨5,4⟩ 要 3 橫 2 縱）。
* **橫向方向 d_h**：目的地軌道號較大 → 介面 3；較小 → 介面 1；同軌 → −1。（注意：不能跨 seam，所以只有一個方向，不像 Walker Delta 可以左右繞。）
* **縱向方向 d_v**：軌道是環，所以看 |d_j − c_j| 和 M − |d_j − c_j| 哪個小，選順時針或逆時針；同相位 → −1。
* **選 main 方向**（只有兩個方向都存在時才要選）：
    * cpr = 0（路徑不經過極區）：**目前緯度 ≥ 目的地緯度 → 先橫**；否則先縱。理由：L_h 在高緯比較短，橫向跳要在**高緯那一端**做。
    * cpr = 1（路徑要經過極區）：如果自己已經是**最靠近極點的那顆**（latC = lat_max）→ 先橫；否則先縱往極區走，到最靠近極點時再橫。
* 每顆衛星都知道自己的緯度，目的地緯度可以從相位差推出來，所以這些都可以本地算。

### 2.4 Alternate path direction（§3.4）
* 兩個方向都存在時：main 是橫 → alternate 是縱；main 是縱 → alternate 是橫。
* 只有一個方向時（同軌或同相位）：
    * main 是橫、沒有縱向跳 → alternate = **遠離極區**（緯度變低）的縱向方向。
    * main 是縱、沒有橫向跳 → alternate = **遠離 seam** 的橫向方向。
* 和 Qi 2020 比：DTDR **只有 1 個** alternate 方向，Qi 有 2 個 secondary 方向。

### 2.5 Failure area routing（§3.5，Algorithm 2）
**故障資訊傳遞**
* 只有直接連著壞鏈路的路由器知道它壞了，周圍的節點不會少送流量進來，所以會塞。
* 全網廣播：開銷大，而且離得遠的衛星收到時資訊已經過時，對它們也沒什麼用；ISL 故障通常很短。
* 所以：壞鏈路兩端只 **flood 到 n 跳內**。收到的衛星進入 **FAS（failure area state）**，採取「提早繞道」。因為收到的是 n 跳內的故障資訊，加上自己的鏈路，每顆衛星知道 **n+1 跳內所有鏈路**的即時狀態。

**繞道路由表怎麼算**（每收到一次故障或恢復就重算）
1. 初始化：detour 表裡所有目的地的介面設為 −1，FAS = 0。
2. 掃過 n+1 跳內所有鏈路，找到壞的就記 FAS = 1，進入步驟 3。
3. 對這條壞鏈路，檢查每個目的地的 **main path 前 n+1 跳**有沒有經過它（main path 很便宜，可以即時算，不用存）。有經過 → 這個目的地的介面 = alternate 方向；沒經過 → −1。
4. 把這張表合併進 detour 表（只填還是 −1 的格子，所以**第一條**被掃到的壞鏈路優先），回到步驟 2。

Fig. 4 是一個菱形區域（以壞鏈路為中心的 n 跳範圍），區域內 main path 會穿過壞鏈路的流量會提早轉向，等於把流量撒到故障區的兩側。

### 2.6 n 要選多少（§4.1）
* 作者說衛星段的業務大多 3–4 跳，所以在 **3×3 網格**上比較 n = 1、2、3。例子：V1 → V9，main path V1-V2-V3-V6-V9，壞鏈路 V6-V9。latV1 = latV3 > latV7（上面緯度高）。
* 兩個指標：
    * **傳播延遲** t = L/c（L = 路徑總長）→ 服務品質。
    * **繞道距離 d** = 繞道路徑和 main path 圍出的格子面積 → 分散負載的能力（越大越分散）。
* 單一故障 V6-V9 時：

| n | 從哪裡開始繞 | 繞道路徑 | t | d |
|---|---|---|---|---|
| 1 | V3 | V1-V2-V3-V2-V5-V8-V9（走回頭路） | (3L_h1 + L_h3 + 2L_v)/c | 2 |
| 2 | V2 | V1-V2-V5-V8-V9 | (L_h1 + L_h3 + 2L_v)/c | 2 |
| 3 | V1 | V1-V4-V7-V8-V9 | (2L_h3 + 2L_v)/c | 4 |

* 假設 V1-V2、V2-V3、V3-V6、V6-V9 四條等機率壞，平均：
    * n = 1：t1 = (4L_h1 + 5L_h2 + L_h3 + 8L_v)/4c，d1 = 1.75
    * n = 2：t2 = (L_h1 + 6L_h2 + L_h3 + 8L_v)/4c，d2 = 2
    * n = 3：t3 = (6L_h2 + 2L_h3 + 8L_v)/4c，d3 = 2.5
* 結論：n = 1 有回頭路，淘汰。t3 − t2 = (L_h3 − L_h1)/4c > 0，因為 V7 緯度較低、L_h3 較長，所以 **n = 2 延遲較低、n = 3 分散較好**。高緯時 L_h 隨緯度變化大，延遲差距明顯 → **高緯用 n = 2**；低緯 L_h 變化平緩、而且業務多集中在低緯（[19]）→ **低緯用 n = 3**。
* ⚠ 模擬時實際用哪個 n、是否依緯度切換，論文**沒寫**。

### 2.7 轉送流程和迴圈避免（§4.2）
* 優先順序：**1. detour 表 → 2. main 表 → 3. alternate 表**。往下一張表的條件：
    1. 衛星不在 FAS（附近沒有壞鏈路）：1 → 2
    2. detour 表裡這個目的地是 −1（main path 沒經過壞鏈路）：1 → 2
    3. 查到的介面是未建鏈狀態（指向 seam）或那條鏈路壞了：1 → 2、2 → 3
    4. 查到的介面就是封包**進來的那個介面**（會走回頭）：1 → 2、2 → 3
* Fig. 6（封包從 4 進到 5，p1 = main、p2 = alternate）：
    * p1、p2 都不能用 → 選 **p3 = alternate 的反方向**，這時**允許回頭**（p3 可能就是進來的方向）。
    * p3 也不能用 → 用剩下的 **p4**，並且在封包頭把**本衛星標成 invalid**，之後經過鄰近衛星時不會再把封包送回來。
* 和 Qi 2020 的 partial-record 比：DTDR 只在「三個方向都不行」時才在封包頭記一筆，更省，但論文**沒有證明不會無窮迴圈**。

## 3. Evaluation Setup（§5.1）
| 項目 | 值 |
|---|---|
| 模擬器 | **OPNET** |
| 星座 | Walker **128/8/3**（8 軌 × 16 顆），1050 km，**89°**，有 seam |
| 流量 | 重力模型 TR_ij = (SD_i·SD_j)^θ / d_ij^φ，θ = 0.5、φ = 1.5；SD = 該衛星服務區的設備覆蓋密度（[20]）；Poisson |
| 故障 | 「每單位時間的鏈路故障機率」，參考值 2.5‰；故障持續多久**沒寫** |
| 負載 | 全網平均鏈路負載，參考值 24% |
| Baselines | **DRA [6]**（Ekici 2001：靜態算路，直接相連的鏈路壞了就換備援）、**OPSPF [15]**（作者描述：故障時全域廣播、跑最短路）、**DODR [13]**（virtual node＋限制範圍的路徑發現，壞了就擴大範圍重找） |
| 指標 | 掉包率、端到端延遲 |

## 4. Main Results（§5.2；數字是我從圖讀的）
* **不同故障率（負載 24%，故障率 0.3‰–8‰，Fig. 7）**
    * 掉包率：**DTDR 一直最低**；OPSPF 最差。8‰ 時大約 DTDR 0.039、DODR 0.047、DRA 0.056、OPSPF 0.063。
        * 作者解釋：OPSPF 收斂慢，對短暫故障反應慢，容易算出錯誤路徑形成迴圈。DRA 在故障少時比 DODR 好（備援路徑可靠），故障多時比 DODR 差（需要即時找路）。
    * 延遲：**DODR 最好**，DTDR ≈ OPSPF，DRA 最差。8‰ 時大約 DODR 39.9 ms、OPSPF 40.9 ms、DTDR 41.5 ms、DRA 42.5 ms。
        * 作者解釋：DTDR 的故障資訊只傳 n 跳，在收斂區邊界可能有多餘繞道。
* **不同負載（故障率 2.5‰，負載 0.16–0.4，Fig. 8）**
    * 掉包率：DTDR 一樣一直最低，**中低負載時優勢最明顯**（0.16 時約 0.001 vs 其他 0.005–0.01）；負載到 0.4 時大家都到 0.11–0.12，差距縮小。
    * 延遲：DTDR、OPSPF、DODR 差不多，DODR 略好；DRA 在負載 < 28% 或 > 36% 時明顯較高。
* 作者總結：DTDR 的主要優勢是**低掉包率**，延遲不是它的強項。

## 5. Novelty／貢獻
* **新的部分**：
    * 用**緯度**決定先橫還是先縱（高緯做橫向跳），以及經過極區時「到最靠近極點那顆再換軌」的規則。
    * 三張表（main / alternate / detour）的優先順序設計，讓直接故障（alternate）和附近故障（detour）有不同反應。
    * 用 3×3 分析把「n 跳 flooding」和「延遲 vs 負載分散」連起來，給出依緯度選 n 的建議。
* **不新的部分**：主備方向（DRA 2001）、限制範圍 flooding＋提早繞道（**Qi 2020 已經提出，DTDR 沒有引用**；我查過 DTDR 的 21 篇參考文獻）、virtual topology 時間切片（Werner 1997）、重力流量模型都是既有的。

## 6. Limitations
* 作者自己承認的：
    * 沒有明確的全域或局部負載平衡策略。
    * 收斂區邊界可能有多餘繞道，故障率高時延遲比 DODR 高。
* 我額外看到的：
    * **極區 ISL 不斷的假設很強**：89° 星座在極區跨軌 ISL 會快速收攏、相對角速度很大，多數文獻（OPSPF、SIDA、StableRoute）都假設會斷。DTDR 用「介面方位互換」帶過，沒有討論天線追蹤是否可行。如果極區 ISL 實際會斷，cpr = 1 時「到最靠近極點那顆再換軌」的規則就會指向一條不存在的鏈路。
    * **和 OPSPF 的比較條件不對等**：OPSPF 的主要賣點是處理可預測的極區斷線，但 DTDR 的模型裡極區不斷，OPSPF 的靜態模組幾乎沒有發揮空間，只剩動態模組（全網 LSU）在比。而且 OPSPF 是作者在 OPNET 上自己重寫的（原論文用 Exata），實作細節（Hello 間隔、Dead interval）沒交代，OPSPF「最差」這個結果要保守看。
    * **以單一 3×3 例子推 n 的選擇**：只考慮 V1→V9 一對、main path 上四條鏈路等機率壞，沒有在大網格或多故障下驗證；模擬裡用的 n 也沒寫。
    * **路徑以方向為單位，不是真正的最短路**：「先橫還是先縱」只決定第一步，整條路徑是否是最短距離沒有證明（Algorithm 1 的「證明」只是直覺說明）。StepClimb 的分析顯示，最短路應該在 MHP 矩形裡某個特定緯度換軌，不一定在端點。
    * **Detour 表合併規則**：多條鏈路同時壞時，只用第一條被掃到的那條決定繞道方向，繞過去的 alternate 方向本身可能也經過另一條壞鏈路，論文沒處理。
    * **沒有迴圈避免的證明**，也沒量封包頭開銷或迴圈發生率。
    * **評估不完整**：沒有訊令開銷（明明是 DTDR 宣稱的優點之一）、沒有收斂時間、沒有故障持續時間、沒有重複次數或信賴區間。結論說「在延遲和掉包率都勝過現有演算法」，但 Fig. 7(b)、8(b) 顯示延遲是 DODR 較好，**結論過度宣稱**。
    * 星座只有 128 顆，不是 mega-constellation；沒有測 Walker Delta。
    * 論文 PDF 有一些排版問題（審稿版文字疊在正式版上），Algorithm 1 有「L_h 隨緯度增加」的筆誤（見 §0）。

## 7. 與其他筆記的關聯
### 和 OPSPF（[[OPSPF_Orbit_Prediction_Shortest_Path_First_Routing_for_Resilient_LEO_Satellite_Networks]]）
* **直接把 OPSPF 當 baseline**（DTDR ref [15]），並說 OPSPF「故障時路由資訊開銷大、大星座收斂慢、容易形成錯誤路徑」。這正是 OPSPF 筆記裡提到的：OPSPF 的動態模組本質上還是 OSPF 的全網 LSU。
* **但 DTDR 沒有補 OPSPF 的 Walker Star 缺口**：DTDR 也是極軌（89°）、也有 seam，一樣只測了 Walker Star 型星座。真正補「Walker Delta」缺口的是 Qi 2020（[[A_Distributed_Survivable_Routing_Algorithm_for_Mega-Constellations_With_Inclined_Orbits]]）。
* **兩篇對極區的假設相反**：OPSPF：極區跨軌 ISL 一定斷，而且可預測；DTDR：極區跨軌 ISL 不斷，只是長度變短、介面方位互換。如果要把兩者結合（推論）：用 OPSPF 的靜態模組處理可預測的極區斷線，再用 DTDR 的「n 跳 flooding＋detour 表」取代 OPSPF 動態模組的全網 LSU。

| | OPSPF | DTDR |
|---|---|---|
| 星座 | Walker Star，LEO-48 | Walker Star（89°），128 顆 |
| 極區跨軌 ISL | 斷（±70°） | 不斷 |
| 平常怎麼算路 | 每秒預測位置＋Dijkstra | 每個時間區間依緯度選方向，不跑 Dijkstra |
| 非預期故障 | 全網 LSU＋重算 | 直接相連：立刻換 alternate；n 跳內：detour 表 |
| 強項 | 可預測變化 0 封包 | 掉包率低 |

### 和 Qi 2020（[[A_Distributed_Survivable_Routing_Algorithm_for_Mega-Constellations_With_Inclined_Orbits]]）
* 兩篇的骨架幾乎一樣：**網格上算主/備方向（不跑 Dijkstra）＋故障只 flood 到 n 跳＋收到的衛星對「路徑會經過壞鏈路」的目的地提早繞道＋迴圈避免**。DTDR 晚兩年，沒有引用 Qi。

| | Qi 2020 | DTDR 2022 |
|---|---|---|
| 星座 | Walker Delta 53°，1600 顆 | Walker Star 89°，128 顆 |
| 路徑標準 | 最少 hop | 依緯度選先橫/先縱（想近似最短距離） |
| 備援 | 2 個 secondary 方向，各自有 N_block | 1 個 alternate 方向 |
| 故障範圍 | f_ttl 跳（試了 2/5/10） | n 跳（建議高緯 2、低緯 3） |
| 迴圈避免 | partial-record＋回溯，有證明 | 允許回頭＋標記 invalid，無證明 |
| 量了什麼 | 延遲、訊令、封包頭開銷、吞吐量 | 掉包率、延遲 |

### 和 survey 容錯章節（[[LEO_Satellite_Networking_Relaunched_Survey_and_Current_Research_Challenges]]）
* Survey §4.6 的描述：DTDR「只廣播壞掉的鏈路資訊（而不是 link status update）來降低開銷；利用星座相對靜態或週期性的網格拓樸，用靜態路由加上動態演算法來算路由表」。
* 讀完原文的補充：
    * 「只廣播故障」**不是全網廣播**，而是只 flood 到 **n 跳**（2 或 3），這點 survey 沒寫，而這才是開銷低的主要原因。
    * 「靜態路由」= main/alternate 方向表（每個時間區間依緯度更新），「動態演算法」= 收到故障後重算的 detour 表。
* Survey 筆記裡說「DTDR 和 OPSPF『平常用預測、故障才發 LSU』非常像」，原文看下來要修正：相像的是「平常不需要訊令」，但故障處理不同，OPSPF 是**全網** LSU，DTDR 是 **n 跳**。DTDR 其實更像同一節的 [123]（Qi 2020）。
* 名字容易和 DTN（RFC 9171 Bundle Protocol）搞混：DTDR 是路由演算法，不是 store-and-forward 的 bundle 協定，也沒有在衛星上長時間存封包（作者反而強調 buffer 很小）。

### 其他
* [[Shortest_Path_in_LEO_Satellite_Constellation_Networks_An_Explicit_Analytic_Approach]]（StepClimb）：DTDR 的「在高緯那端做橫向跳」是 StepClimb 想法的簡化版：StepClimb 用解析式算出在 MHP 矩形裡哪一個緯度換軌最短，DTDR 只在兩端點二選一。兩篇都假設高緯 ISL 不斷，這點和 OPSPF 衝突。
* [[A Load Balancing Routing Strategy for LEO Satellite Network]]（SIDA/SSLB）：同樣是極軌＋seam＋+Grid。SSLB 是「塞了才分流給鄰居」，DTDR 的 detour 是「附近壞了就提早分流」，觸發條件不同但動作很像。SIDA 筆記說 SSLB 只適用低緯（高緯鄰軌 ISL 斷），而 DTDR 假設高緯不斷，兩篇的模型又不一致。
* [[StableRoute_When_Dijkstras_Algorithm_Meets_Topology-Varying_Satellite_Networks]]：DTDR 每個時間區間依緯度重新決定先橫或先縱，main 方向會隨衛星移動跳換，這是 StableRoute 想避免的「不必要換路」（推論；DTDR 沒量路徑變動次數）。
