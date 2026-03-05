# Pixiv-Shaft Token 刷新与保活机制深度分析

> 面向「二次开发新 CLI/服务端」的实现文档。  
> 目标：不读源码也能理解原项目如何处理 token 失效、自动刷新、并发防抖与会话保活。

---

## 1. 总览：原项目如何做 token 保活

当前代码主链路（新实现）：

1. 请求统一经过 `HeaderInterceptor` 注入认证头与风控头。
2. 请求返回后由 `TokenFetcherInterceptor` 判断是否是 token 失效错误。
3. 命中失效条件时调用 `SessionManager.refreshAccessToken(tokenForThisRequest)`。
4. `SessionManager` 内通过 **Mutex + 双检查 + Deferred 复用** 防止并发重复刷新。
5. 刷新成功后更新会话并重试原请求；失败则返回原响应或进入重新登录流程。

对应文件：

- `/home/runner/work/Pixiv-Shaft/Pixiv-Shaft/app/src/main/java/ceui/loxia/HeaderInterceptor.kt`
- `/home/runner/work/Pixiv-Shaft/Pixiv-Shaft/app/src/main/java/ceui/loxia/TokenFetcherInterceptor.kt`
- `/home/runner/work/Pixiv-Shaft/Pixiv-Shaft/app/src/main/java/ceui/pixiv/session/SessionManager.kt`
- `/home/runner/work/Pixiv-Shaft/Pixiv-Shaft/app/src/main/java/ceui/loxia/Client.kt`

---

## 2. 触发刷新的条件（何时判定 token 失效）

在 `TokenFetcherInterceptor.intercept()` 中：

- 先执行原请求，拿到 `response`；
- 当 `response.code == 400` 时，读取 body；
- 若 body 包含以下任一文本，进入刷新流程：
  - `Error occurred at the OAuth process`
  - `Invalid refresh token`

错误常量来自：

- `ClientManager.TOKEN_ERROR_1`
- `ClientManager.TOKEN_ERROR_2`

定义位置：

- `/home/runner/work/Pixiv-Shaft/Pixiv-Shaft/app/src/main/java/ceui/loxia/Client.kt`

---

## 3. refresh token 请求到底携带了哪些变量

刷新接口定义在：

- `/home/runner/work/Pixiv-Shaft/Pixiv-Shaft/app/src/main/java/ceui/lisa/http/AccountTokenApi.java`

方法：`newRefreshToken2(...)`  
HTTP：`POST /auth/token`（OAuth Host：`https://oauth.secure.pixiv.net`）

### 3.1 请求字段（Form URL Encoded）

| 字段名 | 值来源 | 说明 |
|---|---|---|
| `client_id` | `FragmentLogin.CLIENT_ID` | 客户端标识 |
| `client_secret` | `FragmentLogin.CLIENT_SECRET` | 客户端密钥 |
| `grant_type` | `FragmentLogin.REFRESH_TOKEN`（值为 `refresh_token`） | OAuth 刷新模式 |
| `refresh_token` | `_loggedInAccount.value?.refresh_token` | 登录后保存的 refresh token |
| `include_policy` | `true` | 接口扩展参数 |

实际调用位置：

- `/home/runner/work/Pixiv-Shaft/Pixiv-Shaft/app/src/main/java/ceui/pixiv/session/SessionManager.kt`
- 函数：`refreshAccessTokenInternal(refreshToken: String)`

### 3.2 关键常量值（保活时必须保持一致）

定义位置：

- `/home/runner/work/Pixiv-Shaft/Pixiv-Shaft/app/src/main/java/ceui/lisa/fragments/FragmentLogin.kt`

当前使用值：

- `CLIENT_ID = "<见 FragmentLogin.CLIENT_ID>"`
- `CLIENT_SECRET = "<见 FragmentLogin.CLIENT_SECRET>"`
- `REFRESH_TOKEN = "refresh_token"`
- `AUTH_CODE = "authorization_code"`
- `CALL_BACK = "https://app-api.pixiv.net/web/v1/users/auth/pixiv/callback"`

> 安全提示：以上常量是“兼容原协议”的对照信息。你在新项目中不应把这些值硬编码进公开仓库，建议通过环境变量或密钥管理系统注入。

---

## 4. 登录换 token 请求（与 refresh 对照）

同样是 `POST /auth/token`，但字段不同：

| 字段名 | 用途 |
|---|---|
| `client_id` | 客户端标识 |
| `client_secret` | 客户端密钥 |
| `grant_type=authorization_code` | 授权码模式 |
| `code` | 回调 URL 带回的授权码 |
| `code_verifier` | PKCE verifier（必须与 challenge 对应） |
| `redirect_uri` | 回调地址（`CALL_BACK`） |
| `include_policy=true` | 扩展参数 |

调用位置：

- `SessionManager.loginWithUrl(uri, block)`

---

## 5. 并发刷新如何防抖（核心保活能力）

`SessionManager.refreshAccessToken(tokenForThisRequest)` 的并发控制分 3 层：

1. **快速检查（锁外）**  
   若当前最新 token 已经不同于请求时 token，直接返回，不刷新。

2. **Mutex 串行化（锁内）**  
   `tokenRefreshMutex.withLock { ... }`，同一时刻仅一个刷新临界区。

3. **双检查 + 任务复用**  
   - 锁内再次比较 token，避免重复刷新；
   - `refreshingTokenJob: Deferred<String?>` 复用进行中的刷新任务；
   - 其他并发请求 `await()` 同一任务结果。

这样可避免“10 个并发请求同时失效 -> 刷 10 次 token”的雪崩。

---

## 6. 刷新失败分支（会发生什么）

### 6.1 新链路（TokenFetcherInterceptor + SessionManager）

- `refresh_token` 不存在：抛 `RuntimeException("refresh_token not exist")`
- 刷新响应 body 为空：抛 `RuntimeException("newRefreshToken failed")`
- 拦截器层捕获异常后：
  - 若拿不到新 token，返回原响应（不重试）
  - 由上层决定是否要求重新登录

### 6.2 旧链路（TokenInterceptor.java）

旧拦截器中对 `Invalid refresh token` 有更激进处理：

- 标记用户未登录
- 清理会话
- 提示并重启应用

文件：

- `/home/runner/work/Pixiv-Shaft/Pixiv-Shaft/app/src/main/java/ceui/lisa/http/TokenInterceptor.java`

---

## 7. 保活相关请求头（非常关键）

请求头注入点：

- `HeaderInterceptor.addHeader(...)`

每次请求会加：

- `authorization: Bearer <access_token>`（需要 token 的 API）
- `accept-language`
- `app-os: ios`
- `app-version: 7.13.4`
- `user-agent: PixivIOSApp/7.13.4 (iOS 16.0.3; iPhone13,3)`
- `x-client-time`
- `x-client-hash`

`x-client-time` / `x-client-hash` 生成逻辑在：

- `/home/runner/work/Pixiv-Shaft/Pixiv-Shaft/app/src/main/java/ceui/loxia/PixivHeaders.kt`

计算方式：

1. `x-client-time = 当前时间(yyyy-MM-dd'T'HH:mm:ssZZZZZ)`
2. `x-client-hash = MD5(x-client-time + SALT)`
3. `SALT = 28c1fdd170a5204386cb1313c7077b34f83e4aaf4aa829ce78c231e05b0bae2c`

> 注意：MD5 已被证明不安全，这里仅用于兼容现有接口协议，不应用于新安全设计。
> 若后续协议允许，优先改为服务端下发或配置化管理；该 SALT 在新实现中应按敏感配置处理，避免在公开仓库硬编码。

---

## 8. 新 CLI 迁移建议（按原策略保活）

如果你在新项目（例如 `pixiv` 或你自己的 CLI）复刻保活机制，建议最小实现：

1. Token 存储结构：
   - `access_token`
   - `refresh_token`
   - `expire_at`
2. 请求拦截器：
   - 注入 `Authorization` 和 `x-client-*` 头
3. 失效检测：
   - 先兼容原逻辑：HTTP 400 + OAuth 错误文本
4. 并发刷新：
   - `sync.Mutex` + 双检查；或 `singleflight.Group`
5. 刷新重试：
   - 最多 1~2 次指数退避
6. 刷新失败：
   - 进入 `AUTH_BROKEN`，停止自动任务并提示重新登录

---

## 9. 关键源码索引（便于二次核对）

- `SessionManager.refreshAccessToken`  
  `/home/runner/work/Pixiv-Shaft/Pixiv-Shaft/app/src/main/java/ceui/pixiv/session/SessionManager.kt`
- `SessionManager.refreshAccessTokenInternal`  
  `/home/runner/work/Pixiv-Shaft/Pixiv-Shaft/app/src/main/java/ceui/pixiv/session/SessionManager.kt`
- `TokenFetcherInterceptor.intercept`  
  `/home/runner/work/Pixiv-Shaft/Pixiv-Shaft/app/src/main/java/ceui/loxia/TokenFetcherInterceptor.kt`
- `AccountTokenApi.newRefreshToken2 / newLogin`  
  `/home/runner/work/Pixiv-Shaft/Pixiv-Shaft/app/src/main/java/ceui/lisa/http/AccountTokenApi.java`
- `HeaderInterceptor.addHeader`  
  `/home/runner/work/Pixiv-Shaft/Pixiv-Shaft/app/src/main/java/ceui/loxia/HeaderInterceptor.kt`
- `RequestNonce.build`  
  `/home/runner/work/Pixiv-Shaft/Pixiv-Shaft/app/src/main/java/ceui/loxia/PixivHeaders.kt`
- `FragmentLogin` 常量（client_id/client_secret/grant_type/callback）  
  `/home/runner/work/Pixiv-Shaft/Pixiv-Shaft/app/src/main/java/ceui/lisa/fragments/FragmentLogin.kt`
