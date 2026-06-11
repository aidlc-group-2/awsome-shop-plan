# AWSomeShop 五服务 Code Review 问题追踪

> 生成日期：2026-06-11 | 来源：对 auth / gateway / order / points / product 五个服务的全量 review
> 状态图例：⬜ 待修复 | 🔧 修复中 | ✅ 已修复 | ⏸️ 暂缓

## ✅ 已完成的 P0 修复（2026-06-11，均已远程编译通过 BUILD SUCCESS）

| 服务 | 提交分支 | 修复项 |
|------|---------|--------|
| product | intermediate | **P1-1** 库存预占/释放/确认内部接口（条件UPDATE防超卖+预占表幂等）；**OR-1配套** 商品权威快照接口 `/private/product/{id}/snapshot` |
| points | intermediate | **PT-1** FOR UPDATE悲观锁串行化余额变动；**PT-2** deduct/refund 按orderRef+type幂等+唯一约束；**PT-5** 双V2脚本冲突(重命名V5)；**PT-9** negateExact防溢出；**PT-10** 批次消耗一致性校验；**PT-11** 并发首次创建 |
| order | intermediate | **OR-1** 服务端向product核价拒绝信任客户端pointsCost；**OR-2** requestId幂等键+唯一约束；**OR-3** updateStatus改CAS+受影响行数检查；**OR-7** long防int溢出；**OR-8** order_no唯一索引 |
| gateway | baseline | **GW-1** 剥离客户端X-User-*头+set覆盖语义；**GW-2** 移除免认证后门代理路由；**GW-3** 路由/services/admin-paths/validate-url下沉application.yml |
| auth | intermediate | **AU-1** 用户列表移出/public；**AU-2** 管理接口校验X-User-Role==ADMIN；**AU-3** 弱口令种子账户移出生产迁移(改db/seed) |

> 注：order/points/auth 用 `intermediate` 分支，gateway 用 `baseline` 分支。剩余 🟡/🔵 项见下方清单，未勾选者待后续迭代处理。

## 总体结论

当前不具备整体联调/上线条件。两类硬伤：
1. **核心功能缺失**：product-service 库存接口、points-service 并发/幂等机制根本未实现 → 下单链路跑不通或必然资损。
2. **权限体系可绕过**：网关身份头可伪造、免认证后门路由、auth 自我提权等。

架构文档声称的"悲观锁并发控制"在五个服务中均未落地（零处 `FOR UPDATE`/条件 UPDATE）。

---

## 修复优先级（依赖顺序）

| 阶段 | 内容 | 理由 |
|------|------|------|
| **P0-1 联调阻断** | product 库存接口、points 并发+幂等+双V2脚本、order 核价+幂等 | 下单链路依赖项，最底层 |
| **P0-2 安全** | gateway 三漏洞、auth 三缺口 | 入口安全 |
| **P1** | order 超时补偿/乐观锁、points 定时任务锁、删 Test 脚手架、统一 Result | |
| **P2** | 其余 🟡/🔵 | |

---

## 1. awsome-shop-product-service（完成度最低，下单链路最底层依赖）

### 🔴 严重
- [x] **P1-1** 库存扣减/回补接口完全不存在，无悲观锁，无供 order 调用的内部端点 — `ProductController.java:29-39`、`ProductRepository.java:11-31`、`ProductMapper.xml`
- [ ] **P1-2** 文件上传功能完全不存在；`imageUrl` 仅校验长度不校验 URL — `CreateProductRequest.java:44-45`
- [ ] **P1-3** 商品/分类管理接口零权限控制，且挂在 `/public/` 路径 — `ProductController.java:30`、`CategoryController.java:30`
- [ ] **P1-4** 商品列表不按上架状态过滤，员工可见已下架商品 — `ProductMapper.xml:13-26`

### 🟡 中等
- [ ] **P1-5** 商品 CRUD 残缺，update/delete 是死代码；分类无增删改 — `ProductApplicationService.java:11-16`、`ProductRepositoryImpl.java:62-71`
- [ ] **P1-6** 创建商品缺数值范围校验，可建负库存/负积分价 — `CreateProductRequest.java:33-40`
- [ ] **P1-7** SKU 唯一性 check-then-act 竞态 + 全工程零 `@Transactional` — `ProductDomainServiceImpl.java:46-49`
- [ ] **P1-8** 乐观锁失效：version 不映射到 Entity — `ProductRepositoryImpl.java:87-133`、`ProductPO.java:73-75`
- [ ] **P1-9** 商品-分类靠名称字符串关联，改名失联/计数错配 — `CategoryApplicationServiceImpl.java:79`

### 🔵 轻微
- [ ] **P1-10** Test 桩代码全套暴露在生产路径 — `TestController.java:24-58`
- [ ] **P1-11** 两套 Result 类并存 — `ProductController.java:8` vs `GlobalExceptionHandler.java:6`
- [ ] **P1-12** 审计字段 createdBy/updatedBy 永远 null — `UserContext.java:27`、`CustomMetaObjectHandler.java:30-33`
- [ ] **P1-13** LIKE 通配符未转义 — `ProductMapper.xml:20`、`CategoryMapper.xml:13`
- [ ] **P1-14** 孤儿分类树形组装静默丢失 — `CategoryApplicationServiceImpl.java:48-68`
- [ ] **P1-15** Flyway 校验全局放水 — `application.yml:32`
- [ ] **P1-16** 工程卫生（javadoc 错误、空壳模块、AES 硬编码默认 key、Druid admin/admin）

---

## 2. awsome-shop-points-service（资金类，核心机制全缺）

### 🔴 严重
- [x] **PT-1** 余额更新无并发控制（无 FOR UPDATE/条件 UPDATE），version 因 toPO 不拷贝被静默绕过 → 双花 — `PointsGrantDomainServiceImpl.java:86-103`、`PointsAccountRepositoryImpl.java:46-49,61-67`
- [x] **PT-2** deduct/refund 无幂等，orderRef 仅存档不查重，无唯一约束 — `PointsGrantDomainServiceImpl.java:80-153`、`PointsErrorCode.java:24`
- [ ] **PT-3** 定时发放无分布式锁、无周期防重记录，多实例重复发放 — `PointsScheduler.java:44-53`、`PointsExpiryDomainServiceImpl.java:91-118`
- [ ] **PT-4** 权限缺失：adjust 无 ADMIN 校验、operatorId 请求体可伪造、员工可查他人流水、/private 无内部鉴权 — `PointsController.java:55-59`、`PointsApplicationServiceImpl.java:51-53`、`InternalPointsController.java`
- [x] **PT-5** 两个 V2 Flyway 脚本冲突，启动失败 — `V2__create_points_tables.sql` + `V2__create_point_rule_table.sql`

### 🟡 中等
- [ ] **PT-6** runPeriodicGrant 事务边界错误，一人失败整批回滚 — `PointsExpiryDomainServiceImpl.java:90-114`
- [ ] **PT-7** 过期处理账实不符且静默吞掉 — `PointsExpiryDomainServiceImpl.java:54-72`
- [ ] **PT-8** 已到期未处理批次仍可消费 — `PointsBatchRepositoryImpl.findActiveBatchesByUserId:27-36`
- [x] **PT-9** adjust `Math.abs(Long.MIN_VALUE)` 溢出 — `PointsGrantDomainServiceImpl.java:180`
- [x] **PT-10** deduct FIFO 消耗与余额可能不一致 — `PointsGrantDomainServiceImpl.java:92-99`
- [x] **PT-11** getOrCreateAccount 并发首次创建抛 DuplicateKeyException — `PointsAccountDomainServiceImpl.java:19-28`
- [ ] **PT-12** 两套规则模型并存（point_rule vs points_rule） — `PointsRuleRepositoryImpl.java:22`
- [ ] **PT-13** Test 脚手架暴露在生产路由 — `TestController.java`
- [ ] **PT-14** 入参解析异常变 500（parseLong/valueOf） — `PointsApplicationServiceImpl.java:39,53,80,104`

### 🔵 轻微
- [ ] **PT-15** cron 固定每月 1 号无视 periodicCycle 配置 — `PointsScheduler.java:44`
- [ ] **PT-16** expirePoints 长事务 — `PointsExpiryDomainServiceImpl.java:35-81`
- [ ] **PT-17** deduct 流水不记录 batchId — `PointsGrantDomainServiceImpl.java:106-113`
- [ ] **PT-18** 未使用脚手架 + UserContext 审计失效
- [ ] **PT-19** Result 类重复定义

---

## 3. awsome-shop-order-service（Saga 骨架对，但闭环未闭合）

### 🔴 严重
- [x] **OR-1** 价格/快照完全信任客户端，可 1 积分兑任意商品 — `CreateExchangeRequest.java:34-36`、`ExchangeRecordApplicationServiceImpl.java:64-70`
- [x] **OR-2** 下单无幂等，orderNo 每次随机生成 — `ExchangeRecordApplicationServiceImpl.java:62,181-184`
- [x] **OR-3** 乐观锁冲突被静默吞掉，并发 cancel+ship 双成功 — `ExchangeRecordRepositoryImpl.java:53-58`
- [ ] **OR-4** 积分扣减超时≠失败未补偿，资损窗口 — `RedemptionSagaOrchestrator.java:32-33`、`PointsClientImpl.java:57-60`
- [ ] **OR-5** 库存控制默认整体短路（stub），所有环境默认 false — `ProductClientImpl.java:52-57`、`application.yml:45-48`

### 🟡 中等
- [ ] **OR-6** 补偿失败只打日志，无补偿任务表/重试 — `RedemptionSagaOrchestrator.java:67-83`
- [x] **OR-7** 积分总额 int 溢出，quantity 无上限 — `ExchangeRecordApplicationServiceImpl.java:70`、`CreateExchangeRequest.java:26-28`
- [x] **OR-8** order_no 无唯一索引 — `V2__create_exchange_record_table.sql`
- [ ] **OR-9** 统计把已取消订单计入总消耗 — `ExchangeRecordMapper.xml:36`
- [ ] **OR-10** 调用下游 private 接口无内部鉴权 — `PointsClientImpl.java:44-50`、`ProductClientImpl.java:61-67`
- [ ] **OR-11** 业务错误码 HTTP 映射混乱（余额不足返回 200+500004） — `GlobalExceptionHandler.java:270-283`
- [ ] **OR-12** 两个 Result 类，成功/失败 code 类型不一致
- [ ] **OR-13** 零测试

### 🔵 轻微
- [ ] **OR-14** updateStatus 潜在 NPE — `ExchangeRecordRepositoryImpl.java:55-56`
- [ ] **OR-15** shipping_info phone/address 明文落库
- [ ] **OR-16** 模板/脚手架残留
- [ ] **OR-17** WebClient 无重试与连接级超时细化
- [ ] **OR-18** 虚拟商品履约失败后状态不可见 — `RedemptionSagaOrchestrator.java:56-63`

---

## 4. awsome-shop-gateway-service（三漏洞使认证体系形同虚设）

### 🔴 严重
- [x] **GW-1** 外部可伪造 X-User-Id/X-User-Role，header 追加非覆盖且不剥离入站值 — `AuthenticationGatewayFilter.java:84-91`
- [x] **GW-2** /auth/** /product/** /point/** /order/** 代理路由整体绕过认证 — `application-local.yml:136-171`
- [x] **GW-3** 路由/admin-paths/validate-url 仅配在 local profile — `application.yml:3`、`application-local.yml`

### 🟡 中等
- [ ] **GW-4** 限流键取自可伪造 X-Forwarded-For + buckets 无界增长 — `RateLimitFilter.java:70-84`
- [ ] **GW-5** RoleAuthorizationFilter 隐式依赖认证过滤器先执行 — `RoleAuthorizationFilter.java:46-52`
- [ ] **GW-6** AuthServiceClient 把 4xx/5xx 一律吞为 503 — `AuthServiceClient.java:43-63`
- [ ] **GW-7** CORS 允许任意来源 + allowCredentials=true — `CorsConfig.java:19-24`

### 🔵 轻微
- [ ] **GW-8** Test 脚手架死代码 — `TestApplicationServiceImpl.java` 等
- [ ] **GW-9** OperatorIdInjectionFilter 缓冲区工厂混用 — `OperatorIdInjectionFilter.java:67`
- [ ] **GW-10** Application javadoc 错误 — `Application.java:8`

---

## 5. awsome-shop-auth-service（基本面最好，三个权限缺口）

### 🔴 严重
- [x] **AU-1** 用户列表挂在 /public 免认证路径，匿名枚举全部用户 PII — `UserController.java:30-34`
- [x] **AU-2** changeRole 接口零角色校验，可自我提权 — `UserController.java:42-46`、`UserDomainServiceImpl.java:40-48`
- [x] **AU-3** Flyway 在所有环境植入 admin/admin123 弱口令账号 — `V2__create_user_table.sql:21-27`

### 🟡 中等
- [ ] **AU-4** 账户锁定提示构造函数重载选错，只显示裸数字 — `AuthDomainServiceImpl.java:83`
- [ ] **AU-5** 注册 check-then-insert 竞态，并发返回 500 — `AuthDomainServiceImpl.java:46-63`
- [ ] **AU-6** 乐观锁失效，登录失败计数丢失更新 — `UserRepositoryImpl.java:90-103`、`UserPO.java:52-54`
- [ ] **AU-7** token 校验不查用户实时状态；无刷新机制 — `AuthDomainServiceImpl.java:115-129`
- [ ] **AU-8** 非 prod 环境 JWT secret 硬编码默认值 — `application-staging.yml:53`
- [ ] **AU-9** 错误码→HTTP 映射两处漏洞（PARAM 无分支、多下划线拆分错） — `GlobalExceptionHandler.java:226-233,260-264`
- [ ] **AU-10** Test 桩代码暴露为公开接口 — `TestController.java:29-57`

### 🔵 轻微
- [ ] **AU-11** PasswordEncoder 直接 new 而非注入 — `AuthDomainServiceImpl.java:30`
- [ ] **AU-12** 错误码语义不准（USER_NOT_FOUND 映射 401）
- [ ] **AU-13** 解锁后 lock_expired_at 残留 — `UserEntity.java:51`
- [ ] **AU-14** 大量死代码/模板残留
- [ ] **AU-15** 逻辑删除 + 唯一索引隐患 — `V2__create_user_table.sql:18`
- [ ] **AU-16** logout 黑名单 TTL 用全量 7200s — `AuthDomainServiceImpl.java:110`
- [ ] **AU-17** JWT 解析未校验 issuer — `JwtServiceImpl.java:88-94`
- [ ] **AU-18** Druid 控制台 admin/admin（仅 local）
- [ ] **AU-19** 文档/注释错乱

---

## 跨服务系统性问题（同一脚手架带出来的）

- [ ] **SYS-1** Test 桩代码全套暴露（每个服务都有 TestController + V1 建表）→ 全删
- [ ] **SYS-2** 乐观锁普遍失效（PO 有 @Version 但 Entity 不透传）→ auth/points/product
- [ ] **SYS-3** 两套 Result 类并存（String code vs Integer code）
- [ ] **SYS-4** 死代码模板（RequireOwnerPermission 无切面、UserContext 无 Filter、AES 无调用方且硬编码 key、空壳 mq/cache 模块、javadoc 写 "AIOps Service"）
- [ ] **SYS-5** 服务间 /private 调用无内部鉴权
