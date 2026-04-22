# Sub2API 仓库学习文档（中文）

> 面向第一次接触本项目的开发者：帮助你在 1~2 天内建立完整心智模型，能快速定位代码、跑通本地、理解核心请求链路。

## 1. 项目是什么

Sub2API 是一个 **AI API 网关平台**：

- 上游：管理多个 AI 账号（OAuth / API Key / 平台差异）
- 下游：给用户发平台 API Key
- 中间：做鉴权、路由调度、计费、限流、并发控制、观测、支付与运维

核心价值：把“多平台、多账号、复杂策略”的细节封装起来，对下游提供统一 API 体验。

---

## 2. 技术栈与目录结构

## 2.1 技术栈

- **后端**：Go + Gin + Ent + PostgreSQL + Redis
- **前端**：Vue3 + Vite + Pinia + Vue Router + Vitest
- **部署**：二进制安装、Docker Compose

## 2.2 关键目录

```text
backend/
  cmd/server/             # 后端启动入口、Wire 注入
  internal/
    server/               # 路由与中间件组装
    handler/              # HTTP Handler
    service/              # 业务逻辑层
    repository/           # 数据访问层
    config/               # 配置结构、加载与默认值
  ent/schema/             # Ent 数据模型定义
  migrations/             # SQL 迁移

frontend/
  src/main.ts             # 前端启动入口
  src/router/             # 前端路由
  src/views/              # 页面（用户端/管理端）
  src/stores/             # Pinia 状态管理

docs/                     # 项目文档（本文件位置）
```

---

## 3. 后端启动与生命周期

后端入口位于 `backend/cmd/server/main.go`，主流程：

1. 解析命令参数（`-setup` / `-version`）
2. 首次安装判断：是否进入 Setup Wizard
3. 正常模式加载配置并初始化应用（Wire）
4. 启动 HTTP 服务并等待优雅退出

其中 Setup 模式支持自动初始化（适合容器环境）。

---

## 4. 路由与协议兼容设计

路由组装发生在 `internal/server/router.go` + `internal/server/routes/*`。

可以把路由分成四层：

- **公共路由**：健康检查等
- **认证路由**：注册登录、OAuth 回调、刷新 token
- **用户路由**：个人资料、API Key、使用记录、订阅
- **管理路由**：账号/分组/运营/告警/渠道/系统设置

### 4.1 网关路由（最关键）

`routes/gateway.go` 做了“多平台协议兼容 + 自动分发”：

- `/v1/messages`：按分组平台分发到 Anthropic 或 OpenAI 兼容实现
- `/v1/responses`、`/chat/completions`、`/images/*`：同样平台感知
- `/v1beta/*`：Gemini 原生兼容层
- `/antigravity/*`：专用路由，强制平台上下文

这是本仓库“统一入口，内部异构”的核心设计。

---

## 5. 一条核心请求如何流动（/v1/messages）

在 `internal/handler/gateway_handler.go` 中可看到典型处理链路：

1. 从 context 提取 API Key 与用户身份
2. 读取并解析请求体，提取模型与流式参数
3. 进行渠道映射、客户端识别与版本检查
4. 执行等待队列与并发槽位控制
5. 二次计费资格检查（避免等待后状态变化导致脏放行）
6. 生成会话哈希并处理粘性会话
7. 进入具体平台调用、计费落库与返回

这条链路体现了网关系统的三个目标：

- 稳定性（并发控制与错误处理）
- 成本可控（计费资格、倍率、订阅）
- 兼容性（多协议/多平台）

---

## 6. 数据模型：先理解三个实体

建议先看 Ent Schema：

- `User`：用户余额、并发、2FA、通知等
- `Account`：上游账号凭据、状态、调度相关字段
- `Group`：平台归属、倍率、策略、模型映射

这三者的关系可近似理解为：

- 用户属于（可访问）某些分组
- 分组绑定若干账号
- 请求到来后，在分组内基于策略调度到具体账号

---

## 7. 配置系统（建议重点）

配置定义在 `internal/config/config.go`，包含：

- server / cors / security / jwt
- database / redis
- gateway / concurrency / rate_limit
- pricing / billing / dashboard / ops
- token_refresh / idempotency 等

实践经验：排查线上问题时，先看配置生效值是否符合预期。

---

## 8. 运维与可观测能力

管理路由里包含完整 Ops 模块：

- 并发与实时流量
- 告警规则与告警事件
- 请求错误、上游错误、系统日志
- 仪表盘概览、吞吐趋势、延迟分布、错误分布

这意味着 Sub2API 除“网关转发”外，还内建了较完善的运营观测能力。

---

## 9. 数据库迁移约定

`backend/migrations/README.md` 重点规则：

- 迁移文件不可修改（checksum 校验）
- `_notx.sql` 仅用于并发索引（不包事务）
- 启动会自动执行迁移

建议：

- 永远新增迁移，不修改已发布迁移
- 迁移保持小而单一，便于审查与回滚

---

## 10. 前端学习路径

前端入口在 `frontend/src/main.ts`，关键点：

- 先应用主题与注入配置，再挂载应用
- 初始化 i18n
- 等待 router ready，避免首次渲染竞态

路由在 `frontend/src/router/index.ts`，可明显看到：

- setup/public/auth 回调
- user 区域
- admin 区域

这与后端 API 分层一一对应。

---

## 11. 新人上手建议（两天计划）

### Day 1：建立全局模型

1. 阅读 `README.md`（定位、部署）
2. 阅读 `cmd/server/main.go`（启动流程）
3. 阅读 `server/router.go` 和 `routes/*.go`（接口地图）
4. 跟一次 `/v1/messages` 的 handler 代码

### Day 2：深入业务策略

1. 阅读 `ent/schema` 三大实体（User/Account/Group）
2. 阅读 `service` 中 gateway / billing / subscription / ops
3. 结合管理端页面，理解配置项与后端策略映射
4. 跑一组测试并尝试改一个小策略（如限流阈值）

---

## 12. 常见排查思路（实用）

- **请求被拒绝**：先看 API Key 鉴权，再看分组分配，再看并发/限流
- **有余额却报不可用**：看订阅状态、计费资格检查、分组策略
- **某模型频繁失败**：看模型路由配置、账号可调度状态、上游错误分类
- **线上偶发超时**：看 ops 仪表盘延迟分位与上游错误趋势

---

## 13. 结语

Sub2API 的难点不在“单个函数复杂”，而在“多模块策略耦合”。

建议你用“请求生命周期”串起代码：

> 路由入口 → 鉴权/上下文 → 调度策略 → 上游调用 → 计费落库 → 运维观测

只要这条链路理解清楚，后续扩展任何功能（新平台、新计费、新风控）都会变得可控。
