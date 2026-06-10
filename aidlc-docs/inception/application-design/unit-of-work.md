# 工作单元定义（Unit of Work）

> 阶段：INCEPTION - 单元生成
> 时间：2026-06-10T16:01:50+08:00
> 决策：7 个工作单元 · 多仓库 Polyrepo · 各服务独立 Maven 项目（common 本地依赖）· docker-compose 归 Unit7
> 注：工作单元 = 用户故事的逻辑分组（独立可部署服务）；`common` 无用户故事，定位为支撑性共享库（非工作单元）。

---

## 工作单元清单

| 单元 | 名称 | 仓库 | 端口 | 类型 | 技术 |
|------|------|------|------|------|------|
| Unit 1 | 前端 SPA | awsome-shop-frontend | 3000 | 前端 | React + TS + Nginx |
| Unit 2 | 认证服务 | awsome-shop-auth-service | 8001 | 微服务 | Spring Boot |
| Unit 3 | 商品服务 | awsome-shop-product-service | 8002 | 微服务 | Spring Boot |
| Unit 4 | 积分服务 | awsome-shop-points-service | 8003 | 微服务 | Spring Boot |
| Unit 5 | 兑换服务 | awsome-shop-order-service | 8004 | 微服务 | Spring Boot |
| Unit 6 | API 网关 | awsome-shop-api-gateway | 8080 | 网关 | Spring Cloud Gateway |
| Unit 7 | 基础设施/部署 | awsome-shop-deploy | 3306(MySQL) | 编排 | Docker Compose + MySQL + Nginx |

**支撑性共享库（非工作单元）**：`awsome-shop-common`（Maven 库，先构建并 `mvn install` 到本地仓库，供 Unit2~6 引用）。

---

## 各单元职责

### Unit 1 · 前端 SPA（awsome-shop-frontend）
- 员工端（9 页）+ 管理端（7 页）+ 弹窗（10）+ 详情（2）
- 双语 i18n、基于角色的路由守卫、统一 API 客户端（指向网关）
- WCAG 2.1 AA 可访问性
- **覆盖**：FR-F1~F6；渲染所有用户故事的 UI

### Unit 2 · 认证服务（awsome-shop-auth-service）
- 注册（企业邮箱白名单）、登录、JWT 签发、登出、令牌校验
- 用户与角色管理；注册成功后同步调用积分服务发放入职奖励
- **覆盖**：FR-A1~A7

### Unit 3 · 商品服务（awsome-shop-product-service）
- 商品 CRUD/上下架/搜索/详情、二级分类、图片上传
- 库存预占/释放/扣减（悲观锁），供兑换服务调用
- **覆盖**：FR-PR1~PR7

### Unit 4 · 积分服务（awsome-shop-points-service）
- 余额、批次(FIFO)、变动流水、规则配置
- 发放（入职/周期/手动）、扣减、退回；定时发放与积分过期
- **覆盖**：FR-P1~P8

### Unit 5 · 兑换服务（awsome-shop-order-service）
- 兑换订单生命周期、编排式 Saga（扣积分→预占库存→建单 + 补偿）
- 履约：实物发货（库存正式扣减）/虚拟即时；发货前取消退回
- **覆盖**：FR-O1~O10

### Unit 6 · API 网关（awsome-shop-api-gateway）
- 统一入口、JWT 校验、角色鉴权、路由、限流、安全头、CORS
- **覆盖**：FR-G1~G5

### Unit 7 · 基础设施/部署（awsome-shop-deploy）
- docker-compose 一键启动全部服务 + MySQL（4 schema）+ Nginx
- 数据库初始化脚本、网络、持久化卷
- **覆盖**：FR-I1~I4

---

## 代码组织策略（Polyrepo）

```
（多仓库，每个工作单元一个 Git 仓库）
awsome-shop-common          # 支撑性共享库（Maven，先 mvn install）
awsome-shop-auth-service    # Unit2
awsome-shop-product-service # Unit3
awsome-shop-points-service  # Unit4
awsome-shop-order-service   # Unit5
awsome-shop-api-gateway     # Unit6
awsome-shop-frontend        # Unit1
awsome-shop-deploy          # Unit7（docker-compose、init-sql、nginx 配置）
```

- 各后端服务为**独立 Maven 项目**，编译期依赖 `awsome-shop-common`（本地安装或私服发布）。
- 每个仓库各自构建独立的 Docker 镜像；`awsome-shop-deploy` 通过 docker-compose 编排所有镜像。
- 数据库：单 MySQL 实例，4 个独立 schema（auth/product/points/order）。

> 说明：当前 AI-DLC 工作区为规划/文档仓库；实际代码将在 CONSTRUCTION 阶段按上述 Polyrepo 策略在工作区根目录下分目录生成（每个目录对应一个未来仓库），便于本地一键编排。

---

## 构建/开发顺序
```
awsome-shop-common（支撑库，先行）
  → Unit7 基础设施（MySQL/网络就绪）
  → Unit2 认证
  → Unit6 网关
  → (Unit3 商品 ∥ Unit4 积分)   # 可并行
  → Unit5 兑换
  → Unit1 前端
```
