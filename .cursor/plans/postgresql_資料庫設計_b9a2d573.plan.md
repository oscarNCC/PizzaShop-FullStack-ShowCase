---
name: PostgreSQL 資料庫設計
overview: 依現有 BackEnd 訂單 API、FrontEnd shared-types 與 analytics 需求，設計 PostgreSQL 資料表與關聯，並建議必備與選配欄位、索引與實作方式（含 Prisma）。
todos: []
isProject: false
---

# OpenTable PostgreSQL 資料庫設計計畫

## 現況摘要

- **BackEnd**：[OpenTable-BackEnd](OpenTable-BackEnd/) 目前用 **in-memory** 儲存，型別定義在 [src/types/index.ts](OpenTable-BackEnd/src/types/index.ts)：`CollectDataReq` / `CollectDataRecord`（含 `restaurantId`, `floorplanId`, `tableId`, `diners`, `items[]`, `profit`, `startTime`, `endTime`, `status`, `notes`）。
- **FrontEnd**：[shared-types](OpenTable-FrontEnd/packages/shared-types/src/) 已定義 `Restaurant`, `Floorplan`, `Table`, `Menu`, `MenuItem`, `Order`, `Reservation`, `User`；[api-spec](OpenTable-FrontEnd/docs/api-spec.md) 預留 `/api/me`（User + restaurants）與 `/api/analytics`（turnoverRate, avgProfit, popularByPartySize）。
- **Analytics**：[packages/analytics](OpenTable-FrontEnd/packages/analytics/src/) 需依 `restaurantId` + 時間區間查訂單，計算返桌率、平均盈利、依人數熱門菜。

---

## 建議資料表結構

### 1. 必備核心表（與現有 API / 指標對齊）


| 表名              | 用途   | 關鍵欄位                                                                                                                           |
| --------------- | ---- | ------------------------------------------------------------------------------------------------------------------------------ |
| **restaurants** | 餐廳主檔 | id (PK), name, created_at                                                                                                      |
| **floorplans**  | 樓面   | id (PK), restaurant_id (FK), name, layout (JSONB 可選)                                                                           |
| **tables**      | 桌位   | id (PK), floorplan_id (FK), name, max_people, is_enabled, status                                                               |
| **menus**       | 菜單   | id (PK), restaurant_id (FK), name                                                                                              |
| **menu_items**  | 品項   | id (PK), menu_id (FK), name, price, category, image                                                                            |
| **orders**      | 訂單主表 | id (PK), restaurant_id (FK), floorplan_id (FK), table_id (FK), diners, profit, start_time, end_time, status, notes, created_at |
| **order_items** | 訂單明細 | id (PK), order_id (FK), menu_item_id (FK), qty, pizza_options (JSONB 可選)                                                       |


- **orders** 與 **order_items** 分表可讓「依 menu_item 聚合」的 analytics（如 popular dishes）用 JOIN + GROUP BY 查詢，索引也好做；若先求簡單可暫時把 `items` 存成 orders 的一欄 JSONB，之後再拆。
- **tables** 的 `status` 建議型別：`free | occupied | reserved | disabled`（與 [restaurant.ts](OpenTable-FrontEnd/packages/shared-types/src/restaurant.ts) 一致）。

### 2. 建議新增（支援 api-spec / 擴充）


| 表名                   | 用途       | 關鍵欄位                                                         |
| -------------------- | -------- | ------------------------------------------------------------ |
| **users**            | 後台/登入    | id (PK), name, email, created_at                             |
| **restaurant_users** | 餐廳–人員多對多 | restaurant_id (FK), user_id (FK), role (可選)                  |
| **reservations**     | 預約（若要做）  | id (PK), table_id (FK), time, party_size, guest_name, status |


- **users** + **restaurant_users** 可支援 `GET /api/me` 回傳 `User` 與其 `restaurants[]`。
- **reservations** 對應 shared-types 的 `Reservation`，可之後再加。

### 3. 選配（README 提到的進階指標）


| 表名                                | 用途   | 關鍵欄位                       |
| --------------------------------- | ---- | -------------------------- |
| **labor_costs** 或 **shift_costs** | 人力成本 | 與訂單或時段關聯，用於計算 Labor Cost % |


- 可設計為「依 restaurant + 日期/時段」一筆，或與 orders 關聯，依你未來是否要區分班別再定。

---

## 關聯圖（概念）

```mermaid
erDiagram
  restaurants ||--o{ floorplans : has
  restaurants ||--o{ menus : has
  restaurants ||--o{ orders : has
  menus ||--o{ menu_items : has
  floorplans ||--o{ tables : has
  tables ||--o{ orders : at
  orders ||--o{ order_items : has
  menu_items ||--o{ order_items : referenced_by
  users ||--o{ restaurant_users : has
  restaurants ||--o{ restaurant_users : has
  tables ||--o{ reservations : optional

  restaurants { uuid id PK string name }
  floorplans { uuid id PK uuid restaurant_id FK }
  tables { uuid id PK uuid floorplan_id FK string status }
  menus { uuid id PK uuid restaurant_id FK }
  menu_items { uuid id PK uuid menu_id FK decimal price }
  orders { uuid id PK uuid table_id FK int diners decimal profit timestamptz start_time }
  order_items { uuid id PK uuid order_id FK uuid menu_item_id FK int qty jsonb pizza_options }
```



---

## PostgreSQL 實作要點

### 必備

1. **主鍵**：建議全用 `UUID`（或 `gen_random_uuid()`），與現有前端的 string id 相容，分散式與匯入較安全。
2. **時間欄位**：`start_time`, `end_time`, `created_at` 用 `TIMESTAMPTZ`，查詢時用 `from`/`to` 過濾（與 [HistoryQuery](OpenTable-BackEnd/src/types/index.ts) 一致）。
3. **訂單查詢索引**（[GET /history](OpenTable-BackEnd/README.md)）：
  - `orders(restaurant_id, start_time)` 或 `(restaurant_id, start_time, table_id)`，依實際 filter 取用。
  - 若有 `from`/`to` 區間查詢，`start_time` 上要有索引。
4. **外鍵**：`orders.table_id` → `tables.id`、`orders.restaurant_id` → `restaurants.id`、`order_items.order_id`、`order_items.menu_item_id`，確保關聯一致與可做 JOIN 報表。
5. **order_items.pizza_options**：用 `JSONB` 存 `PizzaOptions`（size, crust, toppings 等），與 [BackEnd types](OpenTable-BackEnd/src/types/index.ts) 的 `pizzaOptions?: unknown` 相容。

### 建議

1. **軟刪除**：若需保留歷史，可在 `orders` 加 `deleted_at TIMESTAMPTZ`，查詢時 `WHERE deleted_at IS NULL`。
2. **審計**：重要表可加 `created_at` / `updated_at`；若有多人編輯可加 `updated_by`（user_id）。
3. **約束**：`orders.diners`、`order_items.qty` 用 `CHECK (diners > 0)`、`CHECK (qty > 0)`；`tables.status` 用 `CHECK (status IN ('free','occupied','reserved','disabled'))`。
4. **若保留 JSONB 方案**：在 `orders` 設 `items JSONB` 也可行，之後要算「依品項銷量」可用 JSONB 函數或應用層展開；若要大量依 menu_item 聚合，仍建議 **order_items** 表 + 索引。

---

## 與現有後端銜接

- **Repository**：將 [orderRepository.ts](OpenTable-BackEnd/src/repositories/orderRepository.ts) 的 `IOrderRepository` 改為實作「Prisma（或 pg）版」：`create` 寫入 `orders` + `order_items`，`findHistory` 用 `restaurantId/from/to/tableId` 查 `orders`，`findById` / `update` 依 id 讀寫。
- **型別**：`CollectDataRecord` 的 `id`、`createdAt` 由 DB 產生；`items` 從 `order_items` 組回（含 `menu_item_id`、`qty`、`pizza_options`）。
- **依 README 建議**：採用 **Prisma + PostgreSQL** 可一次搞定 schema、migration、型別生成與連線，並在 [.env.example](OpenTable-BackEnd/.env.example) 加 `DATABASE_URL`。

---

## 建議新增的環境變數

在 [OpenTable-BackEnd/.env.example](OpenTable-BackEnd/.env.example) 增加：

- `DATABASE_URL=postgresql://user:password@localhost:5432/opentable`

---

## 總結：要加什麼、建議加什麼


| 類別     | 要加                                                                        | 建議加                                                     |
| ------ | ------------------------------------------------------------------------- | ------------------------------------------------------- |
| **表**  | restaurants, floorplans, tables, menus, menu_items, orders, order_items   | users, restaurant_users；reservations、labor/shift 成本表視需求 |
| **欄位** | 上述核心欄位 + orders 的 status/notes、order_items 的 pizza_options (JSONB)        | deleted_at、created_at/updated_at、CHECK 約束               |
| **索引** | orders(restaurant_id, start_time) 或 (restaurant_id, start_time, table_id) | order_items(menu_item_id) 若常做品項報表                       |
| **工具** | Prisma（或 raw pg）+ DATABASE_URL                                            | 用 Prisma migrations 管理 schema 版本                        |


依此設計即可支援現有 `/api/data/collect`、`/history`、`/orders/:id`、`PATCH /orders/:id`，以及未來的 `/api/analytics` 與 `/api/me`，並為返桌率、RevPASH、熱門菜、平均盈利（及可選的 Labor Cost %）提供查詢基礎。