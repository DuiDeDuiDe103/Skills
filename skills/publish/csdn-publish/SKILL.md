---
name: csdn-publish
description: 通过浏览器自动化（builtin_browser Chrome 扩展 MCP）把本地 Markdown 笔记发布到 CSDN 草稿箱。适用于用户要求"发到 CSDN"、"发布到 CSDN"、"把笔记发到 CSDN 博客"、"上传到 CSDN 草稿箱"等场景。触发词：CSDN、发布笔记、发到 CSDN、CSDN 草稿、博客发布。
version: 1.0.0
---

# CSDN 草稿发布

通过 `builtin_browser` Chrome 扩展 MCP 把本地 Markdown 文章自动发布到 CSDN 草稿箱。默认流程保存为草稿，用户确认后再点发布。

## 输入

| 参数 | 必填 | 说明 |
|------|------|------|
| `file_path` | 是 | 本地 Markdown 文件绝对路径 |
| `title` | 否 | 文章标题，省略则取文件首行 `# ...` 或文件名 |
| `tags` | 否 | 标签数组，省略则按内容自动选默认 |
| `visibility` | 否 | 可见范围，默认「全部可见」；可选「仅我可见」「粉丝可见」「VIP可见」 |

## 关键 URL 与 DOM

| 用途 | 地址 / 选择器 |
|------|------|
| 登录态探测 | `https://i.csdn.net/` 若跳转到 `https://i.csdn.net/#/user-center/profile` 则已登录 |
| 发布页（WYSIWYG 模式） | `https://mp.csdn.net/mp_blog/creation/editor` |
| 标题输入框 | `#txtTitle` |
| 富文本编辑器实例 | `window.CKEDITOR.instances.editor`（方法：`setData/getData/insertHtml`） |
| 标签搜索输入框 | `input[placeholder*="请输入文字搜索"]`（id 形如 `el_mcm-id-130-*`） |
| 标签类别 tab | `[role="tab"]`，共 42 个（推荐/Python/Java/编程语言/…/AIGC） |
| 标签面板 | `[role="tabpanel"]`，索引与 tab 一一对应（Java=2，后端=8，职场和发展=29） |
| 已选标签 | `.el_mcm-tag.is-closable` |
| 保存草稿按钮文本 | `保存草稿` |
| 成功提示 | `.el-message` 文本含 `草稿保存成功` |

## 工作流

### 1. 打开浏览器并探测登录

```
tabs_context 拿到 tabId
若没有 tab：用 computer_use 按 Win 键搜 "chrome" 回车打开
navigate → https://i.csdn.net/
等 2s 后检查 URL
  ├─ 包含 user-center/profile → 已登录，继续
  └─ 包含 passport.csdn.net 或登录按钮 → 停住，让用户扫码/输密码后再续
```

### 2. 打开发布页

```
navigate → https://mp.csdn.net/mp_blog/creation/editor
等待 title 包含 "写文章-CSDN创作中心"
注意：navigate 工具可能报 "404 Not Found" 但页面实际加载成功，按 title 判断
```

### 3. 注入标题与正文

```js
// 3.1 设置标题（form_input 直接对 #txtTitle 赋值即可）
form_input(ref: ref_9, value: title)
// 或 JS 兜底
const s = Object.getOwnPropertyDescriptor(HTMLInputElement.prototype, 'value').set;
s.call(document.getElementById('txtTitle'), title);
document.getElementById('txtTitle').dispatchEvent(new Event('input', {bubbles: true}));

// 3.2 加载 marked.js 并把 md 转 HTML
await new Promise((res, rej) => {
  const s = document.createElement('script');
  s.src = 'https://cdnjs.cloudflare.com/ajax/libs/marked/12.0.1/marked.min.js';
  s.onload = res; s.onerror = rej;
  document.head.appendChild(s);
});
const html = marked.parse(mdContent);
CKEDITOR.instances.editor.setData(html);
// 校验 editor.getData().length > 0
```

> 不要在发布页 URL 加 `?not_checkout=1` 或尝试 `/mdeditor`，都返回 404。

### 4. 选标签

1. 用 JS 点击「添加文章标签」按钮（`.click()` 在 `button` 文本匹配 `/添加文章标签/`）。
2. 等标签弹窗出现（`input[placeholder*="请输入文字搜索"]` 的 `offsetParent !== null`）。
3. **推荐标签**：遍历 `[role="tabpanel"]` 里所有 `span`，按文本精确匹配后 `.click()`。
   - Java 相关 → `document.querySelectorAll('[role="tabpanel"]')[2]`（tab「Java」对应 panel）
   - 后端相关 → `panels[8]`（tab「后端」）
   - 面试 / 职场 → `panels[29]`（tab「学习和成长」）
4. **自定义标签**（库里没有的，如 `ThreadLocal`）：
   ```js
   const inp = document.querySelector('input[placeholder*="请输入文字搜索"]');
   inp.focus();
   const setter = Object.getOwnPropertyDescriptor(HTMLInputElement.prototype,'value').set;
   setter.call(inp, 'ThreadLocal');
   inp.dispatchEvent(new Event('input', {bubbles: true}));
   setTimeout(() => {
     inp.dispatchEvent(new KeyboardEvent('keydown', {key:'Enter', code:'Enter', keyCode:13, bubbles:true}));
   }, 200);
   ```
5. 校验：`document.querySelectorAll('.el_mcm-tag.is-closable')` 长度 = 期望标签数。

### 5. 保存草稿

```js
const btn = Array.from(document.querySelectorAll('button'))
  .find(b => /保存草稿/.test(b.textContent));
btn.click();
```

等待 3 秒后校验：

```js
const msgs = document.querySelectorAll('.el-message, [class*=message]');
Array.from(msgs).map(m => m.textContent).some(t => t.includes('草稿保存成功'));
```

### 6. 收尾输出

给用户简短回执：标题、标签列表、草稿箱入口 `https://mp.csdn.net/mp_blog/manage/draft`。不要写长篇解释。

## 默认标签策略

无显式 `tags` 时按内容推断。常见默认组合：

- Java 技术文章 → `Java, spring, jvm, 后端, 面试`
- 前端 → `前端, JavaScript, Vue`
- 算法 → `算法, leetcode, 数据结构`

标签库里没有的关键词一律走「自定义标签」路径（步骤 4.4）。

## Pitfalls

- **浏览器截图常超时**：`computer screenshot` 在 CSDN 页面容易 timeout，优先用 `read_page` / `javascript_tool` 读 DOM。
- **navigate 误报 404**：CSDN SPA 路由下 navigate 返回的 Title/状态码不可靠，用 `title` 字段或 DOM 判断。
- **`使用 MD 编辑器` 按钮别点**：点了会弹确认并清空富文本内容。直接用 WYSIWYG + `setData(html)` 更稳。
- **标签面板索引与 tab 顺序严格对应**：不要按文本搜索 tab，用 `panels[index]` 定位更快。
- **自定义标签必须用 setter 触发 input 事件**：直接赋值 `inp.value = x` 不会触发 Vue/React 响应。
- **草稿保存成功 ≠ 跳转**：成功后页面仍停在编辑器，要看 `.el-message` 文本确认。
- **`CKEDITOR.instances.editor` 为空时**：可能是 micro-frontend 未渲染完，再等 2s 重试；若还失败，放弃并告诉用户。
- **登录态失效**：任何环节跳回 `passport.csdn.net` → 停住，让用户手动登录再继续。

## 调用示例

```
用户：把 E:\笔记\八股文\Java ThreadLocal详解.md 发到 CSDN
Agent：
  1. 读文件 → md
  2. tabs_context → navigate i.csdn.net → 已登录
  3. navigate editor → setData(marked.parse(md)) → 标题="Java ThreadLocal 详解"
  4. 标签：panels[2]点 spring、jvm；panels[8]点 restful；panels[29]点 面试；自定义加 ThreadLocal
  5. 点"保存草稿" → 等成功提示
  6. 回执：标题 / 标签 / 草稿箱链接
```

## Verification

发布完成后必须满足：

- [ ] `CKEDITOR.instances.editor.getData().length > 0`（正文非空）
- [ ] `#txtTitle.value` 等于用户给定标题
- [ ] `.el_mcm-tag.is-closable` 数量 ≥ 1
- [ ] 页面 `.el-message` 文本含 `草稿保存成功`
- [ ] 给用户回执包含草稿箱 URL
