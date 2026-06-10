# 组件定义（Components）

> 阶段：INCEPTION - 应用设计
> 时间：2026-06-10T15:54:10+08:00
> 架构决策：Spring Boot 微服务 · 全部同步 REST · 编排式 Saga · 内网信任 · 经典三层 · common 共享模块

本文件定义各服务的主要组件与高层职责（详细业务逻辑在 CONSTRUCTION 功能设计阶段细化）。每个微服务统一采用三层：`Controller`（API）→ `Service`（业务）→ `Repository`（数据访问），并辅以 `Entity`/`DTO`、`Client`（跨服务调用）等组件。

---

## 0. 公共共享模块（common）

| 组件 | 职责 |
|------|------|
| `JwtUtil` | JWT 解析/校验工具（网关与服务复用） |
| `ApiResponse<T>` / `ErrorResponse` | 统一响应与错误结构 |
| `ErrorCode` | 统一错误码枚举 |
| `PageRequest` / `PageResponse<T>` | 分页请求与响应封装 |
| `RoleEnum` | 角色枚举（EMPLOYEE / ADMIN） |
| `ServiceException` / `GlobalExceptionHandler` | 通用异常与全局异常处理 |
| `InternalAuthFilter` | 内网信任校验（标识来自网关的请求） |

---

## 1. 认证服务（Auth Service，Unit 2，:8001）

| 组件 | 类型 | 职责 |
|------|------|------|
| `AuthController` | Controller | 暴露注册、登录、登出、令牌校验、用户信息接口 |
| `UserController` | Controller | 管理员侧用户查询/角色管理接口 |
| `AuthService` | Service | 注册（含企业邮箱域名校验）、登录、JWT 签发、密码校验 |
| `UserService` | Service | 用户查询、角色管理 |
| `PasswordEncoderComponent` | Service 辅助 | bcrypt 加密/校验 |
| `EmailDomainValidator` | Service 辅助 | 企业邮箱白名单域名校验 |
| `PointsClient` | Client | 注册成功后同步调用积分服务发放入职奖励 |
| `UserRepository` | Repository | 用户持久化（auth schema） |
| `User` | Entity | 用户实体（id、邮箱、用户名、密码哈希、角色、状态、创建时间） |
| `RegisterRequest`/`LoginRequest`/`AuthResponse`/`UserDTO` | DTO | 接口数据传输对象 |

**接口（高层）**：注册、登录、登出、`/internal/validate` 令牌校验、用户列表/详情、角色变更。

---

## 2. 商品服务（Product Service，Unit 3，:8002）

| 组件 | 类型 | 职责 |
|------|------|------|
| `ProductController` | Controller | 商品浏览/搜索/详情（员工）、商品 CRUD/上下架/库存/图片（管理员） |
| `CategoryController` | Controller | 二级分类的查询与管理 |
| `InternalStockController` | Controller | 供兑换服务调用的库存查询/预占/释放/扣减接口 |
| `ProductService` | Service | 商品业务：CRUD、上下架、搜索、售罄判定 |
| `CategoryService` | Service | 分类树管理（父/子两级） |
| `StockService` | Service | 库存预占/释放/正式扣减，悲观锁防超兑 |
| `ImageStorageComponent` | Service 辅助 | 图片上传到本地卷、URL 生成 |
| `ProductRepository`/`CategoryRepository`/`StockRepository` | Repository | 持久化（product schema） |
| `Product`/`Category`/`StockReservation` | Entity | 商品、分类、库存预占记录 |
| `ProductDTO`/`CategoryDTO`/`StockOpRequest` 等 | DTO | 数据传输对象 |

**关键点**：`Product` 含 `productType`（PHYSICAL / VIRTUAL）；`StockReservation` 记录订单对库存的预占。

---

## 3. 积分服务（Points Service，Unit 4，:8003）

| 组件 | 类型 | 职责 |
|------|------|------|
| `PointsController` | Controller | 员工：余额查询、变动记录；管理员：规则配置、手动调整、查任意用户记录 |
| `InternalPointsController` | Controller | 供认证/兑换服务调用：发放（入职奖励）、扣减（兑换）、退回（取消） |
| `PointsAccountService` | Service | 余额管理（按批次/FIFO） |
| `PointsTransactionService` | Service | 变动记录写入与查询 |
| `PointsRuleService` | Service | 入职奖励、周期性发放、有效期规则配置 |
| `PointsGrantService` | Service | 发放（入职/周期/手动）、扣减、退回逻辑 |
| `PointsExpiryScheduler` | Service（定时任务） | 周期性发放调度 + 积分到期失效（FIFO） |
| `PointsAccountRepository`/`PointsBatchRepository`/`PointsTransactionRepository`/`PointsRuleRepository` | Repository | 持久化（points schema） |
| `PointsAccount`/`PointsBatch`/`PointsTransaction`/`PointsRule` | Entity | 账户余额、积分批次（含到期时间，FIFO 消耗）、变动流水、规则 |
| `AdjustRequest`/`GrantRequest`/`DeductRequest`/`TransactionDTO` 等 | DTO | 数据传输对象 |

**关键点**：积分按 `PointsBatch` 管理有效期，消耗按 FIFO（最早到期/最早获得优先）；`PointsTransaction` 记录类型 GRANT/DEDUCT/REFUND/EXPIRE/ADJUST。

---

## 4. 兑换服务（Order Service，Unit 5，:8004）

| 组件 | 类型 | 职责 |
|------|------|------|
| `OrderController` | Controller | 员工：创建兑换、订单详情/历史、取消；管理员：兑换记录管理、发货状态更新 |
| `OrderService` | Service | 订单生命周期管理、状态机 |
| `RedemptionSagaOrchestrator` | Service | **编排式 Saga 协调者**：依次调用积分扣减、库存预占、创建订单；失败时触发补偿 |
| `FulfillmentService` | Service | 履约：实物发货状态流转（含库存正式扣减）、虚拟商品即时履约 |
| `PointsClient`/`ProductClient` | Client | 调用积分服务（扣减/退回）、商品服务（预占/释放/扣减/查询） |
| `OrderRepository`/`ShippingInfoRepository` | Repository | 持久化（order schema） |
| `Order`/`OrderItem`/`ShippingInfo` | Entity | 订单、订单项、配送信息（实物） |
| `CreateOrderRequest`/`OrderDTO`/`ShippingUpdateRequest` 等 | DTO | 数据传输对象 |

**关键点**：`Order` 状态机：CREATED → SUCCESS（自动，无审批）→ (实物) PENDING_SHIPMENT → SHIPPED → COMPLETED；可在发货前 CANCELLED；虚拟商品 SUCCESS → COMPLETED（即时）。

---

## 5. API 网关（API Gateway，Unit 6，:8080）

| 组件 | 类型 | 职责 |
|------|------|------|
| `GatewayRouteConfig` | 配置 | 路由规则：/auth、/products、/points、/orders → 对应服务 |
| `JwtAuthenticationFilter` | 全局过滤器 | 校验 JWT，注入用户身份到下游请求头 |
| `RoleAuthorizationFilter` | 过滤器 | 基于角色对管理端路径做访问控制 |
| `RateLimitFilter` | 过滤器 | 限流（保护后端） |
| `SecurityHeadersFilter` | 过滤器 | CSRF/XSS 等安全响应头 |
| `CorsConfig` | 配置 | 跨域配置（前端域） |

**实现**：基于 Spring Cloud Gateway（成熟中间件）。

---

## 6. 前端应用（Frontend，Unit 1，:3000）

| 组件 | 类型 | 职责 |
|------|------|------|
| `AuthModule` | 功能模块 | 登录/注册/会话管理（JWT 存储与附加） |
| `ShopModule` | 功能模块 | 商城首页、商品详情、搜索、分类浏览 |
| `RedemptionModule` | 功能模块 | 确认兑换、配送信息、兑换成功 |
| `OrderModule` | 功能模块 | 兑换历史、订单详情、取消 |
| `PointsModule` | 功能模块 | 积分中心（余额、变动、到期提示） |
| `AdminModule` | 功能模块 | 仪表盘、商品/分类管理、积分规则/调整、兑换记录/发货、用户管理 |
| `ApiClient` | 基础设施 | 统一 HTTP 客户端（指向网关，附加 JWT、错误处理） |
| `I18nProvider` | 基础设施 | 中英文双语（react-i18next） |
| `RouteGuard` | 基础设施 | 基于角色的路由守卫（员工端/管理端） |
| `A11yComponents` | 基础设施 | 满足 WCAG 2.1 AA 的通用可访问组件 |

**技术**：React + TypeScript SPA，通过 Nginx 提供静态资源并反向代理到网关。

---

## 7. 基础设施（Infrastructure，Unit 7）

| 组件 | 类型 | 职责 |
|------|------|------|
| `docker-compose.yml` | 编排 | 启动 6 服务 + MySQL + Nginx |
| `MySQL`（4 schema） | 数据 | auth/product/points/order 独立 schema |
| `init-scripts` | 数据 | 各 schema 建表与种子数据 |
| `Docker Network` | 网络 | 内部服务网络（内网信任） |
| `Volumes` | 存储 | MySQL 数据卷 + 上传文件卷 |
| `Nginx` | 网关前置 | 前端静态资源 + 反向代理到 API 网关 |
