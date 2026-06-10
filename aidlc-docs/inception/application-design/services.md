# 服务层与编排设计（Services & Orchestration）

> 阶段：INCEPTION - 应用设计
> 时间：2026-06-10T15:54:10+08:00
> 决策：全部同步 REST · 编排式 Saga（兑换服务为协调者）· 内网信任 · 网关统一鉴权

---

## 1. 服务清单与职责

| 服务 | 端口 | Schema | 核心职责 |
|------|------|--------|----------|
| Auth Service | 8001 | auth | 注册/登录/JWT/角色/用户管理；注册后同步触发发积分 |
| Product Service | 8002 | product | 商品与二级分类、图片、库存预占/释放/扣减 |
| Points Service | 8003 | points | 积分账户、批次(FIFO)、变动流水、规则、发放/扣减/退回、定时发放与过期 |
| Order Service | 8004 | order | 兑换订单生命周期、Saga 编排、履约（实物发货/虚拟即时） |
| API Gateway | 8080 | - | 统一入口、JWT 校验、角色鉴权、路由、限流、安全头 |
| Frontend | 3000 | - | 员工端/管理端 SPA，双语，经 Nginx 反代到网关 |

---

## 2. 关键编排流程

### 2.1 员工注册 + 入职奖励（同步）
```
前端 → 网关 → Auth.register
  Auth: 校验企业邮箱域名 + 唯一性 → bcrypt 存储 → 创建用户
  Auth → Points.grant(onboarding)   [同步 REST]
  发放成功 → 返回注册成功(含登录态/提示)
  若发积分失败 → 记录并重试/告警；用户已创建（注册不回滚），积分补偿由重试保证最终发放
```
> 说明：注册主事务（建用户）与发积分为同步调用；为避免注册因下游抖动失败，发积分失败采用重试/补偿而非回滚用户。

### 2.2 兑换商品（编排式 Saga，核心）
协调者：`RedemptionSagaOrchestrator`（Order Service）

```
前端 → 网关 → Order.createRedemption
  步骤1  Order → Points.deduct(userId, amount, orderRef)      [扣减积分]
  步骤2  Order → Product.reserveStock(productId, qty, orderRef) [悲观锁预占库存]
  步骤3  Order: 创建订单(SUCCESS)
         - 实物: 状态 PENDING_SHIPMENT，要求/保存配送信息
         - 虚拟: FulfillmentService.completeVirtualOrder → COMPLETED（即时履约）
  返回兑换成功

补偿（任一步失败，按逆序）：
  步骤2失败 → 补偿步骤1：Points.refund
  步骤3失败 → 补偿步骤2：Product.releaseStock；补偿步骤1：Points.refund
```
**一致性**：编排集中在 Order Service；每步具备对应补偿动作（refund / releaseStock）。库存预占用悲观锁防超兑（NFR-4）。

### 2.3 发货（实物）/ 取消
```
发货（管理员）: Order.shipPhysicalOrder
  → Product.confirmDeduct(reservationId)  [预占转正式扣减]
  → 状态 SHIPPED → COMPLETED

取消（员工，发货前）: Order.cancelOrder
  → Points.refund(amount)        [退回积分]
  → Product.releaseStock(resId)  [释放预占]
  → 状态 CANCELLED
  (虚拟商品已即时履约则不可取消)
```

### 2.4 周期性发放 / 积分过期（积分服务内定时任务）
```
PointsExpiryScheduler.runPeriodicGrant()  [按规则周期发放, 创建批次]
PointsExpiryScheduler.expirePoints()       [到期批次 FIFO 失效, 写 EXPIRE 流水]
```

---

## 3. 编排模式与原则
- **编排式 Saga**：兑换服务作为唯一协调者，业务流程集中、便于追踪与补偿。
- **同步调用 + 补偿**：跨服务一律同步 REST；失败即触发补偿，保证最终一致（NFR-5）。
- **幂等性**：内部接口（deduct/refund/reserve/release/confirm）以 `orderRef`/`reservationId` 保证幂等，支持重试。
- **故障与超时**：跨服务调用设置超时与有限重试；超时按失败处理并补偿。

---

## 4. 鉴权与安全编排
- 所有外部请求经网关：`JwtAuthenticationFilter` 校验 JWT → 注入 `X-User-Id`/`X-User-Role`。
- 管理端路径经 `RoleAuthorizationFilter` 校验 ADMIN。
- 服务间内部接口仅在内网暴露，`InternalAuthFilter` 拒绝外部直达。
- 网关负责限流、安全响应头、CORS。

---

## 5. 与需求/故事的对齐
- 注册发积分：FR-A6 / FR-P3 / US-01 / US-22
- 兑换 Saga：FR-O1~O4、FR-O10、BR-5~8 / US-08~10、US-28
- 取消退回：FR-O7、BR-7 / US-14
- 发货：FR-O6、AS-1 / US-26
- 周期发放/过期：FR-P4、FR-P7、BR-2~3 / US-17、US-22
- 鉴权：FR-G1~G5 / US-02、US-18
