---
name: feishu-app-rename
description: 通过浏览器自动化给飞书自建应用改名。Use when user asks to rename a Feishu/Lark app, change bot display name, update app name in Feishu Open Platform, or modify bot name. 飞书开放平台不提供改名 API，需通过 Web 控制台操作。负责人需扫码登录一次。改名后自动创建并发布新版本使变更生效。
---

# feishu-app-rename

通过浏览器自动化修改飞书自建应用的名称。飞书开放平台不提供改名 API，所有操作必须通过 Web 控制台完成。

**需要人工介入：** 应用 Owner 或管理员需用手机飞书扫码登录一次。

## 前提条件

- 飞书 App ID（格式：`cli_xxxxxxxxxxxxxxxx`）
- 新的应用名称
- 应用 Owner 或管理员手机可用于扫码

## 操作流程

### Step 1 — 打开登录页

打开飞书开放平台登录页，登录后直接跳转到目标应用的基本信息页：

```
https://accounts.feishu.cn/accounts/page/login?app_id=7&redirect_uri=https%3A%2F%2Fopen.feishu.cn%2Fapp%2F{APP_ID}%2Fbaseinfo
```

将 `{APP_ID}` 替换为实际的 App ID。

### Step 2 — 截图二维码并发给管理员

截图二维码，通过飞书消息发送给有权限的管理员，等待其扫码。扫码成功后浏览器会自动跳转到应用基本信息页。

确认登录成功（当前 URL 应为 `open.feishu.cn/app/{APP_ID}/baseinfo`）。

### Step 3 — 定位名称输入框并注入新名称

名称输入框位于「多语言应用详情」区域（页面下方），该区域在内部滚动容器中，**不要用滚动命令**，直接用 JS 操作。

先枚举所有 input，确认哪个是应用名称字段（通常是第二个，index 1，value 为当前应用名）：
```js
const inputs = Array.from(document.querySelectorAll('input'));
inputs.map((el, i) => `${i}: value="${el.value}" placeholder="${el.placeholder.substring(0,30)}"`)
```

用原生 setter 注入新名称（React 框架下直接赋值不触发状态更新，必须用此方式）：
```js
const input = document.querySelectorAll('input')[1]; // 根据上一步确认的序号调整
const setter = Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype, 'value').set;
setter.call(input, '新的应用名称');
input.dispatchEvent(new Event('input', { bubbles: true }));
input.dispatchEvent(new Event('change', { bubbles: true }));
```

> ⚠️ 直接 `input.value = '...'` 不会触发 React 状态更新，必须用原生 setter + 手动 dispatch 事件。

### Step 4 — 保存

```js
const btns = Array.from(document.querySelectorAll('button'));
btns.find(b => b.textContent.trim() === 'Save')?.click();
```

成功标志：页面顶部出现黄色提示条 *"The changes will take effect after the current version is published."*

### Step 5 — 创建版本并发布

名称变更必须发布新版本才对用户生效。

**点击创建版本：**
```js
const btns = Array.from(document.querySelectorAll('button'));
btns.find(b => b.textContent.trim() === 'Create Version')?.click();
```

**填写版本号和更新说明（React 兼容方式）：**
```js
(() => {
  const vInput = document.querySelectorAll('input')[0];
  const textarea = document.querySelector('textarea');
  const iS = Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype, 'value').set;
  const tS = Object.getOwnPropertyDescriptor(window.HTMLTextAreaElement.prototype, 'value').set;
  iS.call(vInput, '1.x.x'); // 版本号，根据实际情况填写
  vInput.dispatchEvent(new Event('input', { bubbles: true }));
  tS.call(textarea, '更新应用名称');
  textarea.dispatchEvent(new Event('input', { bubbles: true }));
})()
```

**保存版本**（版本对话框中第 7 个按钮通常是 Save）：
```js
document.querySelectorAll('button')[6].click()
```

**发布：**
```js
const btns = Array.from(document.querySelectorAll('button'));
btns.find(b => b.textContent.trim() === 'Publish')?.click();
```

成功标志：顶部绿色提示 *"The current changes have been published"*，版本状态变为 "Released"。

## 常见问题

| 问题 | 解决方法 |
|------|---------|
| 直接赋值 `input.value =` 无效 | 使用原生 setter + `dispatchEvent` |
| 名称输入框不可见，页面滚动无效 | 页面使用内部滚动容器，直接用 JS 操作 DOM |
| input 序号不固定 | 先枚举所有 input 确认序号再操作 |
| 浏览器 session 超时 | 关键操作前先确认当前 URL，断开时重新扫码 |
| 发布需要管理员审核 | 企业自建应用通常即时生效；若需审核，联系飞书工作空间管理员 |

## 多语言名称

若应用配置了多语言名称，页面会有多个名称 input，分别对应不同语言，按序号依次修改即可。

## 参考资料

- 完整端到端脚本参见 `references/full-workflow.md`


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

