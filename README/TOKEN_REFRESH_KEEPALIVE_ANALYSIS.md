# Pixiv-Shaft Token 刷新与保活实现文档（面向只看 Markdown 开发）

> 这份文档是“实现规格”，目标是：你不看任何项目源码，也能在新 CLI/服务里完整复刻 token 自动刷新与保活流程。

---

## 0. 你要实现的能力（先看这一段）

你最终需要具备 6 个能力：

1. 每次业务请求都带上认证头与保活头。
2. 当接口返回“token 失效”特征时，自动触发 refresh。
3. 多并发请求下只允许一次真实 refresh（防止 token 刷新风暴）。
4. refresh 成功后自动重放原请求。
5. refresh 失败后进入“需要重新登录”状态。
6. 整个流程可观测（日志与状态机清晰）。

---

## 1. 术语与基础常量

### 1.1 Host

- App API Host: `https://app-api.pixiv.net`
- OAuth Host: `https://oauth.secure.pixiv.net`

### 1.2 OAuth 关键常量（实现时必须提供）

- `CLIENT_ID = <YOUR_CLIENT_ID>`
- `CLIENT_SECRET = <YOUR_CLIENT_SECRET>`
- `GRANT_TYPE_REFRESH = refresh_token`
- `GRANT_TYPE_AUTH_CODE = authorization_code`
- `REDIRECT_URI = https://app-api.pixiv.net/web/v1/users/auth/pixiv/callback`
- `TOKEN_HEAD = Bearer `（注意有空格）

> 安全提示：建议通过环境变量或密钥管理注入，不要硬编码到公开仓库。  
> 若你需要对齐本仓库默认实现，可从以下文件读取当前常量：  
> `/home/runner/work/Pixiv-Shaft/Pixiv-Shaft/app/src/main/java/ceui/lisa/fragments/FragmentLogin.kt`

### 1.3 Token 失效判定文本

当响应状态码是 `400`，并且响应体包含任一文本时，视为 token 失效：

- `Error occurred at the OAuth process`
- `Invalid refresh token`

---

## 2. refresh token 请求到底携带哪些变量

### 2.1 请求定义

- Method: `POST`
- URL: `https://oauth.secure.pixiv.net/auth/token`
- Content-Type: `application/x-www-form-urlencoded`

### 2.2 字段字典（必须带齐）

| 字段名 | 类型 | 必填 | 固定/动态 | 说明 |
|---|---|---|---|---|
| `client_id` | string | 是 | 固定 | 客户端 ID |
| `client_secret` | string | 是 | 固定 | 客户端 Secret |
| `grant_type` | string | 是 | 固定 | 必须为 `refresh_token` |
| `refresh_token` | string | 是 | 动态 | 来自当前会话存储 |
| `include_policy` | bool | 是 | 固定 | 固定 `true` |

### 2.3 请求示例（可直接抄）

```bash
curl -X POST 'https://oauth.secure.pixiv.net/auth/token' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'client_id=<YOUR_CLIENT_ID>' \
  --data-urlencode 'client_secret=<YOUR_CLIENT_SECRET>' \
  --data-urlencode 'grant_type=refresh_token' \
  --data-urlencode 'refresh_token=<YOUR_REFRESH_TOKEN>' \
  --data-urlencode 'include_policy=true'
```

### 2.4 响应后你必须做的事

拿到刷新响应后至少更新：

- `access_token`
- `refresh_token`（有些场景服务端会旋转）
- `expires_in` / `expire_at`
- 用户信息快照（如你有这部分模型）

---

## 3. 请求保活头（每次请求都要）

业务请求建议统一注入如下头：

- `authorization: Bearer <access_token>`（仅需鉴权 API）
- `accept-language: <按你的语言策略>`
- `app-os: ios`
- `app-version: 7.13.4`
- `user-agent: PixivIOSApp/7.13.4 (iOS 16.0.3; iPhone13,3)`
- `x-client-time: <当前时间>`
- `x-client-hash: <MD5(time + SALT)>`

### 3.1 x-client-time 生成

格式：`yyyy-MM-dd'T'HH:mm:ssZZZZZ`

例：`2026-03-05T14:03:12+08:00`

### 3.2 x-client-hash 生成

- `SALT = <YOUR_X_CLIENT_HASH_SALT>`
- 拼接：`plain = x_client_time + SALT`
- 计算：`x_client_hash = md5(plain)`（小写十六进制 32 位）

> 注意：MD5 不安全，这里是“协议兼容”用途，不建议用于新的安全设计。
> 若你需要对齐本仓库默认实现，可从以下文件读取当前 SALT：  
> `/home/runner/work/Pixiv-Shaft/Pixiv-Shaft/app/src/main/java/ceui/loxia/PixivHeaders.kt`

---

## 4. 自动刷新触发与重放流程（时序版）

### 4.1 正常流程

1. 发送请求（带 access token）。
2. 若返回非 token 失效，直接返回结果。

### 4.2 token 失效流程

1. 收到 `HTTP 400`。
2. 响应体命中失效文本。
3. 从原请求头取出 `tokenForThisRequest`（去掉 `Bearer ` 前缀）。
4. 调用 `refreshAccessToken(tokenForThisRequest)`。
5. 刷新成功后，用新 token 重建请求并重放一次。
6. 把重放结果返回上层。

---

## 5. 并发刷新防抖（最关键）

核心原则：**同一时刻最多一次真实 refresh 请求**。

你可以按下面的三层结构实现：

### 层 1：锁外快速检查

- 读取当前缓存 token：`currentAccessToken`
- 如果 `currentAccessToken != tokenForThisRequest`，说明别的请求已刷新成功，直接返回 `currentAccessToken`

### 层 2：互斥锁串行

- 使用一个全局锁（如 Go `sync.Mutex`）进入临界区

### 层 3：锁内双检查 + in-flight 任务复用

- 进入锁后再比较一次 token（双检查）
- 若确实还没刷新：
  - 启动一个刷新任务（future/promise/goroutine result）
  - 其他并发请求等待同一个任务结果

这样能避免“10 个并发失败请求触发 10 次 refresh”。

---

## 6. 状态机（建议照着做）

定义 5 个状态：

- `AUTH_OK`：token 可用
- `AUTH_REFRESHING`：正在刷新
- `AUTH_RETRYING`：已刷新，正在重放原请求
- `AUTH_BROKEN`：refresh 失败，需要用户重新登录
- `AUTH_LOGGED_OUT`：用户主动退出

状态迁移：

- `AUTH_OK -> AUTH_REFRESHING`：命中失效判定
- `AUTH_REFRESHING -> AUTH_RETRYING`：refresh 成功
- `AUTH_RETRYING -> AUTH_OK`：重放成功
- `AUTH_REFRESHING -> AUTH_BROKEN`：refresh 失败
- `AUTH_BROKEN -> AUTH_OK`：用户重新登录成功

---

## 7. 失败分支与处理策略

### 7.1 refresh_token 不存在

- 直接判定不可恢复，进入 `AUTH_BROKEN`

### 7.2 refresh 接口返回空/异常

- 记录错误日志（含 request id）
- 不要无限重试
- 建议最多 1~2 次指数退避
- 最终失败进入 `AUTH_BROKEN`

### 7.3 命中 `Invalid refresh token`

- 视为 refresh token 已失效
- 清空本地会话
- 引导重新登录

---

## 8. 可直接实现的伪代码（语言无关）

```text
function sendWithAutoRefresh(request):
    attachHeaders(request)
    response = http.send(request)

    if not isTokenExpired(response):
        return response

    oldToken = extractBearerToken(request.Authorization)
    newToken = refreshAccessTokenWithDedup(oldToken)

    if newToken is null:
        markAuthBroken()
        return response

    retryReq = clone(request)
    retryReq.Authorization = "Bearer " + newToken
    return http.send(retryReq)
```

```text
function refreshAccessTokenWithDedup(tokenForThisRequest):
    current = session.accessToken
    if current != tokenForThisRequest:
        return current

    lock(refreshMutex)
    defer unlock(refreshMutex)

    current = session.accessToken
    if current != tokenForThisRequest:
        return current

    # isCompleted 表示该任务已结束（无论成功/失败），可创建新任务
    if refreshingJob is nil or refreshingJob.isCompleted:
        refreshingJob = async doRefreshHttpCall()

    result = await refreshingJob
    if result.success:
        session.update(result.tokens)
        return result.accessToken
    else:
        return null
```

---

## 9. 登录换 token（和 refresh 的区别）

初次登录使用授权码模式（不是 refresh 模式）：

- `grant_type=authorization_code`
- 额外需要：
  - `code`
  - `code_verifier`（PKCE）
  - `redirect_uri`

而 refresh 模式只需要：

- `grant_type=refresh_token`
- `refresh_token`

---

## 10. 排错清单（按优先级）

1. **总是 400**：检查是否命中失效文本，避免把普通 400 当 token 失效。
2. **refresh 成功但仍 401/400**：确认重放请求用了“新 token”。
3. **偶发并发失败**：检查是否真的只有一个刷新 in-flight。
4. **hash 错误**：检查时间格式、时区、SALT、MD5 小写输出。
5. **refresh 循环**：限制最大刷新重试次数，失败进入 `AUTH_BROKEN`。

---

## 11. 最小落地清单（CLI 可执行）

- [ ] 会话存储：`access_token / refresh_token / expire_at`
- [ ] 请求拦截器：统一注入 headers
- [ ] 失效判定器：`400 + 关键错误文本`
- [ ] 刷新器：`POST /auth/token` + 字段齐全
- [ ] 并发去重：锁 + 双检查 + in-flight 任务复用
- [ ] 请求重放：刷新成功后重发原请求
- [ ] 失败降级：进入 `AUTH_BROKEN` 并提示重新登录

---

## 12. 来源索引（便于后续人工核对）

- `/home/runner/work/Pixiv-Shaft/Pixiv-Shaft/app/src/main/java/ceui/loxia/TokenFetcherInterceptor.kt`
- `/home/runner/work/Pixiv-Shaft/Pixiv-Shaft/app/src/main/java/ceui/pixiv/session/SessionManager.kt`
- `/home/runner/work/Pixiv-Shaft/Pixiv-Shaft/app/src/main/java/ceui/lisa/http/AccountTokenApi.java`
- `/home/runner/work/Pixiv-Shaft/Pixiv-Shaft/app/src/main/java/ceui/loxia/HeaderInterceptor.kt`
- `/home/runner/work/Pixiv-Shaft/Pixiv-Shaft/app/src/main/java/ceui/loxia/PixivHeaders.kt`
- `/home/runner/work/Pixiv-Shaft/Pixiv-Shaft/app/src/main/java/ceui/lisa/fragments/FragmentLogin.kt`
- `/home/runner/work/Pixiv-Shaft/Pixiv-Shaft/app/src/main/java/ceui/loxia/Client.kt`
