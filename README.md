# 用 Claude Code + wechat-cli 导出微信聊天记录

> 让 AI 帮你把微信里几千上万个会话一次性导出成 Markdown / 纯文本，按人/群分文件管理。

## 这份指南能做什么

- **一键导出全部聊天记录**：个人聊天每人一个文件，群聊可合并为一个文件
- **本地运行，数据不出机器**：所有解密和查询都在你自己的电脑上完成
- **AI 辅助操作**：通过 Claude Code 让 AI 自动调用 `wechat-cli`，适合导出几千上万个会话的场景
- **跨平台**：支持 macOS、Windows、Linux

底层工具 [wechat-cli](https://github.com/freestylefly/wechat-cli) 直接从本地微信数据库读取消息，支持查看会话、聊天记录、搜索、联系人、群成员、统计、导出等功能。

## 前提条件

- 微信桌面版 4.0+ 已安装并登录
- Node.js v18+（macOS 用 npm 安装时需要）或 Python 3.10+（Windows 用 pip 安装时需要）
- [Claude Code](https://claude.com/claude-code) 已安装（推荐但非必须，也可以手动跑命令）

> **💡 模型建议**：如果 AI 执行过程中频繁出错、理解偏差大或批量任务卡住，优先尝试**切换到更强的模型**（如 Claude Opus / Sonnet 最新版）。导出几千个会话、写批处理脚本、排查密钥和路径问题这类工作，模型能力差异会很明显，换个更好的模型通常比反复调试更省时间。

---

## macOS 使用指南

### 第一步：关闭 SIP（System Integrity Protection）

提取密钥需要读取微信进程内存，必须关闭 SIP。

1. 关机
2. 长按电源键开机，进入恢复模式
3. 顶部菜单栏 → 实用工具 → 终端
4. 执行 `csrutil disable`
5. 重启

> 完成全部导出后记得重新开启 SIP（同样进入恢复模式，执行 `csrutil enable`）。

### 第二步：给终端授予"完全磁盘访问权限"

1. 打开 **系统设置 → 隐私与安全性 → 完全磁盘访问权限**
2. 把你用的终端应用加进去（Terminal、iTerm2，或 IDE 自带终端等）
3. 重启终端

如果不给这个权限，工具读不到微信数据目录，密钥提取会失败。

### 第三步：安装 wechat-cli

```bash
npm install -g @canghe_ai/wechat-cli
```

> macOS 的 npm 版本目前只打包了 **arm64 二进制**。Intel Mac 请改用 pip 安装（见 Windows 章节的 pip 步骤，通用）。

验证安装：

```bash
wechat-cli --version
# 输出 0.2.4 或更高
```

### 第四步：提取微信数据库密钥

确保微信已打开并登录，然后用管理员权限初始化：

**方法 A：在 Claude Code 中直接运行（会弹系统密码框）**

```bash
/usr/bin/osascript -e 'do shell script "wechat-cli init 2>&1" with administrator privileges'
```

**方法 B：在 Terminal 里手动运行**

```bash
sudo wechat-cli init
```

成功后会输出类似：

```
[+] 提取到 X 个密钥，保存到: ~/.wechat-cli/all_keys.json
[+] 初始化完成!
```

#### 常见问题

**1. 配置指向了错误的微信账号目录**

如果本机登录过多个微信账号，`init` 可能自动选中了旧账号。检查：

```bash
cat ~/.wechat-cli/config.json
# 看 db_dir 里的 wxid 是不是当前登录的账号

ls -lt ~/Library/Containers/com.tencent.xinWeChat/Data/Documents/xwechat_files/
# 按修改时间排序，最新的那个目录通常是当前账号
```

如果不对，直接编辑 `~/.wechat-cli/config.json`，把 `db_dir` 改成正确的 `wxid_xxx/db_storage` 路径。

**2. 配置文件权限属于 root**

因为用了 sudo 跑 init，修复一下：

```bash
sudo chown -R $(whoami):staff ~/.wechat-cli/
```

**3. `task_for_pid failed` 错误**

这是 macOS 的进程内存访问限制。wechat-cli 会自动尝试重新签名微信并提示你后续步骤，按提示操作即可：退出微信 → 重开微信登录 → 重跑 `sudo wechat-cli init`。

如果自动重签失败，手动执行（[参考](https://github.com/freestylefly/wechat-cli)）：

```bash
# 先完全退出微信
sudo codesign --force --sign - --entitlements /dev/stdin /Applications/WeChat.app <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.security.get-task-allow</key>
    <true/>
</dict>
</plist>
EOF
```

> 重签名不会导致账号问题，但可能影响微信自动更新。需要更新时直接去[官网](https://mac.weixin.qq.com/)重新下载安装即可，已提取的密钥继续有效。

### 第五步：完成后重新开启 SIP

1. 关机
2. 长按电源键开机 → 恢复模式 → 实用工具 → 终端
3. 执行 `csrutil enable`
4. 重启

日常查询/导出不需要关 SIP。只有微信大版本更新或重新登录导致密钥变化时，才需要再次关 SIP 重跑 `wechat-cli init`。

---

## Windows 使用指南

### 第一步：安装 Python

从 [python.org](https://www.python.org/downloads/) 下载安装 Python 3.10 或更高版本，安装时**勾选 "Add Python to PATH"**。

验证：

```powershell
python --version
# 应输出 Python 3.10 或更高
```

### 第二步：安装 wechat-cli

Windows 使用 pip 安装：

```powershell
pip install wechat-cli
```

验证：

```powershell
wechat-cli --version
```

### 第三步：提取微信数据库密钥

1. 确保微信已打开并登录
2. 以**管理员身份**打开 PowerShell 或 cmd（右键 → "以管理员身份运行"）
3. 执行：

```powershell
wechat-cli init
```

Windows 下工具会直接读取 `Weixin.exe` 进程内存，无需关 SIP 之类的系统安全设置，但**必须用管理员权限**。

成功后密钥会保存在 `%USERPROFILE%\.wechat-cli\` 目录下。

#### 常见问题

**多账号选择**：如果本机登录过多个微信，会提示选择，默认第一个是当前账号。如果选错了，编辑 `%USERPROFILE%\.wechat-cli\config.json` 修正 `db_dir`。

**管理员权限**：普通窗口运行会报权限错误，必须"以管理员身份运行"。

---

## 第六步（通用）：使用 wechat-cli

初始化完成后，以下命令在任何平台都一样，也不再需要 sudo / 管理员权限。

### 查看最近会话

```bash
wechat-cli sessions --limit 20 --format text
```

### 查看聊天记录

```bash
wechat-cli history "联系人或群聊名称" --limit 50 --format text
```

### 搜索消息

```bash
# 全局搜索
wechat-cli search "关键词" --limit 20 --format text

# 在指定聊天中搜索
wechat-cli search "关键词" --chat "聊天名称" --limit 20 --format text
```

### 导出单个聊天

```bash
# 导出为 Markdown
wechat-cli export "聊天名称" --format markdown --output export.md

# 导出为纯文本
wechat-cli export "聊天名称" --format txt --output export.txt

# 指定时间范围和条数上限
wechat-cli export "聊天名称" --format markdown \
  --start-time "2025-01-01" --end-time "2026-04-09" \
  --limit 100000 --output export.md
```

### 其他常用命令

```bash
wechat-cli contacts --query "姓名"    # 搜索联系人
wechat-cli members "群聊名称"          # 查看群成员
wechat-cli stats "聊天名称"            # 聊天统计分析
wechat-cli unread                      # 未读消息
wechat-cli favorites                   # 微信收藏
wechat-cli new-messages                # 增量新消息
```

---

## 用 Claude Code 批量导出全部聊天记录

单条命令适合查询特定聊天，**批量导出几千个会话**时建议让 Claude Code 写脚本跑。

### 直接对话

在 Claude Code 里这样说即可：

- "帮我把所有个人聊天和群聊都导出，个人每人一个 Markdown 文件，群聊合并成一个文件，放到 `~/wechat_export/`"
- "把和张三的聊天记录导出来"
- "搜索群聊 XX 中关于 YY 的消息"
- "分析一下文因互联群的聊天统计"

Claude 会自动调用 `wechat-cli`，处理文件名冲突、空记录、分页等边界情况。

### 全量导出时的几个坑

- **`sessions --limit` 要设够大**：默认可能只返回最近 1000 个会话，但微信里可能有 8000+ 个历史会话。批量导出时用 `--limit 20000` 或更大值，或告诉 AI "请先确认会话总数再导出"。
- **同名联系人**：用显示名做文件名会冲突（如两个"明明"），让 AI 对冲突的做去重编号。
- **空聊天**：很多联系人列表里的人从未聊过天，导出为空文件属正常。
- **媒体文件不在导出范围**：`wechat-cli export` 只导出文字内容，图片/视频/语音等存在微信本地目录的加密 `.dat` 文件里，当前工具不直接解密媒体。
- **50GB 数据 vs 百 MB 导出**：微信本地数据大部分是媒体文件，纯文本导出通常只有几十到几百 MB，这是正常的。

### 💡 如果 AI 执行不顺畅

批量导出涉及脚本编写、路径处理、编码问题、失败重试等一堆细节，对模型能力要求较高。如果遇到：

- AI 反复出错或逻辑混乱
- 脚本跑到一半卡住
- 导出数量明显不对但 AI 找不到原因
- 解释不清楚哪里出了问题

**优先建议直接切换到更强的模型**（Claude Opus 最新版 / Sonnet 最新版），而不是反复调试。很多看似是工具或数据的问题，换个更好的模型之后一把就过了。

---

## 在 Claude Code 中持久化配置

把下面这段加到项目的 `CLAUDE.md`，让 Claude 每次都知道怎么用 `wechat-cli`：

````markdown
## WeChat CLI

可以用 `wechat-cli` 查询我的本地微信数据。常用命令：

- `wechat-cli sessions --limit 20` — 最近会话
- `wechat-cli history "姓名" --limit 50 --format text` — 聊天记录
- `wechat-cli search "关键词" --chat "聊天名"` — 搜索消息
- `wechat-cli contacts --query "姓名"` — 搜索联系人
- `wechat-cli unread` — 未读会话
- `wechat-cli new-messages` — 自上次检查以来的新消息
- `wechat-cli members "群名"` — 群成员
- `wechat-cli stats "聊天名" --format text` — 聊天统计
- `wechat-cli export "聊天名" --format markdown --output out.md` — 导出

所有命令默认本地运行，数据不出本机。
````

然后就可以直接问 Claude：

- "看看我最近的未读微信"
- "搜一下团队群里关于 deadline 的消息"
- "这周 AI 讨论群谁发言最多？"

---

## 相关链接

- 工具本体：[freestylefly/wechat-cli](https://github.com/freestylefly/wechat-cli)
- 底层解密库：[ylytdeng/wechat-decrypt](https://github.com/ylytdeng/wechat-decrypt)
- Claude Code：[claude.com/claude-code](https://claude.com/claude-code)

---

## 免责声明

本指南和 `wechat-cli` 仅用于个人查询、备份本地微信数据：

- **只读**：只读取本地已存在的数据，不发送、修改、删除任何消息
- **本地运行**：所有数据都在本机处理，不上传任何服务器
- **不干扰微信**：不自动化任何微信操作，不违反微信使用条款
- **自行承担风险**：仅供个人学习研究使用，使用者需自行遵守当地法律法规
