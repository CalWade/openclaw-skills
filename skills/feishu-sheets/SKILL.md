---
name: feishu-sheets
description: 创建飞书电子表格并读写数据。Use when user asks to create a spreadsheet, fill in data, generate a data table, export data to Feishu Sheets, or automate report generation. 适用于数据录入、报表自动化、数据分析输出等场景。前提：飞书 App 已开通 sheets:spreadsheet 权限。
allowed-tools: Bash(curl:*)
---

# feishu-sheets

创建飞书电子表格（Sheets）、读写单元格数据、批量插入/追加数据。

## 前提条件

飞书自建应用已开通以下 **Tenant token** 权限：
- `sheets:spreadsheet` — 创建和管理电子表格

> 权限缺失报错 99991672 时，参考《飞书开放平台权限开通全流程》开通权限。

## 核心流程

### Step 1：获取 tenant_access_token

```bash
TOKEN=$(curl -s -X POST \
  "https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal" \
  -H "Content-Type: application/json" \
  -d '{"app_id":"YOUR_APP_ID","app_secret":"YOUR_APP_SECRET"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['tenant_access_token'])")
```

### Step 2：创建电子表格

```bash
POST https://open.feishu.cn/open-apis/sheets/v3/spreadsheets
Body: { "title": "表格标题", "folder_token": "" }  # folder_token 留空 → 根目录
```

返回：
- `spreadsheet_token` — 后续所有操作的唯一标识
- `url` — 表格访问链接

### Step 3：获取工作表 sheet_id

```bash
GET https://open.feishu.cn/open-apis/sheets/v3/spreadsheets/{spreadsheet_token}/sheets/query
```

返回所有 sheet 的 `sheet_id`、`title`、`index`。**写入数据前必须先拿到 sheet_id。**

### Step 4：写入数据

```bash
PUT https://open.feishu.cn/open-apis/sheets/v2/spreadsheets/{token}/values
Body:
{
  "valueRange": {
    "range": "SheetId!A1:C3",
    "values": [
      ["姓名", "部门", "入职日期"],
      ["张三", "技术部", "2026-01-01"],
      ["李四", "市场部", "2026-02-01"]
    ]
  }
}
```

> ⚠️ `range` 格式：`{sheet_id}!{起始单元格}:{结束单元格}`，sheet_id 区分大小写。

### Step 5：读取数据

```bash
GET https://open.feishu.cn/open-apis/sheets/v2/spreadsheets/{token}/values/{range}
# range 示例：SheetId!A1:C10
```

返回 `values` 二维数组。

### Step 6：追加数据（不覆盖现有内容）

```bash
POST https://open.feishu.cn/open-apis/sheets/v2/spreadsheets/{token}/values_append
# Body 结构同写入，自动追加到已有数据末尾
```

## 常见错误

| 错误 | 原因 | 解决 |
|------|------|------|
| 99991672 | 权限不足 | 确认开通 `sheets:spreadsheet` |
| range 格式错误 | sheet_id 不对或格式有误 | 确认格式为 `SheetId!A1:C3` |
| 数据不写入 | values 不是二维数组 | 每行数据必须包在 `[]` 里，整体再包一层 `[]` |

完整可运行脚本见 [references/scripts/sheets.sh](references/scripts/sheets.sh) 和 [references/scripts/feishu_sheets.py](references/scripts/feishu_sheets.py)。

## 权限缺失处理

若调用飞书 API 时返回错误码 `99991672`，按以下流程处理：

1. 从错误信息中提取缺少的权限名（如 `attendance:task:readonly`）
2. 告知用户：「检测到缺少权限 `[权限名]`，请问要我帮你自动开通，还是你自己去飞书开放平台设置？」
3. **用户选「帮我开通」** → 调用 `feishu-permission-setup` 技能，自动完成权限申请与版本发布（需扫码登录一次）
4. **用户选「我自己设置」** → 提示前往 [飞书开放平台](https://open.feishu.cn/app) → 权限管理 → 开通对应权限并发布新版本，完成后告知继续

