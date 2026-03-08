---
name: feishu-attendance
description: 查询飞书考勤打卡记录和统计数据。Use when user asks to check attendance, view clock-in records, generate attendance reports, detect late arrivals, or summarize monthly attendance.
allowed-tools: Bash(curl:*)
---

# feishu-attendance

AI Agent 查询飞书考勤打卡记录、考勤组配置、月度统计数据，适用于考勤汇总、异常提醒、月度报表自动生成等场景。

## 前提条件

飞书自建应用已开通以下 Tenant token 权限：

| 权限 | 说明 |
|------|------|
| `attendance:task:readonly` | 查询打卡记录 |
| `attendance:rule:readonly` | 查询考勤组规则 |

> ⚠️ 以上权限**默认未开通**，需在开放平台申请并发版。  
> 参见《飞书开放平台权限开通全流程》

---

## 一、核心 API

### 1.1 获取考勤组列表

```
GET https://open.feishu.cn/open-apis/attendance/v1/groups?page_size=10
Authorization: Bearer {token}
```

返回所有考勤组的 `group_id`、`name`、时区、考勤规则等。

### 1.2 查询打卡明细

```
POST https://open.feishu.cn/open-apis/attendance/v1/user_tasks/query
Authorization: Bearer {token}
Content-Type: application/json
```

```json
{
  "locale": "zh",
  "staff_type": 1,
  "include_terminated_user": false,
  "user_ids": ["ou_xxx", "ou_yyy"],
  "check_date_from": 20260301,
  "check_date_to": 20260307,
  "user_id_type": "open_id"
}
```

> ⚠️ `check_date_from` / `check_date_to` 为 **整数 YYYYMMDD**，不是字符串。

### 1.3 查询统计汇总

```
POST https://open.feishu.cn/open-apis/attendance/v1/user_stats_datas/query
Authorization: Bearer {token}
Content-Type: application/json
```

```json
{
  "locale": "zh",
  "stats_type": "month",
  "start_date": 20260301,
  "end_date": 20260331,
  "user_ids": ["ou_xxx"],
  "user_id_type": "open_id"
}
```

返回字段：出勤天数、迟到次数、早退次数、缺勤天数等。

---

## 二、Agent 使用流程

```
1. get_token()
   ↓
2. GET /attendance/v1/groups
   → 确认考勤组（可选）
   ↓
3. POST /attendance/v1/user_tasks/query
   → 查指定用户、指定日期范围的打卡明细
   ↓
4. POST /attendance/v1/user_stats_datas/query
   → 查月度统计（迟到、缺勤等汇总）
   ↓
5. 格式化数据，发飞书消息 / 写 Wiki 报表
```

---

## 三、常见错误排查

| 错误 / 现象 | 原因 | 解决 |
|-------------|------|------|
| `99991672` 权限不足 | `attendance:task:readonly` 未开通或未发版 | 开放平台申请权限后重新发版 |
| 返回空数据 | 日期范围内无考勤记录，或用户不属于任何考勤组 | 确认日期范围和考勤组成员 |
| 日期格式错误 | `check_date_from` 传了字符串而非整数 | 使用整数 `YYYYMMDD`，例如 `20260301` |

---

## 四、触发场景

- 「查一下小王本周的打卡记录」
- 「生成全组 3 月份考勤报表」
- 「列出本月迟到 3 次以上的员工」
- 「昨天有没有人漏打卡？」
- 「自动汇总每月考勤发到 HR 群」

## 权限缺失处理

若调用飞书 API 时返回错误码 `99991672`，按以下流程处理：

1. 从错误信息中提取缺少的权限名（如 `attendance:task:readonly`）
2. 告知用户：「检测到缺少权限 `[权限名]`，请问要我帮你自动开通，还是你自己去飞书开放平台设置？」
3. **用户选「帮我开通」** → 调用 `feishu-permission-setup` 技能，自动完成权限申请与版本发布（需扫码登录一次）
4. **用户选「我自己设置」** → 提示前往 [飞书开放平台](https://open.feishu.cn/app) → 权限管理 → 开通对应权限并发布新版本，完成后告知继续

