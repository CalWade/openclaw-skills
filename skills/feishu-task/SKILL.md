---
name: feishu-task-oauth
description: 通过 OAuth User Access Token 操作飞书任务（创建/分配/更新/完成）。Use when building custom scripts or shell automation that calls Feishu Task API directly with user credentials. 区别于内置 feishu-task skill（使用 OpenClaw 工具），本 skill 适合需要手写 curl/Python 脚本的场景。
allowed-tools: Bash(curl:*)
---

# feishu-task-oauth

飞书任务（Task v2）OAuth 脚本参考。

> **与内置 `feishu-task` skill 的区别：**
> - 内置 skill → 使用 OpenClaw 封装工具（`feishu_task_create` 等），由平台负责鉴权
> - 本 skill → 手写 curl/Python，需要自行管理 OAuth User Access Token

---

## ⚠️ 前提条件

飞书任务 API **必须使用 User Access Token**，不支持 Tenant token。

所需权限：
- `task:task` — 创建和管理任务
- `task:task:readonly` — 只读任务（查询用）

---

## OAuth 流程（获取 User Access Token）

### 1. 引导用户授权

```
https://open.feishu.cn/open-apis/authen/v1/authorize
  ?app_id={APP_ID}
  &redirect_uri={CALLBACK_URL}
  &scope=task:task
```

### 2. 用回调 code 换 token

```bash
curl -s -X POST \
  'https://open.feishu.cn/open-apis/authen/v1/access_token' \
  -H "Authorization: Basic $(echo -n 'APP_ID:APP_SECRET' | base64)" \
  -H 'Content-Type: application/json' \
  -d '{"grant_type":"authorization_code","code":"回调中的code"}'
```

返回：`user_access_token`（有效期约 2 小时）

---

## 核心 API

### 创建任务

```
POST https://open.feishu.cn/open-apis/task/v2/tasks
Authorization: Bearer {user_access_token}
Body:
{
  "summary": "完成 API 文档",
  "description": "任务详情描述",
  "due": {
    "timestamp": "1772956800000",
    "is_all_day": false
  },
  "origin": {
    "platform_i18n_name": "{\"zh_cn\": \"我的应用\"}"
  }
}
```

> ⚠️ `timestamp` 是 **毫秒时间戳，字符串格式**（如 `"1772956800000"`）

---

### 添加负责人

```
POST /task/v2/tasks/{task_guid}/add_members
Body:
{
  "members": [
    { "id": "ou_xxx", "type": "user", "role": "assignee" }
  ],
  "user_id_type": "open_id"
}
```

---

### 查询任务列表

```
GET /task/v2/tasks?page_size=20&completed=false
```

按完成状态筛选：`completed=false`（未完成）| `completed=true`（已完成）

---

### 标记完成 / 重新打开

```
PATCH /task/v2/tasks/{task_guid}
Body:
{
  "task": {
    "completed_at": "1772870400000"
  },
  "update_fields": ["completed_at"]
}
```

> 传 `"0"` 可重新打开已完成任务

---

## 常见错误

| 错误码 | 原因 | 解决 |
|--------|------|------|
| 99991663 | Invalid access token | 必须用 User token，不能用 Tenant token |
| task_guid 不存在 | 任务 ID 错误 | 先 GET /tasks 确认 |
| timestamp 格式错误 | 传了 int 而非 string | 必须传字符串 `"1772956800000"` |
| 权限不足 | 缺少 task:task scope | 重走 OAuth 带 scope=task:task |

---

完整可运行脚本见：
- [references/scripts/task.sh](references/scripts/task.sh) — Shell
- [references/scripts/feishu_task.py](references/scripts/feishu_task.py) — Python

## 权限缺失处理

若调用飞书 API 时返回错误码 `99991672`，按以下流程处理：

1. 从错误信息中提取缺少的权限名（如 `attendance:task:readonly`）
2. 告知用户：「检测到缺少权限 `[权限名]`，请问要我帮你自动开通，还是你自己去飞书开放平台设置？」
3. **用户选「帮我开通」** → 调用 `feishu-permission-setup` 技能，自动完成权限申请与版本发布（需扫码登录一次）
4. **用户选「我自己设置」** → 提示前往 [飞书开放平台](https://open.feishu.cn/app) → 权限管理 → 开通对应权限并发布新版本，完成后告知继续

