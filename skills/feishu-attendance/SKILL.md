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


## 权限缺失处理（必须执行）

**若任何飞书 API 调用返回错误码 `99991672`，立即执行以下步骤，不得直接向用户报错：**

1. 从错误响应的 `msg` 字段提取缺少的权限名，例如：
   ```
   msg: "应用尚未开通所需的应用身份权限：[attendance:task:readonly]"
   → 提取: attendance:task:readonly
   ```

2. 向用户发送以下提示（用实际权限名替换占位符）：
   > 「检测到应用缺少权限 `[权限名]`，有两种处理方式：
   > A. **帮我自动开通**（推荐）— 我来操作飞书开放平台，需要你扫码登录一次
   > B. **我自己去设置** — 我告诉你在哪里操作
   > 请选择 A 或 B？」

3. **用户选 A** → 读取并执行 `feishu-permission-setup` 技能，完成后重新调用原 API
4. **用户选 B** → 回复：「请前往 https://open.feishu.cn/app → 选择你的应用 → 权限管理 → 搜索 `[权限名]` → 开通 → 创建并发布新版本。完成后告诉我，我继续操作。」然后等待用户确认

