---
name: feishu-doc-create
description: 创建飞书云文档并写入结构化内容（标题、正文、代码块、列表等）。Use when user asks to create a Feishu/Lark document, save report/tutorial/analysis/meeting notes to cloud doc, or generate a shareable document link. 全自动，无需人工扫码介入。需要 App ID 和 App Secret，以及 docx:document:create 和 docx:document:write_only 权限。
---

# feishu-doc-create

通过飞书 REST API 全自动创建云文档并写入结构化内容。无需人工介入。

## 前提条件

- 飞书自建应用的 App ID 和 App Secret
- 已开通以下 Tenant token 权限：
  - `docx:document:create` — 创建文档
  - `docx:document:write_only` — 写入内容
  - `docx:document:readonly` — 读取内容（可选）

## 核心流程

### Step 1 — 获取 tenant_access_token

```bash
TOKEN=$(curl -s -X POST \
  "https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal" \
  -H "Content-Type: application/json" \
  -d "{\"app_id\":\"$APP_ID\",\"app_secret\":\"$APP_SECRET\"}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['tenant_access_token'])")
```

> Token 有效期 2 小时，每次调用前重新获取，无需缓存。

### Step 2 — 创建文档

```bash
DOC_ID=$(curl -s -X POST \
  "https://open.feishu.cn/open-apis/docx/v1/documents" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title": "文档标题"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['data']['document']['document_id'])")
```

文档访问链接：`https://bytedance.larkoffice.com/docx/{document_id}`

### Step 3 — 写入内容块

```bash
curl -s -X POST \
  "https://open.feishu.cn/open-apis/docx/v1/documents/$DOC_ID/blocks/$DOC_ID/children" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "children": [ ...blocks ],
    "index": 0
  }'
```

`index=0` 表示文档开头，追加到末尾用 `9999`。

## Block 类型速查

| block_type | 说明 | 字段 |
|-----------|------|------|
| 2（无 style）| 普通文本 | `text.elements` |
| 2 + `style.headingLevel: 1/2/3` | 一/二/三级标题 | `text.elements + text.style` |
| 3 | 有序列表 | `ordered.elements` |
| 4 | 无序列表 | `bullet.elements` |
| 14 | 代码块 | `code.elements + code.style.language` |

代码块语言编号：`1=Go 2=Python 3=Shell 4=JavaScript 49=Bash`

### Block 构建示例（Python）

```python
def h1(t): return {"block_type": 2, "text": {"elements": [{"text_run": {"content": t}}], "style": {"headingLevel": 1}}}
def h2(t): return {"block_type": 2, "text": {"elements": [{"text_run": {"content": t}}], "style": {"headingLevel": 2}}}
def p(t):  return {"block_type": 2, "text": {"elements": [{"text_run": {"content": t}}]}}
def code(t, lang=3): return {"block_type": 14, "code": {"elements": [{"text_run": {"content": t}}], "style": {"language": lang}}}
def ol(t): return {"block_type": 3, "ordered": {"elements": [{"text_run": {"content": t}}]}}
def ul(t): return {"block_type": 4, "bullet": {"elements": [{"text_run": {"content": t}}]}}
```

## 清空文档内容

先删除再重写：

```bash
curl -s -X DELETE \
  "https://open.feishu.cn/open-apis/docx/v1/documents/$DOC_ID/blocks/$DOC_ID/children/batch_delete" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"start_index": 0, "end_index": 20}'
```

`end_index` 填实际 block 数量。

## 常见错误

| 错误码 | 原因 | 解决方法 |
|--------|------|---------|
| 99991672 | 权限不足 | 检查是否开通 docx 相关权限并发布了新版本 |
| 99991663 | token 过期/无效 | 重新调用获取接口刷新 |
| 99992402 | 字段校验失败 | 检查 Request Body 格式 |
| index 超出范围 | 插入位置超出 block 数量 | 先查 block 总数，index ≤ count-1 |

> ⚠️ 注意区分 Tenant token 和 User token，两者权限不同，不可混用。

## 参考资料

- 完整 Shell 脚本：`references/shell-script.md`
- Python 封装示例：`references/python-helper.md`


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

