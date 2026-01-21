# 喂喂喂 (HelloHello)

> **喂喂喂** 是一个 macOS 系统级网络监控守护程序，通过 ICMP (ping) 协议持续监控目标主机的可达性，并在网络状态变化时发送告警通知。就像打电话时确认对方"喂？在吗？"一样，不知疲倦地守护着您的网络连接。

## 核心特性

- **双运行模式**: 支持 `--daemon` 守护进程模式和交互式 TUI 模式，同一二进制文件满足两种场景
- **智能防抖**: 内置状态机，通过 `fail_count` 和 `success_count` 阈值避免网络瞬时波动误报
- **配置热更**: 基于 fsnotify 的配置热重载，修改配置即刻生效，无需重启服务
- **多维告警**: 支持钉钉 Webhook (HMAC-SHA256 签名) 和 macOS 本地通知
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

配置文件位于: `/Library/Application\ Support/HelloHello/config.yaml`

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
up = [恢复] {name}({ip}) 已恢复，恢复时间 {time}

[hosts]
# 基础示例：仅名称 + IP
# 网关 = 192.168.1.1

# 带手机号 @ 的示例：多个手机号用 | 分隔，正文会自动附带 @手机号
# 业务服务器 = 10.0.0.1, at_mobiles=13800138000|13900139000

# 覆盖间隔/阈值的示例：interval/fail_count/success_count 覆盖默认值
# 核心数据库 = 10.0.0.2, interval=5, fail_count=2, success_count=1

# 可选参数说明：
#   interval / interval_seconds: 主机级检测间隔（秒）
#   fail_count / success_count : 覆盖默认阈值
#   at_mobiles                : 钉钉手机号列表，用 | 分隔
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
sound = default

```

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
  ├─ config.yaml          # 配置文件
  └─ status.json          # 状态文件 (自动生成)

/Library/Logs/HelloHello/
  ├─ hellohello.log       # 应用日志
  ├─ stdout.log           # LaunchDaemon 标准输出
  └─ stderr.log           # LaunchDaemon 标准错误

/Library/LaunchDaemons/
  └─ com.ahua.hellohello.plist
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

Internal Use Only.
