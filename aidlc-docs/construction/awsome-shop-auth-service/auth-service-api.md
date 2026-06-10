# 认证服务对外接口契约（Auth Service — Exposed APIs）

> 单元：Unit2 · 仓库：`awsome-shop-auth-service` · 端口：`8001`
> 阶段：CONSTRUCTION · 版本：v1.0（依据实现生成，2026-06-10）
> 适用对象：API 网关（Unit6）、前端（Unit1）及其他下游。
> **本文档严格对应已实现代码**；未实现项不在此列。变更接口须同步本文档。

---

## 0. 通用约定

### 0.1 统一响应包 `Result<T>`
除「内部接口」外，所有 HTTP 响应均包裹为：

```json
{ "code": "SUCCESS", "message": "操作成功", "data": { } }
```

- 成功：`code = "SUCCESS"`。
- 失败：`code` 为错误码（见 §5），`message` 为提示，`data = null`。

### 0.2 入口分类（决定网关行为）
| 类型 | 路径前缀 | 网关认证 | 说明 |
|------|----------|----------|------|
| public | `/api/v1/public/**` | 免认证 | 注册、登录、登出 |
| protected | `/api/v1/**`（非 public） | 需有效 JWT | 管理端用户接口 |
| internal | `/api/v1/internal/**` | 仅网关/内网可达 | 令牌校验，**不对前端暴露** |

### 0.3 JWT 约定
- 算法：HMAC-SHA256；默认有效期 `7200s`。
- Claims：`sub`=userId、`username`、`role`（`EMPLOYEE`/`ADMIN`）、`iss`、`iat`、`exp`。
- 登出后令牌进入黑名单（Redis），校验时视为无效。
- **网关在认证通过后应向下游注入** `X-User-Id`、`X-User-Role`（来自 §3 校验结果）。

---

## 1. Public 接口（免认证）

### 1.1 用户注册 — `POST /api/v1/public/auth/register`
请求体：
```json
{ "username": "alice", "email": "alice@amazon.com", "password": "secret123", "nickname": "Alice" }
```
| 字段 | 类型 | 必填 | 约束 |
|------|------|------|------|
| username | string | 是 | 3–50 字符，唯一 |
| email | string | 是 | 合法邮箱；域名须在白名单（`shop.auth.email.allowed-domains`） |
| password | string | 是 | 6–64 字符 |
| nickname | string | 否 | — |

响应 `data`（注册成功**自动登录**，返回 JWT）：
```json
{ "token": "<JWT>", "userId": 10, "username": "alice", "email": "alice@amazon.com", "nickname": "Alice", "role": "EMPLOYEE" }
```
说明：新用户角色固定 `EMPLOYEE`；注册后会触发入职奖励积分（依赖积分服务，当前为开关控制，失败不影响注册）。

### 1.2 用户登录 — `POST /api/v1/public/auth/login`
> 登录方式：**仅用户名 + 密码**。
```json
{ "username": "alice", "password": "secret123" }
```
响应 `data`：
```json
{ "token": "<JWT>", "userId": 10, "username": "alice", "nickname": "Alice", "role": "EMPLOYEE" }
```
失败：凭据错误 `AUTH_001`；账户禁用 `AUTH_003`；多次失败锁定 `AUTH_002`（含剩余分钟）。

### 1.3 用户登出 — `POST /api/v1/public/auth/logout`
- 请求头：`Authorization: Bearer <JWT>`（无 body）。
- 行为：将令牌加入黑名单。响应 `data = null`。

---

## 2. Protected 接口（需有效 JWT；管理端）

> ⚠️ 设计定位为**管理员（ADMIN）**操作，服务端当前未做角色校验，由网关 `RoleAuthorizationFilter` 统一鉴权（Unit6 已实现；这两个路径已纳入网关 `gateway.security.admin-paths`）。

### 2.1 用户详情 — `POST /api/v1/auth/user/detail`
```json
{ "userId": 10 }
```
响应 `data`：`UserDTO`（见 §4.1）。不存在返回 `AUTH_005`。

### 2.2 变更用户角色 — `POST /api/v1/auth/user/role`
```json
{ "userId": 10, "role": "ADMIN" }
```
- `role` 仅允许 `EMPLOYEE` / `ADMIN`，否则 `PARAM_102`。
- 响应 `data`：变更后的 `UserDTO`。

### 2.3 用户列表分页 — `POST /api/v1/public/auth/user/list`
> ⚠️ **已知问题**：当前挂在 public 路径，按设计应为受保护的管理员接口，后续应迁移到 `/api/v1/auth/user/list`。
```json
{ "page": 1, "size": 20, "username": null, "role": null, "status": null }
```
响应 `data`：`PageResult<UserDTO>`（见 §4.2）。

---

## 3. Internal 接口（仅网关调用，不对前端暴露）

### 3.1 令牌校验 — `POST /api/v1/internal/auth/validate`
> 供 API 网关在认证过滤器中调用。**响应不包裹 `Result`**，直接返回下列结构，便于网关反序列化。

请求体：
```json
{ "token": "<JWT>" }
```
响应：
```json
{ "success": true, "userId": 10, "role": "EMPLOYEE", "message": null }
```
| 字段 | 类型 | 说明 |
|------|------|------|
| success | boolean | 是否有效 |
| userId | number | 有效时返回；网关据此注入 `X-User-Id` |
| role | string | 有效时返回；网关据此注入 `X-User-Role` |
| message | string | 失败原因（无效/过期/已登出） |

> 网关对接状态：✅ 已对齐。Unit6 网关已采用本契约的 `userId` + `role`，并据此注入
> `X-User-Id` / `X-User-Role`（详见 `construction/awsome-shop-gateway-service/gateway-service-api.md`）。

---

## 4. 数据结构

### 4.1 `UserDTO`
```json
{
  "id": 10, "username": "alice", "email": "alice@amazon.com",
  "nickname": "Alice", "role": "EMPLOYEE", "status": "ACTIVE",
  "lastLoginAt": "2026-06-10T16:00:00", "createdAt": "...", "updatedAt": "..."
}
```
`status`：`ACTIVE` / `LOCKED` / `DISABLED`。

### 4.2 `PageResult<T>`
```json
{ "current": 1, "size": 20, "total": 100, "pages": 5, "records": [ /* T[] */ ] }
```

---

## 5. 错误码与 HTTP 映射

错误码前缀决定 HTTP 状态：`AUTH_`→401、`AUTHZ_`→403、`PARAM_`→400、`NOT_FOUND_`→404、`CONFLICT_`→409、`LOCKED_`→423、`SYS_`→500。

| code | 含义 |
|------|------|
| `AUTH_001` | 用户名或密码错误 |
| `AUTH_002` | 账户已锁定，请 N 分钟后重试 |
| `AUTH_003` | 账户已被禁用 |
| `AUTH_004` | Token 无效或已过期 |
| `AUTH_005` | 用户不存在 |
| `CONFLICT_001` | 用户名已存在 |
| `CONFLICT_002` | 邮箱已被注册 |
| `PARAM_101` | 邮箱域名不在允许范围内 |
| `PARAM_102` | 角色取值非法（仅 EMPLOYEE/ADMIN） |

---

## 6. 接口速览

| 方法 | 路径 | 类型 | 用途 |
|------|------|------|------|
| POST | `/api/v1/public/auth/register` | public | 注册（自动返回 JWT） |
| POST | `/api/v1/public/auth/login` | public | 登录 |
| POST | `/api/v1/public/auth/logout` | public | 登出 |
| POST | `/api/v1/public/auth/user/list` | public⚠️ | 用户分页（应迁移为 protected） |
| POST | `/api/v1/auth/user/detail` | protected | 用户详情 |
| POST | `/api/v1/auth/user/role` | protected | 变更角色 |
| POST | `/api/v1/internal/auth/validate` | internal | 令牌校验（网关用，无 Result 包裹） |

---

## 7. 未在本契约内的能力（备注）

- **入职奖励积分发放（FR-A6）**：为出站调用积分服务，非对外 API；当前由开关 `shop.points.onboarding.enabled` 控制（默认关闭），详见 `awsome-shop-auth-service/docs/IMPLEMENTATION_NOTES.md`。
