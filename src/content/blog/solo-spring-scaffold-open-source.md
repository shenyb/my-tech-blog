---
title: 'Spring Boot 项目还在复制粘贴基建？我开源了一个 30 秒生成生产级骨架的 CLI'
description: '厌倦 Initializr 空壳粘贴？solo-spring-scaffold 一键生成 Spring Boot 3 五模块骨架，H2 即跑，JWT/Docker/测试插件按需装，代码全归你。'
pubDate: 2026-07-05
heroImage: ''
category: engineering
tags: ['Spring Boot', '脚手架', '工程化', 'MyBatis-Plus', '开源']
---

> **TL;DR** 每次新开 Spring Boot 项目，都要重复粘贴统一返回体、MyBatis-Plus、Flyway、JWT、Dockerfile……我开源了 [solo-spring-scaffold](https://github.com/shenyb/solo-spring-scaffold)：Python CLI 一键生成 Spring Boot 3.x 多模块、可运行、带完整工程化基建的项目骨架。Initializr 太轻，RuoYi 太重，我们卡在中间甜点区。

---

## 开篇：空壳项目的隐性成本

每次新开一个 Spring Boot 项目，流程都差不多：

1. 打开 Spring Initializr，勾选 Web + MySQL + Lombok
2. 生成一个单模块空项目
3. 开始复制粘贴：统一返回体、全局异常、MyBatis-Plus 配置、Flyway、Logback、Swagger……
4. 再复制一遍 JWT、Dockerfile、测试模板……

**真正写业务之前，往往已经花掉半天到一天。**

更麻烦的是：每个人复制的「最佳实践」版本不一样。A 同事用 JPA，B 同事用 MyBatis-Plus；A 的分页写法跟 B 不兼容；异常码各搞一套。项目一多，维护成本指数上升。

所以我做了 **[solo-spring-scaffold](https://github.com/shenyb/solo-spring-scaffold)**：一个 Python CLI 工具，**一键生成 Spring Boot 3.x 多模块、可运行、带完整工程化基建的项目骨架**。

它不是框架，不会侵入你的业务代码——**生成完就是你的项目，想改什么改什么。**

---

## 和 Initializr / RuoYi 有什么不同？

| 维度 | Spring Initializr | RuoYi / JeecgBoot | solo-spring-scaffold |
|------|---------------------|-------------------|----------------------|
| 定位 | 依赖选择器 | 完整后台管理系统 | **项目生成器** |
| 模块结构 | 通常单模块 | 固定架构 | **5 模块分层**（common/dao/service/web） |
| 开箱可跑 | 需大量补代码 | 功能很重 | **H2 零配置启动** |
| 可定制 | 低 | 低（fork 改造） | **插件按需拼装** |
| 代码归属 | 你的 | 框架的 | **100% 你的** |

一句话：**Initializr 太轻，RuoYi 太重，我们卡在中间甜点区**——30 秒生成生产级骨架，而不是 30 秒生成一个 `Hello World`。

---

## 30 秒上手

### 安装（当前：克隆即用）

```bash
git clone https://github.com/shenyb/solo-spring-scaffold.git
cd solo-spring-scaffold/python
python solo-spring-scaffold init
```

不带参数会进入**交互式向导**，依次问项目名、包名、JDK、插件、数据库类型——**不用记任何参数**。

### 推荐组合（生产起步）

```bash
python solo-spring-scaffold init my-app \
  --package com.example.myapp \
  --jdk 21 \
  --plugins security,docker,test
```

### 编译启动

```bash
cd my-app
mvn package -DskipTests -q
java -jar my-app-web/target/my-app-web-1.0.0.jar
```

浏览器打开：**http://localhost:8080/swagger-ui.html**

默认账号（启用 security 插件后）：`admin` / `123456`

---

## 生成的项目里有什么？

不是空壳，是**可以直接当项目起点**的骨架：

```
my-app/
├── my-app-common/     # Result、BizException、GlobalExceptionHandler、AOP 日志、MDC traceId
├── my-app-dao/        # MyBatis-Plus + Flyway + User/Order/Product 示例
├── my-app-service/    # Service 层 + DTO 分层
├── my-app-web/        # Controller + Swagger + 多环境配置
├── bin/               # start/stop/deploy 脚本
└── pom.xml            # 父 POM，JDK 17/21/25 对应 Spring Boot 3.2/3.3/3.4
```

### 技术选型（每个都有理由）

| 维度 | 选择 | 为什么 |
|------|------|--------|
| ORM | MyBatis-Plus 3.5.9 | 复杂 SQL 可控，分页/自动填充开箱即用 |
| 迁移 | Flyway | 版本化、可审计、团队协作友好 |
| API 文档 | SpringDoc OpenAPI | 原生 OpenAPI 3，UI 现代 |
| 开发库 | H2 内存库 | PR 评审、本地 demo 零依赖 |
| 生产库 | MySQL（可选） | 配 `--db-url` 即可接 MyBatis-Plus 代码生成器 |

---

## 核心差异化：插件系统

这是我最花时间设计的部分。每个插件 = 一组模板 + 补丁，**按需启用，不污染基础骨架**。

```bash
python solo-spring-scaffold list-plugins
```

| 插件 | 能力 |
|------|------|
| `security` | Spring Security + JWT（登录/注册/刷新令牌） |
| `docker` | Dockerfile 分层构建 + docker-compose |
| `test` | JUnit 5 + Mockito + MockMvc 测试模板 |
| `redis` | Lettuce + JSON 序列化 + Spring Cache |
| `actuator` | Actuator + Prometheus + 健康检查 |

插件架构示意：

```
基础骨架（templates/）
    ↓ 复制
生成项目目录
    ↓ 按 --plugins 合并
plugins/security/templates/  → 新增 SecurityConfig、AuthController…
plugins/security/patches/    → 追加 YAML、注入 POM 依赖、Flyway 脚本
```

**写新插件不需要改 CLI 核心代码**——加一个 `plugins/xxx/` 目录，声明 `plugin.yaml` 即可。

---

## 我踩过的坑，脚手架帮你避开了

1. **Git Bash vs PowerShell 的 curl 差异**  
   README 里给了两套登录命令，Swagger 认证只粘贴 token、不要加 `Bearer` 前缀。

2. **多模块 Maven 依赖方向**  
   web → service → dao → common，单向依赖，避免循环引用。

3. **Flyway 与 H2/MySQL 双环境**  
   `application.yml` / `application-dev.yml` / `application-prod.yml` 分环境，开发用 H2、上线切 MySQL 只改 profile。

4. **CI 已经帮你验过了**  
   GitHub Actions 5 个 Job：CLI 语法检查、裸骨架编译、全插件编译、JDK 17 兼容、Security 组合编译。  
   ![CI](https://github.com/shenyb/solo-spring-scaffold/actions/workflows/ci.yml/badge.svg)

---

## 适合谁用？

- **独立开发者 / 小团队**：快速起项目，架构决策已经做好
- **接外包**：统一交付标准，减少「每个项目架构不一样」的维护噩梦
- **学习 Spring Boot 工程化**：生成一份完整项目，对照看分层、异常、迁移、测试怎么组织
- **不想被重型框架绑定**：代码生成后完全属于你，没有「升级框架版本」的包袱

不太适合：想要「下载就能用的完整 ERP/商城后台」——那是 RuoYi 的路子，不是脚手架的路子。

---

## 路线图（欢迎共建）

已完成（v0.1 已发布）：

- ✅ 5 插件体系
- ✅ 多 Domain 示例（User / Order / Product）
- ✅ 交互式 CLI 向导
- ✅ GitHub Actions CI
- ✅ [v0.1.0 Release](https://github.com/shenyb/solo-spring-scaffold/releases/tag/v0.1.0)

计划中：

- 文件上传/下载示例
- `pip install solo-spring-scaffold` 一键安装
- Agent 规范包（生成项目时附带 AGENTS.md，让 AI 助手按同一套规范写代码）

**Star 和 Issue 是我继续投入的信号。** 有需求直接提 Issue，插件想法也欢迎 PR。

---

## 结语

工程化的价值，不在于代码写得有多炫，而在于**每次开工少踩同样的坑**。

solo-spring-scaffold 把我这几年做 Spring Boot 项目反复复制的基建，打包成可复用的模板和插件。希望它能帮你把「搭架子」的时间，压缩到 30 秒，把精力留给真正有价值的业务逻辑。

👉 **GitHub 地址**：https://github.com/shenyb/solo-spring-scaffold  
👉 **给个 Star**，是对开源作者最好的鼓励
