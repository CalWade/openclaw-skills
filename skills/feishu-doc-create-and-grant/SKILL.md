---
name: feishu-doc-create-and-grant
description: 创建飞书云文档、写入内容，并同步授予指定用户（主人）管理权限。Use when user asks to: create a Feishu/Lark document, generate a report and save it to cloud, create a document and share it with someone, or "帮我建个飞书文档""创建云文档并授权给我". 全自动，无需人工扫码。前提：飞书 App 已开通 docx:document:create、docx:document:write_only、drive:drive 三个 Tenant token 权限。
---

# feishu-doc-create

创建飞书云文档 + 写入内容 + 授权给主人，一次完成。全程自动，无需人工介入。

## 前提条件

1. 飞书自建应用已开通以下 **Tenant token** 权限：
   - `docx:document:create` — 创建文档
   - `docx:document:write_only` — 写入内容
   - `drive:drive` — 云盘操作（含权限管理）
2. 已知主人的飞书 Open ID（格式：`ou_xxxxxxxxxxxxxxxxx`）
3. 持有 App ID 和 App Secret

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

Token 有效期 2 小时，过期重新获取（错误码 99991663）。

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
  -d '{"children": [...blocks], "index": 0}'
```

Block 类型说明见 [references/block-types.md](references/block-types.md)。

### Step 4：授权给主人（full_access）

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

> ⚠️ `type=docx` 必须放在 **query param** 里，放 body 里无效（报错 1770001）。

### Step 5：返回文档链接

```
https://bytedance.larkoffice.com/docx/{DOC_ID}
```

## 权限级别

| perm | 说明 |
|------|------|
| `view` | 只读，不能编辑 |
| `edit` | 可编辑，不能管理权限 |
| `full_access` | 完全控制（推荐给主人） |

## 批量授权多个用户

循环调用授权接口即可，每次一个用户。大团队建议用 `chat_id` 方式批量授权（`member_type: "chat"`）。

## 常见错误

| 错误码 | 原因 | 解决 |
|--------|------|------|
| 99991672 | `drive:drive` 权限未开通 | 用 `feishu-permission-setup` 技能开通并发布新版本 |
| 1770001 | 参数格式错误 | 确认 `type=docx` 在 query param 中 |
| 99991663 | Token 过期 | 重新获取 `tenant_access_token` |

完整可运行脚本见 [references/scripts/create_and_grant.sh](references/scripts/create_and_grant.sh) 和 [references/scripts/feishu_doc.py](references/scripts/feishu_doc.py)。

## 权限缺失处理

若调用飞书 API 时返回错误码 `99991672`，按以下流程处理：

1. 从错误信息中提取缺少的权限名（如 `attendance:task:readonly`）
2. 告知用户：「检测到缺少权限 `[权限名]`，请问要我帮你自动开通，还是你自己去飞书开放平台设置？」
3. **用户选「帮我开通」** → 调用 `feishu-permission-setup` 技能，自动完成权限申请与版本发布（需扫码登录一次）
4. **用户选「我自己设置」** → 提示前往 [飞书开放平台](https://open.feishu.cn/app) → 权限管理 → 开通对应权限并发布新版本，完成后告知继续

