---
name: ablesci-download
description: 科研通(AbleSci.com)文献下载自动化 Skill - 支持DOI或文章标题发布求助、智能提取文献信息、自动监控应助状态、高速通道下载、下载后校验题目/作者/DOI、采纳或驳回文件、文件整理的全流程自动化
triggers:
  - 帮我下这篇论文
  - 从科研通下载
  - DOI 求助
  - 帮我找这篇文章
  - 帮我下这篇文章
  - 根据标题求助
  - ablesci
  - 科研通
  - 文献下载
  - 下载文献
  - download paper
---

# 科研通文献下载 (AbleSci Download)

通过科研通(AbleSci.com)发布文献求助、自动监控应助、下载文献、校验文献真实性、采纳或驳回、关闭工单的全流程自动化。

## When to Use
- 用户提供一个 DOI，需要从科研通下载文献 PDF
- 用户只提供文章标题（没有 DOI），需要从科研通下载文献 PDF
- 需要批量发布文献求助
- 需要处理已上传的文献（下载 → 校验 → 采纳或驳回）
- 用户说"帮我下这篇论文"、"从科研通下载"、"DOI 求助"、"帮我找这篇文章"

## Prerequisites
- 科研通账号已登录（cookies 保存在 OpenClaw 浏览器中）
- Node.js 可用
- 用户的积分 > 10（发布求助最低要求）
- 用户的未处理求助 < 限制（积分 < 500 时有限制）
- 校验 PDF 需要 Python（pypdf/pdfplumber）或 pdftotext（poppler），二选一

## 核心流程

### 第一步：发布文献求助

支持两种模式，根据用户提供的信息选择：

**模式 A：DOI 一键求助（推荐，有 DOI 时）**

1. 打开科研通发布页：`https://www.ablesci.com/assist/create`
2. 在 **一键求助** 输入框（`#onekey`）中输入 DOI
3. 点击 **智能提取文献信息** 按钮（`.onekey-search`）
4. 等待弹窗出现，验证提取出的标题/作者/DOI 与用户期望一致
5. 点击 **信息正确，直接发布**（`.layui-layer-btn0`）
6. 弹窗关闭后，点击 **立即发布**（`#form-submit-btn`）完成提交

**模式 B：标题手动填写（只有文章题目时）**

1. 打开科研通发布页：`https://www.ablesci.com/assist/create`
2. **先获取表单结构**：通过 CDP 在页面执行 JS，列出所有可见输入框的 `name`/`id`/`placeholder`，确认标题字段位置
3. 在标题输入框（可能的选择器：`input[name="title"]`、`#title`、`.form-item input[name*="title"]`，实际以页面为准）填入用户给出的文章题目
4. 如用户还提供了作者、期刊、年份、DOI 等信息，一并填入对应字段（找不到对应字段就跳过）
5. 点击 **立即发布**（`#form-submit-btn`）
6. 确认页面跳转到 `https://www.ablesci.com/my/assist-my`

**关键选择器：**
| 元素 | 选择器 |
|------|--------|
| DOI 输入框 | `#onekey` |
| 智能提取按钮 | `.onekey-search` |
| 弹窗确认按钮 | `.layui-layer-btn0` |
| 立即发布按钮 | `#form-submit-btn` |
| 弹窗内容 | `.layui-layer-dialog` |
| 标题输入框（模式B） | 以页面实际结构为准（`input[name="title"]` 等） |

**发布后必须记录求助元数据（用于后续校验）：**
- 目标标题（原文，尽量完整）
- 目标作者（如已知）
- 目标 DOI（如已知）
- 发布时间

**注意事项：**
- 同一 DOI 一小时内不能重复发布，会提示"系统检测到您在1小时内发布了相同DOI的文献求助"
- 如果有未处理的已上传文件，需先处理（下载 + 校验 + 采纳或驳回）才能发布新求助
- 模式 B 若页面没有独立标题输入框，则退回到模式 A：把标题粘贴到 `#onekey` 尝试智能提取，失败则提示用户补充 DOI
- 发布后页面会自动跳转到 `https://www.ablesci.com/my/assist-my`

---

### 第二步：自动监控应助状态（发布后必须执行）

求助发布成功后，进入持续自动监控阶段。**此步骤不可跳过。**

**监控参数：**
| 参数 | 值 | 说明 |
|------|-----|------|
| 轮询间隔 | 60 秒 | 每 1 分钟扫描一次 |
| 最大监控时长 | 7200 秒 | 2 小时 |
| 超时行为 | 任务终止 | 通知用户"X 小时内无人应助" |

**监控流程：**

```
发布完成 → 记录求助元数据（标题/作者/DOI）+ 开始时间
     ↓
┌─→ 导航到 my/assist-my，检查求助列表第一项
│        ↓
│   状态 = "求助中" ?
│     ├─ 是 → 检查是否超过 2 小时
│     │        ├─ 未超时 → 等待 60 秒 → 回到 ┌─→
│     │        └─ 已超时 → 通知用户"2小时内无人上传文件" → 结束
│     └─ 否（待确认/已完结/已关闭）
│              ↓
│         状态 = "待确认" ?
│           ├─ 是 → 进入【自动下载流程】（见下方）
│           └─ 否（已完结）→ 文件已被其他人采纳 → 通知用户 → 结束
```

**状态判断 — CDP Runtime.evaluate 示例：**

```javascript
// 在 my/assist-my 页面执行，获取第一条求助的状态
const result = await send('Runtime.evaluate', {
  expression: `(() => {
    const rows = document.querySelectorAll('.assist-list .item, table tr, .layui-table tbody tr');
    // 根据实际页面结构调整选择器
    const first = rows[0];
    if (!first) return null;
    const statusEl = first.querySelector('[class*="status"], .label, .badge, td:last-child');
    const doiEl = first.querySelector('a[href*="detail"]');
    return {
      status: statusEl?.textContent?.trim() || 'unknown',
      detailUrl: doiEl?.href || '',
      title: doiEl?.textContent?.trim() || ''
    };
  })()`,
  awaitPromise: false,
  returnByValue: true
});
```

**特别注意：** 页面选择器可能随科研通更新变化。首次扫描时先获取页面 HTML 结构，确认实际选择器后再进行状态判断。

---

### 第三步：自动下载 + 校验 + 处理（检测到「待确认」时触发）

以下操作全部自动化，无需人工干预。

#### 3.1 下载文献（实测 2026-08 页面结构）

1. 从第二步获取的 `detailUrl` 进入求助详情页（`https://www.ablesci.com/assist/detail?id=求助ID`）
2. **文件 ID ≠ 求助 ID**：在时间线中找到 `.pdf` 文件链接（`<a class="able-link name" href="/assist/download?id=文件ID">vcln-3c8m.pdf</a>`），或者读 hidden input `.assist-file-id` 的 value，得到**文件 ID**
3. 点击文件名进入下载页：`https://www.ablesci.com/assist/download?id=文件ID`
4. **高速通道下载**（实测有效流程）：
   - 下载页有两个区块：普通线路按钮 `button.download-progress-line-button[data-server-id]`（如 2/3/4）和高速通道按钮 `button#download-highspeed-direct`
   - 点击 `#download-highspeed-direct` → 弹出 Layui 确认框「高速通道提示」（积分<500 扣 2 积分，≥500 免费）→ 点击「确认使用高速通道」按钮
   - 页面随即显示实时进度（「已接收 X KB / 5.0 MB」），下载完成显示「已接收」并**自动保存到浏览器下载目录**
   - **关键：Edge headless 下 `Browser.setDownloadBehavior` 可能不生效，文件实际落在系统默认下载目录（如 `C:/Users/<user>/Downloads`），文件名形如 `vcln-3c8m(科研通-ablesci.com).pdf`**
5. 底层 API（可选，用于直接抓取 URL）：
   ```
   POST https://www.ablesci.com/file/request-download-token
   Content-Type: application/x-www-form-urlencoded
   _csrf=<csrf>&type=assistFile&id=<文件ID>&channel=highspeed&highspeed=1&fallback=0&file_server=0
   → 响应体里含 https://filehub14vip.ablesci.com/file/download?token=...
   ```
6. 用 `curl -L -o` 或直接从浏览器下载目录取文件

**下载页关键选择器（实测）：**
| 元素 | 选择器 |
|------|--------|
| 高速通道按钮 | `#download-highspeed-direct` |
| 普通线路按钮 | `button[data-server-id]` |
| 高速确认弹窗 | `.layui-layer-dialog .download-progress-highspeed-layer` |
| 确认按钮 | 弹窗内文本含「确认使用高速通道」的按钮 |

**常见坑：**
- 文件状态「待审核」时下载页会提示「文件不存在」——需等待审核通过（通常几分钟）
- 普通线路极慢（8KB/s 甚至更慢），优先高速通道
- 不要用 curl 直接请求 filehub URL 时忽略 `User-Agent`，浏览器请求带了完整 UA 才能下载

#### 3.2 校验下载的文献（下载后必做，不可跳过）

目标：确认下载到的 PDF 就是求助的那篇文献，再决定采纳还是驳回。

**提取 PDF 元信息（标题、作者、DOI）：**

方式 1：Python（pypdf，推荐，可同时读元数据和首页文本）
```bash
python -c "
from pypdf import PdfReader
r = PdfReader(r'<PDF路径>')
meta = r.metadata
title_meta = str(meta.get('/Title','')) if meta else ''
author_meta = str(meta.get('/Author','')) if meta else ''
first_text = (r.pages[0].extract_text() or '')[:3000] if r.pages else ''
print('META_TITLE:', title_meta)
print('META_AUTHOR:', author_meta)
print('PAGE1_TEXT:', first_text.replace(chr(10),' ')[:1500])
"
```

方式 2：pdftotext（poppler）
```bash
pdftotext "<PDF路径>" - | head -c 3000
```

方式 3：Node.js pdf-parse（实测有效；注意新版 API 用类，不是函数）
```bash
npm install pdf-parse
```
```javascript
const { PDFParse } = require('pdf-parse');
const fs = require('fs');
const parser = new PDFParse({ data: fs.readFileSync('<PDF路径>') });
const pdfData = await parser.getText();
const text = pdfData.text || '';   // 全文文本，比元数据更可靠
```

**校验规则（按优先级）：**

| 校验项 | 方法 | 通过条件 |
|--------|------|----------|
| DOI（最可靠） | 在 PDF 全文/元数据中搜索目标 DOI 字符串 | 命中求助记录中的 DOI（忽略大小写；允许 `https://doi.org/` 前缀差异；DOI 通常出现在首页） |
| 标题 | 归一化后比较 | 对求助标题与 PDF 首页文本做归一化（去标点、去空格、统一小写），若求助标题前 60 个字符出现在 PDF 首页文本中，或相似度 ≥ 0.8，判为命中 |
| 作者 | 首页/元数据作者匹配 | 求助记录中的第一作者姓氏（或全名）出现在 PDF 首页文本或元数据 /Author 中 |

**判定规则：**
- **通过**：DOI 命中，**或**（标题命中 且 至少一位作者命中）
- **失败**：以上组合均不满足，判定为「文件与求助不匹配」

> 注意：有些 PDF 提取不到文本（扫描版/加密）。此时先看元数据；元数据也空 → 无法校验 → **按失败处理并驳回**，不要冒险采纳。

#### 3.3 根据校验结果处理

**校验通过 → 采纳并关闭工单（实测 2026-08 流程）：**
1. 回到求助详情页（`/assist/detail?id=求助ID`）
2. 点击 **采纳文件** 按钮（`button[data-type="accept"]`）
3. **连续处理两个弹窗**（实测页面会出现两个 Layui 层）：
   - 第一个信息弹窗（「恭喜您，已经有人上传了文件…」）→ 点击「确定」
   - 第二个确认弹窗（「接受应助 确认接受应助吗？确认后，该求助将完结，应助者将获得积分…」）→ 点击「确定」
4. 底层请求：`POST /assist/file-handle`，data 形如 `_csrf=<csrf>&assist_file_id=<文件ID>&note=&type=accept`，响应 `{"code":0,...}` 即成功
5. 刷新详情页，badge 变为「已完结」，时间线显示「XX 接受了 YY 的文件，本次互助完结」→ 工单关闭

**校验失败 → 驳回文件：**
1. 回到求助详情页
2. 点击 **驳回文件** 按钮（`.btn-handle-file[data-type="reject"]`）
3. 在驳回理由中写明原因，例如：
   `文献不匹配：期望标题=<求助标题>，期望DOI=<目标DOI>；实际检测到标题=<检测到的标题>，作者=<检测到的作者>`
4. 工单回到「求助中」，继续等待其他人应助（监控循环不结束，重新开始第二步轮询）

**关键选择器：**
| 元素 | 选择器 |
|------|--------|
| 采纳文件按钮 | `.btn-handle-file[data-type="accept"]` |
| 驳回文件按钮 | `.btn-handle-file[data-type="reject"]` |

#### 3.4 整理文件名

```bash
node -e "
const fs=require('fs');const path=require('path');
const dir='<paper文件夹路径>';
fs.readdirSync(dir).filter(f=>f.endsWith('.pdf')).forEach(f=>{
  const n=f.replace(/_/g,' ');
  if(n!==f) fs.renameSync(path.join(dir,f),path.join(dir,n));
})
"
```

#### 3.5 完成通知

处理完成后，向用户报告：
- 求助方式（DOI / 标题）
- 求助标题 / 作者 / DOI
- 校验结果（通过 / 失败）
- 处理动作（采纳并关闭工单 / 驳回文件并继续等待）
- 下载路径
- 总耗时（从发布到完成）

---

### 超时处理：无人应助

如果扫描满 2 小时仍为「求助中」状态，执行以下步骤：

1. **不要关闭求助工单**（保留在科研通上，可能有用户稍后应助）
2. 向用户输出以下信息：
   - DOI / 标题
   - 发布求助时间
   - "2 小时内无人上传文件，求助工单仍在科研通上等待应助"
   - 建议用户稍后手动检查或重新提交

---

## 手动流程参考（以下步骤为自动化流程的底层实现参考）

### 手动第二步：检查求助状态

导航到 `https://www.ablesci.com/my/assist-my` 查看求助列表。

**状态说明：**
- **求助中** — 尚未有人应助
- **待确认** — 有人上传了文件，需要下载确认
- **已完结** — 已采纳文件，互助完成
- **已关闭** — 超时或违规关闭

每条求助的详情链接格式：`/assist/detail?id=XXXXX`

### 手动第三步：下载文献

当状态为"待确认"时：

1. 进入求助详情页：`https://www.ablesci.com/assist/detail?id=XXXXX`
2. 在时间线中找到上传的文件名（`.pdf` 链接）
3. 点击文件名进入下载页：`https://www.ablesci.com/assist/download?id=XXXXX`
4. 在下载页点击 **高速通道**（节省积分）
5. 获取文件的实际下载 URL（`https://filehub*.ablesci.com/file/download?token=...`）
6. 使用 curl 下载到 paper 文件夹

**下载脚本：**
```bash
mkdir <路径>/paper
curl -L -o "<路径>/paper/<文件名>.pdf" "<下载URL>"
```

---

## CDP 交互方式（当前可用）

OpenClaw 浏览器 Playwright 模块有 bug，使用 CDP (Chrome DevTools Protocol) 直接控制：

```
CDP 端点: http://127.0.0.1:18801/json
浏览器级 WS: ws://127.0.0.1:18801/devtools/browser  (Node 24 兼容性问题)
页面级 WS: ws://127.0.0.1:18801/devtools/page/<PAGE_ID>  (可用)
```

**创建新页面：**
```bash
curl -s -X PUT "http://127.0.0.1:18801/json/new?https://www.ablesci.com/"
```

**Node.js CDP 交互模板：**
```javascript
const WebSocket = globalThis.WebSocket;
const PAGE_WS = 'ws://127.0.0.1:18801/devtools/page/<PAGE_ID>';

const ws = new WebSocket(PAGE_WS);
let msgId = 0;
const pending = new Map();

ws.onmessage = (event) => {
  const msg = JSON.parse(event.data.toString());
  if (msg.id && pending.has(msg.id)) {
    pending.get(msg.id).resolve(msg);
    pending.delete(msg.id);
  }
};

function send(method, params) {
  return new Promise((resolve, reject) => {
    const id = ++msgId;
    pending.set(id, {resolve, reject});
    ws.send(JSON.stringify({id, method, params}));
    setTimeout(() => {
      if (pending.has(id)) { pending.delete(id); reject(new Error('timeout: ' + method)); }
    }, 15000);
  });
}

// 使用示例：
// await send('Page.navigate', {url: 'https://...'});
// await send('Runtime.evaluate', {expression: 'document.title'});
```

**关键 CDP 方法：**
- `Page.navigate` — 页面导航
- `Runtime.evaluate` — 执行 JS（找元素、填表单、点击按钮）

---

## 常见问题

**Q: 弹窗按钮 `.click()` 不生效？**
A: Layui 对话框使用事件委托，有时 `.click()` 仅关闭弹窗但不触发表单提交。需要手动点击"立即发布"按钮作为兜底。

**Q: 发布求助失败："1小时内发布了相同DOI"？**
A: 换一个未提交过的 DOI，或等待一小时后重试。

**Q: 发布求助失败："有未处理的求助"？**
A: 先到"我的求助"中将"待确认"的求助处理掉（下载并校验，通过则采纳、不通过则驳回）。

**Q: 只有标题没有 DOI，怎么发求助？**
A: 用模式 B：在发布页找到标题输入框手动填写，能补充作者/期刊/年份更好。若页面没有标题输入框，把标题粘贴进 `#onekey` 尝试智能提取，失败则提示用户补充 DOI。

**Q: 下载完怎么确认文件对不对？**
A: 必做 3.2 校验：用 pypdf/pdftotext 提取 PDF 元数据和首页文本，对比求助时的标题、作者、DOI。DOI 命中或（标题+作者命中）→ 采纳；否则驳回。

**Q: 校验时 PDF 提取不到文本（扫描版/加密）？**
A: 先看 PDF 元数据（/Title、/Author）。元数据也提取不到 → 视为无法校验，按失败处理，驳回文件，不要冒险采纳。

**Q: 下载链接失效？**
A: 每次进入下载页都会有新的 token。点击文件名重新进入下载页即可获取新链接。

**Q: 自动监控中页面选择器失效？**
A: 科研通前端可能更新导致选择器变化。首次扫描时必须先获取页面 HTML，验证实际 DOM 结构后再做状态判断。

## Anti-patterns
- 不要用没有 token 的直接链接下载（需要 cookie 认证）
- **不要在未校验文献的情况下直接采纳** — 下载后必须先做 3.2 校验
- 校验失败必须驳回文件并写明理由，不要采纳错误文献
- 不要在未采纳文件的情况下反复下载（浪费积分）
- 不要在 Node 24 中使用浏览器级 CDP WebSocket（有兼容性问题，使用页面级）
- 下载后务必及时处理文件（采纳或驳回），否则 48 小时后系统自动采纳
- **超时后不要关闭工单** — 保留工单让其他用户有机会应助
- 不要跳过第二步的自动监控 — 发布后一定要等文件上传
