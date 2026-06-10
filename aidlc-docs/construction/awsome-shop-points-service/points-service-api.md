# 积分服务对外接口契约（Points Service — Exposed APIs）

> 单元：Unit4 · 仓库：`awsome-shop-points-service` · 端口：`8003`
> 阶段：CONSTRUCTION · 版本：v1.0（依据实现生成，2026-06-10）
> 适用对象：API 网关（Unit6）、前端（Unit1）、认证服务（Unit2）、兑换服务（Unit5）。
> **本文档严格对应已实现代码**；未实现项不在此列。变更接口须同步本文档。

---

## 0. 通用约定

### 0.1 统一响应包 `Result<T>`
所有 HTTP 响应均包裹为：

```json
{ "code": "SUCCESS", "message": "操作成功", "data": { } }
```

- 成功：`code = "SUCCESS"`。
- 失败：`code` 为错误码（见 §5），`message` 为提示，`data = null`。

### 0.2 入口分类（决定网关行为）
| 类型 | 路径前缀 | 网关认证 | 说明 |
|------|----------|----------|------|
| public | `/api/v1/public/point/**` | 需登录（网关注入 operatorId） | 员工查自己的积分 |
| protected | `/api/v1/point/**` | 需登录 + ADMIN 角色 | 管理员操作 |
| private | `/api/v1/private/point/**` | 仅内网可达 | 供 auth-service / order-service 调用 |

### 0.3 operatorId 注入机制
- 网关认证通过后，通过 `OperatorIdInjectionFilter` 将 `operatorId`（字符串类型的用户 ID）注入到 POST JSON body 中。
- 积分服务从 request body 的 `operatorId` 字段获取当前操作人。
- private 接口不依赖 operatorId，直接通过请求体传递 `userId`。

---

## 1. Public 接口（员工端，需登录）

### 1.1 查询积分余额 — `POST /api/v1/public/point/balance/get`
请求体：
```json
{ "operatorId": "10" }
```
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| operatorId | string | 是 | 网关自动注入，当前用户 ID |

响应 `data`：
```json
{ "userId": 10, "balance": 500 }
```

### 1.2 查询积分变动记录 — `POST /api/v1/public/point/transaction/list`
请求体：
```json
{ "operatorId": "10", "page": 1, "size": 20 }
```
| 字段 | 类型 | 必填 | 约束 |
|------|------|------|------|
| operatorId | string | 是 | 网关注入 |
| page | int | 否 | 默认 1，最小 1 |
| size | int | 否 | 默认 20，最小 1，最大 100 |

响应 `data`：`PageResult<PointsTransactionDTO>`
```json
{
  "current": 1, "size": 20, "total": 5, "pages": 1,
  "records": [
    {
      "id": 1, "userId": 10, "type": "GRANT", "amount": 100,
      "balanceAfter": 100, "orderRef": null,
      "reason": "入职奖励", "operatorId": null,
      "createdAt": "2026-06-10T17:25:15"
    }
  ]
}
```

### 1.3 查询即将过期积分 — `POST /api/v1/public/point/expiring/get`
请求体：
```json
{ "operatorId": "10" }
```
响应 `data`：
```json
{ "expiringAmount": 200, "earliestExpireAt": "2026-07-01T00:00:00", "batchCount": 2 }
```
说明：返回 30 天内即将过期的积分汇总。

---

## 2. Protected 接口（管理员端，需 ADMIN 角色）

### 2.1 手动调整用户积分 — `POST /api/v1/point/adjust`
请求体：
```json
{ "operatorId": "1", "userId": 10, "delta": 50, "reason": "特殊奖励" }
```
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| operatorId | string | 是 | 网关注入，管理员 ID |
| userId | long | 是 | 目标用户 ID |
| delta | long | 是 | 调整额度（正数增加，负数减少） |
| reason | string | 是 | 调整原因（1-500 字符） |

响应 `data`：`null`（成功即可）。
失败：余额不足（扣减时）返回 `POINTS_001`。

### 2.2 获取积分规则配置 — `POST /api/v1/point/rule/get`
请求体：`{}`（无需参数）
响应 `data`：
```json
{
  "id": 1, "onboardingBonus": 100, "periodicAmount": 50,
  "periodicCycle": "MONTHLY", "validityDays": 365
}
```

### 2.3 更新积分规则配置 — `POST /api/v1/point/rule/update`
请求体：
```json
{
  "operatorId": "1",
  "onboardingBonus": 200,
  "periodicAmount": 100,
  "periodicCycle": "MONTHLY",
  "validityDays": 180
}
```
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| onboardingBonus | long | 是 | 入职奖励额度（≥0） |
| periodicAmount | long | 是 | 周期发放额度（≥0） |
| periodicCycle | string | 是 | DAILY / WEEKLY / MONTHLY |
| validityDays | int | 是 | 积分有效期天数（0=永不过期） |

响应 `data`：`null`。

### 2.4 查看用户积分变动记录（管理员） — `POST /api/v1/point/transaction/list`
请求体：
```json
{ "operatorId": "1", "userId": 10, "page": 1, "size": 20 }
```
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| operatorId | string | 是 | 管理员 ID |
| userId | long | 是 | 目标用户 ID |
| page / size | int | 否 | 分页参数 |

响应 `data`：`PageResult<PointsTransactionDTO>`（格式同 §1.2）。

---

## 3. Private 接口（仅内网，供其他微服务调用）

### 3.1 发放积分 — `POST /api/v1/private/point/grant`
> 调用方：auth-service（注册后发放入职奖励）、定时任务（周期性发放）

请求体：
```json
{ "userId": 10, "amount": 100, "grantType": "ONBOARDING", "reason": "入职奖励" }
```
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| userId | long | 是 | 目标用户 ID |
| amount | long | 是 | 发放金额（>0） |
| grantType | string | 是 | ONBOARDING / PERIODIC / MANUAL / OTHER |
| reason | string | 否 | 发放说明 |

响应 `data`：`null`。
行为：创建积分批次（含过期时间）→ 更新余额 → 写 GRANT 流水。

### 3.2 扣减积分 — `POST /api/v1/private/point/deduct`
> 调用方：order-service（兑换下单时）

请求体：
```json
{ "userId": 10, "amount": 50, "orderRef": "ORDER-20260610-001" }
```
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| userId | long | 是 | 目标用户 ID |
| amount | long | 是 | 扣减金额（>0） |
| orderRef | string | 是 | 关联订单号（幂等标识） |

响应 `data`：`null`。
行为：校验余额 → FIFO 消耗批次 → 更新余额 → 写 DEDUCT 流水。
失败：余额不足返回 `POINTS_001`。

### 3.3 退回积分 — `POST /api/v1/private/point/refund`
> 调用方：order-service（兑换取消时）

请求体：
```json
{ "userId": 10, "amount": 50, "orderRef": "ORDER-20260610-001" }
```
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| userId | long | 是 | 目标用户 ID |
| amount | long | 是 | 退回金额（>0） |
| orderRef | string | 是 | 关联订单号 |

响应 `data`：`null`。
行为：创建新批次 → 更新余额 → 写 REFUND 流水。

---

## 4. 数据结构

### 4.1 `PointsTransactionDTO`
```json
{
  "id": 1, "userId": 10, "type": "GRANT", "amount": 100,
  "balanceAfter": 100, "orderRef": null,
  "reason": "入职奖励", "operatorId": null,
  "createdAt": "2026-06-10T17:25:15"
}
```
| 字段 | 说明 |
|------|------|
| type | GRANT / DEDUCT / REFUND / EXPIRE / ADJUST |
| amount | 正数=增加，负数=减少 |
| orderRef | 关联订单号（DEDUCT/REFUND 时有值） |
| operatorId | 手动调整时的操作人 ID |

### 4.2 `PageResult<T>`
```json
{ "current": 1, "size": 20, "total": 100, "pages": 5, "records": [ /* T[] */ ] }
```

---

## 5. 错误码与 HTTP 映射

错误码前缀决定 HTTP 状态：`POINTS_`→422、`NOT_FOUND_`→404、`PARAM_`→400、`CONFLICT_`→409。

| code | 含义 | 触发场景 |
|------|------|----------|
| `POINTS_001` | 积分余额不足 | deduct / adjust（扣减超余额） |
| `NOT_FOUND_002` | 积分账户不存在 | — |
| `NOT_FOUND_003` | 积分规则配置不存在 | getRule |
| `PARAM_001` | 扣减金额必须大于0 | deduct |
| `PARAM_002` | 发放金额必须大于0 | grant |
| `CONFLICT_002` | 重复操作 | 幂等校验（预留） |

---

## 6. 接口速览

| 方法 | 路径 | 类型 | 用途 |
|------|------|------|------|
| POST | `/api/v1/public/point/balance/get` | public | 查询余额 |
| POST | `/api/v1/public/point/transaction/list` | public | 查询变动记录 |
| POST | `/api/v1/public/point/expiring/get` | public | 即将过期积分 |
| POST | `/api/v1/point/adjust` | protected | 手动调整积分 |
| POST | `/api/v1/point/rule/get` | protected | 获取规则 |
| POST | `/api/v1/point/rule/update` | protected | 更新规则 |
| POST | `/api/v1/point/transaction/list` | protected | 管理员查用户变动 |
| POST | `/api/v1/private/point/grant` | private | 发放积分 |
| POST | `/api/v1/private/point/deduct` | private | 扣减积分 |
| POST | `/api/v1/private/point/refund` | private | 退回积分 |

---

## 7. 定时任务（非 API，内部调度）

| 任务 | 触发频率 | 说明 |
|------|----------|------|
| 积分过期处理 | 每天 02:00 | 扫描到期批次，FIFO 失效，扣余额，写 EXPIRE 流水 |
| 周期性发放 | 每月 1 号 03:00 | 按规则给所有用户发放积分 |

---

## 8. 与其他服务的交互关系

| 调用方 | 调用我的接口 | 场景 |
|--------|-------------|------|
| auth-service (Unit2) | `POST /private/point/grant` | 注册成功后发放入职奖励 |
| order-service (Unit5) | `POST /private/point/deduct` | 兑换下单扣减积分 |
| order-service (Unit5) | `POST /private/point/refund` | 兑换取消退回积分 |
| 前端 (Unit1) | public + protected 接口 | 员工积分中心 + 管理员积分管理 |
