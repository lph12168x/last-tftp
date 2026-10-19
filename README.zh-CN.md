<div align="center">

<img src="resources/icons/icon_256x256.png" alt="last-tftp" width="112" />

# last-tftp

**跨平台现代 TFTP 工具 —— CLI 客户端、内置服务器、桌面 GUI 合为一个二进制。**

[![CI](https://github.com/lph12168x/last-tftp/actions/workflows/ci.yml/badge.svg)](https://github.com/lph12168x/last-tftp/actions/workflows/ci.yml)
![Rust](https://img.shields.io/badge/rust-1.75%2B-orange)
![License](https://img.shields.io/badge/license-MIT%20OR%20Apache--2.0-blue)

[English](./README.md) · [简体中文](./README.zh-CN.md)

</div>

---

实现 RFC 1350，并支持选项扩展（RFC 2347 / 2348 / 2349）与 RFC 7440 滑动窗口传输，
IPv4 / IPv6 双栈可用。协议细节见 [docs/protocol-notes.md](./docs/protocol-notes.md)，
架构设计见 [DESIGN.md](./DESIGN.md)。

## 特性

- **一个二进制，四种用法** —— `get`、`put`、`server`、`gui`（另有 `probe` 占位）。
- **RFC 7440 滑动窗口** —— `--window` 批量确认，同一链路下约 2× 提速（见[实测数据](#实测数据)）。
- **完整选项协商** —— 通过 OACK 协商 `blksize`、`timeout`、`tsize`、`windowsize`。
- **内置服务器** —— 多 socket 模型，任意目录共享，可开启写入，IPv4/IPv6 双栈监听。
- **可信的进度反馈** —— CLI 与 GUI 都有实时百分比进度条和吞吐统计，`--json` 便于脚本消费。
- **egui 桌面 GUI** —— Server / Client / Transfers / Settings 四个面板，使用原生文件对话框。
- **零参数启动** —— 不带子命令运行（或直接双击）即进入 GUI。

## 安装

每个 [GitHub Release](https://github.com/lph12168x/last-tftp/releases) 由 CI 自动附上
Linux / Windows / macOS 二进制。

也可从源码构建：

```bash
cargo build --release
./target/release/last-tftp --help
```

## 用法

传输目标统一写成 `HOST:FILE`，其中 `HOST` 可以是主机名、IPv4 地址或 IPv6 字面量。

### 下载

```bash
last-tftp get 192.168.1.1:firmware.bin -o ./fw.bin \
    --blksize 1468 --window 8
```

### 上传

```bash
last-tftp put -l ./local.bin 192.168.1.1:remote.bin \
    --blksize 1468 --window 8
```

### 内置服务器

```bash
# 共享 ./share，监听 6969 并允许写入
last-tftp server --root ./share --port 6969 --allow-write

# 以双栈绑定标准端口（需要 root / 管理员）
sudo last-tftp server --root ./share --port 69 --ipv6
```

### GUI

```bash
last-tftp gui
```

### 参数一览

| 参数 | 默认值 | 适用子命令 | 说明 |
|------|--------|-----------|------|
| `--port` | `69` | `get` `put` `probe` | 对端 UDP 端口 |
| `--port` | `6969` | `server` | 监听端口 |
| `--root` | `.` | `server` | 共享目录 |
| `--allow-write` | 关闭 | `server` | 接受 WRQ 写入 |
| `--blksize` | `1468` | `get` `put` | 块大小，范围 18…65464 |
| `--window` | `1` | `get` `put` | RFC 7440 windowsize |
| `--timeout` | `5` | `get` `put` | 单块超时（秒） |
| `--retries` | `6` | `get` `put` | 重试次数上限 |
| `--resume` | 关闭 | `get` | 仍是占位，从 block 1 重传，见[已知限制](#已知限制) |
| `--no-progress` | 关闭 | `get` `put` | 关闭进度条 |
| `--json` | 关闭 | `get` `put` | 以 JSON 输出统计信息 |
| `--ipv6` | 关闭 | 全部（全局） | 优先使用 IPv6 |

## GUI

四个面板：**Server**（共享目录、端口、写入开关）、**Client**（主机 / 端口 / 远端路径 /
本地路径，下载与上传）、**Transfers**（每条传输的实时进度）、**Settings**。

几个平台行为上的注意点：

- Windows 构建使用 `windows_subsystem`，启动 GUI 不会闪现 console 窗口；而 CLI 子命令会
  重新 attach 父 console，因此在 `cmd` 和 PowerShell 中 `stdout` 输出正常。
- **Browse** 调用原生文件对话框。下载时请选择**目录**（会自动拼上远端文件名），
  上传时选择**文件**。
- GUI 依赖显示服务，headless 环境无法启动。

## 协议支持

| RFC | 内容 | 状态 |
|-----|------|------|
| RFC 1350 | TFTP（Revision 2） | ✅ 完整 |
| RFC 2347 | 选项扩展 | ✅ 完整 |
| RFC 2348 | `blksize` 选项 | ✅ 完整 |
| RFC 2349 | `timeout` / `tsize` 选项 | ✅ 完整 |
| RFC 7440 | `windowsize` 选项 | ✅ 客户端 + 服务端 |
| RFC 2090 | TFTP 组播 | ❌ 未实现 |

## 平台支持

| 平台 | 状态 | 说明 |
|------|------|------|
| **Linux x86_64** | ✅ 已验证 | Wayland + X11；CI 自动构建 |
| **Windows x86_64** | ✅ CI 编译通过 | winit win32 backend，原生文件对话框，图标由 `rcedit` 注入 |
| **macOS x86_64 / aarch64** | ✅ CI 编译通过 | winit cocoa backend，原生文件对话框 |

本地跨平台构建：

```bash
cargo build --release                                  # 当前平台
rustup target add x86_64-pc-windows-gnu
sudo apt install gcc-mingw-w64-x86-64                  # mingw 工具链
./scripts/build.sh all                                 # 依次尝试全部平台
```

CI 会在 ubuntu / windows / macos 三个 runner 上构建并上传二进制 artifact；推送 `v*`
tag 时自动把二进制附到 Release。

## 测试

```bash
cargo test --release                    # 整个 workspace
cargo test --release -p last-tftp-core  # 仅协议引擎
```

当前代码树状态：**35 通过、1 忽略、0 失败**。

| 测试套件 | 数量 | 覆盖内容 |
|----------|------|----------|
| `last-tftp-core` 单元测试 | 25 | 报文编解码、选项协商、窗口计算、目标地址解析 |
| `last-tftp-core` 集成测试 | 3 | 带 OACK 的端到端 GET/PUT、远端错误传播 |
| `last-tftp` GUI 测试 | 7 | headless egui 渲染 + 进程内 mock server |

被忽略的是长时间运行的丢包场景 GET，需要显式执行
`cargo test --release -- --ignored`。

## 实测数据

512 KiB 下载的本地回环早期数据，属历史记录，并非每次提交都重跑：

| 场景 | 结果 |
|------|------|
| 524288 B GET，`--window 1` | 约 199 ms |
| 524288 B GET，`--window 8` | 约 92 ms（≈2.2×） |

## 阶段进度

| 阶段 | 内容 | 状态 |
|------|------|------|
| P0 | workspace + 双 crate | ✅ |
| P1 | 报文编解码 | ✅ |
| P2 | 选项协商 | ✅ |
| P3 | 客户端 GET（跟随 server ephemeral port） | ✅ |
| P4 | 客户端 PUT（RFC 7440 滑动窗口） | ✅ |
| P5 | 内置服务器（多 socket 模型） | ✅ |
| P6 | `windowsize`（get/put 两端累积） | ✅ |
| P7 | IPv6 双栈 | ✅ |
| P8 | 进度条 + 吞吐统计（CLI/GUI） | ✅ |
| P9 | GUI（eframe 0.32） | ✅ |
| P10 | 断点续传（`--resume`） | ⚠️ 占位 |
| P11 | 协议文档 | ✅ |

## 已知限制

- **`--resume` 仍是占位。** 实际行为是从 block 1 重传，会覆盖已有文件。完整实现需要服务端
  `tsize` 协商加上客户端 offset。
- **headless 无法启动 GUI。** 没有显示服务就没有窗口。
- **大文件。** 服务器 V1 是整文件读入内存，超出限制需改造为流式 I/O。
- **尾帧 ACK 噪声。** 最后一帧 ACK 之后客户端可能误报 timeout，但文件其实已正确写入，
  用 `cmp` 校验即可确认。
- **`probe` 尚未实现**，调用会直接报错；想确认对端能力请用 `get`。

## 许可

MIT 或 Apache-2.0，任选其一。
