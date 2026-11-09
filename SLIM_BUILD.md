# RustDesk 精简客户端（云端构建）

把 RustDesk 定制成「只做被控」的精简客户端，服务器参数编进程序、设置项全部隐藏。
构建在 GitHub Actions 上完成，产物是一个 zip（便携版，解压即用）。

## 定制内容

| 项目 | 做法 |
| --- | --- |
| 服务器参数 | ID / 中继 / API / 公钥在编译期写入，用户看不到也改不了 |
| 设置项 | `disable-settings` 硬开关 + 隐藏全部设置分区 + 封掉所有设置入口 |
| 远程控制 | `conn-type=incoming`，只保留「本机 ID / 密码」，UI 无连接入口，Rust 层也拒绝主动外连 |
| 账户 / 自动更新 | 一并关闭，避免被升级回官方版丢掉定制 |
| 体积 | 去掉硬件编解码（ffmpeg / mfx-dispatch），不打包虚拟显示器与打印机驱动 |

## 使用

### 1. 配置参数（必做）

仓库 **Settings → Secrets and variables → Actions → New repository secret**，新建一条：

- 名称：`RD_CONFIG`
- 值：

```
ID=qzyn10.wicp.vip:21116;RELAY=qzyn10.wicp.vip:21117;API=http://qzyn10.wicp.vip:21114;KEY=<公钥>
```

`KEY` 必须是服务端 `id_ed25519.pub` 的**完整内容**：

```bash
cat /root/rustdesk/id_ed25519.pub     # 或在 hbbs 数据目录下找 id_ed25519.pub
```

正常是 **44 个字符**的 Base64（32 字节 ed25519 公钥），末尾通常带 `=`。
长度不对时客户端握手会失败，日志里会提示。

> 也支持拆成 `RD_ID_SERVER` / `RD_RELAY_SERVER` / `RD_API_SERVER` / `RD_KEY` 四条独立 Secret。

### 2. 触发构建

**Actions → Build RustDesk Slim Client (Windows x64) → Run workflow**。

约 50~70 分钟（首次含 vcpkg 依赖编译）。完成后在本页 **Artifacts** 下载
`RustDesk-Slim-windows-x64.zip`。

### 3. 分发

解压后直接运行 `rustdesk.exe`。首次运行若弹出防火墙授权，允许即可（需要监听 21118 端口）。

## 换服务器 / 换 Key

改 `RD_CONFIG` 的值，重新 Run workflow 即可，**不需要改代码**。

## 仓库结构

```
.github/workflows/build-windows.yml   # 构建流程
patches/rustdesk-custom.patch         # 主仓库改动：内置配置模块 + 封设置入口
patches/hbb-common-server.patch       # 子模块改动：默认服务器与公钥
```

构建时会把 `rustdesk/rustdesk` 固定版本 `05b9f2d` 拉下来，打上这两个补丁再编译。
源码里的服务器值是占位符（`__RD_KEY__` 等），真实值在构建时注入，所以本仓库可以公开。

## 常见问题

- **构建失败在 vcpkg 阶段**：往上看日志里 `vcpkg 失败日志` 分组，通常是某个库下载超时，重跑一次即可（二进制缓存会复用已编译部分）。
- **能启动但连不上服务器**：八成是 `KEY` 不完整或与服务器不匹配，检查 `RD_CONFIG`。
- **想恢复完整功能**：把 `src/custom_build.rs` 里 `ENABLED` 改成 `false`，或在补丁里去掉 `INCOMING_ONLY` / `DISABLE_SETTINGS`。
