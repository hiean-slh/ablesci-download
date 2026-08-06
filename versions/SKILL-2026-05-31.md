# 科研通文献下载 (AbleSci Download)

通过科研通(AbleSci.com)发布文献求助、下载文献、采纳文件的全流程自动化。

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

### 第二步：检查求助状态

导航到 `https://www.ablesci.com/my/assist-my` 查看求助列表。

**状态说明：**
- **求助中** — 尚未有人应助
- **待确认** — 有人上传了文件，需要下载确认
- **已完结** — 已采纳文件，互助完成
- **已关闭** — 超时或违规关闭

每条求助的详情链接格式：`/assist/detail?id=XXXXX`

### 第三步：下载文献

当状态为"待确认"时：

1. 进入求助详情页：`https://www.ablesci.com/assist/detail?id=XXXXX`
2. 在时间线中找到上传的文件名（`.pdf` 链接）
3. 点击文件名进入下载页：`https://www.ablesci.com/assist/download?id=XXXXX`
4. 在下载页点击 **高速通道**（节省积分）
5. 获取文件的实际下载 URL（`https://filehub*.ablesci.com/file/download?token=...`）
6. 使用 curl 下载到 paper 文件夹

**下载脚本：**
```bash
# 创建 paper 文件夹
mkdir <路径>/paper

# 使用 curl 下载（-L 跟随重定向）
curl -L -o "<路径>/paper/<文件名>.pdf" "<下载URL>"
```

### 第四步：采纳文件

下载完成、确认文件正确后：

1. 回到求助详情页
2. 点击 **采纳文件** 按钮（`button.btn-handle-file[data-type="accept"]`）
3. 在确认弹窗中点击确定
4. 状态变为"已完结"，积分结算

**关键选择器：**
| 元素 | 选择器 |
|------|--------|
| 采纳文件按钮 | `.btn-handle-file[data-type="accept"]` |
| 驳回文件按钮 | `.btn-handle-file[data-type="reject"]` |

### 第五步：整理文件名

下载后统一将文件名中的下划线替换为空格：

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

## 常见问题

**Q: 弹窗按钮 `.click()` 不生效？**
A: Layui 对话框使用事件委托，有时 `.click()` 仅关闭弹窗但不触发表单提交。需要手动点击"立即发布"按钮作为兜底。

**Q: 发布求助失败："1小时内发布了相同DOI"？**
A: 换一个未提交过的 DOI，或等待一小时后重试。

**Q: 发布求助失败："有未处理的求助"？**
A: 先到"我的求助"中将"待确认"的求助处理掉（下载并采纳或驳回）。

**Q: 下载链接失效？**
A: 每次进入下载页都会有新的 token。点击文件名重新进入下载页即可获取新链接。

## Anti-patterns
- 不要用没有 token 的直接链接下载（需要 cookie 认证）
- 不要在未采纳文件的情况下反复下载（浪费积分）
- 不要在 Node 24 中使用浏览器级 CDP WebSocket（有兼容性问题，使用页面级）
- 下载后务必采纳文件，否则 48 小时后系统自动采纳
