# 商品服务对外接口契约（Product Service — Exposed APIs）

> 单元：Unit3 · 仓库：`awsome-shop-product-service` · 端口：`8002`
> 阶段：CONSTRUCTION · 版本：v1.0（依据实现生成，2026-06-11）
> 适用对象：API 网关（Unit6）、前端（Unit1）、兑换服务（Unit5）。
> **本文档严格对应已实现代码**；未实现项不在此列。变更接口须同步本文档。

---

## 0. 通用约定

### 0.1 统一响应包 `Result<T>`

**成功响应**（业务正常返回）：

```json
{ "code": "SUCCESS", "message": "操作成功", "data": { } }
```

- 成功：`code = "SUCCESS"`（字符串）。

**失败响应**（由全局异常处理器返回）：

```json
{ "code": 409001, "message": "资源已存在: SKU-001", "data": null }
```

- 失败：`code` 为整数错误码（HTTP 状态码 × 1000 + 序号，见 §5），`message` 为提示。
- 参数校验失败时 `data` 为字段错误明细列表（见 §5.1）。

> 注意：成功与失败的 `code` 类型不同（字符串 / 整数），调用方判断成功请使用 `code === "SUCCESS"`。

### 0.2 入口分类（决定网关行为）

| 类型 | 路径前缀 | 网关认证 | 说明 |
|------|----------|----------|------|
| public | `/api/v1/public/product/**`、`/api/v1/public/category/**` | ❌ 无需登录 | 商品浏览/搜索、类目查询 |

当前所有已实现接口均为 public 类型；网关对 `/api/v1/product/**`（protected）已预留路由，但本服务暂无该前缀下的接口。

### 0.3 请求风格

- 所有接口统一使用 `POST` + JSON body（包括查询类接口）。
- `Content-Type: application/json`。
- 服务忽略未知 JSON 字段（`fail-on-unknown-properties: false`），兼容网关注入的额外参数。

---

## 1. 商品接口（Product）

### 1.1 创建商品 — `POST /api/v1/public/product/create`

请求体：

```json
{
  "name": "iPhone 15 Pro",
  "sku": "SKU-IP15P-001",
  "category": "数码产品",
  "brand": "Apple",
  "pointsPrice": 9999,
  "marketPrice": 7999.00,
  "stock": 100,
  "status": 1,
  "description": "A17 Pro 芯片，钛金属设计",
  "imageUrl": "https://cdn.example.com/ip15p.jpg",
  "subtitle": "钛金属，超强性能",
  "deliveryMethod": "顺丰包邮",
  "serviceGuarantee": "7天无理由退换",
  "promotion": "新品首发",
  "colors": "原色钛金属,蓝色钛金属,白色钛金属",
  "specs": [ { "屏幕尺寸": "6.1英寸" }, { "存储容量": "256GB" } ]
}
```

| 字段 | 类型 | 必填 | 约束 |
|------|------|------|------|
| name | string | 是 | 商品名称，≤200 字符 |
| sku | string | 是 | 商品编号，≤100 字符，**全局唯一** |
| category | string | 是 | 商品分类（类目名称），≤100 字符 |
| brand | string | 否 | 品牌，≤100 字符 |
| pointsPrice | int | 是 | 积分价格 |
| marketPrice | decimal | 否 | 市场参考价 |
| stock | int | 否 | 库存，默认 0 |
| status | int | 否 | 0=已下架（默认）/ 1=已上架 |
| description | string | 否 | 商品描述 |
| imageUrl | string | 否 | 主图 URL，≤500 字符 |
| subtitle | string | 否 | 副标题/卖点，≤500 字符 |
| deliveryMethod | string | 否 | 配送方式，≤200 字符 |
| serviceGuarantee | string | 否 | 服务保障，≤500 字符 |
| promotion | string | 否 | 促销活动，≤200 字符 |
| colors | string | 否 | 可选颜色（逗号分隔），≤500 字符 |
| specs | array | 否 | 规格参数，`[{ "规格名": "规格值" }]` 列表 |

响应 `data`：`ProductDTO`（见 §3.1，返回完整新建商品）。

失败：SKU 已存在返回 `409001`（HTTP 409）。

### 1.2 商品列表查询 — `POST /api/v1/public/product/list`

请求体：

```json
{ "page": 1, "size": 20, "name": "iPhone", "category": "数码产品" }
```

| 字段 | 类型 | 必填 | 约束 |
|------|------|------|------|
| page | int | 否 | 默认 1，最小 1 |
| size | int | 否 | 默认 20，最小 1，最大 100 |
| name | string | 否 | 商品名称，**模糊匹配**（LIKE %name%） |
| category | string | 否 | 商品分类，**精确匹配** |

响应 `data`：`PageResult<ProductDTO>`

```json
{
  "current": 1, "size": 20, "total": 35, "pages": 2,
  "records": [ /* ProductDTO[]，见 §3.1 */ ]
}
```

说明：结果按 `createdAt` 倒序排列；仅返回未逻辑删除的商品（不区分上下架状态，调用方按需用 `status` 过滤展示）。

---

## 2. 类目接口（Category）

### 2.1 查询类目列表（树形结构） — `POST /api/v1/public/category/list`

请求体：

```json
{ "name": "数码", "status": 1 }
```

| 字段 | 类型 | 必填 | 约束 |
|------|------|------|------|
| name | string | 否 | 类目名称，**模糊匹配** |
| status | int | 否 | 0=禁用 / 1=启用，不传查全部 |

响应 `data`：`List<CategoryDTO>`（两级树形结构）

```json
[
  {
    "id": 1, "name": "数码产品", "parentId": null,
    "icon": "digital", "sortOrder": 100, "status": 1,
    "description": "手机、电脑、配件",
    "productCount": 12,
    "children": [
      {
        "id": 5, "name": "手机", "parentId": 1,
        "icon": null, "sortOrder": 100, "status": 1,
        "description": null, "productCount": 8, "children": []
      }
    ]
  }
]
```

说明：
- 一级类目为 `parentId = null` 的记录，子类目挂载在 `children` 下（仅两级）。
- 一级与子类目均按 `sortOrder` 降序、`id` 升序排列。
- `productCount` 为该**类目名称**下的商品数量（商品表按 `category` 名称关联统计，未匹配则为 0）。
- 模糊查询命中子类目时，若其父类目未同时命中，该子类目不会出现在结果树中（树仅由命中记录组装）。

---

## 3. 数据结构

### 3.1 `ProductDTO`

```json
{
  "id": 1,
  "name": "iPhone 15 Pro",
  "sku": "SKU-IP15P-001",
  "category": "数码产品",
  "brand": "Apple",
  "pointsPrice": 9999,
  "marketPrice": 7999.00,
  "stock": 100,
  "soldCount": 5,
  "status": 1,
  "description": "A17 Pro 芯片，钛金属设计",
  "imageUrl": "https://cdn.example.com/ip15p.jpg",
  "subtitle": "钛金属，超强性能",
  "deliveryMethod": "顺丰包邮",
  "serviceGuarantee": "7天无理由退换",
  "promotion": "新品首发",
  "colors": "原色钛金属,蓝色钛金属,白色钛金属",
  "specs": [ { "屏幕尺寸": "6.1英寸" }, { "存储容量": "256GB" } ],
  "createdAt": "2026-06-11T10:00:00",
  "updatedAt": "2026-06-11T10:00:00"
}
```

| 字段 | 说明 |
|------|------|
| sku | 商品编号，全局唯一 |
| category | 商品分类（类目名称，字符串关联，非类目 ID） |
| pointsPrice | 积分价格（兑换所需积分） |
| soldCount | 已兑换数量 |
| status | 0=已下架 / 1=已上架 |
| specs | 规格参数列表，每项为单键值对 Map |

### 3.2 `CategoryDTO`

| 字段 | 类型 | 说明 |
|------|------|------|
| id | long | 类目 ID |
| name | string | 类目名称 |
| parentId | long/null | 上级类目 ID，null 表示一级类目 |
| icon | string | 类目图标名称 |
| sortOrder | int | 排序权重（越大越靠前） |
| status | int | 0=禁用 / 1=启用 |
| description | string | 类目描述 |
| productCount | long | 该类目名称下的商品数量 |
| children | array | 子类目列表（`CategoryDTO[]`） |

### 3.3 `PageResult<T>`

```json
{ "current": 1, "size": 20, "total": 100, "pages": 5, "records": [ /* T[] */ ] }
```

---

## 4. 接口速览

| 方法 | 路径 | 类型 | 用途 |
|------|------|------|------|
| POST | `/api/v1/public/product/create` | public | 创建商品 |
| POST | `/api/v1/public/product/list` | public | 商品分页查询（名称模糊 + 分类筛选） |
| POST | `/api/v1/public/category/list` | public | 类目树查询（含商品数量统计） |

---

## 5. 错误码与 HTTP 映射

失败响应 `code` 为整数：HTTP 状态码 × 1000 + 序号（如 `CONFLICT_001` → `409001`）。

| code | HTTP | 含义 | 触发场景 |
|------|------|------|----------|
| `400002` | 400 | 请求参数无效 | Bean Validation 失败（必填缺失、超长等） |
| `404001` | 404 | 资源不存在 | 按 ID 查询商品不存在（内部使用） |
| `409001` | 409 | 资源已存在 | 创建商品时 SKU 重复 |
| `500001` | 500 | 系统异常 | SystemException |
| `500002` | 500 | 系统错误 | 未预期异常兜底 |

### 5.1 参数校验失败响应示例

```json
{
  "code": 400002,
  "message": "请求参数无效",
  "data": [
    { "field": "name", "message": "商品名称不能为空" },
    { "field": "sku", "message": "商品编号不能为空" }
  ]
}
```

---

## 6. 与其他服务的交互关系

| 调用方 | 调用我的接口 | 场景 |
|--------|-------------|------|
| 前端 (Unit1) | `POST /public/product/list`、`POST /public/category/list` | 商品浏览、搜索、分类导航 |
| 前端 (Unit1) | `POST /public/product/create` | 商品录入 |
| 网关 (Unit6) | 路由 `/api/v1/public/product/**`、`/api/v1/public/category/**` → `:8002` | 无需认证直接转发 |

> 兑换服务（Unit5）侧的商品查询/库存扣减等内部（private）接口尚未实现，不在本契约范围内。

---

## 7. 其他说明

- **OpenAPI 文档**：服务集成 SpringDoc，可经网关访问 `/v3/api-docs/product` 或聚合 Swagger UI（`GET /swagger-ui.html`）查看在线文档。
- **Test 接口**：`/api/v1/public/test/**` 为脚手架示例接口（增删改查模板），非业务契约，外部不应依赖。
