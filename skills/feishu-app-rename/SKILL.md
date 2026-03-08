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

## 权限缺失处理

若调用飞书 API 时返回错误码 `99991672`，按以下流程处理：

1. 从错误信息中提取缺少的权限名（如 `attendance:task:readonly`）
2. 告知用户：「检测到缺少权限 `[权限名]`，请问要我帮你自动开通，还是你自己去飞书开放平台设置？」
3. **用户选「帮我开通」** → 调用 `feishu-permission-setup` 技能，自动完成权限申请与版本发布（需扫码登录一次）
4. **用户选「我自己设置」** → 提示前往 [飞书开放平台](https://open.feishu.cn/app) → 权限管理 → 开通对应权限并发布新版本，完成后告知继续

