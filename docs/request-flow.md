# 函数级请求流文档

## 1. 总览

本文档给出从前端到后端的函数级调用链，便于定位问题、排查接口、扩展功能。

- 前端 API 基础路径：`/api/admin`（`admin-ui/src/api/credentials.ts`）
- 后端管理路由前缀：`/api/admin`（在 `src/main.rs` 中 `.nest("/api/admin", admin_app)`）
- 对话 API 前缀：`/v1` 与 `/cc/v1`（在 `src/anthropic/router.rs` 中 `nest`）

---

## 2. 管理端（Admin）函数调用链

### 2.1 获取凭据列表

- 前端：`getCredentials()`
  - `GET /credentials`
- 路由：`create_admin_router()`
  - `GET /credentials -> get_all_credentials`
- Handler：`get_all_credentials(State(state))`
  - 调用：`state.service.get_all_credentials()`
- Service：`AdminService::get_all_credentials()`
  - 调用：`token_manager.snapshot()`

调用链：
`getCredentials -> GET /api/admin/credentials -> get_all_credentials -> AdminService::get_all_credentials -> token_manager.snapshot`

### 2.2 设置禁用状态

- 前端：`setCredentialDisabled(id, disabled)`
  - `POST /credentials/{id}/disabled`
- Handler：`set_credential_disabled(...)`
  - 调用：`state.service.set_disabled(id, payload.disabled)`
- Service：`AdminService::set_disabled(id, disabled)`
  - 调用：`token_manager.set_disabled(id, disabled)`
  - 若禁用当前凭据：`token_manager.switch_to_next()`

调用链：
`setCredentialDisabled -> POST /api/admin/credentials/{id}/disabled -> set_credential_disabled -> AdminService::set_disabled -> token_manager.set_disabled (-> switch_to_next)`

### 2.3 设置优先级

- 前端：`setCredentialPriority(id, priority)`
- Handler：`set_credential_priority(...)`
  - 调用：`state.service.set_priority(id, payload.priority)`
- Service：`AdminService::set_priority(id, priority)`
  - 调用：`token_manager.set_priority(id, priority)`

### 2.4 重置失败计数

- 前端：`resetCredentialFailure(id)`
- Handler：`reset_failure_count(...)`
  - 调用：`state.service.reset_and_enable(id)`
- Service：`AdminService::reset_and_enable(id)`
  - 调用：`token_manager.reset_and_enable(id)`

### 2.5 查询余额

- 前端：`getCredentialBalance(id)`
- Handler：`get_credential_balance(...)`
  - 调用：`state.service.get_balance(id).await`
- Service：`AdminService::get_balance(id).await`
  - 先查本地缓存（TTL）
  - 未命中时调用内部 `fetch_balance(id).await`

### 2.6 添加凭据

- 前端：`addCredential(req)`
- Handler：`add_credential(...)`
  - 调用：`state.service.add_credential(payload).await`
- Service：`AdminService::add_credential(req).await`
  - 组装 `KiroCredentials`
  - 调用：`token_manager.add_credential(new_cred).await`
  - 调用：`token_manager.get_usage_limits_for(credential_id).await`（预热）

### 2.7 删除凭据

- 前端：`deleteCredential(id)`
- Handler：`delete_credential(...)`
  - 调用：`state.service.delete_credential(id)`
- Service：`AdminService::delete_credential(id)`
  - 调用：`token_manager.delete_credential(id)`
  - 清理余额缓存

### 2.8 负载均衡模式

- 前端：
  - `getLoadBalancingMode()` -> `GET /config/load-balancing`
  - `setLoadBalancingMode(mode)` -> `PUT /config/load-balancing`
- Handler：
  - `get_load_balancing_mode(...)` -> `state.service.get_load_balancing_mode()`
  - `set_load_balancing_mode(...)` -> `state.service.set_load_balancing_mode(payload)`
- Service：
  - `get_load_balancing_mode()` -> `token_manager.get_load_balancing_mode()`
  - `set_load_balancing_mode(req)` -> `token_manager.set_load_balancing_mode(req.mode)`

---

## 3. Anthropic 兼容 API 调用链

### 3.1 路由层

`create_router_with_provider()` 中：
- `/v1/models` -> `get_models`
- `/v1/messages` -> `post_messages`
- `/v1/messages/count_tokens` -> `count_tokens`
- `/cc/v1/messages` -> `post_messages_cc`
- `/cc/v1/messages/count_tokens` -> `count_tokens`

`/v1` 与 `/cc/v1` 都挂载 `auth_middleware`。

### 3.2 非流式消息

`post_messages` 关键链路：
1. 解析请求 `MessagesRequest`
2. `convert_request(&payload)` 完成协议转换
3. `provider.call_api(request_body).await`
4. 读取上游响应并回包

调用链：
`POST /v1/messages (stream=false) -> post_messages -> convert_request -> provider.call_api -> handle_non_stream_request`

### 3.3 流式消息

`post_messages`（`stream=true`）关键链路：
1. `convert_request(&payload)`
2. `provider.call_api_stream(request_body).await`
3. `handle_stream_request(...)`
4. `StreamContext` 将上游事件映射为 SSE

调用链：
`POST /v1/messages (stream=true) -> post_messages -> convert_request -> provider.call_api_stream -> handle_stream_request -> SSE`

### 3.4 Claude Code 兼容流式

`post_messages_cc`（`stream=true`）走缓冲逻辑：
- `handle_stream_request_buffered(...)`
- `BufferedStreamContext` 先缓冲再输出，确保 `contextUsageEvent` 后发送准确 `message_start`

---

## 4. 认证链路

### 4.1 Admin 认证

- 中间件：`admin_auth_middleware`
- API Key 提取：`common::auth::extract_api_key(&request)`
- 常量时间比较：`common::auth::constant_time_eq(...)`
- 失败返回：`401 UNAUTHORIZED`

### 4.2 Anthropic 认证

- 中间件：`auth_middleware`
- 同样使用 `extract_api_key` + `constant_time_eq`
- 失败返回 Anthropic 风格错误体

---

## 5. 入口装配（main）

`src/main.rs` 关键流程：
1. `Config::load(...)`
2. `CredentialsConfig::load(...)`
3. 构建 provider / token manager
4. 构建 Anthropic 路由：`create_router_with_provider(...)`
5. 若 `adminApiKey` 非空：
   - 挂载 `.nest("/api/admin", admin_app)`
   - 挂载 `.nest("/admin", admin_ui_app)`
6. `TcpListener::bind("{host}:{port}")`
7. `axum::serve(listener, app)`
