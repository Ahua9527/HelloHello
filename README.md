# 喂喂喂 (HelloHello)

> **喂喂喂** 是一个 macOS 系统级网络监控守护程序，通过 ICMP (ping) 协议持续监控目标主机的可达性，并在网络状态变化时发送告警通知。就像打电话时确认对方"喂？在吗？"一样，不知疲倦地守护着您的网络连接。
>
> **授权说明**: 本项目采用专有授权。版权方可通过官方渠道公开发布；未经授权不得二次分发、镜像或商用。

## 核心特性

- **双运行模式**: 支持 `--daemon` 守护进程模式和交互式 TUI 模式，同一二进制文件满足两种场景
- **智能防抖**: 内置状态机，通过 `fail_count` 和 `success_count` 阈值避免网络瞬时波动误报
- **配置热更**: 基于 fsnotify 的配置热重载，修改配置即刻生效，无需重启服务；配置校验失败时保留旧配置
- **配置校验**: 启动时校验配置合法性 (interval/threshold > 0)，非法配置直接退出并提示
- **多维告警**: 支持钉钉告警通道 (Webhook，HMAC-SHA256 签名，指数退避重试) 与灵动岛通知，关键状态变化即时触达
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

## 发布通道

- 稳定版 Tag 规则：`vX.Y.Z`（如 `v1.2.3`）
- Beta Tag 规则：`vX.Y.Z-beta.N`（如 `v1.2.3-beta.1`）
- Beta 发布仅进入私有仓库 `Ahua9527/HelloHello-Private`，并标记为 GitHub Pre-release
- 安装/更新脚本默认使用稳定版 `releases/latest`，不会自动拉取 Beta 版本

## 配置说明

配置文件位于: `/Library/Application\ Support/HelloHello/config.toml`

### 示例配置

```toml
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
down = "[告警] {name}({ip}) 不可达，连续失败 {fail_count} 次"
up = "[恢复] {name}({ip}) 已恢复"

[[hosts]]
name = "网关"
ip = "192.168.1.1"

[[hosts]]
name = "业务服务器"
ip = "10.0.0.1"
at_mobiles = ["13800138000", "13900139000"]

[[hosts]]
name = "核心数据库"
ip = "10.0.0.2"
interval_seconds = 5
fail_count = 2
success_count = 1

[webhook]
# 启用钉钉通知（true/false）
enabled = true
# 钉钉机器人 Webhook 地址
url = ""
# 钉钉机器人签名密钥
secret = ""

[macos]
# 是否启用灵动岛通知通道（默认 true）
dynamic_island_enabled = true
# 灵动岛通知显示时长（毫秒），最小 500，默认 2500
dynamic_island_duration_ms = 2500
# 是否启用手动回收（默认 false，true 时仅点击后收回）
dynamic_island_manual_dismiss = false
# 启动静默期（秒），避免服务启动瞬间通知刷屏
startup_silence_seconds = 15

```

### macOS 灵动岛通知字段说明

- `macos.dynamic_island_enabled`: 是否启用灵动岛通知通道，类型为 `bool`，默认 `true`
- `macos.dynamic_island_duration_ms`: 灵动岛通知显示时长（毫秒），类型为 `int`，最小 `500`，默认 `2500`。`dynamic_island_manual_dismiss=false` 时用于自动回收；手动回收模式下该值仅保留兼容
- `macos.dynamic_island_manual_dismiss`: 是否启用手动回收模式，类型为 `bool`，默认 `false`。为 `true` 时通知仅在点击后回收；若旧通知未回收且新通知到达，会先收回旧通知再展示新通知
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
  ├─ config.toml          # 主配置文件
  ├─ notifier.sock        # 守护进程 -> notifier 的 UDS 通道
  ├─ HelloHelloNotifier.app # 用户级 notifier app（灵动岛渲染）
  └─ status.json          # 状态文件 (自动生成)

/Library/Logs/HelloHello/
  ├─ hellohello-core.log  # 应用日志
  ├─ stdout.log           # LaunchDaemon 标准输出
  └─ stderr.log           # LaunchDaemon 标准错误

~/Library/Logs/HelloHello/
  ├─ notifier.stdout.log  # notifier 标准输出（用户级 LaunchAgent）
  └─ notifier.stderr.log  # notifier 标准错误（用户级 LaunchAgent）

/Library/LaunchDaemons/
  └─ com.ahua.hellohello.plist

/Library/LaunchAgents/
  └─ com.ahua.hellohello.notifier.plist      # 用户级 notifier，消费 UDS 并渲染灵动岛
```

notifier 标准输出/错误日志由用户级 LaunchAgent 写入 `~/Library/Logs/HelloHello/notifier.*.log`。

## 高级使用

### 启用灵动岛通知

1. 确保配置文件启用灵动岛通知通道:
   ```toml
   [macos]
   dynamic_island_enabled = true
   dynamic_island_duration_ms = 2500
   dynamic_island_manual_dismiss = false
   ```
   完整配置示例见上方「示例配置」。
2. 登录后会启动用户级通知服务（LaunchAgent），守护进程通过 UDS (`notifier.sock`) 投递通知请求。
3. 通知服务暂不可用时不会阻断守护进程主流程，Webhook 通道仍照常发送。

## FAQ

**Q: 为什么没收到灵动岛通知?**
A:
- 检查 `[macos]` 配置：`dynamic_island_enabled=true`
- 检查 LaunchAgent 状态：`launchctl print gui/$(id -u)/com.ahua.hellohello.notifier`
- 检查通知服务日志：`~/Library/Logs/HelloHello/notifier.stderr.log`
- 检查 socket 文件：`ls -l \"/Library/Application Support/HelloHello/notifier.sock\"`
- 手动测试：`hellohello --test-notify`

**Q: 安装时出现 `getcwd` / `job-working-directory` 报错怎么办?**
A:
- 这通常表示执行命令时所在目录已被删除或失效。
- 新版安装脚本会自动切换到 `/` 并继续执行，避免重复刷屏。

**Q: 使用 `LOCAL_TEST` 安装后 notifier 起不来，提示签名或 socket 问题怎么办?**
A:
- 新版脚本在 `LOCAL_TEST` 模式会自动对 `hellohello` 与 `HelloHelloNotifier.app` 重签并校验，降低本地调试签名异常概率。
- 若仍异常，请依次检查：
  - `launchctl print gui/$(id -u)/com.ahua.hellohello.notifier`
  - `codesign --verify --deep --strict "/Library/Application Support/HelloHello/HelloHelloNotifier.app"`
  - `ls -l "/Library/Application Support/HelloHello/notifier.sock"`

## 构建

构建需要启用 CGO 并安装 Xcode Command Line Tools:

```bash
xcode-select --install
CGO_ENABLED=1 go build ./cmd/hellohello
```

最低系统要求: macOS 13.0+

### 自托管 CI Runner（macOS）初始化

首次配置 self-hosted macOS Runner 时，必须先同意 Xcode License，否则 CI 中的 `git`/`xcodebuild` 可能以 `exit code 69` 失败。

```bash
xcodebuild -license check
sudo xcodebuild -license accept
sudo xcodebuild -runFirstLaunch
```

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
