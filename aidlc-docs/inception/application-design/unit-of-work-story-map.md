# 故事到单元映射（Unit of Work Story Map）

> 阶段：INCEPTION - 单元生成
> 时间：2026-06-10T16:01:50+08:00
> 说明：每条用户故事映射到承载其**核心业务逻辑**的后端单元；UI 部分统一由 Unit1 前端承载。Unit7 基础设施与 common 不承载用户故事（支撑性）。

---

## 1. 故事 → 单元映射表

| 故事 | 标题 | 后端单元 | 前端 | 优先级 |
|------|------|----------|:----:|--------|
| US-01 | 员工自助注册 | Unit2 认证（+Unit4 发积分） | Unit1 | Must |
| US-02 | 员工登录 | Unit2 认证（+Unit6 鉴权） | Unit1 | Must |
| US-03 | 退出登录 | Unit2 认证 | Unit1 | Should |
| US-04 | 浏览首页商品 | Unit3 商品 | Unit1 | Must |
| US-05 | 按分类浏览 | Unit3 商品 | Unit1 | Should |
| US-06 | 搜索商品 | Unit3 商品 | Unit1 | Should |
| US-07 | 查看商品详情 | Unit3 商品 | Unit1 | Must |
| US-08 | 确认兑换商品 | Unit5 兑换（→Unit3/Unit4） | Unit1 | Must |
| US-09 | 填写配送信息(实物) | Unit5 兑换 | Unit1 | Must |
| US-10 | 虚拟商品即时履约 | Unit5 兑换 | Unit1 | Must |
| US-11 | 兑换成功反馈 | Unit5 兑换 | Unit1 | Should |
| US-12 | 查看兑换历史 | Unit5 兑换 | Unit1 | Must |
| US-13 | 查看订单详情 | Unit5 兑换 | Unit1 | Must |
| US-14 | 取消兑换订单(发货前) | Unit5 兑换（→Unit3/Unit4） | Unit1 | Must |
| US-15 | 查看积分余额 | Unit4 积分 | Unit1 | Must |
| US-16 | 查看积分变动记录 | Unit4 积分 | Unit1 | Must |
| US-17 | 积分到期提示 | Unit4 积分 | Unit1 | Should |
| US-18 | 查看管理仪表盘 | Unit1（聚合各服务数据）+Unit6 鉴权 | Unit1 | Should |
| US-19 | 管理商品(增删改/上下架) | Unit3 商品 | Unit1 | Must |
| US-20 | 调整库存与上传图片 | Unit3 商品 | Unit1 | Must |
| US-21 | 管理二级分类 | Unit3 商品 | Unit1 | Must |
| US-22 | 配置积分规则 | Unit4 积分 | Unit1 | Must |
| US-23 | 手动调整用户积分 | Unit4 积分 | Unit1 | Must |
| US-24 | 查看用户积分变动 | Unit4 积分 | Unit1 | Should |
| US-25 | 管理兑换记录 | Unit5 兑换 | Unit1 | Must |
| US-26 | 更新发货状态(实物) | Unit5 兑换（→Unit3 扣减） | Unit1 | Must |
| US-27 | 管理用户 | Unit2 认证（+Unit4 余额展示） | Unit1 | Should |
| US-28 | 跨服务一致性 | Unit5 兑换（Saga）+Unit3/Unit4 | - | Must |

---

## 2. 单元 → 故事汇总

| 单元 | 承载故事 | 数量 |
|------|----------|------|
| Unit2 认证 | US-01, US-02, US-03, US-27 | 4 |
| Unit3 商品 | US-04, US-05, US-06, US-07, US-19, US-20, US-21 | 7 |
| Unit4 积分 | US-15, US-16, US-17, US-22, US-23, US-24 | 6 |
| Unit5 兑换 | US-08, US-09, US-10, US-11, US-12, US-13, US-14, US-25, US-26, US-28 | 10 |
| Unit6 网关 | （横切）US-02, US-18 鉴权/权限 | 横切 |
| Unit1 前端 | 全部故事的 UI（US-01~US-27） | 全部 UI |
| Unit7 基础设施 | 无（支撑性） | 0 |

---

## 3. 覆盖校验
- **故事全覆盖**：US-01~US-28 全部已分配到承载单元。✅
- **跨单元故事**：US-01（认证+积分）、US-08/US-14（兑换+商品+积分）、US-26（兑换+商品）、US-27（认证+积分）、US-18（前端聚合+网关）、US-28（兑换+商品+积分）——其编排已在 services.md 定义。
- **横切关注点**：鉴权（Unit6）、双语/可访问性（Unit1）贯穿多故事。
