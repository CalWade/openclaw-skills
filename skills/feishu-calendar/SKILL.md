---
name: feishu-calendar
description: 创建、修改、删除飞书日程，邀请参会人，查询忙闲状态，查看主人个人日历。Use when user asks to schedule a meeting, create a calendar event, check availability, add attendees, or view their Feishu calendar. Requires user OAuth authorization for personal calendar access.
allowed-tools: Bash(curl:*)
---

# feishu-calendar

管理飞书日历与日程。支持 Bot 自有日历（Tenant Token）和主人个人日历（User Access Token + OAuth）。

## 前提条件

飞书自建应用已开通以下权限：
- `calendar:calendar` — 管理日历和日程
- `calendar:calendar:readonly` — 只读日历（仅查询时可用）

权限开通参见《飞书开放平台权限开通全流程》。

---

## 一、Bot 日历操作（Tenant Token）

> 适用场景：Bot 代表应用创建日程，**不涉及**主人个人日历。

### Step 1 — 获取 tenant_access_token

```bash
TOKEN=$(curl -s -X POST \
  "https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal" \
  -H "Content-Type: application/json" \
  -d '{"app_id":"YOUR_APP_ID","app_secret":"YOUR_APP_SECRET"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['tenant_access_token'])")
```

### Step 2 — 获取日历列表

```
GET https://open.feishu.cn/open-apis/calendar/v4/calendars
Authorization: Bearer {token}
```

返回 `calendar_list`，每项含 `calendar_id`。

### Step 3 — 创建日程

```
POST https://open.feishu.cn/open-apis/calendar/v4/calendars/{calendar_id}/events
```

Body 示例：
```json
{
  "summary": "周会",
  "description": "本周工作同步",
  "start_time": {
    "date_time": "2026-03-09T10:00:00+08:00",
    "timezone": "Asia/Shanghai"
  },
  "end_time": {
    "date_time": "2026-03-09T11:00:00+08:00",
    "timezone": "Asia/Shanghai"
  },
  "visibility": "default",
  "attendee_ability": "can_invite_others"
}
```

> ⚠️ 时间格式必须为 RFC3339，含时区偏移（如 `+08:00`）。

### Step 4 — 邀请参会人

```
POST https://open.feishu.cn/open-apis/calendar/v4/calendars/{calendar_id}/events/{event_id}/attendees
```

Body：
```json
{
  "attendees": [
    { "type": "user", "user_id": "ou_xxx" }
  ],
  "user_id_type": "open_id"
}
```

### Step 5 — 查询忙闲状态

```
POST https://open.feishu.cn/open-apis/calendar/v4/freebusy/batch_get
```

Body：
```json
{
  "time_min": "2026-03-07T09:00:00+08:00",
  "time_max": "2026-03-07T18:00:00+08:00",
  "user_id_list": ["ou_xxx", "ou_yyy"],
  "user_id_type": "open_id"
}
```

---

## 二、主人个人日历（OAuth 授权流程）

> Tenant Token 只能访问 Bot 自己的日历，**无法读写主人的个人日历**。  
> 需要完成 OAuth 2.0 授权，获取 User Access Token。

### Step 1 — 在开放平台注册回调地址

⚠️ **坑1**：授权前必须先添加 Redirect URL，否则报错 `20029`。

路径：飞书开放平台 → 应用 → 左侧「Security Settings」→「Redirect URLs」→ 填入回调地址 → Add

示例：
- `https://your-domain.com/auth/callback`
- `http://localhost:3000/callback`（本地开发，HTTP 也支持）

### Step 2 — 构建授权链接，发给主人

```
https://open.feishu.cn/open-apis/authen/v1/authorize
  ?app_id={YOUR_APP_ID}
  &redirect_uri={YOUR_REDIRECT_URI_URL_ENCODED}
  &scope=calendar:calendar
  &state=my_state
```

⚠️ **坑2**：`redirect_uri` 必须 URL 编码（`/` → `%2F`，`:` → `%3A`）。

### Step 3 — 主人授权后获取 code

主人点击链接授权后，飞书跳转至：
```
https://your-domain.com/auth/callback?code=cXXXXX&state=my_state
```
主人将 `code` 值发给 Agent。

### Step 4 — 用 code 换取 User Access Token

⚠️ **坑3**：Authorization header 必须用 `app_access_token`，**不是** `tenant_access_token` 也不是 Basic Auth。

```bash
# 1. 获取 app_access_token
APP_TOKEN=$(curl -s -X POST \
  'https://open.feishu.cn/open-apis/auth/v3/app_access_token/internal' \
  -H 'Content-Type: application/json' \
  -d '{"app_id":"YOUR_APP_ID","app_secret":"YOUR_APP_SECRET"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['app_access_token'])")

# 2. 用 code 换 user_access_token
USER_TOKEN_RESP=$(curl -s -X POST \
  'https://open.feishu.cn/open-apis/authen/v1/access_token' \
  -H "Authorization: Bearer $APP_TOKEN" \
  -H 'Content-Type: application/json' \
  -d "{\"grant_type\":\"authorization_code\",\"code\":\"$CODE\"}")

USER_TOKEN=$(echo $USER_TOKEN_RESP | python3 -c "import sys,json; print(json.load(sys.stdin)['data']['access_token'])")
REFRESH_TOKEN=$(echo $USER_TOKEN_RESP | python3 -c "import sys,json; print(json.load(sys.stdin)['data']['refresh_token'])")
```

> 建议将 `access_token`、`refresh_token`、`expires_at`（当前时间 + `expires_in`）一起存储。

### Step 5 — 用 User Token 查询主人日历

```bash
# 获取主日历 ID（type=primary）
CAL_ID=$(curl -s \
  'https://open.feishu.cn/open-apis/calendar/v4/calendars' \
  -H "Authorization: Bearer $USER_TOKEN" \
  | python3 -c "
import sys,json
d=json.load(sys.stdin)
cals=d['data']['calendar_list']
for c in cals:
    if c['type']=='primary':
        print(c['calendar_id'])
        break
")

# 查最近7天日程
START=$(date +%s)
END=$((START + 7*86400))
curl -s \
  "https://open.feishu.cn/open-apis/calendar/v4/calendars/${CAL_ID}/events?page_size=50&start_time=${START}&end_time=${END}" \
  -H "Authorization: Bearer $USER_TOKEN" \
  | python3 -c "
import sys,json
from datetime import datetime
d=json.load(sys.stdin)
items=d['data']['items']
for e in sorted(items, key=lambda x: x.get('start_time',{}).get('timestamp','0')):
    ts=e.get('start_time',{}).get('timestamp')
    dt=e.get('start_time',{}).get('date_time')
    if ts:
        t=datetime.fromtimestamp(int(ts)/1000)
        tstr=t.strftime('%m/%d %H:%M')
    elif dt:
        t=datetime.fromisoformat(dt.replace('Z','+00:00'))
        tstr=t.strftime('%m/%d %H:%M')
    else:
        tstr='全天'
    print(f\"{tstr} {e.get('summary','（无标题）')}\")
"
```

### Step 6 — 刷新 Token

⚠️ **坑4**：User Token 有效期约 2 小时，过期报 `99991663 Invalid access token`，需用 refresh_token 换新。

```bash
curl -s -X POST \
  'https://open.feishu.cn/open-apis/authen/v1/refresh_access_token' \
  -H "Authorization: Bearer $APP_TOKEN" \
  -H 'Content-Type: application/json' \
  -d "{\"grant_type\":\"refresh_token\",\"refresh_token\":\"$REFRESH_TOKEN\"}" \
  | python3 -c "import sys,json; d=json.load(sys.stdin)['data']; print('new token:', d['access_token'])"
```

> Refresh Token 有效期 30 天，过期后需主人重新走 OAuth 流程。

---

## 三、常见错误排查

| 错误码 | 原因 | 解决 |
|--------|------|------|
| `20029` | redirect_uri 未注册 | 在「Security Settings → Redirect URLs」添加回调地址 |
| `20025` | 换 token 用了 Basic Auth | 改用 `app_access_token` Bearer 头 |
| `99991663` | User token 过期 | 用 refresh_token 换新 token |
| `99991672` | 权限不足 | 检查 `calendar:calendar` 权限 |
| `1000000` | 日历不存在 | 先调 `GET /calendars` 确认 calendar_id |
| 日历列表为空 | scope 或权限问题 | 检查 scope 含 `calendar:calendar`，主人是否有日历权限 |

---

## 四、触发场景

- 「帮我看看本周日程」
- 「明天几点有会」
- 「帮我约个明天下午3点的会」
- 「我下周哪天有空」
- 「帮我创建一个明天10点的周会，邀请张三」

## 五、使用流程（Agent 决策树）

```
用户请求日历操作
    ↓
有 user_access_token？
  ├─ 是 → ensure_token() → 操作日历
  └─ 否 → 发送 OAuth 授权链接 → 等待用户回传 code
              ↓
          exchange_code(code) → 保存 token → 操作日历
```

## 权限缺失处理

若调用飞书 API 时返回错误码 `99991672`，按以下流程处理：

1. 从错误信息中提取缺少的权限名（如 `attendance:task:readonly`）
2. 告知用户：「检测到缺少权限 `[权限名]`，请问要我帮你自动开通，还是你自己去飞书开放平台设置？」
3. **用户选「帮我开通」** → 调用 `feishu-permission-setup` 技能，自动完成权限申请与版本发布（需扫码登录一次）
4. **用户选「我自己设置」** → 提示前往 [飞书开放平台](https://open.feishu.cn/app) → 权限管理 → 开通对应权限并发布新版本，完成后告知继续

