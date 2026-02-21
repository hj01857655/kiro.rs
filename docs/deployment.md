# 部署与运行文档

## 1. 本地构建运行

### 1.1 前置

- Rust 工具链
- Node.js（用于构建 admin-ui）
- pnpm

### 1.2 构建步骤

先构建前端管理台（会嵌入二进制）：

```bash
cd admin-ui && pnpm install && pnpm build
```

再构建 Rust：

```bash
cargo build --release
```

### 1.3 启动

默认路径启动：

```bash
./target/release/kiro-rs
```

指定配置路径：

```bash
./target/release/kiro-rs -c /path/to/config.json --credentials /path/to/credentials.json
```

---

## 2. Docker 部署

### 2.1 Dockerfile 行为

`Dockerfile` 使用三阶段：
1. `node:22-alpine` 构建 `admin-ui`
2. `rust:1.92-alpine` 构建 `kiro-rs`
3. `alpine:3.21` 运行镜像

运行命令：
`./kiro-rs -c /app/config/config.json --credentials /app/config/credentials.json`

暴露端口：`8990`

### 2.2 docker-compose

`docker-compose.yml` 默认：
- 端口映射：`127.0.0.1:8990:8990`
- 配置挂载：
  - `./config.json -> /app/config/config.json`
  - `./credentials.json -> /app/config/credentials.json`

启动：

```bash
docker-compose up
```

---

## 3. 启动后检查项

服务启动后应可访问：
- `GET /v1/models`
- `POST /v1/messages`
- `POST /v1/messages/count_tokens`

若 `adminApiKey` 非空，还会启用：
- `GET /api/admin/credentials`
- `GET /admin`

---

## 4. 验证请求示例

```bash
curl http://127.0.0.1:8990/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: sk-kiro-rs-your-key" \
  -d '{
    "model": "claude-sonnet-4-20250514",
    "max_tokens": 256,
    "stream": false,
    "messages": [{"role":"user","content":"hello"}]
  }'
```

---

## 5. 日志与排障

- 设置日志级别：

```bash
RUST_LOG=debug ./target/release/kiro-rs
```

常见问题：
- 401：检查 `x-api-key` 或 `Authorization: Bearer ...`
- Admin 接口 404：检查 `config.json` 是否设置了 `adminApiKey`
- 代理/TLS 问题：尝试 `tlsBackend: "native-tls"`
- 凭据刷新失败：检查 `refreshToken`、`authMethod` 与 region 配置

---

## 6. 生产建议

- `apiKey` 与 `adminApiKey` 使用强随机密钥
- 仅监听内网地址或放到反向代理后
- 对 `/admin` 增加网络访问控制
- 将 `config.json` 与 `credentials.json` 放在安全目录并限制权限
- 开启日志轮转，避免敏感信息长期保留
