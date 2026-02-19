# 喂喂喂 (HelloHello)

> **喂喂喂** 是一个 macOS 系统级网络监控守护程序，通过 ICMP (ping) 协议持续监控目标主机的可达性，并在网络状态变化时发送告警通知。就像打电话时确认对方"喂？在吗？"一样，不知疲倦地守护着您的网络连接。
>
> **授权说明**: 本项目采用专有授权。版权方可通过官方渠道公开发布；未经授权不得二次分发、镜像或商用。

## 核心特性

- **双运行模式**: 支持 `--daemon` 守护进程模式和交互式 TUI 模式，同一二进制文件满足两种场景
- **智能防抖**: 内置状态机，通过 `fail_count` 和 `success_count` 阈值避免网络瞬时波动误报
- **配置热更**: 基于 fsnotify 的配置热重载，修改配置即刻生效，无需重启服务；配置校验失败时保留旧配置
- **配置校验**: 启动时校验配置合法性 (interval/threshold > 0)，非法配置直接退出并提示
- **多维告警**: 支持钉钉 Webhook (HMAC-SHA256 签名，指数退避重试) 和 macOS 本地通知
- **并发架构**: 每个监控主机运行在独立 goroutine，高效稳定
- **极简交互**: 提供 Bubble Tea 实现的 TUI 管理界面，操作简单直观

## 技术栈

| 组件 | 技术 |
|------|------|
| 后端语言 | Go 1.24.0 |
| TUI 框架 | Bubble Tea + Bubbles + Lipgloss |
| 配置热重载 | fsnotify |
| 日志轮转 | lumberjack |
| 前端 Landing | Next.js 15.4 + React 19 + Tailwind CSS v4 |
| 系统集成 | macOS LaunchDaemon |

## 快速开始

### 一键安装

```bash
curl -fsSL https://hello.ahua.space/sh | sudo bash
```

### 更新和卸载

```bash
# 更新到最新版本
curl -fsSL https://hello.ahua.space/sh | sudo bash -s update

# 卸载 HelloHello
curl -fsSL https://hello.ahua.space/sh | sudo bash -s uninstall
```

### 使用

```bash
# TUI 交互模式
sudo hello
```

## 配置说明

配置文件位于: `/Library/Application\ Support/HelloHello/config.conf`

### 示例配置

```ini
[general]
# 配置文件版本号，固定 1
version = 1
# 默认检测间隔（秒），主机未单独设置时使用
interval_seconds = 3
# 默认连续失败/成功阈值
fail_count = 3
success_count = 3

[notify]
# 通知模板，占位符: {name} {ip} {fail_count} {success_count} {time}
down = [告警] {name}({ip}) 不可达，连续失败 {fail_count} 次
up = [恢复] {name}({ip}) 已恢复

[hosts]
# 基础示例：仅名称 + IP
# 网关 = 192.168.1.1

# 带手机号 @ 的示例：多个手机号用 @ 前缀分隔，正文会自动附带 @手机号
# 业务服务器 = 10.0.0.1, at_mobiles=@13800138000@13900139000

# 覆盖间隔/阈值的示例：interval_seconds/fail_count/success_count 覆盖默认值
# 核心数据库 = 10.0.0.2, interval_seconds=5, fail_count=2, success_count=1

# 可选参数说明：
#   interval_seconds          : 主机级检测间隔（秒，推荐）
#   interval                  : 主机级检测间隔（秒，兼容历史别名）
#   fail_count / success_count : 覆盖默认阈值
#   at_mobiles                : 钉钉手机号列表，用 @ 前缀分隔
# 注释以 # 开头，空行会被忽略

[webhook]
# 启用钉钉通知（true/false）
enabled = true
# 钉钉机器人 Webhook 地址
url =
# 钉钉机器人签名密钥
secret =

[macos]
# 是否启用 macOS 系统通知
enabled = true
# 通知声音，可填 default 或系统可用声音名称
# 启动静默期（秒），避免服务启动瞬间通知刷屏
startup_silence_seconds = 15
sound = default

```

### macOS 通知字段说明

- `macos.enabled` (即 notifications.macos.enabled): 是否启用 macOS 系统通知，类型为 `bool`，默认 `false`
- `macos.sound` (即 notifications.macos.sound): 通知声音，类型为 `string`，默认 `default`；设置为空字符串表示静音
- `macos.startup_silence_seconds` (即 notifications.macos.startup_silence_seconds): 启动静默期（秒），避免服务启动瞬间通知刷屏，默认 `15`

### 模板变量

| 变量 | 说明 |
|------|------|
| `{name}` | 主机名称 |
| `{ip}` | 主机 IP |
| `{fail_count}` | 连续失败次数 |
| `{success_count}` | 连续成功次数 |
| `{time}` | 当前时间 |

## 系统目录

```
/Library/Application Support/HelloHello/
  ├─ hellohello           # Go 二进制
  ├─ config.conf          # 配置文件
  ├─ .notify_setup_done   # 通知权限状态文件 (JSON 格式)
  ├─ notify_queue.jsonl   # 守护进程写入的通知事件队列
  ├─ HelloHelloHelper.app # 用户级 Helper (LaunchAgent 负责拉起并弹窗)
  └─ status.json          # 状态文件 (自动生成)

/Library/Logs/HelloHello/
  ├─ hellohello-core.log  # 应用日志
  ├─ stdout.log           # LaunchDaemon 标准输出
  ├─ stderr.log           # LaunchDaemon 标准错误
  ├─ notify-setup.stdout.log  # 通知权限请求输出
  └─ notify-setup.stderr.log  # 通知权限请求错误

/Library/LaunchDaemons/
  └─ com.ahua.hellohello.plist

/Library/LaunchAgents/
  ├─ com.ahua.hellohello.notify-setup.plist  # 首次权限触发
  └─ com.ahua.hellohello.helper.plist        # 用户级 Helper，消费队列并弹窗
```

Helper 标准输出/错误日志由 LaunchAgent 写入 `/tmp/com.ahua.hellohello.helper.*.log`。

## 高级使用

### 启用 macOS 系统通知

1. 确保配置文件启用系统通知并设置声音:
   ```ini
   [macos]
   enabled = true
   sound = default
   ```
   完整配置示例见上方「示例配置」。
2. 登录后会启动用户级 Helper（LaunchAgent），它从 `notify_queue.jsonl` 读取并弹出通知；无登录用户时仅发送 Webhook。
3. 首次用户登录时会自动弹出系统权限请求对话框。
4. 授权成功后会发送一条测试通知，并更新状态文件:
   `/Library/Application Support/HelloHello/.notify_setup_done`

### 权限状态说明

状态文件 `.notify_setup_done` 为 JSON 格式，记录以下状态：
- `granted`: 用户已授权，可正常发送通知
- `requested`: 已触发权限请求，等待用户在系统对话框中授权
- `denied`: 用户拒绝授权
- `timeout`: 权限请求超时

### 重新触发权限请求

如果之前点击了"不允许"或需要重新授权，删除状态文件后重新登录即可:

```bash
sudo rm -f "/Library/Application Support/HelloHello/.notify_setup_done"
```

### 权限与故障排查

- 权限请求仅在 GUI 会话中生效，SSH/无头环境会跳过请求。
- 通知权限日志位于:
  `/Library/Logs/HelloHello/notify-setup.stdout.log`
  `/Library/Logs/HelloHello/notify-setup.stderr.log`

## FAQ

**Q: 为什么没收到系统通知?**
A:
- 检查系统设置 → 通知 → HelloHello 是否已允许
- 查看通知权限日志: `/Library/Logs/HelloHello/notify-setup.*.log`
- 检查状态文件内容: `cat "/Library/Application Support/HelloHello/.notify_setup_done"`
  - 如果状态为 `requested`，表示权限请求已触发但未完成授权
  - 如果状态为 `denied`，表示用户拒绝了授权
- 删除状态文件后重新登录触发权限请求:
  `/Library/Application Support/HelloHello/.notify_setup_done`

## 构建

构建需要启用 CGO 并安装 Xcode Command Line Tools:

```bash
xcode-select --install
CGO_ENABLED=1 go build ./cmd/hellohello
```

最低系统要求: macOS 10.14+

## 状态机设计

```
UNKNOWN ──成功x3──> UP
   │                 │
  失败x3           失败x1
   │                 │
   v                 v
  DOWN <──成功x3── (重置计数器)
```


## 许可证

本项目采用专有授权策略。
版权方可通过官方渠道公开发布本项目及其发布产物。
未经授权，禁止第三方二次分发、镜像托管或商用本项目及其衍生产物。
第三方依赖仍遵循其各自许可证条款。
