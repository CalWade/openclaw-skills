# feishu-calendar（优化版 v2）

管理飞书日历与日程。支持 Bot 自有日历（Tenant Token）和主人个人日历（User Access Token + OAuth）。

## 前提条件

- `calendar:calendar` — 管理日历和日程
- `calendar:calendar:readonly` — 只读日历

权限未开通时返回 99991672，执行 feishu-permission-setup 技能自动开通。

## 一、Bot 日历操作（Tenant Token）

适用场景：Bot 代表应用创建日程，邀请用户参加，日程出现在用户个人日历上。**不需要 OAuth，是最简单的方式。**

### 获取 Bot 日历 ID

`GET /calendar/v4/calendars` → 找 `type=primary` 的 `calendar_id`。

### 创建日程

`POST /calendar/v4/calendars/{calendar_id}/events`

### ⚠️ 关键坑：时间戳必须是字符串 + 正确年份

推荐用 timestamp（字符串）而非 date_time，避免时区格式出错：

```json
{
  "summary": "日程测试",
  "start_time": { "timestamp": "1773028800" },
  "end_time": { "timestamp": "1773032400" },
  "visibility": "private"
}
```

**踩坑记录：**

1. timestamp 字段是**字符串类型**（`"1773028800"`），不是整数
2. 必须用**当前真实年份**计算！Agent 容易把 2026 算成 2025，务必用代码验证：

```python
from datetime import datetime, timezone, timedelta
tz = timezone(timedelta(hours=8))
dt = datetime(2026, 3, 9, 12, 0, 0, tzinfo=tz)
print(int(dt.timestamp()))  # → 1773028800
```

如果用 date_time 格式，必须严格 RFC3339 + 时区偏移：`"2026-03-09T12:00:00+08:00"`

### 邀请参会人（让日程出现在用户日历上）

💡 不走 OAuth 就能让用户看到日程的方法：Bot 创建日程 → 加 attendee → 日程出现在用户日历。

`POST /calendar/v4/calendars/{calendar_id}/events/{event_id}/attendees?user_id_type=open_id`

```json
{ "attendees": [{ "type": "user", "user_id": "ou_xxxxx" }] }
```

设 `visibility=private` 则只有被邀请人可见。

### 查询忙闲

`POST /calendar/v4/freebusy/batch_get`（Tenant Token 即可，无需 OAuth）

### 修改 / 删除日程

- `PATCH /calendar/v4/calendars/{cal}/events/{event}` — 修改
- `DELETE /calendar/v4/calendars/{cal}/events/{event}` — 删除

---

## 二、主人个人日历（OAuth）

仅当需要**读取**用户已有日程时才走 OAuth。创建日程用 Bot + attendee 即可。

### 四个坑

| #    | 坑                     | 说明                                                         |
| ---- | ---------------------- | ------------------------------------------------------------ |
| 1    | redirect_uri 未注册    | 必须先在 Security Settings → Redirect URLs 添加，否则报 20029 |
| 2    | redirect_uri 未编码    | 必须 URL encode                                              |
| 3    | 换 token 用错了 header | 必须用 app_access_token，不是 tenant_access_token            |
| 4    | User Token 2小时过期   | 报 99991663 时用 refresh_token 换新（30天有效）              |

### 授权链接

```
https://open.feishu.cn/open-apis/authen/v1/authorize?app_id={APP_ID}&redirect_uri={URL_ENCODED}&scope=calendar:calendar&state=my_state
```

### 换 Token

先拿 app_access_token，再用 code 换 user_access_token：

```
POST /authen/v1/access_token
Authorization: Bearer {app_access_token}
Body: {"grant_type":"authorization_code","code":"xxx"}
```

---

## 三、常见错误速查

| 错误码       | 原因                | 解决                          |
| ------------ | ------------------- | ----------------------------- |
| 20029        | redirect_uri 未注册 | 添加到 Redirect URLs          |
| 20025        | 用了 Basic Auth     | 改用 app_access_token Bearer  |
| 99991663     | User token 过期     | refresh_token 换新            |
| 99991672     | 权限不足            | 开通 calendar:calendar 并发版 |
| 时间戳日期错 | 年份算错            | 用代码算 + 打印验证           |

---

## 四、Agent 决策树

```
用户请求日历操作
  ↓
创建日程？── 是 → Tenant Token + 创建 + attendee（最简单，无需 OAuth）
  │
  否（读取已有日程）
  ↓
有 user_access_token？
├─ 是 → ensure_token() → 读取日历
└─ 否 → OAuth 授权 → 换 token → 读取
```

## 五、触发场景

- 「帮我创建一个明天10点的会议」→ Bot 创建 + attendee
- 「帮我看看本周日程」→ 需要 OAuth 读取
- 「帮我约个会，邀请张三」→ Bot 创建 + 多个 attendee
- 「我下周哪天有空」→ freebusy（Tenant Token 即可）