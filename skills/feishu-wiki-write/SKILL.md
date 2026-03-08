---
name: feishu-wiki
description: 读取和创建飞书 Wiki 知识库页面。Use when user asks to save content to Wiki, create a knowledge base page, read Wiki documentation, or archive meeting notes automatically.
allowed-tools: Bash(curl:*)
---

# feishu-wiki

AI Agent 通过 API 读取 Wiki 空间/页面内容、自动创建新页面。适用于会议纪要自动归档、知识库自动更新、文档批量生成等场景。

## 前提条件

飞书自建应用已开通以下 Tenant token 权限：
- `wiki:wiki` — 读写 Wiki（含创建页面）
- `wiki:wiki:readonly` — 只读 Wiki（若只需读取）

> ⚠️ 此权限**默认未开通**，需在开放平台申请并发版。  
> 权限开通方法参见《飞书开放平台权限开通全流程》

---

## 一、获取 tenant_access_token

```bash
TOKEN=$(curl -s -X POST \
  'https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal' \
  -H 'Content-Type: application/json' \
  -d '{"app_id":"YOUR_APP_ID","app_secret":"YOUR_APP_SECRET"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['tenant_access_token'])")
```

---

## 二、核心 API

### 2.1 获取 Wiki 空间列表

```
GET https://open.feishu.cn/open-apis/wiki/v2/spaces?page_size=10
Authorization: Bearer {token}
```

返回每个空间的 `space_id`、`name`、`description`。

### 2.2 获取空间内的页面节点

```
GET https://open.feishu.cn/open-apis/wiki/v2/spaces/{space_id}/nodes?page_size=20
Authorization: Bearer {token}
```

每个节点包含：

| 字段 | 说明 |
|------|------|
| `node_token` | 节点唯一标识 |
| `obj_token` | 对应的文档 token（用 docx API 读写内容）|
| `obj_type` | 文档类型（docx/sheet/bitable 等）|
| `title` | 页面标题 |
| `parent_node_token` | 父节点 |

### 2.3 创建 Wiki 页面

```
POST https://open.feishu.cn/open-apis/wiki/v2/spaces/{space_id}/nodes
Authorization: Bearer {token}
Content-Type: application/json
```

```json
{
  "obj_type": "docx",
  "parent_node_token": "父节点token（留空则创建在根目录）",
  "node_type": "origin",
  "title": "新页面标题"
}
```

创建后返回 `node_token` 和 `obj_token`，可继续用 docx API 写入内容。

### 2.4 读取/写入页面内容

Wiki 页面本质是 docx 文档，通过 `obj_token` 操作：

```bash
# 读取内容
GET https://open.feishu.cn/open-apis/docx/v1/documents/{obj_token}/blocks?document_revision_id=-1

# 写入内容
POST https://open.feishu.cn/open-apis/docx/v1/documents/{obj_token}/blocks/{obj_token}/children
```

写入 Body 示例（段落文本）：

```json
{
  "children": [{
    "block_type": 2,
    "text": {
      "elements": [{"text_run": {"content": "会议内容在这里"}}]
    }
  }],
  "index": 0
}
```

---

## 三、常见错误排查

| 错误 / 现象 | 原因 | 解决 |
|-------------|------|------|
| `99991672` Access denied | `wiki:wiki` 权限未开通或未发版 | 开放平台申请权限后重新发版 |
| 节点列表为空 | Bot 没有访问该 Wiki 空间的权限 | Wiki 空间设置 → 添加 Bot 成员 |
| 创建页面返回 403 | Bot 无编辑权限 | Wiki 空间管理员给 Bot 添加编辑权限 |

---

## 四、Agent 使用流程

```
1. get_token()
   ↓
2. GET /wiki/v2/spaces
   → 拿到 space_id
   ↓
3. POST /wiki/v2/spaces/{space_id}/nodes
   → 创建页面节点，拿到 obj_token
   ↓
4. POST /docx/v1/documents/{obj_token}/blocks/{obj_token}/children
   → 写入内容
```

---

## 五、触发场景

- 「把这份会议纪要保存到 Wiki」
- 「在知识库创建一篇新文档」
- 「列出所有 Wiki 空间」
- 「读取某个 Wiki 页面的内容」
- 「自动归档今日日报到 Wiki」

## 权限缺失处理

若调用飞书 API 时返回错误码 `99991672`，按以下流程处理：

1. 从错误信息中提取缺少的权限名（如 `attendance:task:readonly`）
2. 告知用户：「检测到缺少权限 `[权限名]`，请问要我帮你自动开通，还是你自己去飞书开放平台设置？」
3. **用户选「帮我开通」** → 调用 `feishu-permission-setup` 技能，自动完成权限申请与版本发布（需扫码登录一次）
4. **用户选「我自己设置」** → 提示前往 [飞书开放平台](https://open.feishu.cn/app) → 权限管理 → 开通对应权限并发布新版本，完成后告知继续

