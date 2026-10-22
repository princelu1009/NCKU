---
title: "OSPF Version 2 (RFC 2328) — 讀 LEO 路由論文需要的重點"
authors: [John Moy]
venue: "IETF RFC 2328, April 1998（STD 54）"
tags: [OSPF, link-state, LSA, LSU, LSAck, LSDB, Hello, flooding, background]
---

## Citation
J. Moy, "OSPF Version 2," IETF RFC 2328 (STD 54), April 1998. 全文約 240 頁，免費公開：https://www.rfc-editor.org/rfc/rfc2328

## 一句話總結
OSPF 是 link-state 路由協定：**每台路由器用 Hello 確認鄰居還活著，把自己的鏈路狀態（LSA）裝進 LSU 用 flooding 傳給全網，收到的人要回 LSAck，沒回就重送**。最後每台都有一模一樣的 LSDB，各自以自己為根跑 Dijkstra，算出自己的路由表。讀 LEO 路由論文（OPSPF、StableRoute、OSPF in LEO）只需要 RFC 的 §1–§2、§4、§9–§10 的 Hello 和鄰居狀態、§13 flooding、§16 路由表計算，以及附錄的計時器，大約 30 頁。其他是封包格式、area、DR 選舉等實作細節，可以跳過。

> 讀法建議：先看課本（例如 Kurose & Ross《Computer Networking》第 5 章 link-state routing）打底，再讀本筆記的 §3（flooding）。OPSPF 和 OSPF 在突發故障時的差距，幾乎都出在 §3 講的 LSAck 和重送。

## 0. 先記住的名詞

| 名詞 | 全名 | 一句話 | 比喻 |
|---|---|---|---|
| LSA | Link State Advertisement | 一台路由器描述「我連到誰、成本多少」的一筆資料 | 一張明信片 |
| LSU | Link State Update | 裝一張或多張 LSA 的封包 | 裝明信片的信封 |
| LSAck | Link State Acknowledgment | 回覆「這張 LSA 我收到了」，只帶 LSA 的 header | 掛號回執 |
| LSDB | Link State Database | 收集到的所有 LSA，等於全網地圖；**同一個 area 內每台都一樣** | 全班共用的一本相簿 |
| 路由表 | Routing table | 用 LSDB 跑 Dijkstra 得到「去哪裡、下一跳走誰」；**每台都不同** | 每個路口自己的指路牌 |
| Hello | — | 定期對鄰居喊「我還在」，也用來確認雙向連通 | 打招呼 |
| Adjacency | 鄰接關係 | 真正要交換 LSDB 的鄰居關係；**LSU 只在 adjacency 上傳** | 正式交換筆記的同學 |

## 1. 整體概念（§1–§2）

* OSPF 是一個自治系統（AS）內部用的 **IGP**（Interior Gateway Protocol），用 link-state（SPF）技術，不是 Bellman-Ford（distance-vector）。
* **每台路由器有同一份 LSDB**：每台只負責描述自己的介面和鄰居，再把這份描述 flood 給全網。大家收集後，手上的地圖就一樣了。
* **每台以自己為根算最短路樹**：原文 "each router constructs a tree of shortest paths with itself as root"。轉送時只用到樹上的**下一跳**。
* **Equal-cost multipath（§2.4）**：同一個目的地有好幾條一樣短的路時，可以同時用，流量平分。
  * 👉 這就是 StableRoute 要處理的地方。OSPF 允許多條等長路，但一般實作的 Dijkstra 只取「第一個找到的」，所以拓樸一變就可能換路。
* §2.1 的 LSDB 是一張**有向圖**：點是路由器和網路，邊代表「這台路由器有介面接到這個網路」，邊上帶成本。
* §2.2 用 RT6 當例子，示範怎麼從 LSDB 算出 RT6 的路由表。

## 2. Hello 和鄰居狀態（§7.1、§9.5、§10）

### 2.1 Hello 在做什麼
* 每個介面定期送 Hello，間隔是 **HelloInterval**。
* **雙向確認**：如果鄰居的 Hello 裡列出了「我」，代表雙方都聽得到對方（2-Way）。
* 在廣播型網路上，Hello 也用來選 Designated Router（DR）。LEO 的 ISL 是點對點，這部分可以跳過。

### 2.2 怎麼判斷鄰居斷了
* 每個鄰居有一個 **Inactivity Timer**，長度是 **RouterDeadInterval**。收到 Hello 就重設；計時到了還沒收到，就判定鄰居 Down，拆掉 adjacency。
* 原文（附錄 C.3）建議 RouterDeadInterval 是 HelloInterval 的倍數，大約 4 倍。

### 2.3 鄰居狀態機（§10.1，只要知道順序）

| 狀態 | 意思 |
|---|---|
| Down | 還沒聽到對方 |
| Init | 聽到對方的 Hello，但對方還沒列出我 |
| 2-Way | 雙向確認了 |
| ExStart → Exchange | 互相交換 LSDB 的摘要（Database Description） |
| Loading | 用 Link State Request 要自己缺的 LSA |
| Full | LSDB 同步完成，正式的 adjacency |

* 👉 對 OPSPF 的意義：OSPF 發現鄰居斷了，要等 RouterDeadInterval（預設 40 s）。OPSPF 對極區斷線不用等，因為事先就預測到了；對突發故障則把 Hello 間隔設成 1 s，偵測得比較快。

## 3. Flooding（§13）：LSU、LSAck、重送 ⭐ 最重要

### 3.1 收到一個 LSU 時，對裡面每一張 LSA 做什麼（§13 步驟）
1. **檢查 checksum**，錯了就丟。
2. **檢查 LS type**，不認得就丟。
3. 外部路由 LSA 不送進 stub area（LEO 用不到）。
4. 收到 MaxAge（宣告作廢）且資料庫裡沒有的 LSA：回 ack，不安裝。
5. **比資料庫裡的新**（看 sequence number、checksum、age 判斷）：
   * 如果資料庫那份才剛裝不久（RFC 2328 是 MinLSArrival，1 s 內），丟掉，避免太頻繁。
   * 否則：**往其他介面 flood 出去**、從各鄰居的重送清單移除舊版、**裝進 LSDB**、（必要時）回 ack、**觸發重算路由表**。
6. 對方在 Database Exchange 中途要過的 LSA 又出現了：代表同步出錯，重新開始交換。
7. **和資料庫裡的一樣**（重複）：如果這張剛好在自己的重送清單上，就當作對方的「隱含 ack」（implied acknowledgment），從清單移除。
8. **比資料庫裡的舊**：丟掉。

* 👉 這就是 OPSPF 說的「收到 LSU 先做 coherence check，重複就丟」的來源。

### 3.2 什麼時候回 LSAck（§13.5）
* **Delayed ack**：收到比較新的 LSA、但沒有從同一個介面 flood 回去時，等一下再跟其他 ack 一起打包回。
* **Direct ack**：收到重複、而且不能算隱含 ack 時，立刻單獨回給對方。
* **不回**：如果自己已經把同一張 LSA 從收到的介面 flood 回去，對方收到就等於確認了。

### 3.3 重送（§13.6）
* flood 出去的 LSA 會放在該鄰居的 **Link state retransmission list**，**直到收到 ack 才移除**。
* 沒收到 ack，每隔 **RxmtInterval**（建議 5 s）重送一次，用 unicast，不用 multicast。
* 原文："these advertisements are retransmitted until they are acknowledged"。
* 鄰居真的斷了：一直重送，直到 Hello 機制判定 adjacency 拆掉，清單才清空。

### 3.4 小例子
A 發現 A–C 斷了：

1. A → B：**LSU**「A–C 斷了（seq 101）」
2. B → A：**LSAck**「seq 101 收到」
3. B → D：轉送 **LSU**
4. D → B：**LSAck**
   （如果 D 的 LSAck 掉了，B 等 RxmtInterval 後重送給 D）

每一跳都是「一個 LSU＋一個 LSAck」，封包數大約翻倍。

* 👉 **和 OPSPF 的關係**：OPSPF 的 dynamic module 只做 1、3，不做 2、4（沒有 LSAck，也沒有重送）。所以突發故障時 OPSPF 的封包少約 57%、收斂快約 82%。代價是 LSU 掉了沒人發現，那顆衛星的 LSDB 會一直是舊的，直到下一次更新。OPSPF 原文也承認自己的實作 "less reliable"。

## 4. 路由表計算（§16）
* 每個 area 用 LSDB 跑 **Dijkstra**，以自己為根建最短路樹（§16.1），再把 area 之間、AS 外部的路由加上去（§16.2–§16.4）。
* 只要 LSDB 有變（例如 §3.1 第 5 步裝了新 LSA），就要重算。
* 一個目的地可以有多個等長下一跳（equal-cost multipath）。
* 👉 OPSPF 的 static module 每 1 s 跑一次的「Shortest path first algorithm」就是這一步。StableRoute 改的也是這一步：保留等長下一跳的集合，再和舊表比。

## 5. 預設計時器和常數（附錄 B、C）

| 參數 | 意思 | 建議值 | 出處 |
|---|---|---|---|
| HelloInterval | Hello 間隔 | 10 s（LAN）；X.25 PDN 為 30 s | C.3 |
| RouterDeadInterval | 多久沒聽到 Hello 就判定鄰居 Down | HelloInterval 的倍數，約 4 倍 → 40 s | C.3 |
| RxmtInterval | LSA 沒收到 ack 時的重送間隔 | 5 s（LAN） | C.3 |
| InfTransDelay | 送一個 LSU 估計要的時間，加到 LSA 的 age 上 | 1 s | C.3 |
| MinLSInterval | 同一台路由器重新產生同一張 LSA 的最短間隔 | 5 s | B |
| MinLSArrival | 收到同一張 LSA 新版本的最短間隔 | 1 s | B |
| LSRefreshTime | 沒變也要定期重新廣播 LSA 的週期 | 30 min | B |
| MaxAge | LSA 最長存活時間，到了就作廢 | 1 hr | B |

* 👉 OSPF 收斂慢的來源：偵測要等 RouterDeadInterval（40 s 等級），每一跳又要等 ack、可能等重送（5 s 等級）。這是 OPSPF、"OSPF in LEO" 這類論文都會去調整或繞過的地方。OPSPF 的實驗把 Hello 設成 1 pkt/s。

## 6. 可以跳過的部分
* §3、§12、附錄 A：area 結構、LSA 的五種類型和封包格式（寫程式才需要）。
* §7.3–§7.4、§9.4：Designated Router／Backup DR 選舉（廣播型網路才用，LEO 的 ISL 是點對點）。
* §15 virtual link、§D 認證。

## 7. 和其他筆記的關聯
### 和 OPSPF（[[OPSPF_Orbit_Prediction_Shortest_Path_First_Routing_for_Resilient_LEO_Satellite_Networks]]）
* OPSPF 的 dynamic module = 本筆記 §2 的 Hello ＋ §3 的 LSU flooding，**但拿掉了 LSAck 和重送**（§3.3）。
* OPSPF 的 static module 每 1 s 做一次本筆記 §4 的 Dijkstra，輸入是用 GPS 預測出來的拓樸，不是 flood 來的 LSA。
* OPSPF 量收斂時間是「第一個 LSU 到最後一個 LSU 或 LSAck」，所以不包含 §2.2 的偵測等待時間。

### 和 StableRoute（[[StableRoute_When_Dijkstras_Algorithm_Meets_Topology-Varying_Satellite_Networks]]）
* StableRoute 改的是 §4：保留 §2.4 說的 equal-cost 下一跳集合，再和舊路由表比，能不換就不換。

### 和 OSPF in LEO（[[OSPF_in_Low_Earth_Orbit_Stability_overhead_and_routing_dynamics]]）
* 那篇直接在 Teledesic 288 顆的模擬器上跑沒改過的 FRR OSPF，用 ECMP 備援下一跳做快速復原。可以對照本筆記的計時器，看實際 OSPF 在 LEO 上卡在哪裡。

## 出處說明
* §1–§4 的內容和引文直接讀自 RFC 2328 原文。
* §13 的步驟、§13.5–§13.6 的 ack／重送規則和附錄 C.3 的參數，是對照 RFC 1583（OSPFv2 的前一版，這幾節幾乎一樣）在 freesoft.org 的分節頁面。RFC 2328 在第 5 步把 MinLSInterval 改成了 MinLSArrival，本筆記用 2328 的版本。
* §10 的鄰居狀態、§16 的流程和附錄 B 的常數，是依我對 RFC 的了解整理的，這次沒有逐字核對。引用前請以原文為準。
