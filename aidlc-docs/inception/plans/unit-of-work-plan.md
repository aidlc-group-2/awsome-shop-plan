# 单元拆解计划（Unit of Work Plan）

> 阶段：INCEPTION - 单元生成（Part 1 规划）
> 时间：2026-06-10T15:58:36+08:00
> 输入：requirements.md、stories.md、application-design/*、execution-plan.md

本阶段将系统拆解为可独立开发/部署的工作单元，并建立依赖与故事映射。请先回答文末问题中的所有 `[Answer]:`。

---

## A. 执行清单（Execution Checklist）

- [x] A1. 定义工作单元与职责 → `unit-of-work.md`（含代码组织策略，Greenfield）
- [x] A2. 建立单元依赖矩阵 → `unit-of-work-dependency.md`
- [x] A3. 故事到单元映射（确保所有 US 已分配）→ `unit-of-work-story-map.md`
- [x] A4. 校验单元边界与依赖（无环、可并行）
- [x] A5. 确认构建/开发顺序

---

## B. 强制产出物（Mandatory Artifacts）
- `aidlc-docs/inception/application-design/unit-of-work.md`
- `aidlc-docs/inception/application-design/unit-of-work-dependency.md`
- `aidlc-docs/inception/application-design/unit-of-work-story-map.md`

---

## C. 拆解决策问题（请填写所有 [Answer]:）

## 问题 1
工作单元的划分是否沿用 README/执行计划的 7 单元？

A) 沿用 7 单元：Unit1 前端、Unit2 认证、Unit3 商品、Unit4 积分、Unit5 兑换、Unit6 网关、Unit7 基础设施（推荐）
B) 增加独立的 common 共享库作为第 8 个单元
C) 其他划分（请在 [Answer]: 后描述）
X) 其他（请在 [Answer]: 后描述）

[Answer]: A

## 问题 2
代码组织方式（Greenfield 多单元）？

A) 单仓库 Monorepo：在工作区根目录下按单元分子目录（如 awsome-shop-auth-service/、awsome-shop-frontend/、awsome-shop-common/、deploy/）（MVP 简单、一键编排，推荐）
B) 多仓库 Polyrepo：每个单元独立 Git 仓库（贴合 README"关联代码仓库"，但本工作区内不便一键启动）
C) 其他（请在 [Answer]: 后描述）
X) 其他（请在 [Answer]: 后描述）

[Answer]: B

## 问题 3
后端各服务与 common 共享模块的构建管理方式（Java）？

A) Maven 多模块（parent POM 聚合各服务 + common），统一依赖管理（推荐）
B) 各服务独立 Maven 项目，common 以本地依赖/发布方式引用
C) Gradle 多项目
X) 其他（请在 [Answer]: 后描述）

[Answer]: B

## 问题 4
部署编排（docker-compose）作为哪个单元的产出？

A) 归入 Unit7 基础设施（在 deploy/ 或根目录提供 docker-compose.yml）（推荐）
B) 独立的 deploy 单元
X) 其他（请在 [Answer]: 后描述）

[Answer]: A
