# 注册与激活 Sora 协议流程摘要

本文档基于 `protocol_register.py` 整理，供「开启注册」对接时参考。具体实现以代码为准。

---

## 1. 域名与常量

- **chatgpt.com**：`CHATGPT_ORIGIN`，CSRF、signin、callback
- **auth.openai.com**：`AUTH_ORIGIN`，授权、注册、OTP、create_account、OAuth token
- **sora.chatgpt.com**：`SORA_ORIGIN`，激活 Sora（设置 username）
- OAuth：`client_id`、`redirect_uri` 从 Step2 的 authorize URL 解析；换 token 用 `auth.openai.com/oauth/token`

---

## 2. 注册流程（8 步）

| 步骤 | 说明 | 接口/动作 |
|------|------|-----------|
| **1/8** | 拿 CSRF | GET `chatgpt.com/api/auth/csrf`，取 `csrfToken`；可先 GET 首页再请求以降低 403 |
| **2/8** | 发起 signin | POST `chatgpt.com/api/auth/signin/openai`（form: callbackUrl, csrfToken, json=true），得到 302，Location 为 auth.openai.com 的 **authorize URL**；从该 URL 解析 `state`、`client_id`、`redirect_uri` |
| **3/8** | 跟 authorize | GET 上一步的 authorize URL（可加 `screen_hint=signup`），**跟到底**；落地页可能是 `create-account/password` 或 `email-verification`。若落地 `log-in`，则 **3b**：GET `auth.openai.com/create-account/password?state=xxx` 进入注册流 |
| （4 省略） | 按 HAR 分析，不调 authorize/continue，直接 Step5 | - |
| **5/8** | 声明注册 | POST `auth.openai.com/api/accounts/user/register`，body: `username`(邮箱)、`password`；Referer 带 state |
| **6/8** | 发邮箱验证码 | GET `auth.openai.com/api/accounts/email-otp/send`（会话已带上下文） |
| **7/8** | 校验验证码 | POST `auth.openai.com/api/accounts/email-otp/validate`，body: `code`；返回 `continue_url`，**follow 该 URL**；若某次 302 的 Location 含 `code=`（chatgpt callback），**不要跟这次请求**，直接拿该 URL 解析 `code` |
| **8/8** | 换 token 或 create_account | 若 7 已拿到 callback 的 `code` → 用 `code` 调 `auth.openai.com/oauth/token` 换 `access_token`/`refresh_token`，**注册完成**。否则 POST `auth.openai.com/api/accounts/create_account`，body: `name`、`birthdate`；成功后再跟 continue 可能得到 callback 含 `code`，同样换 token |

**两种分支**：

- **email-verification 先行**：Step3 落地 `email-verification` 时，先 6 发码 → 7 验码 → 再 4b（authorize/continue）→ 8 create_account。
- **常规**：Step3 落地 create-account → 5 user/register → 6 发码 → 7 验码 → 若 7 的 follow 已到 callback 则直接换 token；否则 8 create_account。

**OTP 来源**：当前实现由调用方提供 `get_otp_fn()`（如从邮箱/临时邮箱拉信取验证码）。Web 端可用 Hotmail007 拉信或接码 API 实现该回调。

---

## 3. Code 换 Token

- **URL**：`auth.openai.com/oauth/token`
- **Body**：`grant_type=authorization_code`、`code`、`redirect_uri`、`client_id`
- **返回**：`refresh_token`、`access_token`、`expires_in`  
保存 **refresh_token** 用于后续登录/刷新；Web 端写入 `accounts.refresh_token`。

---

## 4. 激活 Sora

- **输入**：注册得到的 `tokens`（含 `access_token` 或 `refresh_token`）、`email`
- **步骤**：  
  1. 若无 `access_token`，用 `refresh_token` 调 `auth.openai.com/oauth/token`（grant_type=refresh_token）换 `access_token`。  
  2. 从 `email` 生成 **username**：本地部分仅保留字母数字，截 14 位 + 6 位随机数，总长 ≤20。  
  3. POST `sora.chatgpt.com/backend/project_y/profile/username/set`，Header `Authorization: Bearer {access_token}`，Body `{"username": "xxx"}`。  
- **成功**：HTTP 200 即视为 Sora 已激活。

---

## 5. 调用约定（当前实现）

- **register_one_protocol(email, password, jwt_token, get_otp_fn, user_info)**  
  - `jwt_token`：临时邮箱 Worker 用（若用 Hotmail007 拉信可不必传）。  
  - `get_otp_fn`：无参可调用，返回 6 位验证码字符串，超时返回 None。  
  - `user_info`：至少 `name`、`year`、`month`、`day`（字符串）。  
  - 返回：`(email, password, success, status_extra, tokens)`；若 7 直接拿到 code 则 `tokens` 非空，否则 Step8 成功后需从 callback 再换 token（当前实现中部分分支在 8 之后未再换 token，需以代码为准）。

- **activate_sora(tokens, email)**  
  - 返回 True 表示设置 username 成功。

- **Session**：`_make_session()` 使用 curl_cffi 模拟 Chrome 指纹（若可用），代理从配置取；整轮注册到激活建议同一 session/代理，避免 IP 不一致。

---

## 6. 与 Web 端「开启注册」的衔接

- 从 **emails** 表取未注册邮箱，得到 `email`、`password`、`uuid`、`token`（Hotmail007 的 client_id / refresh_token）。
- **get_otp_fn**：用 Hotmail007 拉该邮箱最新邮件（或接码 API），解析验证码后返回。
- **代理**：从系统设置读 `proxy_url` 或 `proxy_api_url`，注入到 `_make_session()` 使用的配置。
- 注册成功并拿到 `refresh_token` 后：调用 `activate_sora(tokens, email)`；将 email、password、refresh_token、status、has_sora=1、has_plus=0、phone_bound=0 等写入 **accounts** 表；邮箱管理「已注册」由 accounts 存在性自动体现。

以上为协议流程摘要，实现细节以 `protocol_register.py` 为准。
