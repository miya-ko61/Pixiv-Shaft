# Pixiv Linux Go 后台服务 / CLI 开发文档（基于 Pixiv-Shaft）

> 目标：实现一个运行在 Linux 的 Go 后台服务，登录后自动定时下载「收藏」原图到指定路径，带日志，并且保证去重不重复下载。

## 1. 现有仓库可复用能力总览

- 登录与 Token：
  - `app/src/main/java/ceui/lisa/http/AccountApi.java`
  - `app/src/main/java/ceui/lisa/http/AccountTokenApi.java`
  - `app/src/main/java/ceui/lisa/http/TokenInterceptor.java`
  - `app/src/main/java/ceui/pixiv/session/SessionManager.kt`
  - `app/src/main/java/ceui/lisa/feature/HostManager.java`
- Pixiv API 定义：
  - `app/src/main/java/ceui/lisa/http/AppApi.java`
- 下载与去重：
  - `app/src/main/java/ceui/lisa/download/IllustDownload.java`
  - `app/src/main/java/ceui/lisa/download/FileCreator.java`
  - `app/src/main/java/ceui/lisa/utils/Common.java`
  - `app/src/main/java/ceui/lisa/helper/DeduplicateArrayList.java`
  - `app/src/main/java/ceui/lisa/database/DownloadEntity.java`
- 日志：
  - `app/src/main/java/ceui/lisa/utils/Common.java` (`showLog`)
  - `app/src/main/java/ceui/lisa/http/Retro.java` / `ceui/loxia/Client.kt`（HTTP 日志拦截）

---

## 2. 登录流程与 Token 流程梳理

## 2.1 OAuth + PKCE 登录流程

1. 生成 PKCE：
   - `HostManager.getPkce()` 内调用 `PkceUtil.generateCodeVerifier()` / `generateCodeChallenge()`
2. 生成登录 URL：
   - `HostManager.getLoginUrl()` → `https://app-api.pixiv.net/web/v1/login?...`
3. 用户网页登录后回调：
   - `OutWakeActivity` 处理 `pixiv://account/login?code=...`
4. 用授权码换 Token：
   - `AccountApi.newLogin(...)`（`grant_type=authorization_code`）
5. 本地持久化与会话更新：
   - `Local.saveUser(...)`
   - `SessionManager.updateSession(...)`

对应关键参数常量：
- `FragmentLogin.CLIENT_ID`
- `FragmentLogin.CLIENT_SECRET`
- `FragmentLogin.AUTH_CODE`
- `FragmentLogin.CALL_BACK`

## 2.2 Token 刷新流程

- 触发点：
  - Java 旧链路：`TokenInterceptor` 遇到 400 且响应包含 `Error occurred at the OAuth process`
  - Kotlin 新链路：`TokenFetcherInterceptor` 遇到同类 token 错误
- 刷新接口：
  - `AccountTokenApi.newRefreshToken(...)` / `newRefreshToken2(...)`
  - `grant_type=refresh_token`
- 并发控制：
  - Java：`TokenInterceptor.getNewToken(...)` 使用 `synchronized`
  - Kotlin：`SessionManager.refreshAccessToken(...)` 使用 `Mutex` + `Deferred`

---

## 3. Pixiv 相关 API（Go 端重点）

核心定义在 `AppApi.java`，Go CLI 最常用接口：

- 收藏列表（重点）：
  - `GET v1/user/bookmarks/illust`
  - 方法：`getUserLikeIllust(...)`
- 收藏标签（可选）：
  - `GET v1/user/bookmark-tags/illust`
- 收藏详情（可选）：
  - `GET v2/illust/bookmark/detail`
- 分页：
  - 普遍返回 `next_url`，仓库中通过 `getNextIllust(...)` 等方法继续拉取

建议 Go 端先实现最小闭环：
1. 登录/refresh 获取 access token  
2. 拉取书签插画分页  
3. 提取原图 URL 下载落盘  
4. 记录状态并定时重复

---

## 4. 下载、命名、去重机制梳理

## 4.1 原图 URL 选取

- `IllustDownload.getUrl(...)` 最终从：
  - 单图：`meta_single_page.original_image_url`
  - 多图：`meta_pages[i].image_urls.original`

## 4.2 命名策略

- `FileCreator.customFileName(...)` 按配置拼接：标题、作品 ID、页码、作者等
- `FileCreator.deleteSpecialWords(...)` 清理文件名特殊字符

推荐 Go 端稳定命名模板（避免重复）：
- `illust_{illust_id}_p{page_index}.{ext}`

## 4.3 去重策略（必须）

仓库里已有两类去重思想：

1. **集合去重**：`DeduplicateArrayList`（按 `getDuplicateKey()`）
2. **文件存在性去重**：
   - `Common.isIllustDownloaded(...)`
   - `FileCreator.isExist(...)` / `SAFile.isFileExists(...)`

Go 服务建议采用“三层去重”：

1. **任务层**：内存 `map[string]struct{}`，key=`illustID_page`
2. **文件层**：目标文件已存在则跳过
3. **状态层**：SQLite/BoltDB 记录成功下载（`illustID,page,url_hash,path,download_at`）

---

## 5. 代理/绕过代理能力梳理

已有能力：

- `HostManager.replaceUrl(...)`
  - `isUsePixivCat()` 时将 `i.pximg.net` 改为 `i.pixiv.re`
  - `isAutoFuckChina()` 时可替换为 IP 直连（HTTP）
- `HostManager.updateHost()` 通过 DoH（Cloudflare / DNSSB）更新可用 IP
- `Retro.fuckChinaWithConfig(...)` 可启用自定义 DNS + SSL 策略

Go 服务建议：

1. 默认支持系统代理：
   - `HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY`
2. 可选配置自定义代理：
   - `proxy_url: socks5://...` 或 `http://...`
3. 可选镜像/域名替换：
   - 与 `HostManager.replaceUrl(...)` 等价
4. 可选 DNS over HTTPS：
   - 仅在默认解析失败时启用

---

## 6. 保活 / 定时任务 / 后台服务建议

仓库中不存在标准 Linux daemon，但有队列与任务执行思路：

- `ceui/lisa/feature/worker/Worker.java`：串行任务执行
- `ceui/pixiv/ui/task/*`：协程任务与队列管理

Go 端建议：

1. `systemd` 方式托管进程（推荐）
2. 服务内 ticker 周期执行：
   - 例如每 5~15 分钟扫描收藏增量
3. 每轮执行流程：
   - refresh token（必要时）→ 拉取分页 → 去重 → 下载 → 持久化状态
4. 失败重试：
   - 网络错误指数退避（1s/2s/4s/... 上限）
   - 401/400 token 错误优先 refresh 后重试一次

---

## 7. 日志建议（对应“含有日志”要求）

仓库现状：

- `Common.showLog(...)` → `Log.d("==SHAFT==>", ...)`
- OkHttp `HttpLoggingInterceptor` 记录请求/响应

Go 服务建议日志字段化（JSON）：

- `time`, `level`, `msg`
- `task_id`, `illust_id`, `page`, `url`, `file_path`
- `http_status`, `retry`, `error`

最少应有四类日志：
1. 登录与 token 刷新
2. 每轮定时任务开始/结束与统计
3. 单文件下载成功/跳过/失败
4. 异常与重试

---

## 8. Go CLI 推荐模块设计（可直接落地）

- `cmd/pixivd/main.go`：启动入口
- `internal/config`：YAML/ENV 配置
- `internal/auth`：PKCE、登录、token 刷新
- `internal/api`：Pixiv 接口封装（bookmarks、next_url）
- `internal/downloader`：并发下载与限速
- `internal/dedup`：去重（内存+DB+文件）
- `internal/store`：状态持久化（SQLite/BoltDB）
- `internal/scheduler`：定时轮询
- `internal/logx`：结构化日志

建议配置项：

```yaml
download_dir: /data/pixiv/bookmarks
scan_interval: 10m
concurrency: 4
proxy_url: ""
replace_pximg_host: ""
request_timeout: 20s
retry_max: 3
```

---

## 9. 最小可用实现路径（MVP）

1. 手动登录一次，保存 refresh_token
2. 服务启动先 refresh 获取 access_token
3. 拉取 `v1/user/bookmarks/illust` 全分页
4. 解析每个 illust 的原图 URL（含多页）
5. 按 `illustID_page` 去重后下载
6. 定时重复第 2~5 步
7. 输出轮次统计日志（新增/跳过/失败数量）

---

## 10. 风险与注意事项

1. 本仓库含明文客户端参数（`CLIENT_ID/CLIENT_SECRET`），Go 独立服务建议改为配置注入，不要硬编码。
2. 代理与域名替换能力可能受网络环境、证书策略和目标站点策略变化影响，需要可开关配置。
3. 下载去重不要只依赖文件名，建议同时记录 `illustID+page` 唯一键。
4. API 风控（频率限制）需要控制并发与请求间隔。

