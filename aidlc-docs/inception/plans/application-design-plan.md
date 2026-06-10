# 应用设计计划（Application Design Plan）

> 阶段：INCEPTION - 应用设计（Part 1 规划）
> 时间：2026-06-10T15:47:10+08:00
> 输入：requirements.md(v1.1)、stories.md、personas.md、execution-plan.md

本阶段聚焦**高层组件识别与服务层设计**（不含详细业务逻辑，详细逻辑在 CONSTRUCTION 的功能设计阶段）。请先回答文末问题中的所有 `[Answer]:`。

---

## A. 执行清单（Execution Checklist）

- [x] A1. 识别各服务的主要组件与职责 → `components.md`
- [x] A2. 定义组件方法签名（高层，输入/输出）→ `component-methods.md`
- [x] A3. 定义服务层与编排模式（含 Saga、跨服务调用）→ `services.md`
- [x] A4. 定义组件/服务依赖矩阵与通信模式、数据流 → `component-dependency.md`
- [x] A5. 校验设计完整性与一致性（覆盖 FR、与 US 对齐）

---

## B. 强制产出物（Mandatory Artifacts）
- `aidlc-docs/inception/application-design/components.md`
- `aidlc-docs/inception/application-design/component-methods.md`
- `aidlc-docs/inception/application-design/services.md`
- `aidlc-docs/inception/application-design/component-dependency.md`

---

## C. 设计决策问题（请填写所有 [Answer]:）

## 问题 1
微服务之间的通信方式？

A) 全部同步 REST（HTTP）调用
B) 同步 REST 为主 + 关键异步事件（如注册→发积分、积分过期）用轻量消息/事件
C) 事件驱动为主（消息队列）
X) 其他（请在 [Answer]: 后描述）

[Answer]: A

## 问题 2
兑换流程的跨服务事务（积分扣减 + 库存预占 + 订单创建）采用哪种 Saga 模式？

A) 编排式（Orchestration）：由兑换服务作为协调者，依次调用积分/库存并在失败时触发补偿（推荐，逻辑集中清晰）
B) 协同式（Choreography）：各服务通过事件相互触发与补偿
X) 其他（请在 [Answer]: 后描述）

[Answer]: A

## 问题 3
服务间调用的认证/信任方式？

A) 内部网络信任：服务部署在 docker 内网，网关校验 JWT 后服务间直接调用（MVP 简化，推荐）
B) 透传 JWT：服务间调用携带用户 JWT，各服务自行校验
C) 服务间使用独立的内部令牌/密钥（mTLS 或共享密钥）
X) 其他（请在 [Answer]: 后描述）

[Answer]: A

## 问题 4
新员工注册后的入职奖励积分如何触发？

A) 认证服务在注册成功后同步调用积分服务发放（简单直接）
B) 认证服务发布"用户已注册"事件，积分服务订阅后发放（解耦，最终一致）
X) 其他（请在 [Answer]: 后描述）

[Answer]: A

## 问题 5
每个微服务的内部分层结构？

A) 经典三层：Controller（API）→ Service（业务）→ Repository（数据访问），DTO/Entity 分离（Spring Boot 常规，推荐）
B) 更简化的两层（Controller → Service 直连数据）
X) 其他（请在 [Answer]: 后描述）

[Answer]: A

## 问题 6
是否需要一个跨服务共享的公共库（如通用 DTO、错误码、JWT 工具、分页封装）？

A) 需要，抽取一个 common 共享模块供各服务复用（推荐，减少重复）
B) 不需要，各服务自包含，避免耦合
X) 其他（请在 [Answer]: 后描述）

[Answer]: A

## 问题 7
周期性积分发放的调度由谁承担？

A) 积分服务内置定时任务（如 Spring Scheduler / Quartz）（MVP 简单，推荐）
B) 由独立的调度组件/外部触发
X) 其他（请在 [Answer]: 后描述）

[Answer]: A 

---
