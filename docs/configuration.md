# 配置文档（config / credentials）

## 1. 配置文件位置

- 默认配置文件：`config.json`
- 默认凭据文件：`credentials.json`
- 可通过命令行参数指定：
  - `-c, --config <path>`
  - `--credentials <path>`

启动参数定义在：`src/model/arg.rs`

---

## 2. config.json 字段说明

字段定义见：`src/model/config.rs` 的 `Config` 结构体。

关键字段：
- `host`：监听地址，默认 `127.0.0.1`
- `port`：监听端口，默认 `8080`
- `apiKey`：主 API 认证密钥（客户端访问 `/v1` 时使用）
- `region`：默认区域，默认 `us-east-1`
- `authRegion`：Token 刷新区域（可选）
- `apiRegion`：API 请求区域（可选）
- `kiroVersion`：Kiro 版本标识
- `machineId`：机器标识（可选）
- `systemVersion` / `nodeVersion`：客户端标识字段
- `tlsBackend`：`rustls` 或 `native-tls`，默认 `rustls`
- `countTokensApiUrl`：外部 count_tokens API 地址（可选）
- `countTokensApiKey`：外部 count_tokens API 密钥（可选）
- `countTokensAuthType`：`x-api-key` 或 `bearer`，默认 `x-api-key`
- `proxyUrl` / `proxyUsername` / `proxyPassword`：全局代理配置（可选）
- `adminApiKey`：Admin API 密钥（配置后启用管理 API/UI）
- `loadBalancingMode`：`priority` 或 `balanced`，默认 `priority`

示例（最小）：

```json
{
  "host": "127.0.0.1",
  "port": 8990,
  "apiKey": "sk-kiro-rs-your-key"
}
```

示例（启用 Admin）：

```json
{
  "host": "127.0.0.1",
  "port": 8990,
  "apiKey": "sk-kiro-rs-your-key",
  "adminApiKey": "sk-admin-your-secret-key",
  "tlsBackend": "rustls",
  "region": "us-east-1",
  "loadBalancingMode": "priority"
}
```

---

## 3. credentials.json 格式

定义见：`src/kiro/model/credentials.rs`。

支持两种格式：
- 单对象（兼容旧格式）
- 数组（多凭据）

常用字段：
- `id`：可选，管理用 ID
- `accessToken`：可选，访问令牌
- `refreshToken`：刷新令牌（核心）
- `profileArn`：可选
- `expiresAt`：可选，RFC3339
- `authMethod`：`social` / `idc`（`builder-id`、`iam` 会规范化为 `idc`）
- `clientId` / `clientSecret`：IDC 场景可选
- `priority`：数字越小优先级越高
- `region`：兼容字段
- `authRegion`：刷新区域
- `apiRegion`：请求区域
- `machineId`：凭据级机器 ID
- `email` / `subscriptionTitle`：可选元数据
- `proxyUrl` / `proxyUsername` / `proxyPassword`：凭据级代理
- `disabled`：是否禁用

单凭据示例：

```json
{
  "refreshToken": "your-refresh-token",
  "authMethod": "social"
}
```

多凭据示例：

```json
[
  {
    "refreshToken": "token-a",
    "authMethod": "social",
    "priority": 0
  },
  {
    "refreshToken": "token-b",
    "authMethod": "idc",
    "priority": 1,
    "disabled": false
  }
]
```

---

## 4. 区域优先级规则

### Auth Region（刷新）
优先级：
`凭据.authRegion > 凭据.region > config.authRegion > config.region`

### API Region（请求）
优先级：
`凭据.apiRegion > config.apiRegion > config.region`

注意：`凭据.region` 不参与 API Region 回退链。

---

## 5. 代理优先级规则

凭据级优先于全局：
`凭据.proxyUrl > config.proxyUrl > 无代理`

特殊值：
- `proxyUrl: "direct"` 表示该凭据强制直连（忽略全局代理）

---

## 6. 负载均衡模式

- `priority`：按优先级优先使用高优先级凭据
- `balanced`：均衡分配调用

该配置可通过 Admin API 动态修改：
- `GET /api/admin/config/load-balancing`
- `PUT /api/admin/config/load-balancing`

---

## 7. 常见配置问题

- `adminApiKey` 留空：不会启用 `/api/admin` 与 `/admin`
- `apiKey` 未设置：`/v1` 请求认证无法通过
- 代理可用但请求失败：尝试将 `tlsBackend` 改为 `native-tls`
- 多凭据顺序不生效：请确认 `priority` 字段（数值越小越优先）
