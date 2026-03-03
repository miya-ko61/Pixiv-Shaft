# Pixiv 收藏自动下载器（Linux Go CLI）完整开发手册

> 本文是**可独立开发**的说明书。你只看这份文档，不看任何仓库源码，也可以从 0 实现一个 Linux 后台 Go CLI：
>
> - 登录后自动获取/刷新 token
> - 定时扫描 Pixiv 收藏（Bookmarks）
> - 下载完整原图到指定路径
> - 全程结构化日志
> - 多层去重（不能重复下载）

---

## 0. 免责声明与合规边界（必须先看）

1. 本文仅用于技术学习与个人非商业用途。
2. 你下载的作品版权归原作者所有，请遵守当地法律法规与平台条款。
3. 不要在公开仓库提交任何账户 token、cookie、client secret。
4. 默认实现应控制请求频率，避免对目标服务造成压力。

---

## 1. 目标系统定义

你要实现一个可长期运行的 `pixivd`：

- 运行环境：Linux（建议 systemd 托管）
- 运行模式：
  - 前台执行（调试）
  - 守护执行（生产）
- 核心职责：
  1. 拿到 access_token（首次登录 / refresh）
  2. 定时拉取收藏插画列表（含分页）
  3. 解析每张图原图 URL（单图/多图）
  4. 去重并下载文件到本地
  5. 写入状态库与日志

---

## 2. 术语与数据模型

## 2.1 术语

- `access_token`：调用 Pixiv API 的短期令牌
- `refresh_token`：用于换新 `access_token` 的长期令牌
- `illust`：插画对象
- `page`：多图作品中的第几张（`p0/p1/...`）
- `next_url`：分页下一页 URL

## 2.2 最小数据结构（Go）

```go
type AccountToken struct {
    AccessToken  string `json:"access_token"`
    RefreshToken string `json:"refresh_token"`
    ExpiresIn    int64  `json:"expires_in"`
    TokenType    string `json:"token_type"`
}

type IllustListResp struct {
    Illusts []Illust `json:"illusts"`
    NextURL string   `json:"next_url"`
}

type Illust struct {
    ID         int64  `json:"id"`
    Title      string `json:"title"`
    PageCount  int    `json:"page_count"`
    CreateDate string `json:"create_date"`

    MetaSinglePage struct {
        OriginalImageURL string `json:"original_image_url"`
    } `json:"meta_single_page"`

    MetaPages []struct {
        ImageURLs struct {
            Original string `json:"original"`
            Large    string `json:"large"`
            Medium   string `json:"medium"`
        } `json:"image_urls"`
    } `json:"meta_pages"`

    User struct {
        ID   int64  `json:"id"`
        Name string `json:"name"`
    } `json:"user"`
}
```

---

## 3. API 端点与请求规范（可直接实现）

## 3.1 OAuth token 接口

- Host: `https://oauth.secure.pixiv.net`
- 路径: `POST /auth/token`
- Content-Type: `application/x-www-form-urlencoded`

### 3.1.1 授权码换 token

字段：

- `client_id`
- `client_secret`
- `grant_type=authorization_code`
- `code`
- `code_verifier`
- `redirect_uri`
- `include_policy=true`

### 3.1.2 refresh token 换新 access token

字段：

- `client_id`
- `client_secret`
- `grant_type=refresh_token`
- `refresh_token`
- `include_policy=true`

## 3.2 应用 API Host

- `https://app-api.pixiv.net`

## 3.3 收藏列表接口（核心）

- `GET /v1/user/bookmarks/illust`
- 常见参数：
  - `user_id=<当前用户ID>`
  - `restrict=public|private`（按需要）
  - `max_bookmark_id=<分页用，可选>`
  - `tag=<标签过滤，可选>`

## 3.4 分页接口

- 推荐：直接请求响应里的 `next_url`
- 你需要把 `next_url` 原样作为下一次 GET URL

## 3.5 请求头（关键）

```text
Authorization: Bearer <access_token>
User-Agent: PixivIOSApp/7.13.4 (iOS 16.0.3; iPhone13,3)
accept-language: zh-CN
app-os: ios
app-version: 7.13.4
x-client-time: <UTC time>
x-client-hash: <hash(x-client-time + salt)>
```

> 说明：`x-client-time` 与 `x-client-hash` 是常见风控字段。建议你在实现中封装 HeaderBuilder，每次请求动态生成。

---

## 4. 登录逻辑深度分析（你开发 Go CLI 重点看这章）

这一章按“可直接实现”的标准写：你照着实现即可，不需要再读仓库源码。

## 4.1 登录完整时序（OAuth + PKCE）

```text
[1] 生成 PKCE:
    code_verifier (随机串) -> SHA256 -> code_challenge

[2] 打开登录页:
    https://app-api.pixiv.net/web/v1/login?code_challenge=...&code_challenge_method=S256&client=pixiv-android

[3] 用户登录授权后回调:
    https://app-api.pixiv.net/web/v1/users/auth/pixiv/callback?code=AUTH_CODE

[4] 用 code 换 token:
    POST https://oauth.secure.pixiv.net/auth/token
    grant_type=authorization_code
    code=AUTH_CODE
    code_verifier=步骤[1]原始值

[5] 保存 token:
    access_token + refresh_token + expires_in

[6] 后续请求遇到 token 失效:
    refresh_token -> 新 access_token
```

## 4.2 CLI 推荐登录策略（工程上最稳）

因为 CLI 无 UI，建议采用以下方式之一：

1. **方式 A（推荐）**：你手动登录一次拿到 `refresh_token`，写入配置；服务启动后仅走 refresh。
2. 方式 B：实现完整 OAuth + PKCE（打开浏览器、回调本地端口）获取授权码，再换 token。

如果你只追求稳定可用，先实现方式 A，再迭代方式 B。

## 4.3 授权码换 token：请求细节（必须对齐）

- URL：`POST https://oauth.secure.pixiv.net/auth/token`
- Content-Type：`application/x-www-form-urlencoded`
- 表单字段：
  - `client_id`
  - `client_secret`
  - `grant_type=authorization_code`
  - `code=<回调拿到的授权码>`
  - `code_verifier=<生成PKCE时的原始verifier>`
  - `redirect_uri=https://app-api.pixiv.net/web/v1/users/auth/pixiv/callback`
  - `include_policy=true`

`code_verifier` 必须与当时生成 `code_challenge` 的那一个严格对应，否则会登录失败。

## 4.4 refresh token 刷新：触发条件与策略

你需要在请求层统一处理 token 失效，触发 refresh 的推荐条件：

- HTTP `400` 且响应包含 OAuth/token 相关错误文本
- 或 HTTP `401` 且业务可判定为 token 失效

刷新请求：

- URL：`POST https://oauth.secure.pixiv.net/auth/token`
- 表单字段：
  - `client_id`
  - `client_secret`
  - `grant_type=refresh_token`
  - `refresh_token=<本地保存值>`
  - `include_policy=true`

刷新成功后必须立即：

1. 更新内存 token
2. 原子化写回磁盘（防进程重启丢失）
3. 使用新 token 重试一次原请求

## 4.5 并发刷新防抖（必须实现）

场景：多个请求同时 400/401，不能并发打 N 次 refresh。

做法：

- 用 `sync.Mutex` 或 `singleflight.Group`
- 双重检查：进入锁后再次判断 token 是否已被其它协程刷新

伪代码：

```go
func (m *Manager) RefreshIfNeeded(oldToken string) (string, error) {
    m.mu.Lock()
    defer m.mu.Unlock()

    if m.currentAccessToken != oldToken && m.currentAccessToken != "" {
        return m.currentAccessToken, nil
    }
    return m.refreshLocked()
}
```

## 4.6 登录请求头深度说明（风控关键）

除 `Authorization` 外，建议统一注入以下头：

```text
User-Agent: PixivIOSApp/7.13.4 (iOS 16.0.3; iPhone13,3)
accept-language: zh-CN
app-os: ios
app-version: 7.13.4
x-client-time: <UTC ISO-8601>
x-client-hash: <MD5(x-client-time + 固定salt)>
```

建议你实现：

- `BuildClientTimeHeaders(now time.Time) (xClientTime, xClientHash string)`
- HTTP 中间件统一注入，避免每个请求手写
- `x-client-hash` 的 MD5 用法是为兼容接口协议，不是用于本地安全加密

## 4.7 错误分支设计（必须覆盖）

至少区分两类错误：

1. **可恢复错误**（网络波动、临时 OAuth 失败）  
   - 先 refresh（或退避重试）再重试原请求
2. **不可恢复错误**（refresh_token 无效）  
   - 进入 `AUTH_BROKEN` 状态
   - 停止任务并提示人工重新登录

不要无限重试 refresh，建议最多 1~2 次， 并使用指数退避（例如第 1 次等待 1s，第 2 次等待 2s）。

## 4.8 Token 持久化规范（防损坏）

推荐把 token 存在 `token.json`，写入使用“临时文件 + rename”原子替换：

```go
type StoredToken struct {
    AccessToken  string    `json:"access_token"`
    RefreshToken string    `json:"refresh_token"`
    ExpireAt     time.Time `json:"expire_at"`
    UpdatedAt    time.Time `json:"updated_at"`
}
```

启动时加载策略：

1. 文件不存在：报错并提示先登录
2. 有 refresh_token：优先 refresh 获取新 access_token
3. refresh 失败：退出并给出明确错误日志

## 4.9 可直接抄用的 Go 接口骨架

```go
type TokenManager interface {
    GetAccessToken(ctx context.Context) (string, error)
    ExchangeCode(ctx context.Context, code, codeVerifier string) error
    Refresh(ctx context.Context) (string, error)
}
```

---

## 5. 下载对象解析与落盘规则

## 5.1 原图 URL 解析规则

- `page_count == 1`：取 `meta_single_page.original_image_url`
- `page_count > 1`：遍历 `meta_pages[i].image_urls.original`

## 5.2 文件扩展名

从 URL 最后一个 `.` 后缀取扩展名（`jpg/png/webp/...`）。
若解析失败，默认 `jpg`。

## 5.3 文件命名规则（建议固定）

建议使用稳定、可逆、不会冲突的命名：

```text
illust_{illustID}_p{pageIndex}.{ext}
```

示例：

- `illust_116457142_p0.jpg`
- `illust_116457142_p1.jpg`

## 5.4 目录结构建议

```text
/data/pixiv/bookmarks/
  ├── images/
  │   ├── 2026-03/
  │   └── 2026-04/
  ├── state/
  │   ├── pixiv.db
  │   └── token.json
  └── logs/
      └── pixivd.log
```

---

## 6. 去重设计（必须实现三层）

你要求“不能重复下载”，建议三层同时启用：

## 6.1 层 1：任务内去重（内存）

- key：`illustID_page`
- 作用：同一轮扫描中避免重复入队

```go
key := fmt.Sprintf("%d_%d", illustID, pageIndex)
if _, ok := taskSeen[key]; ok { skip }
taskSeen[key] = struct{}{}
```

## 6.2 层 2：文件存在性去重

- 如果目标路径文件已存在，直接跳过
- 适合“服务重启后继续”场景

## 6.3 层 3：状态库去重（推荐 SQLite）

表建议：

```sql
CREATE TABLE IF NOT EXISTS downloaded_files (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  illust_id INTEGER NOT NULL,
  page_index INTEGER NOT NULL,
  url_hash TEXT NOT NULL,
  file_path TEXT NOT NULL,
  file_size INTEGER DEFAULT 0,
  downloaded_at TEXT NOT NULL,
  UNIQUE(illust_id, page_index)
);
```

查询逻辑：

- 若 `(illust_id, page_index)` 已存在 => 跳过
- 下载成功后插入；失败不插入

---

## 7. 定时任务与保活

## 7.1 调度器

- 使用 `time.Ticker` 每 `scan_interval` 执行一轮
- 每轮流程：

```text
refresh token(必要时)
  -> 拉取收藏第一页
  -> while next_url != "": 拉下一页
  -> 生成下载任务
  -> 去重
  -> 并发下载
  -> 写入状态库
  -> 输出轮次统计日志
```

## 7.2 并发下载

- worker pool：`concurrency` 建议 2~6
- 每个下载任务独立超时（如 30s）

## 7.3 重试策略

- 网络错误：指数退避（1s/2s/4s，最多 3 次）
- 429/5xx：可重试
- 401/400 token 错误：先 refresh 再重试一次

---

## 8. 代理、网络与可达性

## 8.1 代理配置

支持两种来源：

1. 环境变量：`HTTP_PROXY/HTTPS_PROXY/NO_PROXY`
2. 配置文件：`proxy_url`

## 8.2 域名替换（可选）

某些网络环境可对 `i.pximg.net` 做可配置替换（如镜像域名或特定入口）。

实现建议：

```go
func replaceImageHost(rawURL, newHost string) string
```

## 8.3 DNS 策略（可选）

- 默认系统 DNS
- 失败时降级到 DoH（可配置）

---

## 9. 日志标准（可直接用于 ELK/Loki）

## 9.1 字段规范

```json
{
  "time": "2026-03-03T06:30:00Z",
  "level": "INFO",
  "msg": "download_success",
  "task_id": "scan-20260303-0630",
  "illust_id": 116457142,
  "page": 0,
  "file_path": "/data/pixiv/bookmarks/images/2026-03/illust_116457142_p0.jpg",
  "duration_ms": 842,
  "retry": 1
}
```

## 9.2 最少日志事件

1. `service_start`
2. `token_refresh_success` / `token_refresh_failed`
3. `scan_started` / `scan_finished`
4. `download_queued`
5. `download_success`
6. `download_skipped_duplicate`
7. `download_failed`

---

## 10. 配置文件模板（生产可用）

`config.yaml`：

```yaml
pixiv:
  oauth_host: "https://oauth.secure.pixiv.net"
  api_host: "https://app-api.pixiv.net"
  client_id: ""
  client_secret: ""
  refresh_token: ""

runtime:
  download_dir: "/data/pixiv/bookmarks/images"
  state_db: "/data/pixiv/bookmarks/state/pixiv.db"
  token_store: "/data/pixiv/bookmarks/state/token.json"
  scan_interval: "10m"
  concurrency: 4
  request_timeout: "30s"
  retry_max: 3

network:
  proxy_url: ""
  replace_pximg_host: ""
  enable_doh_fallback: false

log:
  level: "info"
  format: "json"
  file: "/data/pixiv/bookmarks/logs/pixivd.log"
```

---

## 11. 推荐项目结构（可直接创建）

```text
pixivd/
  ├── cmd/pixivd/main.go
  ├── internal/config/
  ├── internal/logx/
  ├── internal/auth/
  ├── internal/api/
  ├── internal/downloader/
  ├── internal/dedup/
  ├── internal/store/
  ├── internal/scheduler/
  ├── internal/service/
  ├── migrations/
  ├── config.example.yaml
  └── Makefile
```

---

## 12. 核心流程伪代码（端到端）

```go
func runOneRound(ctx context.Context) error {
    token, err := tokenManager.GetAccessToken(ctx)
    if err != nil { return err }

    all := []ImageTask{}
    nextURL := api.BuildBookmarksURL()

    for nextURL != "" {
        resp, err := api.GetBookmarks(ctx, token, nextURL)
        if isAuthError(err) {
            token, err = tokenManager.Refresh(ctx)
            if err != nil { return err }
            resp, err = api.GetBookmarks(ctx, token, nextURL)
        }
        if err != nil { return err }

        tasks := parseOriginalImageTasks(resp.Illusts)
        all = append(all, tasks...)
        nextURL = resp.NextURL
    }

    queued := dedup.Filter(all) // 内存 + 文件 + DB
    return downloader.DownloadAll(ctx, queued)
}
```

---

## 13. systemd 部署模板

`/etc/systemd/system/pixivd.service`

```ini
[Unit]
Description=Pixiv Bookmark Downloader Service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=pixiv
Group=pixiv
WorkingDirectory=/opt/pixivd
ExecStart=/opt/pixivd/pixivd --config /etc/pixivd/config.yaml
Restart=always
RestartSec=5
LimitNOFILE=65535
Environment=TZ=Asia/Shanghai

[Install]
WantedBy=multi-user.target
```

常用命令：

```bash
sudo systemctl daemon-reload
sudo systemctl enable pixivd
sudo systemctl start pixivd
sudo systemctl status pixivd
journalctl -u pixivd -f
```

---

## 14. 开发顺序（建议按此落地）

1. 建项目骨架 + 配置加载 + 日志
2. 实现 refresh token 换 access token
3. 实现 bookmarks 拉取（含分页）
4. 实现任务解析（单图/多图）
5. 实现文件下载
6. 实现三层去重
7. 接入 scheduler + worker pool
8. 增加系统信号处理（优雅退出）
9. 接入 systemd 与监控

---

## 15. 验收标准（Definition of Done）

满足以下条件才算完成：

1. 服务启动后可自动拉取收藏并下载原图
2. 重启服务后不会重复下载旧文件
3. 定时轮询稳定运行 24h 无崩溃
4. token 过期后能自动刷新并继续任务
5. 网络抖动场景下有重试且日志可追踪
6. 日志可统计每轮：发现数/新增数/跳过数/失败数

---

## 16. 故障排查手册

## 16.1 现象：全部 401/400

- 检查 refresh token 是否失效
- 检查请求头 Authorization 是否为 `Bearer <token>`
- 检查本机时间是否漂移（影响签名时间相关头）

## 16.2 现象：列表有数据但无下载

- 检查去重逻辑是否误判（UNIQUE 键冲突）
- 检查下载目录权限
- 检查 URL 解析是不是拿到了 non-original URL

## 16.3 现象：频繁超时

- 减小并发
- 提高 timeout
- 配置代理或更换网络出口

## 16.4 现象：下载重复

- 确认三层去重都启用了
- 检查 state DB 是否被清空或路径变动
- 检查 key 是否稳定（必须 `illustID_page`）

---

## 17. 安全建议（生产）

1. token/secret 放在：
   - `/etc/pixivd/config.yaml` + 600 权限，或
   - 环境变量 + Secret 管理系统
2. 禁止在日志打印完整 token
3. 对下载文件名做路径清理，防止目录穿越
4. 限制单文件最大大小，避免磁盘打满

---

## 18. 你可以直接照抄的最小命令行设计

```bash
pixivd run --config /etc/pixivd/config.yaml
pixivd once --config ./config.yaml
pixivd doctor --config ./config.yaml
pixivd migrate --dsn /data/pixiv/bookmarks/state/pixiv.db
```

- `run`：守护模式，定时循环
- `once`：执行单轮，便于调试
- `doctor`：检查 token/目录/数据库/网络
- `migrate`：初始化数据库表

---

## 19. 最后结论

如果你只按本文实现，不看任何源码，也可以完成一个可用的 Linux Go CLI 服务。实现成败最关键的 4 点是：

1. Token 自动刷新 + 并发防抖
2. 收藏分页完整拉取
3. 三层去重严格执行
4. 可观测日志 + 可恢复重试

做到这四点，你的“自动定时下载收藏完整图片且不重复”的目标就能稳定达成。
