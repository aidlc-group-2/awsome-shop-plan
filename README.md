# AWSomeShop - 内部员工福利电商平台

## 项目概述

AWSomeShop 是一个面向企业内部员工的福利积分电商平台 MVP。员工通过积分兑换商品，管理员负责商品管理、积分发放和订单处理。

## 参考系统架构

```
Frontend (Port 3000)
       |
    Nginx (静态资源 + 反向代理)
       |
API Gateway (Port 8080)
       |
+------+------+------+------+
|      |      |      |      |
v      v      v      v      v
Auth  Product Points Order  (微服务)
8001  8002    8003   8004
|      |      |      |
+------+------+------+------+
       |
    MySQL (Port 3306)
    4 个独立 Schema
```

**关键架构模式**：微服务架构、API Gateway 统一入口、SPA 前端、JWT 无状态认证、悲观锁并发控制、Saga 补偿事务、Docker 容器化部署

## 工作单元

| 单元 | 名称 | 端口 | 说明 |
|------|------|------|------|
| Unit 1 | 目前无 | 3000 | SPA 前端应用（员工端 + 管理端） |
| Unit 2 | 目前无 | 8001 | 认证服务（注册/登录/JWT/角色管理） |
| Unit 3 | 目前无 | 8002 | 商品服务（商品 CRUD/分类管理/文件上传） |
| Unit 4 | 目前无 | 8003 | 积分服务（余额管理/自动发放/手动调整） |
| Unit 5 | 目前无 | 8004 | 兑换服务（下单/状态管理/跨服务调用） |
| Unit 6 | 目前无 | 8080 | API 网关（认证校验/路由转发/权限控制） |
| Unit 7 | 目前无 | 3306 | 基础设施（Docker Compose/MySQL/网络） |

**开发顺序**：Unit 7 → Unit 2 → Unit 6 → Unit 3 + Unit 4（可并行） → Unit 5 → Unit 1

## 核心功能

### 员工端
- 浏览和搜索商品
- 积分余额查询和变动记录
- 商品兑换和订单管理
- 个人兑换历史

### 管理端
- 仪表盘数据概览
- 商品和分类管理（支持二级分类）
- 积分规则配置和手动调整
- 兑换记录管理和发货状态跟踪
- 用户管理和积分变动查看

## 需求规模

- **用户故事**：25 个，覆盖 9 条用户旅程
- **用户画像**：3 个（技术型员工、非技术行政、HR 管理员）
- **验收标准**：130+ 条（Given-When-Then 格式）

## 非功能性要求

| 类别 | 目标 |
|------|------|
| 性能 | 页面加载 < 3s，API 响应 < 500ms，网关开销 P95 ≤ 50ms |
| 安全 | bcrypt 密码加密，JWT 令牌，CSRF/XSS/SQL 注入防护 |
| 部署 | Docker 容器化，docker-compose 一键启动 |
| 存储 | MySQL 8.4 LTS，本地文件存储（Docker Volume） |
| 可维护性 | 分层架构，API 文档，单元测试 |
| 可访问性 | WCAG 2.1 AA，支持最近两个主流浏览器版本 |

## UI 设计

项目包含完整的 UI 设计稿（`doc/awsome-shop.pen`），使用 Pencil 设计工具制作，涵盖：

### 员工端页面（9 个）
- 登录页、商城首页、商品详情页
- 确认兑换、配送信息、兑换成功
- 订单详情、兑换历史、积分中心

### 管理端页面（7 个）
- 仪表盘、商品管理（卡片模式）、商品详情
- 分类管理（树形表格）、积分规则管理
- 兑换记录管理、用户管理

### 弹窗/对话框（10 个）
- 商品：下架确认、调整库存、上传图片、编辑商品
- 分类：新增/编辑/删除确认
- 积分规则：新增/编辑
- 发货：修改发货状态
- 用户：调整用户积分

### 详情页面（2 个）
- 兑换记录详情（完整管理页面）
- 用户积分变动记录

## 仓库结构

```
awsome-shop-plan/
├── README.md                           # 本文件
├── doc/
│   ├── original-intent.md              # 原始需求意图
│   └── awsome-shop.pen                 # UI 设计稿
├── aidlc-docs/                         # AI-DLC 流程文档
│   ├── aidlc-state.md                  # 项目状态跟踪
│   ├── audit.md                        # 完整审计日志
│   ├── inception/                      # 启动阶段
│   │   ├── requirements/               # 需求分析（3 个文件）
│   │   ├── user-stories/               # 用户故事和画像
│   │   ├── application-design/         # 应用设计（7 个文件）
│   │   └── plans/                      # 规划文档（5 个文件）
│   └── construction/                   # 构建阶段
│       ├── plans/                      # 构建规划（18 个文件）
│       ├── infrastructure/             # Unit 7 基础设施设计
│       ├── auth-service/               # Unit 2 认证服务设计
│       ├── product-service/            # Unit 3 商品服务设计
│       ├── points-service/             # Unit 4 积分服务设计
│       ├── order-service/              # Unit 5 兑换服务设计
│       ├── api-gateway/                # Unit 6 网关服务设计
│       └── awsomeshop-frontend/        # Unit 1 前端应用设计
└── .kiro/                              # AI-DLC 工作流规则
```

## 关联代码仓库

| 仓库 | 说明 |
|------|------|
| awsome-shop-frontend | 前端 SPA 应用 |
| awsome-shop-auth-service | 认证微服务 |
| awsome-shop-product-service | 商品微服务 |
| awsome-shop-points-service | 积分微服务 |
| awsome-shop-order-service | 兑换微服务 |
| awsome-shop-api-gateway | API 网关 |
| awsome-shop-deploy | 部署编排（Docker Compose） |


