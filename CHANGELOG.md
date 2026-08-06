# 更新日志 (Changelog)

所有历史版本按上传日期命名，并存放在 `versions/` 目录中。

## [2026-08-06] 实测下载流程完善版

### 更新内容

1. **支持标题模式发求助**
   - 原版本仅支持输入 DOI 自动填充文献信息。
   - 新版支持直接输入文章标题，在发布页表单中手动填写标题、作者、期刊、年份、DOI、原文链接后发布求助。

2. **下载后自动校验，正确才采纳、错误才驳回**
   - 新增 PDF 文本提取与校验步骤。
   - 校验维度：标题、作者、DOI。
   - 命中 DOI 或（标题 + 至少一位作者）则点击「采纳文件」关闭工单；否则驳回并继续监控。
   - 扫描版/加密 PDF 无法提取文本时，按失败处理并驳回。

3. **高速通道下载实测流程**
   - 文件上传后进入 `/assist/download?id=文件ID`。
   - 点击「高速通道」→ 积分确认弹窗 → 页面实时进度。
   - 完成后文件落到浏览器默认下载目录，脚本再复制到目标位置。
   - 记录 `/file/request-download-token` 接口与参数。

4. **采纳流程双弹窗**
   - 点击「采纳文件」后存在两个连续弹窗（确认采纳 + 确认完结）。
   - 底层请求为 `POST /assist/file-handle (type=accept)`，成功返回 `{"code":0}`。

5. **SKILL.md 结构补全**
   - 新增 YAML frontmatter（name / description / triggers / author / version）。
   - 新增 FAQ 与 Anti-patterns 章节。

6. **README.md 重写**
   - 补充功能说明、触发词、安装方式、使用前提、版本目录说明。

---

## [2026-05-31] 初始版

### 功能

- 科研通 (AbleSci.com) 文献下载自动化 Skill。
- 输入 DOI → 自动提取文献信息 → 发布求助。
- 自动监控应助状态（每 60 秒轮询，最多 2 小时）。
- 检测到文件上传后下载 PDF。
- 整理文件名并保存到本地 `paper/` 目录。

### 注意

- 此版本未做下载后校验，直接下载后结束流程。
- SKILL.md 无 frontmatter，格式不符合 WorkBuddy Skill 规范，需手动适配后安装。

---

## 版本命名规则

- `versions/SKILL-YYYY-MM-DD.md`
- `versions/README-YYYY-MM-DD.md`
- 日期为该版本实际上传/发布的日期。
- 根目录的 `SKILL.md` 和 `README.md` 始终为最新版本，方便直接安装。

---

## 如何回滚

```bash
# 查看历史版本
git log --oneline

# 恢复到某个日期版本（以 2026-05-31 为例）
git checkout 2026-05-31 -- versions/SKILL-2026-05-31.md
cp versions/SKILL-2026-05-31.md SKILL.md
cp versions/README-2026-05-31.md README.md
```

或直接在 GitHub 页面进入 `versions/` 目录下载对应日期的文件。

---

*本 CHANGELOG 随每次版本更新而追加，确保每个版本的改动都清晰可查。*
