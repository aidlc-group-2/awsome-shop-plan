# API 网关对外接口文档（awsome-shop-gateway-service / Unit6）

> 阶段：CONSTRUCTION
> 模块：Unit6 · API 网关（Spring Cloud Gateway，reactive/WebFlux）
> 监听端口：**8080**（local/dev/staging/prod），docker profile 为 8081
> 关联设计：`aidlc-docs/inception/`（FR-G1~G5）；实现说明：网关仓 `docs/IMPLEMENTATION_NOTES.md`
> 生成时间：2026-06-10

---

## 1. 概述

网关是 AWSomeShop 所有外部请求的**统一入口**。它**没有自己的业务 REST 接口**（运行包 `bootstrap` 仅打包
`gateway-impl/common/domain-api/application-api`，源码树中的 `Test*` 控制器属未打包的脚手架残留）。
因此本文档的"对外接口"指：

1. **路由入口**（客户端实际访问的路径 → 转发到的后端服务）；
2. **横切行为契约**（认证、身份透传、角色鉴权、限流、安全头、CORS、统一错误响应）；
3. **运维与文档端点**（actuator、Swagger 聚合）。

```
Client ──HTTP──> Gateway(:8080)
                   ├─ RateLimit → SecurityHeaders → JWT 校验 → 角色鉴权 → 身份注入
                   └─ 路由转发 → auth(8001) / product(8002) / point(8003) / order(8004)
```

---

## 2. 通用约定

### 2.1 认证（请求方 → 网关）
受保护路由需在请求头携带 JWT：

```
Authorization: Bearer <JWT>
```

- 缺失/格式非 `Bearer ` 开头：返回 **401**（`AUTH_001`）。
- 令牌无效/过期/校验失败：返回 **401**（`AUTH_002`）。
- 校验经由认证服务（见 §8）。

### 2.2 身份透传（网关 → 下游服务）
JWT 校验通过后，网关向下游注入：

| 通道 | 字段 | 说明 |
|------|------|------|
| 请求头 | `X-User-Id` | 认证用户 ID |
| 请求头 | `X-User-Role` | 用户角色（`EMPLOYEE` / `ADMIN`） |
| 请求头 | `X-Request-Id` | 链路请求 ID（网关生成，全程透传/日志关联） |
| 请求体 | `userId` | 仅对 `POST` + JSON 请求，注入到 body 顶层字段（兼容下游 `GatewayInjectableRequest` 的 POST-Only 约定） |

> 下游服务应**信任**这些由网关注入的身份字段（内网信任模型），并**不应**接受外部直接传入的同名头。

### 2.3 统一错误响应体
所有由网关产生的错误（4xx/5xx）返回统一 JSON：

```json
{
  "code": "AUTH_002",
  "message": "Authorization token is invalid",
  "requestId": "a1b2c3d4e5f6a7b8",
  "path": "/api/v1/auth/user/role",
  "timestamp": "2026-06-10 18:00:00"
}
```

### 2.4 错误码 → HTTP 状态映射
状态由错误码前缀决定：

| 前缀 | HTTP 状态 |
|------|-----------|
| `AUTH_` | 401 Unauthorized |
| `AUTHZ_` | 403 Forbidden |
| `RATE_` | 429 Too Many Requests |
| `PARAM_` | 400 Bad Request |
| `NOT_FOUND_` | 404 Not Found |
| `SYS_` | 500 / 503 / 504 |

---

## 3. 路由入口表

后端目标（local profile）：`auth=http://localhost:8001`、`product=:8002`、`point=:8003`、`order=:8004`。

| # | 路径模式 | 转发目标 | 需认证 | 需 ADMIN | 说明 |
|---|----------|----------|:------:|:--------:|------|
| 1 | `/api/v1/public/auth/**` | auth | ❌ | ❌ | 认证服务公共接口（登录/注册/登出等） |
| 2 | `/api/v1/public/product/**` | product | ❌ | ❌ | 商品公共接口（浏览/搜索/详情） |
| 3 | `/api/v1/public/point/**` | point | ❌ | ❌ | 积分公共接口 |
| 4 | `/api/v1/public/order/**` | order | ❌ | ❌ | 兑换公共接口 |
| 5 | `/api/v1/auth/**` | auth | ✅ | 视路径 | 认证服务受保护接口 |
| 6 | `/api/v1/product/**` | product | ✅ | 视路径 | 商品受保护接口 |
| 7 | `/api/v1/point/**` | point | ✅ | 视路径 | 积分受保护接口 |
| 8 | `/api/v1/order/**` | order | ✅ | 视路径 | 兑换受保护接口 |
| 9 | `/v3/api-docs/{auth\|product\|point\|order}` | 对应服务 | ❌ | ❌ | OpenAPI 文档聚合（RewritePath + SwaggerServersRewrite） |
| 10 | `/{auth\|product\|point\|order}/**` | 对应服务 | ❌ | ❌ | Swagger Try-it-out 代理（`StripPrefix=1`），见 §3.1 安全提示 |

**路由规则**：
- 公共与受保护的区分由路由元数据 `auth-required`（`false`/`true`）控制。
- 路径前缀 `/api/v1/public/` 与 `/v3/api-docs/`、`/swagger-ui`、`/actuator` 一律免认证。

### 3.1 ⚠️ 安全提示（第 10 类代理路由）
`/{service}/**`（StripPrefix）当前 `auth-required=false`，主要服务于 Swagger 在线调试，会**绕过认证**直达后端。生产环境应禁用或加认证，避免成为鉴权旁路。

---

## 4. 认证与鉴权行为

### 4.1 认证（FR-G2）
- 命中免认证路径（`/api/v1/public/**`、docs、`/swagger-ui`、`/actuator`）或路由 `auth-required=false` → 直接放行。
- 其余路由必须通过 JWT 校验，否则 401。

### 4.2 角色鉴权（FR-G3）
对命中 `gateway.security.admin-paths` 配置的路径，要求 `X-User-Role == ADMIN`，否则 **403**（`AUTHZ_001`）。

默认管理端路径（local profile）：

| 路径模式 | 含义 |
|----------|------|
| `/api/v1/auth/user/detail` | 用户详情（管理员） |
| `/api/v1/auth/user/role` | 变更用户角色（管理员） |
| `/api/v1/admin/**` | 约定的管理端前缀（供下游服务使用） |
| `/api/v1/**/admin/**` | 各服务管理端子路径约定 |

> 新增下游管理端接口时，把其路径加入 `gateway.security.admin-paths` 即可纳入 ADMIN 管控。

---

## 5. 限流（FR-G5）

- 算法：进程内**令牌桶**，按客户端 IP（取 `X-Forwarded-For` → `X-Real-IP` → remoteAddr）。
- 超限返回 **429**（`RATE_001`）。
- 配置项（`gateway.security.rate-limit`）：

| 配置 | 默认 | 说明 |
|------|------|------|
| `enabled` | `true` | 是否启用 |
| `capacity` | `100` | 突发容量（令牌） |
| `refill-tokens` | `100` | 每周期补充令牌数 |
| `refill-period-seconds` | `1` | 补充周期（秒）→ 约 100 req/s/IP |

> MVP 为单实例内存方案；多实例部署需改共享存储（如 Redis `RequestRateLimiter`）。

---

## 6. 安全响应头（FR-G5）

所有响应附加以下头：

| 响应头 | 值 | 作用 |
|--------|-----|------|
| `X-Content-Type-Options` | `nosniff` | 禁止 MIME 嗅探 |
| `X-Frame-Options` | `DENY` | 防点击劫持 |
| `X-XSS-Protection` | `1; mode=block` | 反射型 XSS 防护 |
| `Referrer-Policy` | `no-referrer` | 不泄露来源 |
| `Content-Security-Policy` | `default-src 'self'; frame-ancestors 'none'` | 内容注入防护 |

---

## 7. CORS

| 项 | 值 |
|----|----|
| 允许来源 | `*`（`allowedOriginPatterns`） |
| 允许方法 | `GET, POST, PUT, DELETE, PATCH, OPTIONS` |
| 允许头 | `*` |
| 暴露头 | `X-Request-Id, X-User-Id, X-User-Role` |
| 允许凭证 | `true` |
| 预检缓存 | `3600s` |

> 注：生产应将允许来源收紧为前端实际域名。

---

## 8. 运维与文档端点

| 端点 | 说明 |
|------|------|
| `GET /actuator/health` | 健康检查 |
| `GET /actuator/info` | 应用信息 |
| `GET /actuator/prometheus` | Prometheus 指标 |
| `GET /actuator/gateway` | Spring Cloud Gateway 路由查看 |
| `GET /swagger-ui.html` | 聚合 Swagger UI（auth/product/point/order） |
| `GET /v3/api-docs/{service}` | 各后端 OpenAPI 文档（经网关聚合重写） |

---

## 9. 网关错误码一览（GatewayErrorCode）

| code | HTTP | message |
|------|------|---------|
| `AUTH_001` | 401 | Authorization token is missing |
| `AUTH_002` | 401 | Authorization token is invalid |
| `AUTH_003` | 401 | Authorization token has expired |
| `AUTH_004` | 401 | Authentication service is unavailable |
| `AUTHZ_001` | 403 | Access denied: administrator role required |
| `RATE_001` | 429 | Too many requests, please try again later |
| `SYS_001` | 503 | Backend service is unavailable |
| `SYS_002` | 504 | Backend service timeout |
| `SYS_003` | 500 | Gateway internal error |
| `SYS_004` | 502 | Bad gateway response from backend |

---

## 10. 上游依赖契约（网关 → 认证服务，供联调）

> 这是网关**消费**的接口（非网关对外暴露），用于令牌校验。详见认证服务（Unit2）文档。

```
POST {gateway.auth.validate-url}     # local: http://localhost:8001/api/v1/internal/auth/validate
Content-Type: application/json

请求: { "token": "<JWT>" }
响应: { "success": true, "userId": 123, "role": "ADMIN", "message": null }
```

- 调用超时：`gateway.auth.timeout`（默认 5s）；超时/异常 → 401（`AUTH_004`）。

---

## 11. 关键配置项参考

```yaml
server:
  port: 8080

gateway:
  services:                       # 后端服务地址
    auth: http://localhost:8001
    product: http://localhost:8002
    point: http://localhost:8003
    order: http://localhost:8004
  auth:
    validate-url: http://localhost:8001/api/v1/internal/auth/validate
    timeout: 5s
  security:
    admin-paths:                  # FR-G3 ADMIN 路径
      - /api/v1/auth/user/detail
      - /api/v1/auth/user/role
      - /api/v1/admin/**
      - /api/v1/**/admin/**
    rate-limit:                   # FR-G5 限流
      enabled: true
      capacity: 100
      refill-tokens: 100
      refill-period-seconds: 1
```

> 当前 `gateway.security.*`、路由、`validate-url` 仅落在 `application-local.yml`；`docker`/`prod` profile 待补。
