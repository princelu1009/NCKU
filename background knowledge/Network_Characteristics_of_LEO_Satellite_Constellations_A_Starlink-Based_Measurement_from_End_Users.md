---
title: "Network Characteristics of LEO Satellite Constellations: A Starlink-Based Measurement from End Users"
authors: [Sami Ma, Yi Ching Chou, Haoyuan Zhao, Long Chen, Xiaoqiang Ma, Jiangchuan Liu]
venue: "IEEE INFOCOM 2023"
doi: 10.1109/INFOCOM53939.2023.10228912
code: https://codeocean.com/capsule/2088459/tree
tags: [LEO, Starlink, measurement, bent-pipe, GSL, ground-station, latency, throughput, weather, mobility, background]
---

## Citation
S. Ma, Y. C. Chou, H. Zhao, L. Chen, X. Ma, J. Liu, "Network Characteristics of LEO Satellite Constellations: A Starlink-Based Measurement from End Users," *IEEE INFOCOM 2023*. DOI: 10.1109/INFOCOM53939.2023.10228912（Simon Fraser University 與 Douglas College，加拿大溫哥華）

## 一句話總結
這是一篇**實測論文**，不是演算法論文。作者在溫哥華用 4 個 Starlink 天線（3 個 Gen 1、1 個 Gen 2），從 2022 年 1 月起量了約 7 個月，對 9 個 AWS 區域跑 ping、traceroute、iperf3、SCP 和 8K YouTube 串流，並和 800 Mb/s 的地面 Cable 網路比較。結論是：Starlink **能用、但很不穩**。平均延遲只比地面網路高約 10%，但延遲變異約 3.8 倍、jitter 約 5 倍以上；TCP 下載平均約 80 Mb/s，上傳只有約 6 Mb/s。天氣影響很大：下雨平均掉 27% 吞吐量，地磁風暴那兩天從 100 Mb/s 掉到 5 Mb/s。最重要的網路層觀察是：**2022 年的 Starlink 是 single bent-pipe**，封包只走「天線 → 一顆衛星 → 離天線最近的地面站」這一跳，接著就進地面 ISP，不管目的地在哪；沒有 ISL，也沒有多段 bent-pipe。所以 Starlink 在當時只是「最後一哩」，**星座本身沒有參與路由**。偏遠地區、高緯度、山谷和行車中的表現都更差。對我們讀的 ISL 路由論文來說，這篇是「現實的 baseline」：它告訴我們沒有 ISL 時，延遲變動主要來自衛星移動和換手，跨洋流量要靠海底電纜，這正是那些論文想用 ISL 解決的問題。

> 讀法建議：§III-A（Starlink 架構）、§IV-A（延遲）、§IV-C（Routing Strategy）是和路由最相關的部分，先讀這三段。§IV-D 天氣、§V 野外、§VI 行動性可以快速看結果就好。

## 0. 先記住的名詞

| 名詞 | 全名 | 一句話 |
|---|---|---|
| LSN | LEO Satellite Network | 作者對 LEO 衛星網路的簡稱 |
| UL | User Link | 使用者天線 ↔ 衛星（Ku-band） |
| GSL | Ground-to-Satellite Link（GS-to-Satellite Link） | 衛星 ↔ 地面站（Ka-band） |
| GS | Ground Station | 地面站，接到地面網際網路 |
| ISL | Inter-Satellite Link | 衛星間雷射鏈路；**2022 年量測時 Starlink 尚未大規模啟用** |
| Bent-pipe | — | 衛星只當中繼器（repeater），收到 UL 的訊號直接轉給 GS，不做路由 |
| Dishy | Dishy McFlatface | Starlink 使用者天線的暱稱；Gen 1 圓形 7.3 kg，Gen 2 方形 4.2 kg |
| Obstruction ratio | — | Starlink App debug 頁的 `FractionObstructed`，天空被擋住的比例 |
| Buffer health | — | YouTube「Stats for nerds」裡已預先下載、還沒播的影片秒數，掉到 0 就卡住 |

## 1. Introduction
* **Problem**：LEO 星座（Starlink、OneWeb、Telesat）已經商轉，但學界的研究幾乎都靠模擬器（Hypatia、StarPerf），很少有人從**一般使用者**的角度系統性地量過真正的網路表現。Starlink 對使用者是黑盒子：插上電、接 WiFi 就能上網，看不到衛星和路由細節。
* **三個問題**：
    1. 現在的 Starlink 能不能達到和現代地面網路相當的表現？
    2. 影響 Starlink 服務品質的關鍵因素是什麼？
    3. Starlink 真的做到全球覆蓋了嗎？如果沒有，難在哪裡？
* **主要發現（作者整理）**：
    1. 吞吐量和延遲的變動比地面網路大很多，斷線（outage）也頻繁。
    2. 使用者體驗和耗電都受環境影響：地形、太陽風暴、雨、雲、溫度。
    3. Starlink 現在用 **bent-pipe**，而且一般只有**一跳**，只經過**最近的 GS** 就轉進地面網路。跨洋路線因此受限。
    4. 行進中吞吐量變化不大，但斷線頻繁、延遲很高；行動性還需要很多改進。
    5. 偏遠地區有技術上的挑戰（電力、遮蔽、野生動物），也有文化上的挑戰。

## 2. Related Work（作者的分類）
* 衛星通訊的理論分析、系統設計、實際部署；早期量測多是 GEO。
* LEO 研究：衛星與 GS 的互動、拓樸管理、定址與路由、環境因素（雨 [15]、太陽風暴 [23]）。
* 模擬器：**Hypatia [2]**、**StarPerf [3]**，適合驗證模型，但取代不了實測。
* 早期 Starlink 量測 [24]–[28] 比較偏實體層（光學亮度、訊號定位）。同期的 IMC 2022 兩篇 [29][30] 用 Ookla speedtest 和瀏覽器外掛，看 QUIC 和瀏覽器延遲。
* 本文的差別：目的地分布更廣（9 個洲際 AWS 區域）、加上影片串流、偏遠北方地區、耗電量。

## 3. Starlink 系統概觀（§III-A）
* 截至量測時：超過 2,000 顆衛星、40 萬用戶。大部分在**第一殼層：72 個軌道面、每面最多 22 顆、高度 550 km、傾角 53°**（Fig. 2a）。FCC 已核准 12,000 顆，Gen 2 再加 7,500 顆。
* 衛星用 phased array beamforming，Ku- 和 Ka-band。
* 使用者套件：天線、PoE 電源、路由器（Gen 1 有 Ethernet；Gen 2 只有 WiFi）。
* **資料路徑（Fig. 2b）**：天線 —UL→ 一顆可見的衛星 —GSL（Ka-band）→ GS → 網際網路，反向亦同。
* **Bent-pipe 的幾何限制**：GS 和天線必須**同時**在同一顆衛星的視野內，所以兩者距離大約不能超過 **1,000 km**。
* 一顆 LEO 衛星在地面某一點上空只停留約 **10 分鐘**，飛走就要換手（handover）給下一顆。
* GS 數量：北美至少 65 個、歐洲 23 個，其他零星分布。
* ISL：Starlink 已開始雷射 ISL 實驗，但論文寫作時預期 2022 年底之前不會大規模啟用。

## 4. 量測設定（§III-B、§III-C）

### 4.1 硬體和地點
| 天線 | 世代 | 地點 | 備註 |
|---|---|---|---|
| A | Gen 1 | Burnaby（大溫哥華） | 遮蔽 2.7% |
| B | Gen 1 | Burnaby | 周圍樹多，遮蔽 4.7% |
| C | Gen 1 | Coquitlam | 陽台，遮蔽 24.9%；後來帶去 Manning Park 和車頂 |
| F | Gen 2 | Koeye Point（BC 北部海岸） | 只有 WiFi，遮蔽 0.0% |

* 每個 Starlink 路由器接一台 **Raspberry Pi 4B**（Ethernet，F 只能用 WiFi）跑腳本，減少干擾。
* **目的地**：9 個 AWS 區域：N. California、Tokyo、London、Sydney、Singapore、Sao Paulo、Bahrain、Mumbai、Cape Town。
* **Baseline**：一台 i7 PC，接 800 Mb/s 的地面 Cable 寬頻，打同樣的 AWS 區域。作者說換其他類似速度的寬頻結果差不多。
* **耗電**：Emporia smart plug。
* **天氣**：Vancouver Harbour 氣象站的每小時資料，離天線 A 約 10 km。
* 規模：每個區域累計超過 300 萬筆紀錄。

### 4.2 工具
* **iperf3**：TCP（cubic）和 UDP 吞吐量；每個目的地開兩個 port，上下行同時量。UDP 會逐步加大流量直到飽和。
* **SCP**：用 `fallocate` 產生 1–300 MB 的檔案，雙向傳。
* **ping / traceroute**：延遲和路徑。
* **Starlink App**（含 debug 頁）、**YouTube Stats for nerds**：遮蔽、斷線、串流 buffer 狀態。

## 5. 都市量測：Starlink vs. 地面網路（§IV）

### 5.1 端到端延遲（§IV-A）
* 平均延遲**比地面網路高約 10%**，但**變動大很多**（Fig. 4、5）。
* 平均延遲差 μs − μt（ms，Table I）：

| Bahrain | Cape Town | London | Mumbai | N. California | Sao Paulo | Singapore | Sydney | Tokyo |
|---|---|---|---|---|---|---|---|---|
| 8.4 | 14.3 | 16.0 | 12.9 | 20.9 | 22.8 | 1.8 | **78.4** | 9.2 |

* 一般差 1.8–22.8 ms；**Sydney 是離群值**（78.4 ms，比其他區域的差大 3.4–43.6 倍）。
* 作者的推論：
    * 如果有**多段** bent-pipe，Starlink 對某些目的地應該會比地面網路**快**（衛星走直線、光在真空中比光纖快），而且各區域的差距會變化很大。實際上 Starlink 幾乎都比較慢、而且差距大致固定，表示只有**第一跳**走衛星。
    * Sydney：Starlink 當時的覆蓋只到陸地附近的沿海，跨太平洋必須轉地面網路走海底電纜。traceroute 顯示到 Sydney 的流量在第一跳 bent-pipe 後先繞到 **Seattle** 的 AWS 節點。
    * Bahrain 的延遲上升在地面網路也看得到，所以是 AWS Bahrain 那邊的問題，不是 Starlink。
* **延遲變異**：Starlink 的標準差約是地面網路的 **3.8 倍**。原因是衛星一直在動：連到的衛星 S 從 t0 飛到 t1、離天線越來越遠，延遲就一直上升，直到換手給下一顆（Fig. 7）。
* **Jitter**：Starlink 下載 0.255 ms、上傳 2.715 ms；地面網路 0.041 ms、0.546 ms。

### 5.2 端到端吞吐量（§IV-B）
* 平均吞吐量（Mb/s，Table II，取 9 區平均，由我們從表格計算）：

| | TCP 下載 | TCP 上傳 | UDP 下載 | UDP 上傳 |
|---|---|---|---|---|
| 地面網路 | 483.4 | 48.9 | 807.5 | 102.4 |
| Starlink | 78.4 | 6.3 | 200.3 | 10.4 |

* Starlink TCP 下載約 **80 Mb/s**，但不穩：標準差（相對值）Starlink 50.71%、地面 34.44%。
* **UDP 吞吐量在 9 個區域幾乎一樣**（下載都約 194–206 Mb/s），表示瓶頸在天線這一跳（UL 或 GSL），可能是物理限制，也可能是**刻意做了 traffic engineering／限速**。作者傾向後者，理由是 §5.4 發現吞吐量和耗電沒有正相關，可能是為了控制耗電而限速。
* **突發性掉包**：就算 UDP 只送 20% 的可用頻寬（應該不會壅塞），仍有平均 0.24%（下載）和 1.24%（上傳）的突發掉包。掉包有約 **12 小時的週期**，同一週期內各區域的數值相近，所以作者推測掉包發生在 bent-pipe，尤其是 UL。
* **TCP 效率**：bandwidth utilization（TCP/UDP）Starlink 39.0%、地面 46.8%，作者解讀為 TCP congestion control 對衛星這一跳的變動很敏感。
* Gen 2 天線的 TCP 上傳比 Gen 1 多 1.76 倍，但 UDP 上傳反而只有 0.73 倍，所以改善可能來自路由器或存取方式，不是無線鏈路本身。
* **SCP**：1 MB 沒差（SCP 本身的開銷蓋過去），200 MB 時 Starlink 的傳輸時間約是地面的 **2 倍**，而且變異更大（Fig. 9）。

### 5.3 路由策略（§IV-C）⭐ 和我們最相關
* traceroute 顯示：
    * **只有一段 bent-pipe**（一條 UL ＋ 一條 GSL），然後就進地面網路。
    * **永遠連到離天線地理上最近的 GS**，不管目的地在哪。
    * 路徑上只看到**一個 SpaceX Services 的 ISP 節點**（在最近的 GS），下一跳就換成美國的地面 ISP。
    * 作者沒找到 Starlink 用 L2 routing 的證據。
* **Pearson 相關性（Fig. 10）**：Starlink 到各區域的延遲時間序列**兩兩之間都是中度相關（0.35–0.70）**，很平均。地面網路則分成兩群：N. California、Sao Paulo、Singapore、Sydney、Tokyo 彼此相關 0.6–0.75（大概共用美西出口），其他組合多半只有 0.04–0.3。意思是不管打哪個區域，Starlink 都用相似的衛星、GS 和路徑，延遲一起漲一起跌，**變動來自共同的第一跳**。
* 作者的反證論點：如果 Starlink 會為了去 London 把流量用多段 bent-pipe 送到美東的 GS，那 London 和 N. California、Sydney、Tokyo 這些「不同方向」區域的相關性應該會變低。但實際上都是中度相關，所以沒有這種行為。
* **一句話**：當時的 single bent-pipe 架構**除了當第一跳（或最後一跳）接上網際網路之外，完全沒有參與路由**。
* **兩個 Starlink 使用者互連**：作者把 A、B 放在同一顆衛星範圍內，想讓它們直接透過衛星互傳，不經過 GS。結果 Starlink 只允許 ssh，也不給設定 port forwarding，必須透過外部 proxy（remote.it，在 AWS 上架中繼）。吞吐量平均只有 2–3 Mb/s，偶爾衝到 9–12 Mb/s，而且 **27% 的讀數是 0**。也就是說，「同一顆衛星下的兩個使用者」還是要繞到地面、再繞回來。

### 5.4 環境因素（§IV-D）
1. **遮蔽**：App 的可視圖要 12–24 小時才建好；obstruction ratio 更新比較快，但不一定和可視圖一致。就算位置很空曠，**天線表面上的落葉等異物**也會造成延遲尖峰甚至斷線。結論：要天空乾淨，**也要天線乾淨**。
2. **太陽／地磁風暴**：2022/2/3–4 吞吐量從 100 Mb/s 掉到 5 Mb/s。同時發生兩件事：2/3 發射 49 顆新衛星，2/4 地磁風暴讓其中約 40 顆重返大氣層。可能是風暴干擾 UL/GSL，也可能星座進入維護模式。**只有下載受影響、上傳穩定**，表示兩個方向不對稱。
3. **雨、溫度、耗電**：
    * 有降雨時吞吐量平均掉 **27%**；大雨（> 4 mm/h）時被壓住。UDP 下載從無雨時約 215 Mb/s 掉到 4.1–5.2 mm 降雨時約 120 Mb/s（相關係數 −0.34，下載比上傳受影響大）。原因是 Ka/Ku-band 的雨衰；薄雲也能讓訊號強度掉約 10%。
    * 氣溫 > 12°C 時吞吐量掉 5–26%；天線只有鋁背板被動散熱。
    * 耗電：平均 **56.3 W**，最高 144.5 W（比早期報告的 105 W／190 W 低，可能是韌體優化）。下雪時融雪模式 10 小時，平均 146.3 W、最高 188.6 W。
    * **耗電和吞吐量沒有正相關**，有時甚至負相關，作者推測天線可能為了控制耗電而限速；並和奧地利的早期量測（175/17 Mb/s）比，懷疑韌體降耗電時也降了速度。有雨時耗電持續偏高（Fig. 12）。

## 6. 野外量測（§V）
### 6.1 北方海岸：Koeye Point（天線 F）
* 從溫哥華坐 2 小時飛機到 Bella Bella，再坐 2 小時船。和生物學家、Heiltsuk 原住民合作做鮭魚與森林保育。
* 天空 100% 無遮蔽，但**下載吞吐量比都市低約 68%，延遲多 11–30%**，斷線也更多。
* 作者的解釋：北方能覆蓋的衛星和 GS 比較少，加上第一殼層傾角的限制，換手更吃力。

### 6.2 高海拔山谷：Manning Park（天線 C）
* 海拔約 1,000 m、四周是 2,000 m 以上的山，沒有手機訊號。當時還不在官方服務範圍（預計 2023 年底），開啟 portability 模式。
* 延遲平均 90–350 ms，波動超過 1,000 ms；吞吐量平均下載 13、上傳 4 Mb/s。
* **每 1–3 分鐘斷線一次**，但 App 說遮蔽只有 2%。原因：只要衛星不在頭頂附近就會被山擋住，但 App 只看得到近距離的天空，算錯了遮蔽比例。
* 對照組：天線 C 在都市陽台時遮蔽 24.9%，體驗卻和 A、B 差不多。結論：**遮蔽比例不是全部，地形更重要；不是所有遮蔽都一樣**。
* 兩個地點都沒有電網，只能用太陽能–柴油混合系統或電池。

## 7. 進一步討論（§VI）
### 7.1 Bent-pipe 的穩定性：雙天線同步 8K 串流
* A、B 放在同一顆衛星範圍內：**共用同一條 GSL，各自有自己的 UL**。兩台 PC 同步循環播放 8K YouTube 超過 24 小時，每次播完清空 buffer。
* 結果（Table III）：

| 天線 | Buffer 斷流 | App 回報斷線 | 兩邊同時斷 | 同時斷的比例 |
|---|---|---|---|---|
| A | 8 | 8 | 3 | 37.50% |
| B | 19 | 13 | 3 | 15.79% |

* 兩邊同時斷，代表原因是**共用的 bent-pipe**（衛星或 GSL），不是個別天線。A 比 B 穩，因為 B 周圍樹多。結論：**同一服務區內的使用者，UL 品質也不一樣**。

### 7.2 行動性：車頂測試
* 天線 C 用兩根橫桿綁在小廂型車車頂（Fig. 14），時速 40–70 km/h 開 30 分鐘。天線偵測到移動後會轉成平躺（朝正上方），比固定時的傾斜更適合全方位接收。
* 斷線：中位數**每 16.5 秒斷 5 秒**；第 95 百分位是每 162 秒斷一次、最長 36 秒。
* 延遲平均 100 ms（固定時約 50 ms），最高 2,800 ms；每分鐘至少 2 次、最多 8 次超過 200 ms。轉彎（方向突變）和路樹遮蔽會造成尖峰。
* 下載吞吐量 80–100 Mb/s，和固定時差不多，最後幾分鐘在山谷直線路段突然下降，作者推測換手比預期快。上傳 5–10 Mb/s。
* 開機到可上網要 3–7 分鐘，曾超過 20 分鐘，不利行動使用。

### 7.3 全球覆蓋？技術和文化問題
* 要真正全球覆蓋，需要更多衛星、包括覆蓋南北極的其他殼層。
* **跨洲通訊和地表災害時的韌性，需要 ISL**。
* 實際挑戰：30 m 高的原始林只剩河岸看得到天空（會淹水）；熊、狼、鳥可能破壞天線；只能用太陽能，而天線最高要 145 W，所以只能斷斷續續運作。
* 文化挑戰：當地是 Heiltsuk 族的聖地。生物學家擔心「WiFi 污染」：在真正的野外分心滑手機反而危險。作者呼籲制定相關政策與倫理規範。

## 8. 和我們的 LEO 路由研究的關係
* **這是「沒有 ISL」的現實 baseline**。我們讀的 OPSPF、StableRoute、DoTD、LAMP、Motif 都假設 +Grid ISL 和星座內路由；這篇證明 2022 年的 Starlink 根本還沒有星座內路由，衛星只是彎管。所以那些論文的效益應該拿「single bent-pipe ＋ 地面網路」這個基準來比，而不是只和其他 ISL 路由演算法比。
* **跨洋是 ISL 最有價值的地方**：Sydney 多 78 ms，因為要先繞到 Seattle 再走海底電纜。ISL 可以讓流量在太空中跨洋，直到靠近目的地才落地。這也是 [[Enhancing_LEO_Mega-Constellations_with_Inter-Satellite_Links_Vision_and_Challenges]] 的核心論點。
* **第一跳／最後一跳的選擇很重要**：Starlink 一律接「最近的 GS」。這和 FROG（The Nearest Is Not The Fastest）、CSGI（[[Enabling_Low-latency-capable_Satellite-Ground_Topology_for_Emerging_LEO_Satellite_Networks]]）討論的問題是同一類：最近的不一定最好。有了 ISL 之後，「在哪顆衛星、哪個 GS 落地」會變成一個可以最佳化的變數。
* **延遲變動的來源**：bent-pipe 時代，延遲鋸齒狀上升再下降，來自單一衛星遠離和換手（約每 10 分鐘以內）。有 ISL 之後，換成 ISL 拓樸變動造成的換路，這正是 StableRoute（[[StableRoute_When_Dijkstras_Algorithm_Meets_Topology-Varying_Satellite_Networks]]）想減少的「路由抖動」。可以把這篇的 3.8 倍延遲變異當成「使用者會感受到的不穩」的實際數字。
* **鏈路不是完美的**：0.24–1.24% 的非壅塞突發掉包、雨衰 27%、地磁風暴、遮蔽造成的斷線。很多路由模擬把 GSL/UL 當成穩定鏈路、只考慮 ISL 斷線，這篇提醒 UL/GSL 本身就是主要的不穩定來源。對 OPSPF（[[OPSPF_Orbit_Prediction_Shortest_Path_First_Routing_for_Resilient_LEO_Satellite_Networks]]）這類「可預測的斷線用預測、不可預測的用 flooding」的設計來說，天氣和遮蔽就是典型的「不可預測故障」。
* **共用 GSL 的同時斷線（Table III）**：說明故障是相關的（correlated），不是每條鏈路獨立。做 survivability／failure 模擬時可以考慮共用元件的故障。
* **同一顆衛星下的兩個使用者還要繞地面**：表示 2022 年的 Starlink 沒有 on-board switching，衛星不會自己轉送封包。這是 ISL 路由論文假設的「衛星是路由器」和當時現實的落差。
* **時代背景**：量測是 2022 年上半年。之後 Starlink 已經開始用雷射 ISL（例如極區和海上服務），所以這篇的「沒有 ISL」結論**只適用於 2022 年**。這篇和 [[Starlink_SelfDriving_Walkthrough]]（MobiCom 2023）可以一起讀：那篇也說 2022 年的 Starlink 是 bent-pipe，所以它的 ISL 影響是模擬出來的。

## 9. 我們檢查到的問題
* **Jitter 的「5 倍以上」**：下載 0.255 / 0.041 ≈ 6.2 倍，上傳 2.715 / 0.546 ≈ 4.97 倍，上傳其實略低於 5 倍。
* **Bandwidth utilization 46.8% 對不上**：用 Table II 算，Starlink TCP 下載 / UDP 下載 = 78.4 / 200.3 ≈ 39.1%，和論文的 39.0% 一致；但地面網路的下載是 483.4 / 807.5 ≈ 59.9%，上傳才是 48.9 / 102.4 ≈ 47.8%。論文沒寫清楚 46.8% 怎麼算（可能是逐樣本平均）。如果用下載比，地面網路和 Starlink 的差距其實更大。另外 Starlink 的上傳 utilization 是 6.3 / 10.4 ≈ 61%，反而比地面網路高，論文沒有提。
* **Koeye Point 的解釋值得懷疑（我們的解讀）**：Koeye Point 約在北緯 51.8°，論文說是「北極圈以南 14°」（約 52.6°N）。對傾角 53° 的 Walker 殼層來說，衛星密度在接近 ±53° 緯度時反而**最高**（衛星在轉折緯度停留最久），所以「第一殼層傾角限制，衛星比較少」不太成立。比較可能的原因是附近 GS 少：bent-pipe 需要天線和 GS 在 1,000 km 內同時看到同一顆衛星。另外 F 是 Gen 2、只用 WiFi，和都市的 Gen 1 有線量測也不是完全同條件比較。
* **電池續航**：論文說「1,500 Ah 的電池只能撐約 1 小時」。以平均 56.3 W 算，就算是 1,500 Wh 也能撐 20 多小時，1,500 Ah（12 V 約 18 kWh）更久。數字或單位應該有誤（也可能是指某種特定的小型電源包）。
* **「single bent-pipe」的推論是間接的**：證據是 traceroute（只看得到 IP 層，看不到衛星層的轉送）、延遲差大致固定、延遲相關性。這些都和「只有一跳」一致，但不能完全排除 L2 層的轉送；作者也只說「沒找到 L2 routing 的證據」。
* **只有一個城市**：所有都市量測都在溫哥華，最近的 GS 在美國西北，所以「到美西最快」這種結果有地理偏差。9 個區域的比較其實是「從溫哥華出發的地面網路路徑」差異。
* **天氣分析沒有控制其他變數**：降雨、溫度、時段（12 小時週期）可能互相相關，論文只看單變數相關。
* **地磁風暴那兩天**同時有發射新衛星，兩個原因分不開，作者自己也承認。

## 出處說明
* 全文直接讀自使用者上傳的 INFOCOM 2023 PDF（10 頁）。Table I、II、III 的數字照原文；§5.2 的四欄平均和 §9 的倍數、utilization 是我們用 Table II 算的。
* Fig. 10 的相關係數是對照 PDF 頁面的熱圖讀的。
* §8、§9 中標「我們的解讀」的部分不是論文的說法。
