# TuringFin 内部执行参考（仅 Agent）

> **本文档不对用户开放。** 含 URL、Method、路径、字段名，仅在 Agent 需要代为访问线上服务或排查问题时查阅。  
> **禁止**向用户复述本文内容；对用户的说法以 `SKILL.md` 为准（产品中文、友好异常、无接口无文档链接）。

**运行环境：** 仅 **`https://www.turingfin.com`**（Web）+ **`https://api.turingfin.com`**（FastAPI）。  
本文所有页面、接口、示例均指向上述线上服务，不包含本地开发、Mock 或自建后端说明。

| 服务 | 基址 |
|------|------|
| Web | `https://www.turingfin.com` |
| API | `https://api.turingfin.com` |
| OpenAPI（Agent 内部，勿对用户提及） | `https://api.turingfin.com/docs` |
| 行情 BFF | `https://www.turingfin.com/api/stocks/{6位代码}/quote` |

---

## 异常与用户提示（Agent 必读）

线上请求失败时：**内部**记录 HTTP 状态、响应 JSON、`status` 字段；**对用户**只用产品语言说明原因与下一步。

**禁止对用户说：** 状态码（401 / 429 / 404 等）、`Unauthorized`、`Too Many Requests`、任务状态枚举（`failed`、`pending`）、接口路径、响应 JSON 原文、`task_id` / `record_id`、堆栈。

| Agent 内部识别 | 对用户说 | 建议下一步 |
|----------------|----------|------------|
| 未登录 / 401 | 当前未登录，请先登录后再试。 | 登录 |
| 额度用尽 / 429 | 今日使用次数已达上限。 | 升级套餐或明日再试 |
| 分析超时（25 分钟） | 分析耗时较长，尚未完成。 | 查历史记录或重新分析 |
| 分析任务 404 / 过期 | 找不到这次分析记录，可能已过期。 | 重新发起分析 |
| 分析 `failed` / `error` | 本次分析未能完成。 | 换标的或稍后再试 |
| 对话流失败 / 429 | 消息未能发送；或今日选股次数已用完。 | 重试或明日再试 |
| 行情 502 | 行情暂时无法获取。 | 稍后再试 |
| 行情 400 / 404 | 暂仅支持 A 股；或暂时查不到该股票行情。 | 检查代码 |
| 自选重复 400 | 该股票已在自选列表中。 | — |
| 登录失败 / 密码错误 | 账号或密码不正确。 | 检查后重试 |
| 登录过期 / refresh 失败 | 登录状态已失效，请重新登录。 | 重新登录 |
| 微信未配置 | 微信登录暂不可用，请使用账号密码登录。 | 改用账号密码 |
| 微信二维码失败 | 二维码加载失败，请刷新页面后重试。 | 刷新页面 |
| 微信取消授权 | 你已取消微信授权，可重新扫码。 | 重新扫码 |
| 注册失败 | 注册未成功，请检查填写信息后重试。 | 检查表单 |
| 5xx / 网络错误 | 服务暂时繁忙。 | 稍后再试 |
| 其它 | 操作未能完成。 | 稍后再试 |

各功能章节下的错误表仅供 Agent 对照；对用户只说 `SKILL.md`「异常时对用户说什么」表中的中文。

---

## 登录与退出

对用户的说法见 `SKILL.md`「登录与退出」；本节供 Agent 联调与排错。

### 能力 vs 登录态

| 能力 | 未登录 | 已登录 |
|------|--------|--------|
| `POST /api/analyze`、分析历史 | 401 | Bearer |
| `POST /api/dialogue/stream` | 401 | Bearer |
| `GET /api/stocks/search` | 空列表 | 正常 |
| `GET/POST/DELETE /api/stocks/portfolio` | 401 | Bearer |
| `GET www.../api/stocks/{code}/quote` | 401 | Bearer |
| 自选 UI | `localStorage` `app-watchlist-v2` | 云端 `portfolio` 为准 |

### Web 用户流程（与线上一致）

```text
/login 或 /login?next=/app/...
  ├─ 账号密码（默认）
  │    账号：11 位手机号 或 用户名 /^[a-zA-Z0-9_-]{3,50}$/
  │    密码：8～16 位
  │    须勾选法律条款 → POST /api/auth/login → 存 token → redirect next（默认 /app/analyze）
  ├─ 微信 ?mode=wechat
  │    GET /api/auth/wechat/qrcode → 展示扫码（约 120s 过期自动 refresh）
  │    回调带 code → POST /api/auth/wechat/login?code= → 存 token → redirect
  └─ 注册 /register
       POST /api/auth/register?grant_tokens=true（优先）→ 直接带 token 或再 password login
```

已登录用户访问 `/login`：`session===user` 时自动 `replace` 到 `next` 或 `/app/analyze`。

会话持久化：Zustand `persist` 键 `zhputian-auth`（`accessToken`、`refreshToken`、`user`）。  
`authenticatedFetch`：401 时用 `refresh_token` 调 `POST /api/auth/refresh`，失败则清空会话并抛「登录已过期，请重新登录」（对用户改为 SKILL 友好文案）。

**无短信登录 API**；登录页重置密码对话框存在，短信为本地/mock 流程，不以短信接口为准。

### 接口

| 步骤 | 请求 |
|------|------|
| 密码登录 | `POST https://api.turingfin.com/api/auth/login`（`username`+`password`，form 或 JSON） |
| 注册 | `POST https://api.turingfin.com/api/auth/register`（`username`,`email`,`password`,`phone?`；`grant_tokens=true` 时直接返回 token） |
| 微信二维码 | `GET https://api.turingfin.com/api/auth/wechat/qrcode` |
| 微信换 token | `POST https://api.turingfin.com/api/auth/wechat/login?code=` |
| 刷新 token | `POST https://api.turingfin.com/api/auth/refresh` body `{ "refresh_token" }` |
| 当前用户 | `GET https://api.turingfin.com/api/users/me` 或 `/api/auth/me`，Bearer |
| 退出 | `POST https://api.turingfin.com/api/auth/logout` Bearer + `{ "refresh_token" }` |
| 全设备退出 | `POST https://api.turingfin.com/api/auth/logout/all` Bearer |

登录成功响应含：`access_token`、`refresh_token`、`token_type`、`user`。

### 退出（Web）

入口：`https://www.turingfin.com/app/account` → **退出登录**。

`use-auth-store.logout`：有 token 则 `POST /api/auth/logout`；`finally` 清空会话并 `router.push("/")`。

### 登录异常 → 对用户文案（勿暴露 HTTP）

| 内部 | 对用户说 |
|------|----------|
| 401 未登录 | 当前未登录，请先登录后再试 |
| 401 刷新失败 | 登录状态已失效，请重新登录 |
| 密码错误 | 账号或密码不正确 |
| 注册失败 | 注册未成功，请检查填写信息后重试 |
| 微信未配置 / 503 | 微信登录暂不可用，请使用账号密码登录 |
| 二维码加载失败 | 二维码加载失败，请刷新页面后重试 |
| 微信取消授权 | 你已取消微信授权，可重新扫码 |
| state 校验失败 | 登录未完成，请返回登录页重新扫码 |

---

## 功能总览

| 功能 | 页面 | API 域 |
|------|------|--------|
| 股票分析 | `https://www.turingfin.com/app/analyze`、`/app/history` | `api.turingfin.com` |
| 对话选股 | `https://www.turingfin.com/app/pick` | `POST /api/dialogue/stream`（SSE） |
| 股票搜索 | 分析/自选搜索框 | `GET /api/stocks/search` |
| 自选股票 | `https://www.turingfin.com/app/watchlist` | `GET/POST/DELETE /api/stocks/portfolio` |
| 行情查询 | 分析/自选列表 | `www` BFF `/api/stocks/{code}/quote` |

```text
搜索(api) → 选代码 → 行情(www BFF) → 分析(api 异步)
自选：列表(portfolio) + 搜索 + A股行情(BFF) → 可跳转分析
选股(www: D1-D8 本地 → 候选本地 → stream 对话) → 分析(api 异步)
```

---

## 1. 股票分析

### 用户路径（C 端，与 `SKILL.md` 一致）

**配置阶段（`AnalyzeRunConfigDialog`「分析配置」）**

| 项 | UI | 默认 / 约束 |
|----|-----|-------------|
| 标的 | 搜索标的 | 必填；`StockSearchCombobox`，可 `CN.600519` |
| 分析日期 | DatePicker | 默认当天 |
| 分析深度 | 一级～五级单选 | 默认 **3**（标准）；耗时见 `ANALYZE_DEPTH_SUMMARY` |
| 分析师团队 | 四角色多选 | 默认 market+fundamental；**至少 1 人**；A 股禁用 social |
| 情绪分析 | Switch | 默认开 |
| 风险评估 | Switch | 默认开 |
| 报告语言 | zh / en | 默认中文 |
| 算力预览 | 侧栏 | 随深度与团队变化 |

持久默认：`useAnalyzePreferencesStore` + **设置 → 分析设置**（含 quickModel/deepModel，写入 preference themes）。弹窗提交时与 store 同步。

**提交后进度（Web UX）**

1. 按钮变为「分析中」；页内展示进度条 + `progress`% + `progress_message`（对用户原样转述中文阶段文案）。  
2. `localStorage` `analyze-pending-task-v1` 记录 task；**每 5s** `GET /api/analyze/{task_id}`，最长客户端等待 **25min**。  
3. 可离开分析页：`GlobalAnalyzeTaskWatcher` 后台继续；完成弹窗「分析任务已完成」→「去查看」`?record_id=`。  
4. 顶栏 **分析任务**：多任务时「进行中 / 排队中」+ 进度%。  
5. 完成：报告渲染或历史 `record_id`；失败：友好文案，勿贴 `error` JSON。

### Agent 技术路径

1. 打开 `https://www.turingfin.com/app/analyze`，或代用户 `POST /api/analyze`（body 含 depth、分析师开关等）。
2. 拿到 `task_id`、`record_id`。
3. 每 **5s** `GET /api/analyze/{task_id}` 直至完成/失败/超时。
4. `result` 渲染报告；历史 `GET /api/analyze/history/{record_id}`。

### 接口

| 操作 | Method | URL |
|------|--------|-----|
| 创建任务 | POST | `https://api.turingfin.com/api/analyze` |
| 轮询状态 | GET | `https://api.turingfin.com/api/analyze/{task_id}` |
| 历史列表 | GET | `https://api.turingfin.com/api/analyze/history` |
| 历史单条 | GET | `https://api.turingfin.com/api/analyze/history/{record_id}` |
| 导出报告 | GET | `https://api.turingfin.com/api/analyze/report?ticker=000001&format=markdown` |

**POST body 示例：**

```json
{
  "ticker": "000001",
  "depth": 3,
  "market_analyst": true,
  "fundamental_analyst": true,
  "news_analyst": false,
  "social_analyst": false,
  "sentiment_analysis": true,
  "risk_assessment": true
}
```

**创建成功响应示例：**

```json
{
  "task_id": "550e8400-e29b-41d4-a716-446655440000",
  "record_id": "660e8400-e29b-41d4-a716-446655440001",
  "message": "分析任务已创建，请通过 /analyze/{task_id} 接口轮询获取结果"
}
```

### 轮询协议（必读）

分析是**异步任务**，禁止用已废弃的同步 `GET https://api.turingfin.com/api/analyze?ticker=...`。

#### 推荐节奏

| 项 | 值 |
|----|-----|
| 轮询间隔 | **5 秒**（与线上 Web 一致） |
| 客户端最长等待 | **25 分钟**（超时后提示重试） |
| 服务端任务缓存 | Redis 约 **1 小时**（过期后 `GET` 可能 404） |

#### 轮询流程

```text
POST /api/analyze  →  task_id, record_id
        ↓
   [立即] GET /api/analyze/{task_id}
        ↓
   pending / processing  →  等待 5s  →  再 GET
        ↓
   completed + result    →  结束，解析 result
   failed 或 error 字段   →  结束，展示错误
   超过 25 分钟仍未完成   →  按超时处理
```

#### 任务状态（后端字段 `status`，小写）

| status | 含义 | 下一步 |
|--------|------|--------|
| `pending` | 已入队，等待执行 | 继续轮询 |
| `processing` | 执行中 | 继续轮询，读 `progress` / `progress_message` |
| `completed` | 成功 | 读取 `result`，结束轮询 |
| `failed` | 失败 | 读取 `error`，结束轮询 |

兼容判断（与线上 Web 客户端一致）：`status` 为 `completed` / `success` / `done`（大小写不敏感），或响应里已有非空 `result`，视为完成；`failed` / `error` / `cancelled` 或顶层 `error` 非空，视为失败。

#### 轮询响应字段

| 字段 | 说明 |
|------|------|
| `task_id` | 任务 ID |
| `record_id` | 分析记录 ID（创建任务时即生成，可用于历史跳转） |
| `status` | 见上表 |
| `progress` | 0–100 整数进度 |
| `progress_message` | 当前阶段文案（如「正在并行运行分析师…」） |
| `result` | 完成后的分析报告 JSON；未完成时为 `null` |
| `error` | 失败原因；成功时一般为 `null` |

**进行中响应示例：**

```json
{
  "task_id": "550e8400-e29b-41d4-a716-446655440000",
  "record_id": "660e8400-e29b-41d4-a716-446655440001",
  "status": "processing",
  "progress": 50,
  "progress_message": "正在进行多空辩论...",
  "result": null,
  "error": null
}
```

**完成响应示例：**

```json
{
  "task_id": "550e8400-e29b-41d4-a716-446655440000",
  "record_id": "660e8400-e29b-41d4-a716-446655440001",
  "status": "completed",
  "progress": 100,
  "progress_message": "分析完成",
  "result": { "final_signal": "HOLD", "stock_info": { "code": "000001", "name": "平安银行" } },
  "error": null
}
```

#### 异常处理（Agent 内部 → 用户提示见上文总表）

| 场景 | Agent 内部 | Agent 动作 | 对用户说 |
|------|------------|------------|----------|
| 未登录 | 401 | 停止轮询 | 当前未登录，请先登录后再试 |
| 额度用尽 | 429 | 停止轮询 | 今日使用次数已达上限；可升级套餐或明日再试 |
| 任务不存在 | 404 | 停止轮询 | 找不到这次分析记录，可能已过期；请重新发起分析 |
| 查询失败 | 4xx/5xx | 5xx 可间隔重试 | 服务暂时繁忙，请稍后再试 |
| 超时 | 25 分钟无完成 | 结束等待 | 分析耗时较长尚未完成；请稍后在历史记录查看或重新分析 |
| 结果带错 | `result.error` 非空 | 按失败处理 | 本次分析未能完成；请换标的或稍后再试 |

#### Agent 伪代码（直接调线上 API）

```text
r = POST https://api.turingfin.com/api/analyze  (Bearer + JSON body)
task_id = r.task_id
deadline = now + 25min
loop:
  if now > deadline → 报错「分析任务超时」
  t = GET https://api.turingfin.com/api/analyze/{task_id}
  if t.error 非空 → 报错
  if t.status in (completed) 或 t.result 非空 → 使用 t.result，退出
  if t.status in (failed) → 报错
  sleep 5s
```

#### 与 Web 产品的关系（说明用，非实现任务）

- 用户在分析页点击「开始分析」时，客户端会在同一会话内完成「创建 + 轮询」直至出报告。
- 任务 ID 会写入浏览器 `localStorage`（键 `analyze-pending-task-v1`），刷新页面或离开分析页后，仍由页面或全局组件每 5 秒查询一次任务状态。
- 完成后用 `record_id` 跳转 `https://www.turingfin.com/app/analyze?record_id=...` 查看报告。

### 结果映射

后端 `BUY` / `SELL` / `HOLD` → 前端展示 `add` / `reduce` / `wait`。

**额度：** 创建任务时若会员额度不足，内部为 429；对用户说「今日使用次数已达上限」。

---

## 2. 对话选股

页面：`https://www.turingfin.com/app/pick`

**发消息接口（唯一）：** `POST https://api.turingfin.com/api/dialogue/stream`（SSE 流式）。  
`/api/dialogue` 仅为路由前缀，**不存在** `POST /api/dialogue` 根路径。

选股产品是 **「本地偏好向导 + 流式 AI 对话 + 本地候选演示」** 的组合，不是单一后端筛股 API。

### 产品流程总览

```text
进入 /app/pick
    ├─ 「随便问问」→ dialogueMode=direct → 自由输入 → POST /api/dialogue/stream (mode=direct)
    └─ 「帮我选股」→ dialogueMode=prompt
            ├─ D1～D8 按钮向导（纯前端，不调 API）→ 写入 PreferenceSnapshot
            ├─ 「生成候选」→ 前端本地生成演示候选列表（不调 API）
            ├─ 自由输入 / 点追问 chip → POST /api/dialogue/stream (mode=prompt)
            └─ 选候选标的 → 打开分析配置 → POST /api/analyze（见「股票分析」）
```

| 阶段 | 是否调 API | 说明 |
|------|------------|------|
| D1–D8 偏好向导 | 否 | 点选按钮更新 `PreferenceSnapshot` |
| 生成候选 | 否 | 前端按市场生成演示 `candidate_stocks` |
| 用户发文字 / 点 `extension_questions` | 是 | `POST /api/dialogue/stream` |
| 候选 → 单票分析 | 是 | `POST /api/analyze` + 轮询 |

### 两种对话模式

| 入口 | `mode` | 用途 |
|------|--------|------|
| 「随便问问」 | `direct` | 咨询市场、术语、策略；无 D1–D8 向导 |
| 「帮我选股」及之后自由对话 | `prompt` | 选股语境；服务端带选股提示词 |

同一 `session_id` 应在多轮 `stream` 请求中保持不变，以延续上下文。

### D1–D8 偏好向导（纯前端）

向导通过页面按钮驱动，**不调用** `/api/dialogue/*`。结果写入前端结构 `PreferenceSnapshot`：

| 步骤 | 字段 | 可选值 / 说明 |
|------|------|----------------|
| D1 交易市场 | `market` | `CN` / `HK` / `US` |
| D2 行业范围 | `sector_mode`, `sectors` | `unrestricted` 或 `specified` + GICS 一级行业中文名数组 |
| D4 持有周期 | `holding_horizon` | `intraday_to_days` / `w1_to_w4` / `m1_to_m3` / `m3_plus` |
| D5 风格 | `style` | `value` / `growth` / `momentum` / `no_preference` |
| D6 风险档 | `risk_tier` | `conservative` / `balanced` / `aggressive`（与择时分析一致） |
| D7 市值流动性 | `cap_liquidity` | `unrestricted` / `large_mid_liquid` / `small_volatile_ok` |
| D8 排除规则 | `exclusions` | `exclude_st` / `exclude_illiquid` / `exclude_high_leverage`（可多选） |
| D3 主题叠加 | `themes` | 如「红利/高股息」「新能源产业链」「人工智能与硬科技」（可选多项） |

实际顺序：**D1 → D2 → D4 → D5 → D6 → D7 → D8 → D3 → 完成偏好**。  
完成后 `conversation_phase` 为 `ready_to_screen`，可点「生成候选」。

`PreferenceSnapshot` 完整字段：

```json
{
  "market": "CN",
  "sector_mode": "unrestricted",
  "sectors": [],
  "themes": [],
  "holding_horizon": "m1_to_m3",
  "style": "no_preference",
  "risk_tier": "balanced",
  "cap_liquidity": "large_mid_liquid",
  "exclusions": [],
  "other_notes": null
}
```

**与后端区别：** 后端 dialogue 会话另有 `criteria`（正则从对话抽取），**不等于** 上表 `PreferenceSnapshot`；勿混用。

### 候选列表（纯前端演示）

点「生成候选」后，`candidate_stocks` 由前端按 `market` 与偏好生成（如 A 股示例 600519、601318），**没有**独立的「筛股」HTTP 接口。

每条候选含：`code`、`name`、`reason`、`snapshot_keys`（对应 D 维度说明）。

`conversation_phase` 变为 `candidates_shown` 后，用户点某标的 → 弹出「分析配置」→ 携带当前 `preference_snapshot` 作为 `AnalysisInput.preference_snapshot` → 调用分析 API。

### 跳转分析

从候选进入分析时：

1. `POST https://api.turingfin.com/api/analyze`（body 含 `ticker`、分析师开关等，偏好来自 `PreferenceSnapshot` 映射）。
2. 按「轮询协议」`GET /api/analyze/{task_id}` 直至完成。
3. 页面跳转：`https://www.turingfin.com/app/analyze?stockCode=CN.600519`（示例）。

### 相关接口

| 操作 | Method | URL |
|------|--------|-----|
| **发消息** | POST | `https://api.turingfin.com/api/dialogue/stream` |
| 会话历史 | GET | `https://api.turingfin.com/api/dialogue/history?session_id=` |
| 清除历史 | DELETE | `https://api.turingfin.com/api/dialogue/history?session_id=` |

### `/api/dialogue/stream` 流式协议（必读）

#### 请求

```http
POST https://api.turingfin.com/api/dialogue/stream?message={用户输入}&session_id={会话ID}&mode=prompt
Authorization: Bearer <access_token>
```

| 查询参数 | 必填 | 说明 |
|----------|------|------|
| `message` | 是 | 用户本轮文本 |
| `session_id` | 否 | 会话 ID；同一选股对话应固定传递以延续上下文 |
| `mode` | 否 | `prompt`（选股对话，带提示词，**默认**）或 `direct`（无提示词直聊） |

请求体为空（参数均在 Query）。

#### 响应

| 项 | 值 |
|----|-----|
| `Content-Type` | `text/event-stream` |
| 格式 | SSE（Server-Sent Events） |
| 事件分隔 | 空行 `\n\n` 分隔多条 `data:` 行 |

每条事件一行（或多行 `data:`），JSON 载荷形如：

```text
data: {"chunk":"片段文字","extension_questions":[]}

```

| JSON 字段 | 说明 |
|-----------|------|
| `chunk` | 本轮回复的**增量**文本，需**顺序拼接**得到完整 `response` |
| `extension_questions` | 追问建议（字符串数组）；可能在流末尾事件中给出，以**最后一次非空**为准 |

#### 客户端处理流程

```text
POST /api/dialogue/stream  (Bearer + Query)
        ↓
读取 response.body 流
        ↓
按 \n\n 切分 SSE 事件块 → 解析 data: 行 JSON
        ↓
累加 chunk → 完整助手回复
        ↓
流结束 → 使用 extension_questions（若有）
```

#### Agent 伪代码

```text
full = ""
questions = []
POST stream?message=...&session_id=...&mode=prompt
for each SSE event:
  parse JSON
  full += chunk
  if extension_questions 非空 → questions = extension_questions
return { response: full, extension_questions: questions, session_id }
```

#### 异常处理（Agent 内部 → 用户提示见总表）

| 场景 | Agent 内部 | Agent 动作 | 对用户说 |
|------|------------|------------|----------|
| 未登录 | 401 | 停止发送 | 当前未登录，请先登录后再试 |
| 额度用尽 | 429 | 停止发送 | 今日选股次数已达上限；可升级套餐或明日再试 |
| 请求失败 | 非 2xx / 无 body | 记录并重试一次 | 消息未能发送成功，请检查网络后重试 |
| 参数无效 | `message` 为空 | 不发起请求 | — |

#### 流式结束后的 UI 行为

- 将 `extension_questions` 转为页面可点的追问按钮（`extq_*`）；用户点击等价于再发一条 `message`。
- `mode=prompt` 时展示追问；`mode=direct` 时常不展示。
- 脚本状态记为 `backend_dialogue`（表示本轮回复来自流式接口）。

#### 额度与登录

- 未登录无法调用 `stream`（需 Bearer token）。
- 「帮我选股」会消耗选股会话额度；用尽时内部为 429，对用户提示额度已用完。
- 访客与会员策略以线上为准。

#### 其它说明

- `POST /api/dialogue/sync` 存在但 **不属于** Web 选股主流程。
- `GET /api/dialogue/history` 可拉取服务端保存的多轮 `history`；与 `stream` 发消息配合。
- 重置对话会生成新的 `session_id`（客户端 UUID）。

---

## 3. 行情查询

行情由 **Web 站点 BFF** 提供，**不**走 `api.turingfin.com`。

| 操作 | Method | URL |
|------|--------|-----|
| A 股报价 | GET | `https://www.turingfin.com/api/stocks/{6位代码}/quote` |

查询参数：`?fresh=1` 强制跳过缓存。

约束：仅沪深 **6 位 A 股**代码。

| Agent 内部 | 对用户说 |
|------------|----------|
| 401 | 当前未登录，请先登录后再试 |
| 400 | 暂仅支持沪深 A 股六位代码 |
| 404 | 暂时查不到该股票行情 |
| 502 | 行情暂时无法获取，请稍后再试 |

---

## 4. 股票搜索

| 操作 | Method | URL |
|------|--------|-----|
| 关键词搜索 | GET | `https://api.turingfin.com/api/stocks/search?keyword=平安&limit=6` |

未登录时接口返回空列表。标的库由服务端 `stock_info.csv` 维护。

---

## 5. 自选股票

页面：`https://www.turingfin.com/app/watchlist`（侧栏「自选」）

管理用户关注的标的列表；**列表增删走 FastAPI**，**搜索与 A 股行情**与其它页面共用能力。

### 重要：接口路径

Web 自选使用 **`/api/stocks/portfolio`**（非 `/api/stocks/watchlist`）。

后端虽另有 `watchlist` 路由，**线上 Web 以 `portfolio` 为准**。

### 用户路径

1. 打开自选页，登录用户自动 `GET` 同步云端列表。
2. 顶部搜索：模糊匹配 → 点选加入，或回车按输入解析加入。
3. 列表展示标的；**A 股（CN）** 展示现价与涨跌幅（东方财富快照，www BFF）。
4. 卡片菜单：**分析** → `/app/analyze?stockCode=CN.600519`；**删除** → 从自选移除。

未登录时列表存于浏览器 `localStorage`（`app-watchlist-v2`），登录后以云端为准。

### 自选列表 API（`api.turingfin.com`）

| 操作 | Method | URL |
|------|--------|-----|
| 获取列表 | GET | `https://api.turingfin.com/api/stocks/portfolio` |
| 添加 | POST | `https://api.turingfin.com/api/stocks/portfolio` |
| 删除 | DELETE | `https://api.turingfin.com/api/stocks/portfolio/{stock_code}` |

均需 `Authorization: Bearer <token>`。

**GET 响应示例：**

```json
{
  "count": 2,
  "stocks": [
    {
      "id": 1,
      "stock_code": "600519",
      "stock_name": "贵州茅台",
      "exchange": "SH",
      "market": "CN",
      "added_date": "2026-05-18T10:00:00"
    }
  ]
}
```

**POST body 示例：**

```json
{
  "stock_code": "600519",
  "stock_name": "贵州茅台",
  "market": "CN",
  "exchange": "SH"
}
```

成功：`{ "success": true, "message": "股票已添加到自选股", "data": { ... } }`  
重复添加：内部 400；对用户说「该股票已在自选列表中」。

**DELETE：** 路径参数为 **6 位代码**（如 `600519`），不含市场前缀。

### 共用能力

| 能力 | 接口 | 说明 |
|------|------|------|
| 搜索加入 | `GET https://api.turingfin.com/api/stocks/search?keyword=&limit=6` | 与分析页相同 |
| A 股行情 | `GET https://www.turingfin.com/api/stocks/{6位代码}/quote` | 见「行情查询」；`?fresh=1` 强制刷新 |
| 跳转分析 | — | `https://www.turingfin.com/app/analyze?stockCode={market}.{symbol}` |

### 行情展示范围

| 市场 | 自选列表行情 |
|------|----------------|
| `CN` A 股 | 有（BFF + 客户端短缓存，约 30 分钟） |
| `HK` / `US` | 暂不展示现价（显示「未接入」） |

### 其它入口

分析页、选股页候选卡片也可 **加自选**，调用同一套 `POST /api/stocks/portfolio`。

---

## 接口速查（Agent 内部，全量线上 URL）

| 能力 | URL |
|------|-----|
| 登录 | `POST https://api.turingfin.com/api/auth/login` |
| 退出 | `POST https://api.turingfin.com/api/auth/logout` |
| 全设备退出 | `POST https://api.turingfin.com/api/auth/logout/all` |
| 刷新 | `POST https://api.turingfin.com/api/auth/refresh` |
| 分析创建 | `POST https://api.turingfin.com/api/analyze` |
| 分析轮询 | `GET https://api.turingfin.com/api/analyze/{task_id}` |
| 分析历史 | `GET https://api.turingfin.com/api/analyze/history` |
| 分析导出 | `GET https://api.turingfin.com/api/analyze/report?ticker=&format=` |
| 选股对话 | `POST https://api.turingfin.com/api/dialogue/stream?message=&session_id=&mode=prompt` |
| 搜索 | `GET https://api.turingfin.com/api/stocks/search` |
| 自选列表 | `GET/POST https://api.turingfin.com/api/stocks/portfolio` |
| 自选删除 | `DELETE https://api.turingfin.com/api/stocks/portfolio/{stock_code}` |
| 行情 | `GET https://www.turingfin.com/api/stocks/{code}/quote` |

---

## 常见踩坑

1. **域名拆分：** 分析 / 搜索 / 选股 / 登录 → `api.turingfin.com`；行情 → `www.turingfin.com`。
2. **分析轮询：** `POST` 创建 → 每 **5s** `GET /api/analyze/{task_id}`，最长等 **25 分钟**；见「轮询协议」。
3. **选股：** D1–D8 / 生成候选为**前端本地**；仅自由对话走 **`POST /api/dialogue/stream`**；无筛股 API、无 `POST /api/dialogue` 根路径。
4. **认证：** 业务与行情 BFF 均需有效 Bearer token（搜索未登录返回空）。
5. **自选 API：** Web 用 **`/api/stocks/portfolio`**，勿与 `/api/stocks/watchlist` 混用。
6. **退出：** 须同时提交 `refresh_token`；退出后勿继续用旧 Bearer 调业务 API。
7. **异常提示：** 内部分辨 401 / 429 等；对用户统一用「异常与用户提示」表，勿暴露状态码。

---

## 不在本 skill 范围

后台管理、小程序、桌面客户端、纸面交易、营销静态页（如站内导航「产品介绍」`/welcome`，无 API，不参与联调）。未列出的端点由 Agent 在 OpenAPI 内部自查，勿引导用户查看文档。
