# ablesci-download

科研通(AbleSci.com)文献下载自动化 Skill — 支持 DOI 或文章标题发布求助、智能提取文献信息、自动监控应助状态、高速通道下载、下载后校验题目/作者/DOI、采纳或驳回文件、文件整理的全流程自动化。

## 功能

1. **发布文献求助**：支持 DOI 一键求助，也支持只有文章标题时手动填写发布
2. **自动监控应助**：每 60 秒轮询一次，最多监控 2 小时，检测到文件上传自动触发下载
3. **高速通道下载**：自动选择高速下载线路（积分 < 500 扣 2 积分，≥ 500 免费）
4. **下载后校验**：提取 PDF 全文，校验标题 / 作者 / DOI 是否与求助一致
5. **采纳或驳回**：校验通过自动采纳并关闭工单；校验失败自动驳回并写明理由，继续等待应助
6. **文件整理**：下载完成后自动重命名并移动到目标文件夹

## 安装

将本仓库的 `SKILL.md` 放入 WorkBuddy 的 skills 目录：

- 用户级：`~/.workbuddy/skills/ablesci-download/SKILL.md`
- 项目级：`.workbuddy/skills/ablesci-download/SKILL.md`

## 使用前提

- 科研通账号已登录（cookies 保存在浏览器中）
- Node.js 可用
- 用户的积分 > 10（发布求助最低要求）
- CDP 浏览器环境（Edge/Chrome 带 `--remote-debugging-port=18801` 启动）

## 使用方法

直接对 AI 助手说：

- "帮我下这篇论文，DOI 是 xxx"
- "帮我下这篇文章，标题是 xxx"
- "从科研通下载 xxx"

## 技术要点（2026-08 实测）

| 环节 | 关键点 |
|------|--------|
| 登录 | 入口 `/site/login`，表单提交返回 `{"code":0,"msg":"登录成功"}` |
| 发布 | 发布页 `/assist/create`（不是 `/assist/add`） |
| 文件ID | 详情页 hidden input `.assist-file-id`，≠ 求助 ID |
| 下载 | 下载页点 `#download-highspeed-direct` → 弹窗确认 → 实时进度 → 自动保存到浏览器下载目录 |
| 高速API | `POST /file/request-download-token`，参数 `channel=highspeed&highspeed=1` |
| 校验 | Node `pdf-parse`（新版 `new PDFParse({data}).getText()`）提取全文 |
| 采纳 | `button[data-type="accept"]` → 两个确认弹窗 → `POST /assist/file-handle (type=accept)` |

## 免责声明

本站互助的所有文件仅供个人学习研究使用。请遵守相关知识产权规定，勿将文件分享给他人或用于盈利。
