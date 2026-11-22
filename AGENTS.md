# AGENTS.md

## 目的

本檔案定義 `OneDirect Open Data` 目錄的資料規格、修改邊界、驗證流程及安全要求。

任何 coding agent 或開發者修改本目錄前，必須先閱讀本檔案及 `DATA_SCHEMA.md`。如兩者有衝突，以較嚴格、且不影響原始資料真實性的要求為準。

## 授權

- 本資料目錄必須使用 **Apache License 2.0**。
- 根目錄必須保留完整的 `LICENSE` 檔。
- `LICENSE` 必須包含 `SPDX-License-Identifier: Apache-2.0`。
- `DATA_SCHEMA.md` 的 License 狀態必須顯示 `Apache License 2.0`。
- 不得在未明確要求的情況下改成其他授權。
- 發佈或分發時，必須保留 `LICENSE` 及適用的來源標示。

## 最高優先級規則

1. `OneDirect Open Data` 以外的原始專案檔案必須保持唯讀。
2. 不得修改、移動、重新命名或刪除原始巴士、輕軌、伺服器、環境變數或設定檔。
3. 所有整理、轉換及輸出操作只可在本目錄內進行。
4. 不得把原始專案檔案複製到本目錄後保留其敏感或內部欄位。
5. 不得從 `.env`、服務設定、私有接口或未公開來源提取資料。

## 資料來源

### 巴士資料

巴士資料以 `ngsiolei/macau-bus-data` 的以下快照為準：

```text
commit: be48ee689184f718bf66699cfd87d4c54dfc49d8
date:   2023-01-27 07:36 +08:00
```

使用的來源檔：

| 來源 | 用途 |
| --- | --- |
| `routes.json` | 巴士路線、方向、站序及方向名稱 |
| `stops.json` | 平台／月台級站點、父站、名稱及座標 |
| `route-schedule/` | 路線方向、服務日、時段及班距 |
| `geojson/` | 巴士方向線形 |

### 輕軌資料

輕軌資料沿用原始專案內的非 Mock 資料：

| 來源 | 用途 |
| --- | --- |
| `server/data/lrt-lines.json` | 輕軌路線及站序 |
| `server/data/stations.json` | 輕軌站點及座標 |

### 禁止使用

以下資料不得加入本資料集：

- `src/data/mockData.js`
- `src/data/stationRoutesMock.js`
- `OneDirect-OpenSource` 內的虛構 fixtures
- 任何自行建立的虛擬路線、站點、座標、方向、班次、票價或 ETA

## 資料真實性

### 巴士路線

- `routeId` 必須保留來源路線編號，不得自行重新編號。
- `name` 及 `namePt` 必須逐筆對應來源名稱。
- `routeType` 只可使用來源已有的 `2-way` 或 `circular`。
- `directionCount` 必須等於來源方向陣列長度。
- `agencyNames` 只保留營運商名稱，不保留電話或網站。

### 巴士站點

- `stopId` 是平台／月台級代碼，不可當成父站代碼。
- `parentStopId` 保留來源父站關係。
- 同一站名可能有多個 `stopId`，不得為簡化資料而任意合併。
- 站名、葡文名稱及座標必須沿用來源，不得手動修正或推測。

### 路線站序

- `bus-route-stops.json` 的 `stopId` 必須存在於 `bus-stops.json`。
- `routeId` 必須存在於 `bus-routes.json`。
- `directionIndex` 必須存在於對應路線的方向集合。
- `sequence` 必須按照來源 `seq` 保留，不得重新排序。
- 同一路線方向內的站序必須連續。

### 排班

- 每筆排班必須包含來源已有的 `directionIndex` 及 `serviceType`。
- `serviceType` 只可使用 `weekdays`、`weekends` 或 `holidays`。
- `scheduleIndex` 是同一路線方向內按來源 CSV 行序產生的 1-based 索引。
- 不得把不同方向的班次合併後失去方向資訊。
- 不得把不同服務日類型合併為模糊的中文描述。

### 線形

- `bus-route-geometries.json` 必須為每個巴士路線方向提供一條 `LineString`。
- `routeId` 及 `directionIndex` 必須能對應路線站序。
- GeoJSON 座標次序固定為 `[longitude, latitude]`。
- 站點資料使用 `{ latitude, longitude }` 物件，兩者不可混淆。
- GeoJSON 中重複的站點 point features 不另行輸出，站點統一由 `bus-stops.json` 提供。

### 輕軌

- 輕軌來源沒有提供方向標記，不得自行加入 `directionIndex` 或反向站序。
- `lrt-line-stops.json` 的 `sequence` 是來源 `stations` 陣列次序，必須保持原順序。
- 輕軌站點、葡文名稱及線形必須沿用原始非 Mock 資料。

## 已知缺值及特殊情況

這些情況是來源本身的狀態，不是輸出錯誤：

| 情況 | 筆數 | 正確處理 |
| --- | ---: | --- |
| 巴士排班缺少開始或結束時間 | 30 | `start` / `end` 使用 `null` |
| 巴士排班缺少班距 | 30 | 最短／最長班距使用 `null` |
| `71S` 使用單一班距值 `15` | 2 | 最短及最長班距均使用 `15` |
| `701X` 方向 `1` 缺少排班 | 1 個方向 | 保留路線、站序及線形，不補班次 |
| 輕軌來源沒有方向標記 | - | 不建立推測方向 |

不得以平均值、預設值、前後時段或相鄰方向資料填補上述缺值。

## 目前資料筆數

| 檔案 | 筆數 |
| --- | ---: |
| `bus-routes.json` | 91 |
| `bus-route-schedules.json` | 950 |
| `bus-stops.json` | 586 |
| `bus-route-stops.json` | 2,843 |
| `bus-route-geometries.json` | 115 |
| `lrt-lines.json` | 3 |
| `lrt-stations.json` | 15 |
| `lrt-line-stops.json` | 17 |
| `lrt-line-geometries.json` | 3 |

修改資料後，必須同步更新 `DATA_SCHEMA.md` 的筆數、狀態表及相關說明。

## JSON 格式要求

- 編碼必須為 UTF-8。
- 不得包含 BOM。
- 欄位名稱必須為 `camelCase`。
- 不得加入 JSON 註解、undefined、NaN、Infinity 或尾隨逗號。
- 陣列及物件必須使用合法 JSON 語法。
- 日期及時間如有使用，必須保留來源格式；本資料集目前不包含 ETA 或實時資料。
- 不得把 `null` 改成 `0`、空字串、猜測值或不存在的欄位。

## 檔案關係

修改任何一個檔案時，必須同時檢查下列關係：

```text
bus-routes.json
  └── routeId
      ├── bus-route-stops.json
      ├── bus-route-schedules.json
      └── bus-route-geometries.json

bus-stops.json
  └── stopId
      └── bus-route-stops.json

lrt-lines.json
  └── lineId
      ├── lrt-line-stops.json
      └── lrt-line-geometries.json

lrt-stations.json
  └── stopId
      └── lrt-line-stops.json
```

任何未解析的 `routeId`、`stopId` 或 `lineId` 均視為錯誤。

## 安全及去敏化

不得輸出：

- API Key、Token、Cookie、Session 或認證憑證
- Authorization、Bearer 或其他自訂 Headers
- 後台 URL、私密接口參數或未公開端點
- 政府或第三方 API 端點 URL
- `sourceImage`、圖片路徑或抓取流程 metadata
- 營運商電話、網站或聯絡資料
- 內部調試欄位、provider／cache／retry 設定
- 內部映射規則、私有 ID 或未公開欄位

巴士及輕軌的公開路線編號、站點代碼、站名及座標可以保留，因為它們是交通業務資料本體。若來源日後加入新的 metadata，必須逐欄審核，不得直接複製。

## 文件及 Banner 要求

- `DATA_SCHEMA.md` 必須保留頂部 `banner.png`。
- Banner 必須是有效的 PNG，不可只留下 SVG 或失效圖片引用。
- Banner 不得包含 Mock Data、虛假統計或不存在的功能。
- Markdown 內所有相對檔案連結必須存在。
- 修改筆數或 schema 時，必須同步更新 Banner 上的主要統計數字。
- `banner-source.svg` 只作為 PNG 的可編輯來源，不得取代 PNG。

## 修改流程

1. 只讀檢查來源資料及 commit 版本。
2. 記錄修改前的來源檔案 SHA-256。
3. 確認要修改的欄位、筆數及關聯。
4. 只在本目錄產生或覆寫 JSON。
5. 如資料格式有變，先更新 JSON 再更新 `DATA_SCHEMA.md`。
6. 如有主要統計變更，同步更新 Banner 及其 SVG 來源。
7. 執行完整驗證。
8. 重新計算來源 SHA-256，確認原始檔案未變。
9. 回報實際筆數、驗證結果及原始專案狀態。

## 必須執行的驗證

### JSON 語法

- 每個 `.json` 都能被 `JSON.parse()` 解析。
- 所有 JSON 為 UTF-8 且無 BOM。
- 所有陣列及物件欄位符合 `camelCase`。

### 資料完整性

- `bus-routes.json` 的 `routeId` 不重複。
- `bus-stops.json` 的 `stopId` 不重複。
- `bus-route-stops.json` 的 `stopId` 都存在於站點資料。
- `bus-route-stops.json` 的 `routeId` 都存在於路線資料。
- 每條路線方向的站序連續。
- 每筆排班都能對應存在的 `routeId` 及 `directionIndex`。
- 每條巴士方向線形都能對應路線方向。
- 每條輕軌站序及線形都能對應 `lineId`。
- 每個輕軌 `lineIds` 引用都存在。
- 所有座標位於合法經緯度範圍。

### 去敏化

掃描所有輸出檔案，確認沒有以下字詞或實際值：

```text
http://
https://
apiKey
authorization
bearer
cookie
token
secret
signature
user-agent
headers
endpoint
sourcePage
sourceImage
website
phone
service_role
```

任何命中都必須人工審核，不得直接忽略。

### 文件一致性

- `DATA_SCHEMA.md` 的檔名及筆數與實際 JSON 一致。
- 所有 Markdown 本地檔案連結都存在。
- Banner PNG 可正常解碼，尺寸為 `1600 × 520`。
- Banner 顯示的統計與 JSON 一致。

## 完成條件

只有同時符合以下條件，才可視為完成：

- 所有 JSON 合法且可重用。
- 所有跨檔關聯有效。
- 沒有捏造或補值。
- `DATA_SCHEMA.md` 與實際資料一致。
- Banner PNG 有效且內容正確。
- `LICENSE` 存在並保持 Apache License 2.0。
- 敏感資料掃描零命中。
- 原始專案檔案未被修改。
- 修改者已在回覆中列出實際檔案、筆數及驗證結果。
