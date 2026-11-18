# Antigravity 汉化定制仓库 — 智能体接手与维护手册 (MAINTENANCE.md)

本文档专为后续接手维护本项目的 AI 智能体（Antigravity / Claude Code / Codex 等）及开发者编写，记录本仓库与上游项目的关系、本机部署拓扑、定制增强清单及客户端升级后的标准维护流程（SOP）。

---

## 1. 仓库定位与远程分支结构

本仓库基于开源项目 [`Silas-02/antigravity-desktop-cn`](https://github.com/Silas-02/antigravity-desktop-cn) 定制增强，采用双远程（Dual-Remote）结构维护：

- **`origin`**：`git@github.com:cixia-zhao/antigravity-desktop-cn.git`（保存本机 Linux 适配修复、Electron 主进程原生弹窗深度汉化补丁及增量词典）。
- **`upstream`**：`git@github.com:Silas-02/antigravity-desktop-cn.git`（上游社区仓库，使用 SSH 地址可直接走本机 `~/.ssh/config` 代理通道同步新版词典与监控模块）。

---

## 2. 本机部署拓扑与联动文件（维护前必读）

在本机（Linux）环境中，Antigravity 的完整中文化由**「客户端界面汉化（本仓库）」+「官方全局中文规则」**两部分协同实现：

| 组件 | 本机路径 | 说明与维护禁忌 |
| :--- | :--- | :--- |
| **本地工作区仓库** | `/home/cixia/.gemini/antigravity/scratch/antigravity-desktop-cn` | 本汉化工程源码目录，可设为 Antigravity 工作区进行维护。 |
| **客户端安装软链** | `/home/cixia/.local/opt/antigravity` | 符号链接，指向实际版本目录（例如 `antigravity-2.33.0`）。客户端升级后通常更新此软链目标。 |
| **核心资源包与备份** | `/home/cixia/.local/opt/antigravity/resources/app.asar` | 注入目标文件；首次注入时自动生成官方原版备份 `app.asar.bak`。每次重复注入均从 `app.asar.bak` 纯净源重新构建，幂等安全。 |
| **代理启动包装脚本** | `/home/cixia/.local/bin/antigravity` | 负责加载本机代理环境（`phone-clash-env.sh` / `--proxy-server`）并启动客户端。**严禁覆盖、删除或绕过该包装脚本**。 |
| **官方全局中文规则** | `/home/cixia/.gemini/config/rules/zh-cn.md` | `trigger: always_on` 常驻全局规则，从源头要求模型在对话回复、计划交付件（Artifacts）与可见思考过程（Thinking）中使用简体中文，同时保持英文代码标识符与命令原样。 |
| **内置技能源目录** | `/home/cixia/.gemini/antigravity/builtin/skills/` | 客户端内置技能目录，升级客户端后可扫描各子目录 `SKILL.md` 的 `name` 与 `description` 检查是否有新增未汉化技能。 |

---

## 3. 相较上游仓库的定制增强清单

合并 `upstream/main` 时，请务必保留以下四部分定制增强逻辑：

### 3.1 Linux 路径自动探测与包装脚本保护（`localization_engine.js`）
- **`addHomeCandidates()`**：新增 `~/.local/opt/antigravity`、`~/.local/opt/Antigravity`、`~/opt/antigravity`、`~/opt/Antigravity` 目录探测。
- **`addSystemCandidates()`**：解析 `which antigravity` 命中的启动脚本（如 `~/.local/bin/antigravity`）时，自动将脚本内的 `$HOME`、`${HOME}`、`~/` 展开为实际主目录后再提取安装路径。
- **Linux 重启逻辑（`main()`）**：优先通过 `command -v antigravity` 调用 `PATH` 中的包装启动脚本（保留代理参数），未命中时再回退至安装目录内的二进制文件。

### 3.2 Electron 主进程原生弹窗与独立页面深度汉化（`localization_engine.js`）
上游 `preload.js` 仅能翻译主渲染器（`127.0.0.1` / `localhost`）及 `plugin://` 子框架 DOM，无法触达 Electron 主进程原生系统对话框与 `data:text/html` 独立窗口。本仓库在 `installLocalization()` 中扩展了以下补丁模块（动态锚点总数由上游 20 项扩展至 **107 项**）：
- **`7. dist/main.js`**：退出确认弹窗（`Confirm Quit`）、Dock 新建窗口菜单、启动失败/二进制缺失报错弹窗、WSL 分发版缺失提示框。
- **`9. dist/ipcHandlers.js`**：打开工作区/多工作区（`Open workspace` / `Open workspaces`）、选择下载文件夹（`Select Download Folder`）原生文件选择器标题、WSL 文件夹错误提示，以及右键菜单 `Reveal in File Manager`。
- **`9.1 dist/webviewPopupSecurity.js`**：内置浏览器弹窗安全状态标签与证书连接信息弹窗（`Connection info`）。
- **`9.2 dist/deviceChooser.js`**：WebHID / WebUSB / 串口外设连接选择原生弹窗（`Connect a device`）。
- **`9.3 dist/webauthn.js`**：通行密钥（Passkey）账号选择原生弹窗（`Choose a Passkey`）。
- **`9.4 dist/ideInstall/wizardHtml.js`**：首次启动独立 IDE 安装向导页（`Welcome to the new Antigravity!`）。
- **`9.41 dist/remote/disconnectedPage.js`**：SSH 远程窗口断线、重连中、服务端重启中独立状态页。
- **`9.42 dist/remote/remoteManager.js`**：SSH 配置文件选择、连接失败、停止/重启远端服务端、麦克风权限、组织/回环登录确认、共享主目录接管等全部原生弹窗。
- **`9.43 dist/remote/sshPrompts.js`**：SSH 密码/私钥口令/PIN 输入独立模态窗及安全密钥轻触提示窗。
- **`9.44 dist/computer/hud.js` & `9.45 dist/computer/hudState.js`**：Computer Use 桌面控制悬浮 HUD 状态栏与停止/继续按钮。
- **打包前全量语法守卫**：在写入 `app.asar.tmp` 前，对 `filePatches` 中所有 `.js` 模块执行 `new vm.Script()` 语法编译校验，任何语法异常立即中止写入，杜绝客户端白屏或崩溃。

### 3.3 只读巡检脚本增强（`scripts/ag_audit.js`）
- `extractEnginePatchSpecs()` 支持识别小数编号段落（如 `// 9.1` ~ `// 9.45`）以及子目录模块路径（如 `dist/remote/*.js`、`dist/ideInstall/*.js`、`dist/computer/*.js`），并在 `// 9.5` 处精确收束边界。

### 3.4 词典与内置技能补充（`dicts/` & `SKILL_PICKER_DISPLAY_ENTRIES`）
- `dicts/release.json` & `localization_engine.js`：补充 `v2.33.0` 新增内置技能 `swarm_narrator`（蜂群叙事）、`swarm_pane`（蜂群面板）及右键菜单词条。
- `dicts/marketplace_plugins.json`：补充精选合作伙伴插件（Mobbin、Linear、Auth0）描述译文。

---

## 4. 客户端版本升级后的标准维护 SOP（供后续 Agent 执行）

当本机 Antigravity 客户端升级到新版本（例如 `v2.34.x`）后界面恢复英文，或出现新增未翻译文案时，接手 Agent 请严格按以下步骤执行：

### 步骤 1：检查本机版本与备份状态
```bash
ls -la /home/cixia/.local/opt/antigravity /home/cixia/.local/opt/antigravity/resources/
```
- 确认软链指向的新版本目录中 `app.asar` 为全新官方包（若旧目录升级覆盖导致残留了旧版 `app.asar.bak`，先核对时间戳与大小，切勿用旧版备份覆盖新版客户端）。

### 步骤 2：同步上游更新（可选但推荐）
```bash
cd /home/cixia/.gemini/antigravity/scratch/antigravity-desktop-cn
git fetch upstream
git merge upstream/main
```
- 如遇冲突，保留上游新增的 `dicts/*.json` 词条与 DOM 规则，同时保留本仓库第 3 节列出的 Linux 路径探测增强、`dist/*.js` 原生弹窗补丁及 `scripts/ag_audit.js` 正则增强。

### 步骤 3：执行全量只读体检（核心验证关卡）
```bash
node -c localization_engine.js
node scripts/ag_audit.js check
```
检查三项输出：
1. **词典规范质检 (`dicts`)**：确保 `跨文件重键: 0 | 大小写冲突: 0 | 标点规范差异: 0`（注意：`dicts/*.json` 内必须严格遵循英文半角标点 `.`、`:`、`?`、`()` 1:1 对齐规范，详见 `AGENTS.md`）。
2. **注入脚本沙箱校验 (`engine`)**：确保 `generateJs() 只读沙箱编译通过`。
3. **桌面壳动态锚点体检 (`asar`)**：确保全部锚点均为 `[OK]`。若某个锚点报 `[BROKEN]`，说明新版客户端微调了对应 `dist/*.js` 原文，可将新版 `app.asar` 解包到临时目录查阅最新源码并同步更新 `localization_engine.js` 中的 `.replace()` 目标字符串。

### 步骤 4：热注入部署（Agent 会话内严禁强杀宿主！）
> **警告**：如果当前 Agent 正在 Antigravity 客户端内部运行，**必须携带 `--no-kill` 参数**！否则默认流程会调用 `closeAntigravityProcesses()` 强杀当前客户端进程，导致会话中断。

```bash
node /home/cixia/.gemini/antigravity/scratch/antigravity-desktop-cn/localization_engine.js --no-kill --brand-title english
```
部署成功后，提示用户**手动退出并重启一次 Antigravity** 即可生效（同时保留输入框左下角的上下文窗口与速率监控圆环）。

### 步骤 5：提交并推送到个人仓库
```bash
cd /home/cixia/.gemini/antigravity/scratch/antigravity-desktop-cn
git status
git diff
git add -A
git commit -m "chore: sync and update localization for Antigravity vX.Y.Z"
git push origin main
```

---

## 5. 快速故障排查与应急回滚

- **一键无损回滚官方英文原版**（客户端关闭状态下执行，或在会话内加 `--no-kill` 后手动重启）：
  ```bash
  /home/cixia/.gemini/antigravity/scratch/antigravity-desktop-cn/uninstall.sh
  # 或在 Antigravity 会话内热还原（重启生效）：
  node /home/cixia/.gemini/antigravity/scratch/antigravity-desktop-cn/localization_engine.js --huifu --no-kill
  ```
- **输入框左下角 Token 监控显示“待采样”**：发送一条新对话消息后即会自动通过内置 `languageServer` 通道刷新采样。
- **思考过程或回复未用中文**：检查全局规则 `/home/cixia/.gemini/config/rules/zh-cn.md` 是否存在且未被禁用。
