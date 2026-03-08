---
name: feishu-minutes
description: 读取飞书妙记会议转写内容，自动生成会议纪要。Use when user asks to summarize a meeting, get meeting transcript, extract action items from Feishu Minutes, or archive meeting notes automatically.
allowed-tools: Bash(curl:*)
---

# feishu-minutes

AI Agent 读取飞书妙记（Minutes）转写文字，自动生成会议纪要、提取 Action Items、发送到指定频道。

## 前提条件

飞书自建应用已开通以下 Tenant token 权限：
- `minutes:item:read` — 读取妙记内容

> ⚠️ 此权限**默认未开通**，需在开放平台申请并发版。  
> 权限开通方法参见《飞书开放平台权限开通全流程》

---

## 一、核心 API

### 1.1 获取妙记列表

```
GET https://open.feishu.cn/open-apis/minutes/v1/minutes?page_size=10
Authorization: Bearer {token}
```

返回每条妙记的 `minute_token`、`title`、创建时间、时长等。

### 1.2 获取妙记详情

```
GET https://open.feishu.cn/open-apis/minutes/v1/minutes/{minute_token}
Authorization: Bearer {token}
```

返回会议时长、参会人、关键词等元数据。

### 1.3 获取转写内容

```
GET https://open.feishu.cn/open-apis/minutes/v1/minutes/{minute_token}/transcript
Authorization: Bearer {token}
```

返回逐句转写，每条包含：

| 字段 | 说明 |
|------|------|
| `content` | 转写文字 |
| `start_time` | 发言起始时间（毫秒）|
| `end_time` | 发言结束时间（毫秒）|
| `user_id` | 发言人 open_id |

---

## 二、Agent 使用流程

```
1. get_token()
   ↓
2. GET /minutes/v1/minutes
   → 拿到 minute_token（最新 or 指定）
   ↓
3. GET /minutes/v1/minutes/{minute_token}
   → 获取参会人、时长等元信息（可选）
   ↓
4. GET /minutes/v1/minutes/{minute_token}/transcript
   → 获取逐句转写内容
   ↓
5. LLM 总结 + 提取 Action Items
   ↓
6. 发送会议纪要到飞书群组 / 写入 Wiki
```

---

## 三、常见错误排查

| 错误 / 现象 | 原因 | 解决 |
|-------------|------|------|
| `99991672` 权限不足 | `minutes:item:read` 未开通或未发版 | 开放平台申请权限后重新发版 |
| 列表为空 | Bot 没有访问该妙记的权限（妙记默认只有参会人可见）| 让参会人将妙记分享给 Bot，或调整妙记可见范围 |
| 转写内容乱码 | 未用 UTF-8 编码处理响应 | 确认 `python3` 脚本使用 UTF-8（默认即可），检查终端编码 |

---

## 四、触发场景

- 「帮我总结一下刚才的会议」
- 「读取最新妙记，提取 Action Items」
- 「把这次会议纪要发到 XXX 群」
- 「列出本周所有妙记」
- 「自动归档会议纪要到 Wiki」

## 权限缺失处理

若调用飞书 API 时返回错误码 `99991672`，按以下流程处理：

1. 从错误信息中提取缺少的权限名（如 `attendance:task:readonly`）
2. 告知用户：「检测到缺少权限 `[权限名]`，请问要我帮你自动开通，还是你自己去飞书开放平台设置？」
3. **用户选「帮我开通」** → 调用 `feishu-permission-setup` 技能，自动完成权限申请与版本发布（需扫码登录一次）
4. **用户选「我自己设置」** → 提示前往 [飞书开放平台](https://open.feishu.cn/app) → 权限管理 → 开通对应权限并发布新版本，完成后告知继续

