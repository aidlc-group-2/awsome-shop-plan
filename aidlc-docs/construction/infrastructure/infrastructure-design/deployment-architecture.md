# 部署架构 — Unit 7（infrastructure / awsome-shop-deploy）

> 阶段：CONSTRUCTION - 基础设施设计
> 配套文档：infrastructure-design.md
> 目标环境：本地 docker-compose 单环境（同一份 compose 亦可在 Staging EC2 `i-0d1d69a9339074fef` 上运行）

---

## 1. 部署拓扑图（ASCII）

```
                         宿主机 / Staging EC2
                              │  :80
                    ┌─────────▼──────────┐
                    │   nginx:1.27        │  唯一对外入口
                    │  / → 前端静态资源    │
                    │  /api → 反代网关     │
                    └─────────┬──────────┘
                              │ (awsomeshop-net 内网)
                    ┌─────────▼──────────┐
                    │ api-gateway :8080   │  JWT校验/角色鉴权/路由
                    └──┬────┬────┬────┬───┘
            ┌──────────┘    │    │    └──────────┐
            ▼               ▼    ▼               ▼
     ┌───────────┐  ┌───────────┐ ┌───────────┐ ┌───────────┐
     │auth :8001 │  │product8002│ │points :8003│ │order :8004│
     └─────┬─────┘  └─────┬─────┘ └─────┬─────┘ └──┬──┬──┬───┘
           │              │             │          │  │  │
           │              │             │   order→product/points (内网直调)
           └──────────────┴──────┬──────┴──────────┘
                                  ▼
                        ┌──────────────────┐
                        │   mysql:8.4       │
                        │  4 schema:        │
                        │  auth/product/    │
                        │  points/order     │
                        └────────┬──────────┘
                                 │
                   卷: mysql-data │   卷: product-images → product-service
                                 │   卷: frontend-dist  → frontend(写) / nginx(读)
```

文本说明：所有外部流量经 Nginx:80 进入；`/api` 反代到 API 网关，网关按前缀路由到 4 个业务服务；业务服务端口不对宿主暴露，仅在 `awsomeshop-net` 内互通；order-service 在网内直调 product/points；所有服务连同一个 MySQL 实例的各自 schema。

---

## 2. 目录结构（Q3=A · Monorepo 风格同级摆放）

```
<工作区根>/
├── awsome-shop-deploy/            # Unit 7 交付物（本单元）
│   ├── docker-compose.yml
│   ├── .env.example
│   ├── nginx/
│   │   └── nginx.conf
│   └── mysql/
│       └── init/
│           └── 01-create-schemas.sh    # 仅建空 schema + 账号（引用 $MYSQL_USER）
├── awsome-shop-auth-service/      # Unit 2（队友，build 上下文）
├── awsome-shop-product-service/   # Unit 3
├── awsome-shop-points-service/    # Unit 4
├── awsome-shop-order-service/     # Unit 5
├── awsome-shop-api-gateway/       # Unit 6
└── awsome-shop-frontend/          # Unit 1
```

> compose 中各业务服务 `build: ../awsome-shop-<service>`，相对 `awsome-shop-deploy` 目录定位源码。

---

## 3. 启动顺序与健康编排

| 阶段 | 容器 | 就绪条件 |
|------|------|----------|
| 1 | mysql | `mysqladmin ping` 通过（`service_healthy`）|
| 2 | auth / product / points | mysql `service_healthy` 后启动 |
| 3 | order | product/points `service_started`（进程启动）|
| 4 | api-gateway | 4 业务服务 `service_started` |
| - | frontend | 一次性导出 SPA 产物到 `frontend-dist` 卷后退出 |
| 5 | nginx | frontend `service_completed_successfully` + gateway `service_started` |

> **就绪语义说明**：仅 mysql 有真正的健康检查；业务服务用 `service_started`，只代表容器进程已启动、不代表 Spring 应用就绪。启动早期可能短暂 502，配合 `restart: unless-stopped` 最终收敛。为业务服务补 `/actuator/health` 健康检查、升级为 `service_healthy` 属"构建与测试"阶段的增强项。

一键启动：`docker compose up -d --build`；查看状态：`docker compose ps`；日志：`docker compose logs -f <svc>`。

---

## 4. 两种运行位置

### 4.1 本机（主用）
```bash
cp .env.example .env   # 填入本地密钥
docker compose up -d --build
# 浏览器访问 http://localhost
```

### 4.2 Staging EC2（联调，复用同一份 compose）
- 机器已具备 Docker 25 + Compose v2.39.4（见 infra/user-data.sh）。
- 经 SSM 登录（见 infra/ACCESS.md），在 `/opt/awsomeshop` 下拉取代码并 `docker compose up -d --build`。
- 本机经 SSM 端口转发访问（无需开放安全组）：
  ```bash
  aws ssm start-session --target i-0d1d69a9339074fef --region us-east-1 \
    --document-name AWS-StartPortForwardingSession \
    --parameters '{"portNumber":["80"],"localPortNumber":["8088"]}'
  # 然后访问 http://localhost:8088
  ```

> 设计上无需为两种位置维护两套 compose——同一份即可，差异仅在 `.env`。

---

## 5. 当前阶段限制与后续衔接

- **现状**：仅 Unit 7 就绪，业务服务源码目录尚不存在。`docker compose up` 会在 build 业务服务时失败——这是预期的。
- **可独立验证的部分**：单独启动 mysql + nginx 容器，验证 schema 创建、账号、网络、卷、Nginx 配置语法（`nginx -t`）。
- **完整端到端验证**：留待"构建与测试"阶段，待 Units 1~6 代码就绪后统一进行。
- **交接契约**（供队友遵守）：
  - DB 连接：`jdbc:mysql://mysql:3306/awsomeshop_<service>`，账号取自 `.env` 的 `MYSQL_APP_USER/PASSWORD`
  - 建表：各服务自管（Flyway 优先 / JPA ddl-auto 备选）
  - 服务端口：auth 8001 / product 8002 / points 8003 / order 8004 / gateway 8080（容器内）
  - 健康检查：暴露 `/actuator/health`

---

## 6. 回滚与运维（MVP 级）

- 按服务独立重建：`docker compose up -d --build <svc>`
- 数据重置：`docker compose down -v`（⚠️ 删除卷会清空 MySQL 数据与图片）
- 成本控制（EC2）：不用时 `aws ec2 stop-instances`（见 ACCESS.md）
- 生产级监控/告警/CI-CD：属未来 OPERATIONS 阶段，本次不含
