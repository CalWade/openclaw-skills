---
name: feishu-send-message
description: 通过飞书 API 向指定用户或群组发送消息。Use when user asks to send a Feishu message, notify someone on Feishu, send a report/image/file via Feishu bot. 支持文本、图片、文件、富文本（post）、交互卡片等消息类型。全自动，无需人工介入。需要 App ID、App Secret 和 im:message:send_as_bot 权限。
---

# feishu-send-message

通过飞书 Bot 向用户或群组发送消息，全自动，无需人工介入。

## 前提条件

- 飞书自建应用的 App ID 和 App Secret
- 已开通 Tenant token 权限：`im:message:send_as_bot`
- 接收方的 ID（open_id / user_id / union_id / email / chat_id 之一）

## 核心流程

### Step 1 — 获取 tenant_access_token

```bash
TOKEN=$(curl -s -X POST \
  "https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal" \
  -H "Content-Type: application/json" \
  -d "{\"app_id\":\"$APP_ID\",\"app_secret\":\"$APP_SECRET\"}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['tenant_access_token'])")
```

### Step 2 — 发送消息

```bash
curl -s -X POST \
  "https://open.feishu.cn/open-apis/im/v1/messages?receive_id_type=open_id" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"receive_id\":\"$RECEIVER_ID\",\"msg_type\":\"text\",\"content\":\"{\\\"text\\\":\\\"Hello!\\\"}\"}"
```

`receive_id_type` 可选：`open_id` / `user_id` / `union_id` / `email` / `chat_id`（发群组用 `chat_id`）

## 消息类型速查

| msg_type | content 格式 | 说明 |
|----------|-------------|------|
| `text` | `{"text": "内容"}` | 纯文本 |
| `image` | `{"image_key": "img_xxx"}` | 图片（需先上传） |
| `file` | `{"file_key": "file_xxx"}` | 文件（需先上传） |
| `post` | 见下方示例 | 富文本，支持链接/@人/加粗 |
| `interactive` | 卡片 JSON | 交互卡片，支持按钮/表单 |

### 富文本（post）content 示例

```python
import json
content = json.dumps({
    "zh_cn": {
        "title": "通知标题",
        "content": [
            [{"tag": "text", "text": "这是正文内容"}],
            [{"tag": "a", "text": "点击查看", "href": "https://example.com"}],
            [{"tag": "at", "user_id": "ou_xxx", "user_name": "张三"}]
        ]
    }
})
```

## 上传图片 / 文件

发送图片或文件前，需先上传获取 key：

```bash
# 上传图片
IMG_KEY=$(curl -s -X POST \
  "https://open.feishu.cn/open-apis/im/v1/images" \
  -H "Authorization: Bearer $TOKEN" \
  -F "image_type=message" \
  -F "image=@/path/to/image.png" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['data']['image_key'])")

# 上传文件
FILE_KEY=$(curl -s -X POST \
  "https://open.feishu.cn/open-apis/im/v1/files" \
  -H "Authorization: Bearer $TOKEN" \
  -F "file_type=stream" \
  -F "file_name=report.pdf" \
  -F "file=@/path/to/report.pdf" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['data']['file_key'])")
```

## 常见错误

| 错误码 / 现象 | 原因 | 解决方法 |
|-------------|------|---------|
| 230013 | Bot 对该用户不可用 | 用户需先主动给 Bot 发一条消息建立会话 |
| 99991672 | 权限不足 | 检查是否开通 `im:message:send_as_bot` 并发布新版本 |
| 发群组失败 | receive_id_type 错误 | 改为 `chat_id`，receive_id 填群的 chat_id |
| 99991663 | token 过期 | 重新获取 tenant_access_token |

## 参考资料

- 完整 Shell 脚本（文本+图片+文件）：`references/shell-script.md`

## 权限缺失处理

若调用飞书 API 时返回错误码 `99991672`，按以下流程处理：

1. 从错误信息中提取缺少的权限名（如 `attendance:task:readonly`）
2. 告知用户：「检测到缺少权限 `[权限名]`，请问要我帮你自动开通，还是你自己去飞书开放平台设置？」
3. **用户选「帮我开通」** → 调用 `feishu-permission-setup` 技能，自动完成权限申请与版本发布（需扫码登录一次）
4. **用户选「我自己设置」** → 提示前往 [飞书开放平台](https://open.feishu.cn/app) → 权限管理 → 开通对应权限并发布新版本，完成后告知继续

