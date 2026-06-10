# 组件方法签名（Component Methods）

> 阶段：INCEPTION - 应用设计
> 时间：2026-06-10T15:54:10+08:00
> 说明：以下为**高层方法签名**（输入/输出/用途），详细业务规则在 CONSTRUCTION 功能设计阶段细化。签名以 Java/Spring 风格表达，仅作设计参考。

---

## 1. 认证服务（Auth Service）

### AuthService
- `AuthResponse register(RegisterRequest req)` — 校验企业邮箱域名、唯一性，bcrypt 加密存储，创建用户；随后同步触发入职奖励发放。
- `AuthResponse login(LoginRequest req)` — 校验凭据，签发 JWT（含 userId、role）。
- `void logout(String token)` — 登出处理（前端清除令牌为主）。
- `TokenValidationResult validateToken(String token)` — 解析并校验 JWT，返回用户身份与角色（供网关/内部）。

### UserService
- `PageResponse<UserDTO> listUsers(PageRequest page, String keyword)` — 管理员分页查询用户。
- `UserDTO getUser(Long userId)` — 查询用户详情。
- `void changeRole(Long userId, RoleEnum role)` — 变更用户角色。

### PointsClient（出站）
- `void grantOnboardingBonus(Long userId)` — 同步调用积分服务发放入职奖励（注册成功后）。

---

## 2. 商品服务（Product Service）

### ProductService
- `PageResponse<ProductDTO> listProducts(PageRequest page, Long categoryId, String keyword)` — 浏览/搜索/按分类。
- `ProductDTO getProduct(Long productId)` — 商品详情。
- `ProductDTO createProduct(ProductCreateRequest req)` / `updateProduct(Long id, ProductUpdateRequest req)` — 创建/编辑（管理员）。
- `void changeStatus(Long productId, ProductStatus status)` — 上下架。
- `void deleteProduct(Long productId)` — 删除（管理员）。

### CategoryService
- `List<CategoryDTO> getCategoryTree()` — 返回两级分类树。
- `CategoryDTO createCategory(CategoryRequest req)` / `updateCategory(...)` / `void deleteCategory(Long id)` — 分类管理（删除含占用校验）。

### StockService（含内部接口）
- `int getAvailableStock(Long productId)` — 查询可用库存。
- `ReservationResult reserveStock(Long productId, int qty, String orderRef)` — **悲观锁**预占库存，返回预占凭据。
- `void releaseStock(String reservationId)` — 释放预占（取消/补偿）。
- `void confirmDeduct(String reservationId)` — 发货时正式扣减。
- `void adjustStock(Long productId, int newQty)` — 管理员调整库存。

### ImageStorageComponent
- `String uploadImage(Long productId, MultipartFile file)` — 保存到本地卷并返回访问 URL。

---

## 3. 积分服务（Points Service）

### PointsAccountService
- `long getBalance(Long userId)` — 当前可用余额（汇总未过期批次）。
- `List<PointsBatch> getActiveBatchesFifo(Long userId)` — 按 FIFO 顺序返回有效批次（供扣减）。

### PointsGrantService（含内部接口）
- `void grant(Long userId, long amount, GrantType type, String reason)` — 发放（入职/周期/手动），创建批次（含到期时间）与流水。
- `DeductResult deduct(Long userId, long amount, String orderRef)` — 按 FIFO 扣减，写流水；余额不足抛业务异常。
- `void refund(Long userId, long amount, String orderRef)` — 兑换取消时退回，写流水。
- `void adjust(Long userId, long delta, String reason, Long operatorId)` — 管理员手动增减。

### PointsTransactionService
- `PageResponse<TransactionDTO> listTransactions(Long userId, PageRequest page)` — 查询某用户变动记录。

### PointsRuleService
- `PointsRuleDTO getRules()` / `void updateRules(PointsRuleRequest req)` — 配置入职奖励额度、周期发放额度/周期、积分有效期。

### PointsExpiryScheduler
- `void runPeriodicGrant()` — 定时：按规则周期性发放。
- `void expirePoints()` — 定时：将到期批次按 FIFO 失效，写 EXPIRE 流水。

---

## 4. 兑换服务（Order Service）

### OrderService
- `OrderDTO createRedemption(CreateOrderRequest req, Long userId)` — 入口：委托 Saga 协调者执行兑换。
- `OrderDTO getOrder(Long orderId, Long userId)` — 订单详情。
- `PageResponse<OrderDTO> listMyOrders(Long userId, PageRequest page)` — 个人兑换历史。
- `void cancelOrder(Long orderId, Long userId)` — 发货前取消（触发退回 + 释放预占）。
- `PageResponse<OrderDTO> listAllOrders(OrderQuery query, PageRequest page)` — 管理员兑换记录管理。

### RedemptionSagaOrchestrator
- `OrderDTO execute(CreateOrderRequest req, Long userId)` — 编排：1) 积分扣减 → 2) 库存预占 → 3) 创建订单；任一失败按逆序补偿（释放预占、退回积分）。
- `void compensate(SagaContext ctx)` — 执行补偿动作。

### FulfillmentService
- `void shipPhysicalOrder(Long orderId, ShippingUpdateRequest req)` — 实物发货：正式扣减库存、状态推进。
- `void completeVirtualOrder(Long orderId)` — 虚拟商品即时履约完成。

### PointsClient / ProductClient（出站）
- `DeductResult deductPoints(Long userId, long amount, String orderRef)` / `void refundPoints(...)`
- `ReservationResult reserveStock(...)` / `void releaseStock(...)` / `void confirmDeduct(...)`

---

## 5. API 网关（Gateway）

### JwtAuthenticationFilter
- `Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain)` — 校验 JWT，失败返回 401，成功注入 `X-User-Id`/`X-User-Role` 头。

### RoleAuthorizationFilter
- `Mono<Void> filter(...)` — 对 `/admin/**` 等管理端路径校验 role=ADMIN，否则 403。

---

## 6. 前端（Frontend，关键服务方法）

### ApiClient
- `get/post/put/delete<T>(path, payload?)` — 统一请求，自动附加 JWT、处理 401/错误。

### AuthService（前端）
- `login(credentials)` / `register(form)` / `logout()` / `getCurrentUser()`

### 各模块 hooks/services
- `useProducts(query)` / `useProduct(id)` / `useRedeem(productId, shippingInfo?)` / `useOrders()` / `usePointsBalance()` / `usePointsTransactions()`
- 管理端：`useAdminProducts()` / `useCategories()` / `usePointsRules()` / `useAdjustPoints()` / `useOrderManagement()` / `useUsers()`

---

## 7. 跨服务接口约定（内部 REST，内网信任）

| 调用方 → 被调方 | 端点（示意） | 用途 |
|------|------|------|
| Auth → Points | `POST /internal/points/grant` | 注册发放入职奖励 |
| Order → Points | `POST /internal/points/deduct` / `/refund` | 兑换扣减 / 取消退回 |
| Order → Product | `POST /internal/stock/reserve` / `/release` / `/confirm` | 库存预占/释放/扣减 |
| Gateway → Auth | `POST /internal/auth/validate` | 令牌校验（或网关本地公钥校验） |

> 内部接口受 `InternalAuthFilter` 保护，仅接受来自内网/网关的调用。
