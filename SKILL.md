# 科研通文献下载 (AbleSci Download)

通过科研通(AbleSci.com)发布文献求助、自动监控应助、下载文献、采纳文件的全流程自动化。

## When to Use
- 用户提供一个 DOI，需要从科研通下载文献 PDF
- 需要批量发布文献求助
- 需要处理已上传的文献（下载 + 采纳）
- 用户说"帮我下这篇论文"、"从科研通下载"、"DOI 求助"

## Prerequisites
- 科研通账号已登录（cookies 保存在 OpenClaw 浏览器中）
- Node.js 可用
- 用户的积分 > 10（发布求助最低要求）
- 用户的未处理求助 < 限制（积分 < 500 时有限制）

## 核心流程

### 第一步：发布文献求助

1. 打开科研通发布页：`https://www.ablesci.com/assist/create`
2. 在 **一键求助** 输入框（`#onekey`）中输入 DOI
3. 点击 **智能提取文献信息** 按钮（`.onekey-search`）
4. 等待弹窗出现，验证 DOI 正确
5. 点击 **信息正确，直接发布**（`.layui-layer-btn0`）
6. 弹窗关闭后，点击 **立即发布**（`#form-submit-btn`）完成提交

**关键选择器：**
| 元素 | 选择器 |
|------|--------|
| DOI 输入框 | `#onekey` |
| 智能提取按钮 | `.onekey-search` |
| 弹窗确认按钮 | `.layui-layer-btn0` |
| 立即发布按钮 | `#form-submit-btn` |
| 弹窗内容 | `.layui-layer-dialog` |

**注意事项：**
- 同一 DOI 一小时内不能重复发布，会提示"系统检测到您在1小时内发布了相同DOI的文献求助"
- 如果有未处理的已上传文件，需先处理（下载 + 采纳或驳回）才能发布新求助
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
发布完成 → 记录 DOI + 开始时间
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

### 第三步：自动下载 + 采纳 + 整理（检测到「待确认」时触发）

以下操作全部自动化，无需人工干预。

#### 3.1 下载文献

1. 从第二步获取的 `detailUrl` 进入求助详情页
2. 在时间线中找到 `.pdf` 文件链接
3. 点击文件名进入下载页：`https://www.ablesci.com/assist/download?id=XXXXX`
4. 点击 **高速通道**（节省积分）
5. 通过 CDP 拦截或页面提取获取实际下载 URL（`https://filehub*.ablesci.com/file/download?token=...`）
6. 用 `cmd /c curl -L -o` 下载 PDF 到本地 paper 文件夹

**CDP 提取下载链接示例：**
```javascript
// 在下载页获取高速通道的实际下载 URL
const downloadUrl = await send('Runtime.evaluate', {
  expression: `document.querySelector('a[href*="download"], .high-speed a, a:contains("高速通道")')?.href || ''`,
  awaitPromise: false,
  returnByValue: true
});
```

#### 3.2 采纳文件

下载完成后立即采纳：

1. 回到求助详情页
2. 点击 **采纳文件** 按钮（`.btn-handle-file[data-type="accept"]`）
3. 确认弹窗

**关键选择器：**
| 元素 | 选择器 |
|------|--------|
| 采纳文件按钮 | `.btn-handle-file[data-type="accept"]` |
| 驳回文件按钮 | `.btn-handle-file[data-type="reject"]` |

#### 3.3 整理文件名

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

#### 3.4 完成通知

下载 + 采纳 + 改名全部完成后，向用户报告：
- DOI
- 文献标题
- 下载路径
- 总耗时（从发布到完成）

---

### 超时处理：无人应助

如果扫描满 2 小时仍为「求助中」状态，执行以下步骤：

1. **不要关闭求助工单**（保留在科研通上，可能有用户稍后应助）
2. 向用户输出以下信息：
   - DOI
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
A: 先到"我的求助"中将"待确认"的求助处理掉（下载并采纳或驳回）。

**Q: 下载链接失效？**
A: 每次进入下载页都会有新的 token。点击文件名重新进入下载页即可获取新链接。

**Q: 自动监控中页面选择器失效？**
A: 科研通前端可能更新导致选择器变化。首次扫描时必须先获取页面 HTML，验证实际 DOM 结构后再做状态判断。

## Anti-patterns
- 不要用没有 token 的直接链接下载（需要 cookie 认证）
- 不要在未采纳文件的情况下反复下载（浪费积分）
- 不要在 Node 24 中使用浏览器级 CDP WebSocket（有兼容性问题，使用页面级）
- 下载后务必采纳文件，否则 48 小时后系统自动采纳
- **超时后不要关闭工单** — 保留工单让其他用户有机会应助
- 不要跳过第二步的自动监控 — 发布后一定要等文件上传
