---
name: feishu-manage-chat
description: 创建和管理飞书群组，包括添加/移除成员、修改群名称和描述、获取群列表、发送欢迎消息等。Use when user asks to: create a Feishu group chat, add or remove members from a chat, rename a group, manage team channels, 「帮我建个飞书群」「把 XXX 加到群里」「创建项目群并通知成员」. 前提：飞书 App 已开通 im:chat 和 im:chat.members 两个 Tenant token 权限。
---

# feishu-manage-chat

自动化飞书群组管理：建群、加人、改名、发消息，一步到位。

## 前提条件

飞书自建应用已开通以下 **Tenant token** 权限：
- `im:chat` — 管理群组（创建/修改/解散）
- `im:chat.members` — 管理群成员

> 如缺少权限（报错 99991672），先用 `feishu-permission-setup` 技能开通权限。

## 核心流程

### Step 1：获取 tenant_access_token

```bash
TOKEN=$(curl -s -X POST \
  "https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal" \
  -H "Content-Type: application/json" \
  -d '{"app_id":"YOUR_APP_ID","app_secret":"YOUR_APP_SECRET"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['tenant_access_token'])")
```

### Step 2：创建群组

```bash
CHAT_ID=$(curl -s -X POST \
  "https://open.feishu.cn/open-apis/im/v1/chats" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "项目群",
    "description": "群描述",
    "owner_id": "ou_xxx",
    "owner_id_type": "open_id",
    "user_id_list": ["ou_aaa", "ou_bbb"]
  }' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['data']['chat_id'])")
echo "群已创建: $CHAT_ID"
```

### Step 3：添加成员

```bash
curl -s -X POST \
  "https://open.feishu.cn/open-apis/im/v1/chats/$CHAT_ID/members" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"member_id_type": "open_id", "id_list": ["ou_xxx", "ou_yyy"]}'
```

### Step 4：发送欢迎消息

```bash
curl -s -X POST \
  "https://open.feishu.cn/open-apis/im/v1/messages?receive_id_type=chat_id" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"receive_id\": \"$CHAT_ID\",
    \"msg_type\": \"text\",
    \"content\": \"{\\\"text\\\": \\\"欢迎加入项目群！\\\"}\"
  }"
```

## 其他操作

### 移除成员

```bash
curl -s -X DELETE \
  "https://open.feishu.cn/open-apis/im/v1/chats/$CHAT_ID/members" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"member_id_type": "open_id", "id_list": ["ou_xxx"]}'
```

### 修改群信息

```bash
curl -s -X PATCH \
  "https://open.feishu.cn/open-apis/im/v1/chats/$CHAT_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "新群名",
    "description": "新描述",
    "i18n_names": {"zh_cn": "中文群名", "en_us": "English Name"}
  }'
```

### 获取 Bot 所在群列表

```bash
curl -s -X GET \
  "https://open.feishu.cn/open-apis/im/v1/chats?page_size=20" \
  -H "Authorization: Bearer $TOKEN"
```

返回字段包含：`chat_id`、`name`、`description`、`owner_id` 等。

## 常见错误

| 错误码 | 原因 | 解决 |
|--------|------|------|
| 99991672 | `im:chat` 或 `im:chat.members` 权限未开通 | 用 `feishu-permission-setup` 技能开通并发布新版本 |
| 1300007 | `chat_id` 不存在 | 确认群 ID 正确，或重新获取群列表 |
| 用户添加失败 | `open_id` 无效 | 确认 `user_id_list` 中的 Open ID 有效 |

完整可运行脚本见 [references/scripts/manage_chat.sh](references/scripts/manage_chat.sh) 和 [references/scripts/feishu_chat.py](references/scripts/feishu_chat.py)。

## 权限缺失处理

若调用飞书 API 时返回错误码 `99991672`，按以下流程处理：

1. 从错误信息中提取缺少的权限名（如 `attendance:task:readonly`）
2. 告知用户：「检测到缺少权限 `[权限名]`，请问要我帮你自动开通，还是你自己去飞书开放平台设置？」
3. **用户选「帮我开通」** → 调用 `feishu-permission-setup` 技能，自动完成权限申请与版本发布（需扫码登录一次）
4. **用户选「我自己设置」** → 提示前往 [飞书开放平台](https://open.feishu.cn/app) → 权限管理 → 开通对应权限并发布新版本，完成后告知继续

