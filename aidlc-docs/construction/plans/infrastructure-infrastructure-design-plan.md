# 基础设施设计计划 — Unit 7（infrastructure / awsome-shop-deploy）

> 阶段：CONSTRUCTION - 基础设施设计（Infrastructure Design）
> 单元：Unit 7 基础设施/部署
> 负责人：Eric Yan
> 输入：requirements.md（FR-I1~I4, NFR-6/7）、application-design/components.md(§7)、services.md、unit-of-work.md、现有 infra/（Staging EC2 + SSM）

---

## A. 设计任务清单（Checklist）

- [x] A1. 确定部署目标拓扑（→ 本地 docker-compose 单环境，Q1=A）
- [x] A2. 定义 docker-compose 服务编排（6 服务 + MySQL + Nginx）与启动依赖顺序
- [x] A3. 定义镜像策略（→ 混合：官方镜像 + 业务服务 build，Q2=C）
- [x] A4. 设计 MySQL 拓扑（→ 单实例多 Schema + 共享账号，Q4=B；建表交服务自迁移，Q5=C）
- [x] A5. 设计容器网络（网络名、内网信任、服务发现方式）
- [x] A6. 设计持久化卷（MySQL 数据卷 + 上传文件卷，Q6=A）
- [x] A7. 设计 Nginx 前置（静态资源 + 反向代理到网关，Q8=A）
- [x] A8. 设计环境配置（.env 模板，Q7=A）
- [x] A9. 输出 infrastructure-design.md 与 deployment-architecture.md

---

## B. 待你确认的关键决策（请在每个 [Answer]: 后填字母；不合适选 Other 自述）

### Question 1 — 部署目标范围
本次基础设施设计要覆盖哪些部署环境？

A) 仅本地开发环境（每位成员本机 `docker compose up` 一键起全栈）
B) 仅已有的 Staging EC2（团队统一在 `i-0d1d69a9339074fef` 上联调）
C) 两者都要：同一套 compose 既能本机起，也能在 Staging EC2 上起（推荐）
D) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question 2 — 镜像/构建策略（其他单元代码尚未就绪）
当前只有 Unit 7，其余服务代码还没生成。compose 如何引用各服务？

A) 用 `build:` 指向各服务子目录（待代码就绪后可直接构建）；现在先放占位结构
B) 用预构建镜像 `image:`（各单元自行推送镜像到仓库，compose 只拉取）
C) 混合：MySQL/Nginx 用官方镜像，业务服务用 `build:` 指向源码目录
D) Other (please describe after [Answer]: tag below)

[Answer]: C

---

### Question 3 — 代码组织落地方式
unit-of-work.md 规划为 Polyrepo（每单元独立仓库）。本地一键编排时，代码如何摆放？

A) Monorepo 风格：在 `awsome-shop-deploy` 同级放各服务目录，compose 用相对路径 build
B) 纯 Polyrepo：各服务独立仓库，compose 假定它们已 clone 到约定路径
C) 仅交付 compose + 配置，由使用者自行准备各服务镜像/源码
D) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question 4 — MySQL 账号与初始化
4 个 Schema（auth/product/points/order）的数据库账号怎么设计？

A) 单一 root + 每服务一个独立账号（各自仅能访问自己的 schema），最小权限
B) 单一共享应用账号，可访问全部 4 个 schema（简单，MVP 够用）
C) Other (please describe after [Answer]: tag below)

[Answer]: B

---

### Question 5 — 数据库建表脚本归属
各 Schema 的建表 DDL 由谁产出、放在哪里？

A) Unit 7 统一编写并维护全部 4 个 schema 的 init SQL（依据各单元功能设计的数据表）
B) 各单元自己产出本服务的 init SQL，Unit 7 只负责 MySQL 容器与挂载目录
C) Unit 7 先建空 schema + 账号，建表交由各服务启动时自动迁移（如 Flyway/JPA ddl-auto）
D) Other (please describe after [Answer]: tag below)

[Answer]: C （审核后由 A 调整为 C：Unit 7 只建空 schema + 账号，建表交各服务 JPA ddl-auto/Flyway 自动迁移，彻底解耦、零返工）

---

### Question 6 — 上传文件存储（商品图片）
FR-PR6 的图片上传用本地卷。卷如何在服务间共享？

A) 命名卷由 product-service 独占挂载（图片只在商品服务读写，URL 经网关/Nginx 暴露）
B) 命名卷同时挂载给 product-service 与 Nginx（Nginx 直接静态服务图片，减轻后端）
C) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question 7 — 密钥与敏感配置管理（MVP 阶段）
JWT 签名密钥、DB 密码等敏感配置如何管理？

A) 统一放 `.env` 文件（提供 `.env.example` 模板，真实 .env 不入库），MVP 够用
B) 用 Docker secrets / 外部密钥管理（更安全，但配置更复杂）
C) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question 8 — Nginx 与网关的职责边界
对外入口怎么分层？

A) Nginx 仅托管前端静态资源；所有 /api 请求由 Nginx 反代到 API 网关(8080)，网关再路由
B) 前端直连网关(8080)，Nginx 只托管静态资源、不反代 API
C) Other (please describe after [Answer]: tag below)

[Answer]: A
```
