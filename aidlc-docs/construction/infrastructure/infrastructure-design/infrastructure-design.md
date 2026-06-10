# 基础设施设计 — Unit 7（infrastructure / awsome-shop-deploy）

> 阶段：CONSTRUCTION - 基础设施设计
> 单元：Unit 7 基础设施/部署（负责人：Eric Yan）
> 设计依据：infrastructure-infrastructure-design-plan.md（Q1=A, Q2=C, Q3=A, Q4=B, Q5=C, Q6=A, Q7=A, Q8=A）
> 原则：最简可跑（MVP）；Unit 7 完全自有、不依赖其他单元功能设计即可交付

---

## 1. 设计决策总览

| 维度 | 决策 | 来源 |
|------|------|------|
| 部署目标 | 本地单环境 `docker compose up` 一键起全栈（同一份 compose 亦可在 Staging EC2 运行）| Q1=A |
| 镜像策略 | MySQL/Nginx 用官方镜像；6 个业务服务用 `build:` 指向同级源码目录 | Q2=C |
| 代码组织 | Monorepo 风格：`awsome-shop-deploy` 与各服务目录同级，compose 用相对路径 build | Q3=A |
| MySQL 拓扑 | 单实例，4 个独立 schema（auth/product/points/order）| Q4=B |
| DB 账号 | 单一共享应用账号访问全部 schema | Q4=B |
| 建表 DDL | **不由 Unit 7 负责**；各服务用 JPA `ddl-auto` / Flyway 自动迁移；Unit 7 仅建空 schema + 账号 | Q5=C |
| 图片卷 | 命名卷由 product-service 独占挂载 | Q6=A |
| 敏感配置 | `.env` 文件 + `.env.example` 模板（真实 .env 不入库）| Q7=A |
| 对外入口 | Nginx 托管前端静态资源 + 反代 `/api` → API 网关(8080) | Q8=A |

---

## 2. 逻辑组件 → 基础设施映射

| 逻辑组件（来自 application-design §7）| 基础设施实现 | 镜像/来源 |
|------|------|------|
| `docker-compose.yml` 编排 | Compose v2 单文件 | — |
| MySQL（4 schema） | `mysql:8.4` 容器，单实例 | 官方镜像 |
| `init-scripts` | 仅创建 4 个空 schema + 共享账号（**不含建表**）| `/docker-entrypoint-initdb.d` 挂载 |
| Docker Network | 单个 bridge 网络 `awsomeshop-net`（内网信任）| — |
| Volumes | `mysql-data`（DB）+ `product-images`（图片）+ `frontend-dist`（前端产物交接）| 命名卷 |
| Nginx 前置 | `nginx:1.27-alpine`，静态资源 + `/api` 反代 | 官方镜像 |
| Auth/Product/Points/Order/Gateway/Frontend | 6 个业务容器，`build:` 各自源码目录 | 待队友代码就绪 |

---

## 3. 服务编排与依赖

### 3.1 容器清单

| 容器 | 镜像/构建 | 端口（容器内）| 暴露到宿主 | 依赖 |
|------|-----------|--------------|-----------|------|
| `mysql` | `mysql:8.4` | 3306 | 3306（可选，调试用）| — |
| `auth-service` | build ../awsome-shop-auth-service | 8001 | 否（仅内网）| mysql |
| `product-service` | build ../awsome-shop-product-service | 8002 | 否 | mysql |
| `points-service` | build ../awsome-shop-points-service | 8003 | 否 | mysql |
| `order-service` | build ../awsome-shop-order-service | 8004 | 否 | mysql, product, points |
| `api-gateway` | build ../awsome-shop-api-gateway | 8080 | 否（经 Nginx）| auth, product, points, order |
| `frontend` | build ../awsome-shop-frontend | （构建产物）| — | — |
| `nginx` | `nginx:1.27-alpine` | 80 | **80（唯一对外入口）** | api-gateway, frontend 产物 |

> 业务服务端口默认**不暴露到宿主**，仅通过容器网络互通；唯一对外入口是 Nginx:80。调试时可临时为 MySQL/网关加 `ports`。

### 3.2 启动依赖（`depends_on`）

```
mysql (healthcheck: mysqladmin ping → healthy)
  └── auth / product / points 服务（condition: service_healthy 等 mysql）
        └── order 服务（condition: service_started，等 product/points 进程启动）
              └── api-gateway（condition: service_started，等各业务服务进程启动）
frontend（一次性导出产物到 frontend-dist 卷后退出）
  └── nginx（condition: service_completed_successfully 等 frontend；service_started 等 gateway）
```

- **仅 mysql 使用 `service_healthy`**（自带 mysqladmin ping 健康检查）。
- **业务服务（auth/product/points/order/gateway）当前未定义应用级健康检查**，服务间依赖用 `service_started`——只保证"容器进程已启动"，不保证"Spring 应用就绪"。配合 `restart: unless-stopped`，启动早期可能短暂出现 502/连接错误，最终收敛。
- **已知简化（MVP）**：未对业务服务配置 `/actuator/health` 健康检查。后续可在"构建与测试"阶段为各服务补 healthcheck 并把依赖升级为 `service_healthy`，以消除启动期抖动。
- nginx 通过 `service_completed_successfully` 等待 frontend 把 SPA 产物导出到 `frontend-dist` 卷后再启动（避免首批请求 404）。

---

## 4. 数据层设计（Q4=B + Q5=C）

### 4.1 MySQL 实例
- 镜像：`mysql:8.4`（LTS）
- 单实例承载 4 个 schema：`awsomeshop_auth` / `awsomeshop_product` / `awsomeshop_points` / `awsomeshop_order`
- 字符集：`utf8mb4` / `utf8mb4_unicode_ci`（支持中英文双语内容）
- 数据持久化：命名卷 `mysql-data` → `/var/lib/mysql`

### 4.2 初始化脚本职责边界（关键）
Unit 7 的初始化脚本 **只做两件事**，不碰任何业务表：
1. 创建 4 个空 schema
2. 创建/授权 1 个共享应用账号对这 4 个 schema 的权限

实现方式：`awsomeshop_auth` 与应用账号由官方入口环境变量（`MYSQL_DATABASE`/`MYSQL_USER`/`MYSQL_PASSWORD`）自动创建；`01-create-schemas.sh` 补建其余 3 个 schema 并扩展授权。

> **为什么用 `.sh` 而非 `.sql`**：`.sql` 初始化文件不做环境变量替换，会把授权用户名硬编码（一旦改了 `MYSQL_APP_USER` 就会 GRANT 给不存在的用户并中断初始化）。`.sh` 引用 `$MYSQL_USER`，授权对象与 `.env` 始终一致。

```bash
# mysql/init/01-create-schemas.sh（要点）
mysql -uroot -p"${MYSQL_ROOT_PASSWORD}" <<-EOSQL
  CREATE DATABASE IF NOT EXISTS awsomeshop_product CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
  CREATE DATABASE IF NOT EXISTS awsomeshop_points  CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
  CREATE DATABASE IF NOT EXISTS awsomeshop_order   CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
  GRANT ALL PRIVILEGES ON awsomeshop_product.* TO '${MYSQL_USER}'@'%';
  GRANT ALL PRIVILEGES ON awsomeshop_points.*  TO '${MYSQL_USER}'@'%';
  GRANT ALL PRIVILEGES ON awsomeshop_order.*   TO '${MYSQL_USER}'@'%';
  FLUSH PRIVILEGES;
EOSQL
```

> **已知取舍（Q4=B，安全）**：单一共享账号对全部 4 个 schema 持有 `ALL PRIVILEGES`（含 DROP），不满足最小权限原则——任一服务被攻破即可读写其他服务的数据，"各服务拥有自己的表"仅为团队约定、非数据库强制。MVP 内网环境可接受；正式版应改为每服务独立账号、仅授权自己的 schema。

> **与 FR-I2 的偏离说明（Q5=C）**：`requirements.md` FR-I2 与 `components.md` §7 原文写 infra 负责"建表与种子数据"，本单元按已批准的 Q5=C 决策**不做建表**（移交各服务自管）。此处显式标注，避免后续读者把"无业务表"误判为缺陷。

### 4.3 建表责任移交各服务（Q5=C）
- 每个业务服务在启动时自行创建/迁移自己的表。约定（写入团队约定，供各单元遵守）：
  - **推荐 Flyway**：各服务 `src/main/resources/db/migration/` 放版本化迁移脚本，生产可控
  - **或 JPA `spring.jpa.hibernate.ddl-auto=update`**：MVP 快速起步（不建议用于正式数据）
- Unit 7 提供的契约：schema 已存在、账号已就绪、连接串通过环境变量注入（见 §6）。

---

## 5. 网络与卷

### 5.1 网络
- 单个 bridge 网络 `awsomeshop-net`，所有容器加入。
- 服务发现：靠 compose 的服务名 DNS（如 `jdbc:mysql://mysql:3306/...`、`http://api-gateway:8080`）。
- 内网信任（components.md 的 `InternalAuthFilter`）：业务服务端口不对宿主暴露，仅网络内可达；内部接口靠 `InternalAuthFilter` 识别来自网关的请求。

### 5.2 卷
| 卷名 | 挂载到 | 用途 |
|------|--------|------|
| `mysql-data` | `mysql:/var/lib/mysql` | 数据库持久化 |
| `product-images` | `product-service:/app/uploads`（独占，Q6=A）| 商品图片上传持久化 |
| `frontend-dist` | `frontend:/export`（写）+ `nginx:/usr/share/nginx/html:ro`（读）| 前端 SPA 构建产物交接 |

> **前端产物导出流程**：`frontend` 容器为一次性导出器——构建产物在镜像内 `/app/dist`，启动时 `cp` 到 `frontend-dist` 卷后退出；`nginx` 通过 `service_completed_successfully` 等其完成后再启动，从该卷只读提供静态资源。前端镜像最终阶段须含 `sh`/`cp`（不可用 distroless）。

> 图片对外访问：product-service 经网关返回图片 URL；如需更高性能，后续可让 Nginx 直挂 `product-images` 卷做静态服务（当前 MVP 不做）。

---

## 6. 环境配置（Q7=A）

通过 `.env`（不入库）+ `.env.example`（入库模板）注入。关键变量：

| 变量 | 说明 | 示例（占位）|
|------|------|------|
| `MYSQL_ROOT_PASSWORD` | MySQL root 密码 | （随机强密码）|
| `MYSQL_APP_USER` | 共享应用账号 | `awsomeshop` |
| `MYSQL_APP_PASSWORD` | 应用账号密码 | （随机强密码）|
| `JWT_SECRET` | JWT 签名密钥（网关与 auth 共享）| （≥32 字节随机串）|
| `EMAIL_DOMAIN_WHITELIST` | 注册企业邮箱白名单 | `example.com` |
| `ONBOARDING_POINTS` | 入职奖励积分额度 | `1000` |
| `TZ` | 时区 | `Asia/Shanghai` |

各服务通过 `environment:` 读取，如 `SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/awsomeshop_auth`。

---

## 7. Nginx 入口（Q8=A）

- 监听宿主 80 端口，作为**唯一对外入口**。
- 路由规则：
  - `/` → 前端静态资源（来自 `frontend-dist` 卷），SPA 路由回退到 `index.html`
  - `/api/` → 反代到 API 网关；用变量上游 + `resolver 127.0.0.11`（Docker 内置 DNS）**推迟上游解析到请求时**，使 nginx 在网关未就绪时仍能启动（返 502 而非启动失败）
  - `rewrite ^/api/(.*)$ /$1 break` **剥离 `/api` 前缀**（`/api/auth/login → /auth/login`）；若网关期望保留 `/api`，删除该 rewrite
- 单一来源(same-origin)，前端无需处理跨域 CORS。
- 转发时透传 `Authorization`、`X-Forwarded-*` 等头；`client_max_body_size 10m` 支持图片上传（网关与 product-service 也需放宽到 ≥10m）。

---

## 8. Unit 7 现在可交付 vs 依赖他人

| 交付物 | 现在可完成 | 说明 |
|--------|:---------:|------|
| docker-compose.yml 骨架 | ✅ | 业务服务 `build:` 占位，待源码接入 |
| MySQL 容器 + schema/账号 init SQL | ✅ | 完全自有 |
| 网络 + 卷定义 | ✅ | 完全自有 |
| nginx.conf | ✅ | 反代规则固定 |
| .env.example | ✅ | 完全自有 |
| 业务服务实际可构建 | ⛔ 依赖 | 需 Units 1~6 代码就绪 |
| 端到端启动验证 | ⛔ 依赖 | 全员就绪后在"构建与测试"阶段统一验证 |

> 结论：Q5=C 的解耦让 Unit 7 现在即可独立交付全部编排骨架，不必等待任何其他单元的功能设计或数据模型。

---

## 9. 与需求/NFR 对齐

- FR-I1 容器编排 → §3 compose 编排
- FR-I2 数据库初始化（4 schema）→ §4 init SQL（建空库 + 账号）
- FR-I3 网络/服务发现 → §5.1 bridge 网络 + 服务名 DNS
- FR-I4 持久化卷 → §5.2 mysql-data + product-images
- NFR-6 容器化一键启动 → §1/§3 单文件 compose
- NFR-7 存储（MySQL 8.4 + 本地卷）→ §4/§5.2
- 安全（NFR-3 部分）→ §7 单入口 + 业务端口不外露 + .env 管密钥
