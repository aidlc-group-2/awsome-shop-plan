# 代码生成摘要 — Unit 7（infrastructure / awsome-shop-deploy）

> 阶段：CONSTRUCTION - 代码生成（Part 2 完成）
> 时间：2026-06-10
> 负责人：Eric Yan

## 交付物清单

代码位置：`<工作区根>/awsome-shop-deploy/`

| 文件 | 说明 |
|------|------|
| `docker-compose.yml` | 8 容器编排（mysql + 6 业务服务占位 + nginx）、网络、卷、健康检查依赖 |
| `mysql/init/01-create-schemas.sh` | 建 3 个补充 schema + 授权（引用 `$MYSQL_USER`，不含建表，Q5=C）|
| `nginx/nginx.conf` | 单一入口：静态资源 + `/api` 反代网关（变量上游 + Docker DNS + rewrite 剥离前缀）|
| `.env.example` | 环境配置模板（DB/JWT/邮箱白名单/入职积分/时区）|
| `.gitignore` | 排除真实 `.env` |
| `README.md` | 一键启动、本机/EC2 运行、交接契约、运维命令 |

文档：本摘要位于 `aidlc-docs/construction/infrastructure/code/`。

## 验证结果（已实测）

| 验证项 | 结果 |
|--------|------|
| `docker compose config` 语法 | ✅ 通过 |
| `nginx -t` 配置语法 | ✅ 通过 |
| MySQL 启动 + 健康检查 | ✅ healthy |
| 4 个 schema 创建 | ✅ auth/product/points/order |
| 共享账号权限 | ✅ 对 4 schema 均 ALL PRIVILEGES，应用账号登录成功 |
| `docker compose down -v` 清理 | ✅ 通过 |

## 对队友的接入契约

- DB 连接：`jdbc:mysql://mysql:3306/awsomeshop_<service>`，账号取 `.env` 的 `MYSQL_APP_USER/PASSWORD`（默认 `awsomeshop`）
- 建表：各服务自管（Flyway 优先 / JPA ddl-auto）
- 服务端口（容器内）：auth 8001 / product 8002 / points 8003 / order 8004 / gateway 8080
- 健康检查：建议暴露 `/actuator/health`
- 图片卷：product-service 挂载 `product-images` 于 `/app/uploads`
- 前端产物：前端镜像最终阶段须含 `/app/dist` + `sh`/`cp`（不可 distroless）；`frontend` 容器导出到 `frontend-dist` 卷后退出，`nginx` 经 `service_completed_successfully` 等其完成
- 网关前缀：nginx 默认剥离 `/api` 前缀转发；若网关保留 `/api`，改 nginx.conf 的 rewrite
- 上传体积：nginx 限 10m；网关与 product-service 也需放宽到 ≥10m

## 已知取舍（MVP，经批准）

- **共享 DB 账号（Q4=B）**：单一账号对 4 个 schema 持 `ALL PRIVILEGES`（含 DROP），不满足最小权限；服务间数据隔离仅为约定。正式版改为每服务独立账号。
- **无业务建表（Q5=C，偏离 FR-I2 原文）**：FR-I2/components.md §7 原文写 infra 负责建表，本单元按批准的 Q5=C 不做，移交各服务自迁移——"无业务表"非缺陷。
- **业务服务无应用级健康检查**：依赖用 `service_started`，启动早期可能短暂 502；补 `/actuator/health` + `service_healthy` 属"构建与测试"增强项。

## 当前限制

- 业务服务源码目录尚不存在，`up --build` 在构建业务服务时会失败（预期）
- 完整端到端验证留待"构建与测试"阶段（Units 1~6 就绪后）
