---
name: feishu-doc-create-and-grant
description: 创建飞书云文档、写入内容，并同步授予指定用户（主人）管理权限。Use when user asks to: create a Feishu/Lark document, generate a report and save it to cloud, create a document and share it with someone, or "帮我建个飞书文档""创建云文档并授权给我"。
allowed-tools: Bash(curl:*)
---

# feishu-doc-create-and-grant

创建飞书云文档 + 写入内容 + 授权给主人，一次完成。

## 前提条件

1. 飞书自建应用已开通以下 **Tenant token** 权限：
   - `docx:document:create` — 创建文档
   - `docx:document:write_only` — 写入内容
   - `drive:drive` — 云盘操作（含权限管理）
2. 已知主人的飞书 Open ID（格式：`ou_xxxxxxxxxxxxxxxxx`）
3. 持有 App ID 和 App Secret

## 核心流程

### Step 1：获取 tenant_access_token

```bash
TOKEN=$(curl -s -X POST \
  "https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal" \
  -H "Content-Type: application/json" \
  -d '{"app_id":"YOUR_APP_ID","app_secret":"YOUR_APP_SECRET"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin).get('tenant_access_token', ''))")
```

### Step 2：创建文档

```bash
DOC_ID=$(curl -s -X POST \
  "https://open.feishu.cn/open-apis/docx/v1/documents" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title": "文档标题"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['data']['document']['document_id'])")
```

### Step 3：写入内容块

```bash
curl -s -X POST \
  "https://open.feishu.cn/open-apis/docx/v1/documents/$DOC_ID/blocks/$DOC_ID/children" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "children": [
      {
        "block_type": 2,
        "text": {
          "elements": [
            {
              "text_run": {
                "content": "文本内容",
                "text_element_style": { "bold": true }
              }
            }
          ]
        }
      }
    ],
    "index": 0
  }'
```

> ⚠️ **注意**：`text` 内部必须包含 `elements` 数组，且文本内容需放在 `text_run.content` 中。Block 类型说明见 [references/block-types.md](references/block-types.md)。

### Step 4：授权给主人 (full_access)

```bash
curl -s -X POST \
  "https://open.feishu.cn/open-apis/drive/v1/permissions/$DOC_ID/members?type=docx" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "member_type": "openid",
    "member_id": "ou_xxxxxxxxxxxxxxxx",
    "perm": "full_access",
    "perm_type": "container"
  }'
```

> ⚠️ `type=docx` 必须作为 Query Parameter 传递。

### Step 5：返回文档链接

```text
https://bytedance.larkoffice.com/docx/{DOC_ID}
```

## 常见错误排查

| 错误码 | 原因 | 解决 |
|--------|------|------|
| 99991672 | 权限未开通 | 确认应用已发布并包含 drive:drive 等权限 |
| 1770001 | 参数格式错误 | 确认 type=docx 在 URL 查询参数中 |
| 99991663 | Token 过期 | 重新调用 Step 1 获取 Token |

完整脚本见 [references/scripts/create_and_grant.sh](references/scripts/create_and_grant.sh)。
