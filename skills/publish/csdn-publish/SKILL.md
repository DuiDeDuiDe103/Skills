---
name: csdn-publish
description: 通过浏览器自动化（builtin_browser Chrome 扩展 MCP）把本地 Markdown 笔记发布到 CSDN 草稿箱。适用于用户要求"发到 CSDN"、"发布到 CSDN"、"把笔记发到 CSDN 博客"、"上传到 CSDN 草稿箱"等场景。触发词：CSDN、发布笔记、发到 CSDN、CSDN 草稿、博客发布。
version: 1.1.0
---

# CSDN 草稿发布

通过 `builtin_browser` Chrome 扩展 MCP 把本地 Markdown 文章自动发布到 CSDN 草稿箱。默认流程保存为草稿，用户确认后再点发布。

## Hard Rules

以下两条规则**不可违反**，优先级高于工作流步骤中的任何指令：

1. **禁止使用 `computer screenshot`**。CSDN 页面上 screenshot 极易超时导致整个流程中断。所有页面状态读取统一使用 `read_page`（accessibility tree）或 `javascript_tool`（DOM 查询）。
2. **navigate 返回 404 时不得中断流程**。CSDN 是 SPA 路由，navigate 工具返回的 HTTP 状态码 / Title 不可靠。每次 navigate 到发布页后，必须用 `javascript_tool` 执行以下校验，只要 title 包含「写文章」即放行：
   ```js
   const t = document.title;
   JSON.stringify({ ok: t.includes('写文章'), title: t });
   ```
   若 `ok === true` 则继续下一步；若 `ok === false` 则等待 2s 后重试一次，仍失败再报告用户。

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
```

> **navigate 可能报 404，这是已知行为，不要中断。** 立即执行 Hard Rule #2 的校验代码：
> ```js
> const t = document.title;
> JSON.stringify({ ok: t.includes('写文章'), title: t });
> ```
> `ok === true` → 继续。`ok === false` → 等 2s 重试一次，仍失败再报告用户。

### 3. 数据传输与内容注入

#### 3.0 数据传输（CORS HTTP Server 方案）

Markdown 内容通常 10-20KB，无法直接在 `javascript_tool` 的 text 参数里内联（转义问题多），分 chunk 拼接又容易丢失字符。**唯一验证通过的方案**是 Python CORS HTTP server + 浏览器 fetch：

```python
# cors_server.py — 写到工作目录后 python cors_server.py
import http.server, socketserver, os, base64, sys, threading

class H(http.server.SimpleHTTPRequestHandler):
    def end_headers(self):
        self.send_header("Access-Control-Allow-Origin", "*")
        self.send_header("Access-Control-Allow-Methods", "GET, OPTIONS")
        self.send_header("Access-Control-Allow-Headers", "*")
        super().end_headers()
    def do_OPTIONS(self):
        self.send_response(200)
        self.end_headers()

PORT = 18766
os.chdir(sys.argv[1] if len(sys.argv) > 1 else ".")
with socketserver.TCPServer(("127.0.0.1", PORT), H) as s:
    print(f"serving on {PORT}")
    s.serve_forever()
```

操作步骤：

```
1. 用 Python base64 编码 md 文件：
   python -c "import base64; d=open(r'<md_path>','rb').read(); open(r'<workdir>/article.b64','wb').write(base64.b64encode(d))"

2. 后台启动 CORS server（is_background: true）：
   cd <workdir> && python cors_server.py <workdir>

3. 浏览器端 fetch + 解码（javascript_tool）：
   (async () => {
     const r = await fetch('http://127.0.0.1:18766/article.b64', {cache:'no-store'});
     const b64 = await r.text();
     window.__b64 = b64.trim();
     const bin = atob(window.__b64);
     const bytes = new Uint8Array(bin.length);
     for (let i = 0; i < bin.length; i++) bytes[i] = bin.charCodeAt(i);
     window.__md = new TextDecoder('utf-8').decode(bytes);
     return JSON.stringify({ b64Len: window.__b64.length, mdLen: window.__md.length });
   })()

4. 校验返回值 b64Len 和 mdLen 均 > 0
```

> **不要用以下方案**（原始对话中均已验证失败）：
> - 普通 `python -m http.server`：无 CORS 头，fetch 被浏览器拦截
> - 分 chunk 拼 `window.__b64 +=`：边界丢字符，atob 报 InvalidCharacterError
> - 直接 inline base64 到 JS text 参数：转义问题导致语法错误

#### 3.1 设置标题

```js
// form_input 对 #txtTitle 赋值即可；JS 兜底：
const s = Object.getOwnPropertyDescriptor(HTMLInputElement.prototype, 'value').set;
s.call(document.getElementById('txtTitle'), title);
document.getElementById('txtTitle').dispatchEvent(new Event('input', {bubbles: true}));
```

#### 3.2 加载 marked.js 并注入正文

```js
// 先加载 marked（如果页面没有）
if (typeof marked === 'undefined') {
  await new Promise((res, rej) => {
    const s = document.createElement('script');
    s.src = 'https://cdnjs.cloudflare.com/ajax/libs/marked/12.0.1/marked.min.js';
    s.onload = res; s.onerror = rej;
    document.head.appendChild(s);
  });
}
// md → html → CKEditor
const html = marked.parse(window.__md);
CKEDITOR.instances.editor.setData(html);
// 校验
CKEDITOR.instances.editor.getData().length  // 应 > 0
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

### 6. 收尾

关闭 CORS HTTP server（taskkill 对应 python 进程），然后给用户简短回执：标题、标签列表、草稿箱入口 `https://mp.csdn.net/mp_blog/manage/draft`。不要写长篇解释。

## 默认标签策略

无显式 `tags` 时按内容推断。常见默认组合：

- Java 技术文章 → `Java, spring, jvm, 后端, 面试`
- 前端 → `前端, JavaScript, Vue`
- 算法 → `算法, leetcode, 数据结构`

标签库里没有的关键词一律走「自定义标签」路径（步骤 4.4）。

## Pitfalls

- **`使用 MD 编辑器` 按钮别点**：点了会弹确认并清空富文本内容。直接用 WYSIWYG + `setData(html)` 更稳。
- **标签面板索引与 tab 顺序严格对应**：不要按文本搜索 tab，用 `panels[index]` 定位更快。
- **自定义标签必须用 setter 触发 input 事件**：直接赋值 `inp.value = x` 不会触发 Vue/React 响应。
- **草稿保存成功 ≠ 跳转**：成功后页面仍停在编辑器，要看 `.el-message` 文本确认。
- **`CKEDITOR.instances.editor` 为空时**：可能是 micro-frontend 未渲染完，再等 2s 重试；若还失败，放弃并告诉用户。
- **登录态失效**：任何环节跳回 `passport.csdn.net` → 停住，让用户手动登录再继续。
- **大文件注入**：正文 Markdown 超过约 4KB 时，不要直接把字符串拼到 `javascript_tool` 的 `text` 参数里，会触发 atob 截断/非法字符错误。正确做法：先用 Python `base64.b64encode` 编码并写到工作目录（如 `ttl.b64`），再启动 CORS HTTP 服务器（`Access-Control-Allow-Origin: *`），在浏览器里用 `fetch('http://127.0.0.1:<port>/<file>.b64').then(r=>r.text())` 拉到 `window.__ttlB64`，最后 `atob + TextDecoder('utf-8')` 解码得到原文。Chrome 扩展在 https 页面里允许 fetch http://127.0.0.1（安全上下文例外）。
- **自定义标签难以添加**：CSDN Vue 组件对 `input[placeholder*="请输入文字搜索"]` 的 Enter 键有严格绑定，通过 setter + `dispatchEvent(new KeyboardEvent('keydown',{key:'Enter',keyCode:13}))` 能返回 `blocked=true` 但标签不会真正入列；用 `computer.type` + `key Enter` 也无法触发。长标签名（如 `TransmittableThreadLocal`）特别容易失败。**兜底策略**：至少选 3-5 个推荐标签兜底（CSDN 只需 ≥1 即可保存），长尾标签让用户到草稿箱编辑页手工加。
- **本地 CORS 服务器用完即关**：`taskkill /PID <pid> /F`，避免端口占用。

> 以下已提升为顶部 Hard Rules，此处仅保留索引：
> - ~~浏览器截图超时~~ → Hard Rule #1
> - ~~navigate 误报 404~~ → Hard Rule #2

## 调用示例

```
用户：把 E:\笔记\八股文\Java ThreadLocal详解.md 发到 CSDN
Agent：
  1. 读文件 → md
  2. tabs_context → navigate i.csdn.net → 已登录
  3. navigate editor → Hard Rule #2 校验放行
  4. CORS server 传数据 → fetch 解码 → marked.parse → setData(html) → 标题="Java ThreadLocal 详解"
  5. 标签：panels[2]点 spring、jvm；panels[8]点 restful；panels[29]点 面试；自定义加 ThreadLocal
  6. 点"保存草稿" → 等成功提示
  7. 关 CORS server → 回执：标题 / 标签 / 草稿箱链接
```

## Verification

发布完成后必须满足：

- [ ] `CKEDITOR.instances.editor.getData().length > 0`（正文非空）
- [ ] `#txtTitle.value` 等于用户给定标题
- [ ] `.el_mcm-tag.is-closable` 数量 ≥ 1
- [ ] 页面 `.el-message` 文本含 `草稿保存成功`
- [ ] CORS server 已关闭
- [ ] 给用户回执包含草稿箱 URL
