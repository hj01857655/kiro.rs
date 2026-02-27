# 批量导入逻辑说明

## 1. 功能入口

前端入口文件：`admin-ui/src/components/batch-import-dialog.tsx`

核心函数：
- `handleBatchImport()`：主流程（解析、去重、导入、验活、回滚、汇总）
- `rollbackCredential(id)`：回滚流程（先禁用，再删除）

---

## 2. 输入格式与预处理

- 输入框接受 JSON 字符串，支持：
  - 单对象
  - 数组对象
- 内部统一转为数组处理：`Array.isArray(parsed) ? parsed : [parsed]`
- 每个对象支持字段：
  - `refreshToken`（必需）
  - `clientId` / `clientSecret`（IDC 可选）
  - `region` / `authRegion` / `apiRegion`
  - `priority`
  - `machineId`

空数组会直接提示：`没有可导入的凭据`。

---

## 3. 去重逻辑

在导入前，前端先通过 `useCredentials()` 拿到现有凭据列表，提取 `refreshTokenHash` 集合。

对每条待导入凭据：
1. 取 `refreshToken.trim()`
2. 前端计算 SHA-256（`sha256Hex()`）
3. 若哈希已存在，则标记 `duplicate`，不再调用后端新增接口

这样可以减少明显重复凭据的后端请求。

> 注意：后端仍会做重复校验（前端去重不是最终防线）。

---

## 4. 单条导入与验活流程

每条记录按顺序串行处理（for 循环 + await）：

1. 状态切到 `checking`
2. 计算 `authMethod`：
   - 同时有 `clientId` 和 `clientSecret` => `idc`
   - 否则 => `social`
3. 参数归一化：
   - `authRegion = authRegion ?? region`
   - `apiRegion` 原样（可空）
   - `priority` 默认 `0`
4. 调用新增接口：`POST /api/admin/credentials`
5. 成功后等待 1 秒
6. 调用余额接口验活：`GET /api/admin/credentials/{id}/balance`
7. 验活成功标记 `verified`，记录用量信息和邮箱

对应前端 API 封装在：`admin-ui/src/api/credentials.ts`

---

## 5. 失败与回滚策略

如果单条导入或验活任一步失败：

- 若已创建出 `credentialId`，执行回滚：
  1. `POST /api/admin/credentials/{id}/disabled`（先禁用）
  2. `DELETE /api/admin/credentials/{id}`（再删除）
- 若尚未创建成功，则回滚标记为 `skipped`

回滚状态记录为：
- `success`
- `failed`
- `skipped`

失败条目会保留原错误信息 + 回滚错误信息（若有）。

---

## 6. 后端处理链路（新增凭据）

新增请求路径：

`POST /api/admin/credentials`
-> `src/admin/handlers.rs::add_credential`
-> `src/admin/service.rs::add_credential`
-> `token_manager.add_credential(new_cred)`

后端服务层行为：
- 组装 `KiroCredentials`
- 调用 `MultiTokenManager::add_credential`
- 添加后会尝试 `get_usage_limits_for(credential_id)` 预热订阅信息（失败不影响添加成功）

错误分类（`classify_add_error`）会把以下识别为凭据问题：
- `refreshToken` 缺失/为空/被截断
- 凭据已存在、refreshToken 重复
- 凭证过期或无效
- 权限不足

这也是前端会出现“添加失败但已回滚”提示的根本来源。

---

## 7. 进度与结果汇总

前端维护：
- `progress.current / progress.total`
- `results[]`（每条状态）
- `currentProcessing`（当前处理提示）

最终 toast 汇总：
- 成功数 `successCount`
- 重复数 `duplicateCount`
- 失败数 `failCount`
- 失败中的回滚明细：`rollbackSuccessCount / rollbackFailedCount / rollbackSkippedCount`

若存在回滚失败，会额外提示手动禁用并删除。

---

## 8. 关键特性与限制

### 特性
- 支持单条/批量 JSON 一次导入
- 自动去重（哈希）
- 自动验活（余额查询）
- 失败自动回滚（禁用 + 删除）

### 限制
- 当前是串行处理，批量很大时耗时较长
- 依赖余额接口验活，若上游短时不稳定会触发失败回滚
- 前端去重依赖本地拉取到的现有列表，最终一致性仍以后端为准

---

## 9. 调试建议

- 前端看浏览器控制台（已加 `[admin-api]` 请求/响应/错误日志）
- 若 Vite 出现 `http proxy error ECONNREFUSED`，先检查后端是否在 `8990` 端口运行
- 后端看 `cargo run` 输出，重点关注：
  - `refreshToken 已被截断`
  - `凭据已存在` / `refreshToken 重复`
  - 区域/认证方式错误
