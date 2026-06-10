# 代码生成计划 — Unit 7（infrastructure / awsome-shop-deploy）

> 阶段：CONSTRUCTION - 代码生成（Part 1 计划）
> 单元：Unit 7 基础设施/部署（负责人：Eric Yan）
> 设计来源：infrastructure-design.md + deployment-architecture.md
> 本计划是代码生成的唯一事实来源（single source of truth）。

---

## 单元上下文

- **覆盖需求**：FR-I1（编排）、FR-I2（DB 初始化）、FR-I3（网络）、FR-I4（卷）；NFR-6/7
- **关联用户故事**：无（基础设施为支撑性单元，不直接对应 US；详见 stories.md 附录 B）
- **依赖**：被全员运行期依赖；自身无业务依赖（构建顺序最先行）
- **本单元拥有的数据实体**：无（建表交各服务自管，Q5=C）
- **关键约束**：仅 Unit 7 就绪，业务服务源码目录尚不存在 → 业务服务以 `build:` 占位

## 代码位置

- **应用/部署代码**：`<工作区根>/awsome-shop-deploy/`（本机：`/Users/yabolin/Kiro/aidlc-awsome-shop/awsome-shop-deploy/`）
- **文档摘要**：`awsome-shop-plan/aidlc-docs/construction/infrastructure/code/`（仅 markdown）

---

## 生成步骤（按顺序执行）

- [x] **Step 1 · 项目结构搭建**
  创建 `awsome-shop-deploy/` 及子目录 `nginx/`、`mysql/init/`。

- [x] **Step 2 · docker-compose.yml**
  定义 8 个容器（mysql + 6 业务服务 + nginx）；网络 `awsomeshop-net`；卷 `mysql-data`/`product-images`/`frontend-dist`；`depends_on: service_healthy` 编排；业务服务 `build: ../awsome-shop-<service>` 占位；仅 nginx 暴露 80；环境变量从 `.env` 注入。

- [x] **Step 3 · MySQL 初始化脚本** `mysql/init/01-create-schemas.sh`
  创建补充 schema（utf8mb4）+ 授权；引用 `$MYSQL_USER` 避免硬编码；**不含任何业务建表**（Q5=C）。
  （评审后由 `.sql` 改为 `.sh`，修复 #4 硬编码用户名地雷）

- [x] **Step 4 · Nginx 配置** `nginx/nginx.conf`
  `/` 托管前端静态资源；`/api/` 反代网关（变量上游 + Docker DNS 推迟解析）；剥离 /api 前缀；透传 Authorization/X-Forwarded-*。

- [x] **Step 5 · 环境模板** `.env.example`
  DB 凭据、`JWT_SECRET`、`EMAIL_DOMAIN_WHITELIST`、`ONBOARDING_POINTS`、`TZ` 占位变量 + 注释。

- [x] **Step 6 · README.md + .gitignore**
  一键启动说明、本机/Staging EC2 两种运行位置、交接契约、数据重置命令；`.gitignore` 排除真实 `.env`。

- [x] **Step 7 · 文档摘要**
  `aidlc-docs/construction/infrastructure/code/code-summary.md`。

---

## 验证结果（已实测，2026-06-10）

- [x] `docker compose config` 语法校验通过
- [x] `nginx -t` 配置校验通过
- [x] `docker compose up -d mysql` 启动健康，4 个 schema（auth/product/points/order）创建成功
- [x] 共享账号 `awsomeshop` 对 4 个 schema 均有 ALL PRIVILEGES，应用账号登录成功
- [x] `docker compose down -v` 清理完成

---

## 验证方式（本阶段可独立做的）

- `docker compose config` 校验 compose 语法
- `nginx -t`（临时容器内）校验 nginx 配置
- 单独 `docker compose up -d mysql` 验证 4 schema 与账号创建成功
- 完整端到端验证留待"构建与测试"阶段（待 Units 1~6 代码就绪）

## 预估规模
7 个步骤，约 6 个工件文件 + 1 份文档摘要。不含传统业务逻辑/API/仓储层与单元测试（基础设施单元不适用）。

---

## 评审修复记录（2026-06-10，code review 后）

依据独立代码评审意见落地以下修复：

- **#1（Critical）前端→nginx 交接**：compose 中 `nginx` 增加 `depends_on: frontend: service_completed_successfully`；README 写死前端镜像契约（`/app/dist` + `sh`/`cp`，禁 distroless）。
- **#2（Important）文档漏 frontend-dist 卷**：infrastructure-design.md §5.2 + 拓扑图 + deployment-architecture.md §1 补齐第 3 个卷与导出流程。
- **#3（Important）健康检查文档漂移**：设计文档 §3.2 / 部署架构 §3 如实改为"仅 mysql=service_healthy，业务服务=service_started、无应用级健康检查"，并标注为 MVP 已知简化。
- **#4（Important）init 硬编码用户名**：`01-create-schemas.sql` → `01-create-schemas.sh`，引用 `$MYSQL_USER`。已实测：改 `MYSQL_APP_USER=shopapp` 后授权正确跟随 shopapp（旧 .sql 会报错中断）。
- **#5 / #6（已知取舍标注）**：在 infrastructure-design.md §4 与 code-summary.md 显式标注共享账号 ALL 权限（Q4=B）与不建表偏离 FR-I2（Q5=C）为已批准取舍。

**复测结果**：`docker compose config` ✅、`nginx -t` ✅、MySQL 健康 + 4 schema ✅、默认/改名用户授权均正确 ✅、`down -v` 清理 ✅。
