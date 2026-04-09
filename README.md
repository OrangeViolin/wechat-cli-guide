# 用 Claude Code + wechat-cli 导出微信聊天记录

## 前提条件

- macOS (Apple Silicon)
- 微信 for Mac 4.0+ 已安装并登录
- Node.js 已安装（v18+）
- Claude Code 已安装

## 第一步：关闭 SIP（System Integrity Protection）

密钥提取需要读取微信进程内存，必须关闭 SIP。

1. 关机
2. 长按电源键开机，进入恢复模式
3. 顶部菜单栏 → 实用工具 → 终端
4. 执行 `csrutil disable`
5. 重启

> 完成全部操作后记得重新开启 SIP（同样步骤，执行 `csrutil enable`）。

## 第二步：安装 wechat-cli

```bash
npm install -g @canghe_ai/wechat-cli
```

验证安装：

```bash
wechat-cli --version
# 应输出 0.2.4 或更高
```

项目地址：https://github.com/freestylefly/wechat-cli

## 第三步：提取微信数据库密钥

确保微信已打开并登录，然后用管理员权限初始化：

**方法 A：通过 osascript（在 Claude Code 中直接运行，会弹出密码框）**

```bash
/usr/bin/osascript -e 'do shell script "wechat-cli init 2>&1" with administrator privileges'
```

**方法 B：通过 Terminal.app 手动运行**

```bash
sudo wechat-cli init
```

初始化成功后会输出类似：

```
[+] 提取到 X 个密钥，保存到: ~/.wechat-cli/all_keys.json
[+] 初始化完成!
```

### 常见问题

**配置文件指向了错误的微信账号目录：**

如果有多个微信账号登录过，`config.json` 可能自动选择了旧账号的目录。检查方法：

```bash
cat ~/.wechat-cli/config.json
# 查看 db_dir 路径中的 wxid 是否是当前登录账号
```

```bash
ls -lt ~/Library/Containers/com.tencent.xinWeChat/Data/Documents/xwechat_files/
# 按时间排序，最近修改的目录通常是当前账号
```

如需修正，编辑 `~/.wechat-cli/config.json` 中的 `db_dir` 指向正确的 `wxid_xxx/db_storage` 路径。

**配置文件权限问题：**

因为用了 sudo 运行 init，配置文件可能属于 root。修复：

```bash
sudo chown -R $(whoami):staff ~/.wechat-cli/
```

## 第四步：使用

初始化完成后，以下所有命令不再需要 sudo 或 SIP 关闭。

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

### 导出聊天记录

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

### 其他命令

```bash
wechat-cli contacts --query "姓名"    # 搜索联系人
wechat-cli members "群聊名称"          # 查看群成员
wechat-cli stats "聊天名称"            # 聊天统计分析
wechat-cli unread                      # 未读消息
```

## 在 Claude Code 中使用

以上命令都可以直接在 Claude Code 对话中让 Claude 执行。例如：

- "帮我导出和张三的聊天记录"
- "搜索群聊 XX 中关于 YY 的消息"
- "分析一下这个群的聊天统计"

Claude 会自动调用 wechat-cli 并处理结果。

## 完成后：重新开启 SIP

1. 关机
2. 长按电源键开机，进入恢复模式
3. 顶部菜单栏 → 实用工具 → 终端
4. 执行 `csrutil enable`
5. 重启

已提取的密钥会保存在 `~/.wechat-cli/` 中，日常使用不需要 SIP 关闭。只有当微信更新版本或重新登录导致密钥变化时，才需要再次关闭 SIP 重新执行 `wechat-cli init`。
