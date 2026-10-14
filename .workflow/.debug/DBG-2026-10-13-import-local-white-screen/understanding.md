# Understanding Document

**Session ID**: DBG-2026-10-13-import-local-white-screen
**Bug Description**: 首页访问时 `/api/import-local` 返回 400，随后 React #31 白屏；错误对象包含 `$wbEncrypted` 与 `envelope`。
**Started**: 2026-10-13 Asia/Shanghai

---

## Exploration Timeline

### Iteration 1 - Initial Exploration

#### Current Understanding

- WebUI 的 `importLocal()` 通过 HTTP POST 调用 `/api/import-local`。
- 服务端从 `workbuddy-desktop.info` 读取认证 JSON，并把当前账号状态透传给前端。
- 本机认证文件的 `account.nickname`、`auth.accessToken`、`auth.refreshToken` 是
  `{"$wbEncrypted":1,"envelope":"..."}` 对象，不是字符串。
- `/api/status` 原样返回 `account.nickname`，账号页把它作为 React 子节点，符合 React #31 的对象渲染原因。
- 导入逻辑只接受字符串 token，因此加密认证文件无法首次解析为账号记录。

#### Hypotheses

- H1（confirmed）：认证字段类型从字符串变成加密对象，导致导入解析失败。
- H2（confirmed）：状态 API 透传加密对象，首页将其渲染为 React child，触发白屏。
- H3（rejected）：HTTP body 或前端 POST 参数格式错误；服务端 `/api/import-local` 无 body 参数要求，空 JSON 可正常路由。

### Iteration 2 - Evidence Analysis

#### Evidence

- 本机 `workbuddy-desktop.info`：`account.uid` 为明文字符串；`account.nickname` 与 `auth.accessToken` 为对象，键为 `$wbEncrypted`、`envelope`。
- 原服务实现 `get_str()` 只接受 `Value::String`，因此加密 token 使 `imported_account_from_root()` 返回 `None`。
- 原 `/api/status` 和 Tauri `build_app_status()` 直接使用 `acct.get("nickname")`。
- 新编译服务实测：`GET /api/status` 返回 `nickname: null`、明文 UID；`POST /api/import-local` 返回 200，并复用账号库中同 UID 的已有完整记录。

#### Corrected Understanding

- ~~`/api/import-local` 一定是请求参数错误~~ → 认证文件加密升级导致后端无法把 token 当明文解析；空 JSON 请求本身合法。
- ~~只修前端渲染即可~~ → 导入链路也需要处理「加密认证文件 + 已有账号库」的兼容情况。

### Resolution

- 共享核心的账号元数据只输出字符串/数字标量；加密对象变为 `null`。
- HTTP server、CLI、Tauri 状态 API 均使用安全字符串提取，禁止把认证对象透传给 UI。
- 加密认证文件导入时使用明文 UID 查找已有账号库记录；没有已有记录时返回明确的首次导入提示，不写入无 token 半成品。
- 账号页增加运行时字符串保护，旧服务返回异常对象时也不会因状态文案导致 React #31。

#### Verification

- `cargo test -p wb-switch-core account::tests`：10 passed。
- `cargo test -p wb-switch-core auth_file::tests`：4 passed。
- `cargo check -p wb-switch-server`：passed。
- `cargo build -p wb-switch-server`：passed。
- `cargo check -p wb-switch-rust`：passed。
- `npm run build`：passed。
- 新服务实测 `/api/status` 与 `/api/import-local`：分别返回 200；导入返回已有账号元数据。
- 完整 `cargo test -p wb-switch-core` 还有 3 个既有环境相关失败：Node 路径测试、扩展名大小写测试、Windows home 路径测试；与本次改动无关。
