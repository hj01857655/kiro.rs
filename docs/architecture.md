# 项目架构与原理图

## 1. 总体架构原理图

```text
[Browser / Admin UI (React + TS + Tailwind)]
        |
        | HTTPS REST (JSON)
        v
+--------------------------------------------------+
|                Rust HTTP Server                  |
|                 (src/main.rs)                    |
+-------------------------+------------------------+
                          |
             +------------+-------------+
             |                          |
             v                          v
  +---------------------+     +------------------------+
  |   Admin Domain      |     |   Anthropic Domain     |
  | (src/admin/*)       |     | (src/anthropic/*)      |
  +---------------------+     +------------------------+
  | router.rs           |     | router.rs              |
  | middleware.rs       |     | middleware.rs          |
  | handlers.rs         |     | handlers.rs            |
  | service.rs          |     | stream.rs (流式输出)    |
  | types.rs/error.rs   |     | websearch.rs           |
  +----------+----------+     +-----------+------------+
             |                            |
             | 业务调用                   | 模型与流式调用
             v                            v
      +-------------------------------------------+
      |           Kiro Core (src/kiro/*)          |
      +-------------------------------------------+
      | provider.rs       token_manager.rs        |
      | parser/*          machine_id.rs           |
      | model/requests/*  model/events/*          |
      | model/credentials.rs usage_limits.rs      |
      +-------------------+-----------------------+
                          |
                          | outbound HTTP / auth
                          v
                 +----------------------+
                 | Upstream AI Service  |
                 | + optional websearch |
                 +----------------------+

数据与配置侧:
  - config.example.json: 运行配置模板
  - credentials.example.*.json: 凭据结构示例
  - 管理域服务负责凭据读写与校验
```

## 2. 管理端请求时序原理图

```text
User Click
  -> admin-ui component (dialog/card/dashboard)
  -> admin-ui/src/api/credentials.ts (HTTP请求)
  -> Rust main router (src/main.rs)
  -> src/admin/router.rs
  -> src/admin/middleware.rs (鉴权/上下文)
  -> src/admin/handlers.rs (参数解析/响应封装)
  -> src/admin/service.rs (核心业务)
  -> src/kiro/model/credentials.rs (模型与持久化相关)
  -> handlers 返回 JSON
  -> hooks(use-credentials) 更新状态
  -> UI rerender
```

## 3. Anthropic/流式链路原理图

```text
Client Request
  -> src/anthropic/router.rs
  -> src/anthropic/middleware.rs
  -> src/anthropic/handlers.rs
  -> src/anthropic/stream.rs (分块/实时推送)
  -> src/kiro/provider.rs + src/kiro/token_manager.rs + src/http_client.rs
  -> Upstream chunks
  -> src/kiro/model/events/* 事件映射
  -> 实时回传到客户端
```

## 4. 模块依赖约束图（建议与现状对齐）

```text
依赖方向（高层 -> 低层）

admin-ui (React)
  -> Rust HTTP API

src/main.rs (组装层)
  -> src/admin
  -> src/anthropic
  -> src/admin_ui
  -> src/common
  -> src/kiro
  -> src/model

src/admin
  -> src/common
  -> src/kiro/model (必要时)
  -> src/model (通用配置/参数)
  -X-> src/anthropic   (避免横向业务耦合)

src/anthropic
  -> src/common
  -> src/kiro (provider/token/model/events)
  -> src/http_client
  -X-> src/admin       (避免横向业务耦合)

src/kiro (核心域)
  -> src/http_client
  -> src/model (共享类型)
  -X-> src/admin
  -X-> src/anthropic/router层

src/common
  -> (尽量无业务反向依赖)

规则说明:
  - 入口层(main/router)负责装配，不承载复杂业务
  - handler只做边界职责：鉴权后参数解析、调用service、响应映射
  - service负责业务编排，不关心HTTP细节
  - core(kiro)不依赖具体业务域(admin/anthropic)的路由与handler
```

## 5. 快速阅读顺序（建议）

1. src/main.rs（总入口与路由装配）
2. src/admin/router.rs + src/admin/handlers.rs + src/admin/service.rs（管理链路）
3. src/anthropic/router.rs + src/anthropic/handlers.rs + src/anthropic/stream.rs（流式链路）
4. src/kiro/provider.rs + src/kiro/token_manager.rs（核心能力）
5. admin-ui/src/api/credentials.ts + admin-ui/src/hooks/use-credentials.ts（前端到后端调用面）
