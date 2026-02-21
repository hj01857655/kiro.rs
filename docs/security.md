# 安全文档

## 1. 认证机制

项目对两类接口做 API Key 认证：
- Anthropic 兼容接口（`/v1`、`/cc/v1`）
- Admin 接口（`/api/admin`）

认证中间件：
- Anthropic：`src/anthropic/middleware.rs::auth_middleware`
- Admin：`src/admin/middleware.rs::admin_auth_middleware`

认证实现细节：
- 从请求中提取 API Key：`common::auth::extract_api_key`
- 常量时间比较：`common::auth::constant_time_eq`
- 校验失败返回 `401`

支持头：
- `x-api-key: <key>`
- `Authorization: Bearer <key>`

---

## 2. 密钥与凭据安全

- `config.json` 中的 `apiKey` / `adminApiKey` 必须使用高强度随机值
- `credentials.json` 包含刷新凭据，必须：
  - 禁止提交到仓库
  - 使用最小权限文件权限
  - 不通过 IM/工单明文传播

建议：
- 生产环境按环境分离密钥（dev/staging/prod）
- 定期轮换 `adminApiKey`
- 凭据泄露后立即替换对应 `refreshToken`

---

## 3. Admin 面板暴露面

当 `adminApiKey` 非空时，会启用：
- `GET /admin`（管理 UI）
- `/api/admin/*`（管理接口）

风险点：
- 管理接口可增删凭据、调整路由策略
- 若公网暴露，风险显著提升

建议：
- 仅内网开放，或置于反向代理后
- 增加 IP 白名单/WAF/二次认证
- 审计 Admin API 调用日志

---

## 4. CORS 安全提示

当前 CORS 层（`src/anthropic/middleware.rs::cors_layer`）为宽松策略：
- `allow_origin(Any)`
- `allow_methods(Any)`
- `allow_headers(Any)`

这有利于兼容，但在公网部署时建议改为：
- 明确允许来源
- 最小化允许方法
- 限制允许头

---

## 5. 代理与 TLS

- `tlsBackend` 默认 `rustls`
- 如遇证书/代理兼容问题，可切到 `native-tls`

安全建议：
- 避免关闭证书校验
- 代理账号密码不要硬编码到镜像
- 使用安全存储注入代理凭据

---

## 6. 日志与隐私

建议日志实践：
- 不输出完整 API Key、refreshToken、proxyPassword
- 对敏感字段做脱敏（仅显示前后少量字符）
- 错误日志记录必要上下文，不记录明文凭据

---

## 7. 最小权限与隔离

- 容器仅暴露必要端口（默认 8990）
- 宿主机绑定可用 `127.0.0.1:8990:8990` 限制外部访问
- 配置目录单独挂载并限制读写权限
- 非必要不要启用 Admin 模块

---

## 8. 事件响应建议

发现密钥泄露或异常请求时：
1. 立刻轮换 `apiKey` / `adminApiKey`
2. 替换受影响凭据的 `refreshToken`
3. 临时下线 Admin 接口（清空 `adminApiKey`）
4. 回放访问日志确认影响范围
5. 完成后恢复并持续监控
