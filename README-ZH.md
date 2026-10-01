# Podroid 使用教程

> 在 Android 手机上免 root 运行真实的 Linux 虚拟机 —— 跑 Podman、Docker、LXC 和 Linux 桌面

**文档对应版本**：v1.2.9 ｜ **架构**：ARM64 / Android 8+ ｜ **无需 root** ｜ **GPLv2**

---

## 目录

1. [它是什么](#1-它是什么)
2. [前置条件](#2-前置条件)
3. [安装与首次设置向导](#3-安装与首次设置向导)
4. [第一次启动 VM](#4-第一次启动-vm)
5. [基本操作：包管理与服务](#5-基本操作包管理与服务)
6. [跑容器：Podman / Docker / LXC](#6-跑容器podman--docker--lxc)
7. [图形桌面与 X11 查看器](#7-图形桌面与-x11-查看器)
8. [网络、端口转发与文件传输](#8-网络端口转发与文件传输)
9. [进阶：开启 AVF 硬件加速](#9-进阶开启-avf-硬件加速)
10. [内建命令行工具](#10-内建命令行工具)
11. [性能预期](#11-性能预期)
12. [常见问题排查](#12-常见问题排查)
13. [速查表](#13-速查表)

---

## 1. 它是什么

Podroid 是一个 Android 应用，它在你的手机里启动一台**真正的 Linux 虚拟机**——不是 Termux 那种 chroot / proot 环境，而是带完整自定义 Linux 内核的 Alpine Linux 系统。

它把 QEMU、内核、initramfs、Alpine 根文件系统和字体全部打包进 APK（约 200 MB），安装即用，全部运行在用户态，**不需要 root**。

| 项目 | 说明 |
|---|---|
| 默认后端 | QEMU（TCG 软件 CPU 模拟）—— 任何 ARM64 设备都能跑 |
| 可选后端 | AVF / pKVM —— 仅 Pixel 8 及更新机型，速度接近原生 |
| 发行版 | Alpine Linux 3.24（aarch64），OpenRC 作为 init |
| 存储 | squashfs 只读底 + ext4 可写上层（overlay），改动持久保留 |
| 预装 | Podman、Docker、LXC、dropbear(SSH)、tigervnc(Xvnc)、PulseAudio |

---

## 2. 前置条件

| 项目 | 要求 |
|---|---|
| CPU 架构 | 必须 **ARM64（arm64-v8a）**。x86、x86_64、32 位 ARM 设备**装不上** |
| 系统版本 | Android 8（API 26）及以上。v1.2.5 起正式支持 Android 8 / 8.1 |
| 可用空间 | 至少预留 3～5 GB。VM 磁盘可设 2 GB ~ 512 GB |
| 内存 | 建议 8 GB 以上，才能给 VM 分配 2~4 GB |
| Root | 不需要 |

> **⚠️ 先确认你的芯片**
>
> 如果你用的是**高通骁龙**机型（8 Gen 2 及以后），它只有 Gunyah 且只支持受保护 VM，Podroid 的 AVF 后端用不了，只能跑 QEMU。这一点在后面的[第 9 节](#9-进阶开启-avf-硬件加速)会详细说。

---

## 3. 安装与首次设置向导

### 安装

1. 打开 [github.com/ExTV/Podroid/releases](https://github.com/ExTV/Podroid/releases)，下载最新的 `Podroid-release-*.apk`。
2. 在系统设置里给浏览器或文件管理器打开「安装未知应用」权限，然后点开 APK。
3. 点击「安装」。它没有上架 Play Store，只能侧载。

### 首次启动的四步向导

#### 第 1 步 · 持久化存储大小

选择可写 VM 磁盘镜像的大小，范围 **2 GB – 512 GB**，上限受手机剩余空间限制。这个镜像在应用私有目录里，你在 VM 内装的软件包、容器镜像、配置全都存在这里。

> **⛔ 只能变大，永远不能变小**
>
> 存储大小日后可以在 Settings 里**扩大**，但**无法缩小**。所以第一次别贪大也别太小 —— 选一个你愿意长期分配给 VM 的容量。建议比你预想的稍微大一点。

#### 第 2 步 · VM 配置

设置虚拟 CPU 核心数和 RAM（默认 **512 MB**，偏小，建议 2048 MB 起）。同时可以在这里打开 **SSH** 开关 —— 打开后 dropbear 会自动启动，VM 就能通过局域网 `9922` 端口访问。

#### 第 3 步 · Downloads 共享（可选）

授权访问手机的 Downloads 文件夹，方便在 Android 和 VM 之间传文件。两种后端都支持，不需要可以跳过。

> **⚠️ QEMU 后端下的已知不稳定**
>
> 在部分设备（尤其是 **ARMv9.2 的 Pixel 10**）上，QEMU 的 virtio-9p 驱动可能崩溃并拖垮整个 VM。如果开启后 VM 启动不久就崩，**第一步就是去设置里关掉 Downloads sharing**。AVF 后端走 vsock，不受影响。

#### 第 4 步 · 权限

最后一张卡片逐项列出权限并附上理由：

- **通知权限** —— VM 跑在前台服务里，通知是你查看状态和停止它的方式
- **不受限制的后台使用** —— 否则 Android 会在应用切到后台后停掉 VM

两项都可以跳过，但强烈建议都授予。如果日后某个权限缺失，主屏幕顶部会再次出现同一行提示。

---

## 4. 第一次启动 VM

在主屏幕点击 **Start VM**。首次启动需要 **50–60 秒**，之后的启动只要 10–15 秒。慢是因为有两件一次性的事在跑：

1. **SSH 主机密钥生成** —— dropbear 生成 RSA、ECDSA、ED25519 三套密钥，典型手机上耗时 25–35 秒，只发生一次。
2. **文件系统格式化** —— 把 ext4 存储镜像格式化成你在向导第 1 步选的容量。

通知栏会依次显示 `Starting SSH...` → `Almost ready...` → `Ready!`，看到 `Ready!` 就说明 VM 完全启动，终端可以交互了。

### 默认凭据

```sh
# 用户名 / 密码
root / podroid

# 建一个普通用户（带 wheel 组，等效 sudo）
adduser -G wheel <username>
```

`wheel` 组成员可以用 `doas` 提权（装了 `sudo` 的话也能用 sudo）。

### 主屏幕能看到什么

- VM 状态与运行时长
- 网络信息：手机 IP、SSH 端点、当前生效的端口转发规则数量
- 已创建容器数
- Start / Stop 按钮，以及 Backup 和 Status 按钮

点主屏幕的终端图标进入完整 VT100 终端。终端顶栏的显示器图标打开 X11 查看器。

---

## 5. 基本操作：包管理与服务

VM 里就是标准的 Alpine，`/etc/apk/repositories` 已经配好 main 和 community 两个仓库。

```sh
# 刷新索引 / 全面升级
apk update
apk upgrade

# 安装 / 卸载 / 搜索
apk add firefox
apk del firefox
apk search nginx
apk info nginx
```

装的东西会持久保存在可写的 ext4 overlay 上，重启后依然在。

### OpenRC 服务管理

```sh
# 查看所有服务状态
rc-status

# 起停重启
rc-service docker stop
rc-service dropbear start

# 设开机自启 / 取消自启
rc-update add crond default
rc-update del docker default   # 只用 Podman 时禁用 Docker 省内存
```

### Podroid 自己的系统服务

| 服务 | 作用 |
|---|---|
| `podroid-bootstrap` | 最先跑。校时、启用 cgroup v2、加载内核模块、挂载 devpts/shm/mqueue/configfs、配置 ZRAM 交换、设置主机名、把 ext4 存储绑定挂载到容器数据目录 |
| `podroid-network` | 拉起 eth0（QEMU 为 10.0.2.15/24，AVF 走 DHCP），写 `/etc/resolv.conf` |
| `podroid-resize` | 终端缩放守护进程，键盘/旋转变化后让 shell 重绘 |
| `podroid-x11` | 在 `:0`（5900）启动 Xvnc，并在 4713 启动 PulseAudio |
| `podroid-vsock` | **仅 AVF**。vsock 端口转发代理；QEMU 上是 inactive |
| `podroid-hostd` | guest → Android 的桥接守护进程，两种后端都工作 |
| `podroid-ready` | 最后跑。发出 `Ready!` 标记，并调低基础设施进程的 OOM 分数 |
| `docker` / `lxc` | 已加入 default 运行级别，开机自动启动 |

---

## 6. 跑容器：Podman / Docker / LXC

三个容器引擎都预装且**开箱即用**：Podman 是 rootless 且无需守护进程；Docker 和 LXC 已设开机自启。

```sh
# Podman —— rootless，立即能用
podman run --rm alpine echo hi

# Docker —— 预装、自动启动
docker run -d -p 8080:80 nginx

# LXC —— 预装，网桥已就绪
lxc-create -t alpine -n test
lxc-start -n test
```

> **⚠️ 别拉 x86 镜像**
>
> app 只带 arm64-v8a，客户机也是 ARM64。拉 x86 / x86_64 镜像会在双重模拟下跑（ARM64 客户机 + TCG 再模拟 x86），慢到不可用。**只拉 arm64 镜像。**

> **💡 容器端口也要用高位**
>
> 无根 Podman / Docker 在客户机内同样受限，发布端口必须 ≥1024。比如 Pi-hole DNS 容器要发布到客户机 `5353/udp`，再转发到主机 `5353`，然后让局域网设备指向 `<phone-ip>:5353`。

---

## 7. 图形桌面与 X11 查看器

Podroid 内置一个 VNC 查看器，连到 VM 里的 Xvnc（显示器 `:0`，端口 5900，已自动转发）。`DISPLAY=:0` 在环境里已经预设好了。

### 装 Xfce（最小路径）

```sh
# 约 400 MB
apk add xfce4 xfce4-terminal

# 后台启动
startxfce4 &
```

然后点终端顶栏的显示器图标，桌面就出来了。不需要装 VNC 服务器、不需要配端口、不需要任何配置。

### 其他桌面

```sh
startlxqt &      # LXQt
mate-session &   # MATE
```

> **⛔ 只支持 X11，Wayland 桌面不会出现**
>
> 查看器用的是 RFB(VNC) 协议，只能连 X11 显示，**无法显示 Wayland 合成器**。
>
> **可用**：Xfce、LXQt、MATE　　**不可用**：KDE Plasma 6、GNOME、Sway（Alpine 上都是 Wayland）

> **⚠️ 不要用显示管理器**
>
> Alpine 的 `setup-desktop` 会顺手装并启用 `lightdm`。但在 Podroid 里客户机内核没有图形设备，lightdm 只会空转、从不起动 X 服务器，而 Xvnc 已经独占 `:0`。装完请执行 `rc-update del lightdm` 保持启动干净。

### 跑单个 GUI 程序

```sh
apk add firefox
firefox &
```

### 从电脑连同一个桌面

VM 内的 X 会话是**无密码**运行的，所以 Podroid 刻意只把 5900 转发到手机的 `127.0.0.1`，从外部访问不到。要用电脑连，走 SSH 隧道（需先在设置里开 SSH）：

```sh
# 在你自己的电脑上执行（不是手机里）
ssh -p 9922 -L 5900:127.0.0.1:5900 root@<phone-ip>

# 保持连接不断，然后用任意 VNC 客户端连
localhost:5900
```

想用 RDP 的话，客户机内装 `xrdp` 并把 `autorun` 固定为 `Xvnc`，再添加主机 `3389` → VM `3389` 的转发即可。**动手前先用 `passwd` 改掉 root 密码**，因为端口转发绑的是所有接口。

### 查看器设置要点

| 设置 | 说明 |
|---|---|
| 分辨率 | Match viewport / 预设(720p–1440p) / 自定义，可实时调整不用重启 VM |
| 触摸模式 | Direct（点触即点击）或 Trackpad（相对指针，灵敏度 0.5x–3.0x） |
| 通用手势 | 双指点击 = 右键，双指拖动 = 滚动 |
| Render scale | 100% / 75% / 50%，越低越快 |
| Server DPI | 96–192，**下次启动 VM 生效**，不支持热重载 |
| Audio | PulseAudio 通过 4713 端口把 PCM 流到 app，可开关 |

> **⚠️ 两个已知毛病**
>
> ① 外接鼠标的**右键会退出全屏**（被 Android 手势处理器截胡），用双指触摸右键绕过。
> ② 视频编码是**未压缩 Raw**（ZRLE 被禁用），高 DPI 屏上会偏卡，建议选 720p / 900p 预设。

---

## 8. 网络、端口转发与文件传输

### VM 的网络长什么样

| 后端 | 网络实现 | 地址 |
|---|---|---|
| QEMU | SLIRP 用户态 TCP/IP 栈，无需 root、无 TUN/TAP | 客户机 10.0.2.15/24，网关 10.0.2.2 |
| AVF | Android 内置网络共享，udhcpc 取 IP | 由 Android 网络栈分配 |

SLIRP 的 DNS 转发器（10.0.2.3）在 Android 上不可用，所以 `/etc/resolv.conf` 会写入设备自身的解析器，再补上 `8.8.8.8` / `1.1.1.1`。QEMU 的 netdev 上 IPv6 是关闭的。

### 三条自动注入的转发

每次启动 VM 都会自动加，不出现在转发列表里也删不掉：

| 服务 | 主机端口 | 客户机端口 | 条件 |
|---|---|---|---|
| SSH (dropbear) | 9922 | 22 | 设置里 SSH 开关打开时 |
| X11 / VNC | 5900（仅回环） | 5900 | 始终 |
| PulseAudio | 4713（仅回环） | 4713 | 始终 |

### 加自定义转发规则

路径：**Settings → Port forwards → Add**，填主机端口、VM 端口、协议（TCP / UDP / Both）。规则通过 QMP 实时生效，不用重启 VM，且跨重启持久化。

```sh
# 客户机内起服务
podman run -d -p 8080:80 nginx

# 设置里添加：主机 8080 -> VM 8080, TCP

# 同一 Wi-Fi 的笔记本上访问
curl http://<phone-ip>:8080
```

> **⛔ 1024 以下的主机端口绑不上**
>
> Android 应用是非特权用户，没有 `CAP_NET_BIND_SERVICE`，内核直接拒绝绑定 1024 以下的套接字。这是内核规则，无 root 无解。所以 SSH 用 9922 而不是 22；**规则本身会被接受，但 VM 启动时绑定失败并记 bind 错误**。
>
> 变通办法就是全部用高位端口：主机 8080 → 客户机 80，主机 8443 → 客户机 443。

> **💡 选端口的小技巧**
>
> Android 把 32768 及以上分给普通应用，所以**主机端口别用 ≥32768**，容易冲突。规则上限 2048 条。

### 传文件：首选 SFTP

```sh
sftp -P 9922 root@<phone-ip>
```

SSH 已经在 9922 上跑着，SFTP 助手随 dropbear 一起提供，**无需第二个服务、无需额外规则、无需任何设置**。WinSCP、FileZilla 都能用。旧镜像（v1.2.7 之前）如果连上就断，在客户机里跑一次 `apk add openssh-sftp-server`。

FTP 需要钉住被动端口范围并设置 `pasv_address`（要加 11 条转发）；NFSv4 可用但导出必须在 `/mnt/persist` 这类 ext4 路径上且带 `fsid=0`；**SMB / Windows 文件共享完全不可行**（445 是特权端口，且内核没编 cifs）。

---

## 9. 进阶：开启 AVF 硬件加速

这是唯一能让 Podroid 跑到接近原生速度的办法。默认的 QEMU 用 TCG 软件模拟 CPU，CPU 密集型任务慢 5–20 倍；AVF 让客户机直接跑在物理核心上，没有翻译、没有双重 JIT。

### 支持设备

Pixel 8、8a、9、9 Pro、9 Pro XL、10 全系，以及任何出厂自带 pKVM 的其他 Android 设备。

> **⛔ 关键陷阱：只支持受保护 VM 的设备用不了**
>
> Auto 模式检查四个条件，必须全部满足才会用 AVF：
>
> ① 存在 `android.software.virtualization_framework` 特性
> ② 已授予 `MANAGE_VIRTUAL_MACHINE`
> ③ 已授予 `USE_CUSTOM_VIRTUAL_MACHINE`
> ④ 设备声明支持**非受保护 VM**
>
> Podroid 的客户机镜像是标准 Alpine squashfs，不满足受保护 VM 的签名与度量要求。所以**只声明支持受保护 VM 的设备（典型如高通骁龙机型）根本无法在 AVF 下运行 Podroid**，Auto 会静默回退到 QEMU。唯一希望是 root 后直接开 `/dev/kvm` 或 `/dev/gunyah`，而 Podroid 刻意设计为免 root。

### 三步开启

**第 1 步 · 确认设备支持 AVF**

```sh
adb shell pm list features | grep virtualization_framework
```

有 `android.software.virtualization_framework` 输出 = 支持；没输出 = 没 pKVM，AVF 不可用。

**第 2 步 · 授予两个开发模式权限**（系统 UI 里不暴露）

```sh
adb shell pm grant com.excp.podroid android.permission.MANAGE_VIRTUAL_MACHINE
adb shell pm grant com.excp.podroid android.permission.USE_CUSTOM_VIRTUAL_MACHINE
```

没有电脑？装 [Shizuku](https://shizuku.rikka.app/)，用 ADB 或无线调试启动它，然后在它的 `rish` shell 里直接执行上面两条 `pm grant` 命令（去掉 `adb shell` 前缀）。

**第 3 步 · 跑 AVF 诊断** —— **Settings → About → "AVF (pKVM) diagnostic"**。它会检查功能标志、两个权限、`/apex/com.android.virt`、AVF 系统服务、内核报告的 hypervisor 能力。全过之后把 Backend 设为 **Auto**（或强制 AVF）即可启动。

> **💡 权限生命周期**
>
> 两个 pm grant 权限**重启后、就地覆盖安装后都保留**，OTA 或 Podroid 更新也无需重新授予。但**卸载重装会失效**，必须重新执行。

### 切换后端的注意事项

- 切换必须**先停止 VM**，设置在下一次启动时生效。
- AVF **不支持 USB 直通**（没有 QMP 通道），只有 QEMU 后端支持。
- AVF 的 CPU 数量只能选「1 个」或「全部核心」，没有中间值。部分 OEM 的 AVF 构建禁用了客户机网络，那个没有应用内解法，只能退回 QEMU。
- v1.2.5 起多核 AVF 会自动降级重试：如果全核启动失败，会用更少核心重试并记住能启动的最高核心数。

---

## 10. 内建命令行工具

这些工具随 VM 内置，两种后端都能用，专门用来从 VM 里反过来操作 Android 侧。

| 命令 | 用途 | 示例 |
|---|---|---|
| `podroid-forward` | 在 shell 里管理端口转发，效果与设置里添加的完全相同 | `podroid-forward add 8888 80 tcp`<br>`podroid-forward list`<br>`podroid-forward clean` |
| `podroid-notify` | 发 Android 通知，适合长时间构建 / 下载 / cron 任务 | `podroid-notify --title Podroid --priority high "容器崩了"` |
| `podroid-open` | 在手机默认浏览器打开 URL（仅 http/https） | `podroid-open https://github.com/ExTV/Podroid` |
| `podroid-power` | 从 VM 内部控制 VM 生命周期 | `podroid-power stop`<br>`podroid-power restart`<br>`podroid-power status` |
| `podroid-server` | 服务器/无头模式：屏幕转全黑最低亮度，最小化 OLED 耗电与烧屏 | `podroid-server on` / `off` / `status`<br>长按屏幕 3 秒退出 |

原理上是一个叫 `podroid-hostd` 的守护进程，通过 virtio-console（QEMU）或 vsock（AVF）把请求中继到 Android 侧执行。任何能访问客户机套接字的进程都能用这些工具 —— 把 `/run/podroid-host.sock` 绑定挂载进容器，容器内也能用。

> **💡 podroid-power 和 poweroff 的区别**
>
> 用 `podroid-power stop` 是经应用路由的干净停止，应用会回到停止状态。在客户机里直接敲 `poweroff`，应用会把它当成意外退出。

---

## 11. 性能预期

| 后端 | CPU 速度 | 适用设备 | USB 直通 |
|---|---|---|---|
| QEMU (TCG) | 简单负载慢 2–5 倍，CPU 密集型慢 5–20 倍 | 任何 ARM64 Android 8+ | 支持 |
| AVF (pKVM) | 接近原生，无 TCG 翻译 | Pixel 8+ 等 pKVM 设备 | 不支持 |

QEMU 已经做了不少调优（多线程 TCG、更大的 TB 缓存、磁盘 I/O 用 iothread），但 TCG 的开销对「执行大量不同代码路径」的负载最致命 —— 编译器、JVM 启动、`npm install` 属于重灾区；IO 密集型和内存密集型负载相对好很多。

---

## 12. 常见问题排查

### 切应用后 VM 死了 / "QEMU crashed (SIGKILL)"

**原因**：Android 12+ 的幻影进程杀手。QEMU 是前台服务的子进程，一旦被归类为幻影进程就会吃到 SIGKILL（退出码 137）。

**修复**：一次性 ADB 命令，跨重启有效：

```sh
adb shell device_config set_sync_disabled_for_tests persistent
adb shell device_config put activity_manager max_phantom_processes 2147483647
adb shell settings put global settings_enable_monitor_phantom_procs false
```

有 root 的话在设备本机执行：

```sh
su -c "device_config set_sync_disabled_for_tests persistent"
su -c "device_config put activity_manager max_phantom_processes 2147483647"
su -c "setprop persist.sys.fflag.override.settings_enable_monitor_phantom_procs false"
```

### QEMU 崩溃并提示 "Downloads sharing is unstable"

ARMv9.2 设备（含 Pixel 10）上 virtio-9p 驱动的已知问题。去设置里**关掉 Downloads sharing**，崩溃就停了。这是未解决的公开问题。

### AVF 启动即停，reason=5 / STOP_REASON_REBOOT

Pixel 8 / 8a / 9 上的已知问题：pKVM 固件在客户机内核启动完成前触发停止事件。排查流程是启用 **Verbose AVF logging** → 复现 → 用 **Export diagnostic log** 导出日志。期间把 Backend 切回 QEMU 继续用。

### 卡在 "Starting" 起不来

有两个超时：**10 秒**（QEMU 建不出 Unix socket，通常是上次崩溃留下的陈旧 socket 文件，强制停止再打开）和 **60 秒**（socket 开了但客户机一直没发 `Ready!`）。去 **Settings → Export diagnostic log** 看 `console.log`，最后几行通常能看出卡在哪。

### 改了 RAM / CPU 数量但没变化

RAM 和 CPU 在启动时传给 QEMU，**不支持热改**。要从主屏幕停止 VM 再重启。相比之下，**端口转发规则和 SSH 开关的改动无需重启即时生效**。

### X11 画面模糊或卡顿

因为用未压缩 Raw 编码。打开查看器设置，选 **720p 或 900p** 预设，或调低 Render scale。

### 外接鼠标右键退出全屏

Android 全屏手势处理器抢先拦截了右键。用**双指触摸**来右键，这条路径会正确传到客户机 X 会话。

### 首次启动特别慢

正常现象。dropbear 首次生成三套主机密钥，在低端设备的 TCG 模拟下要 15–60 秒。密钥一旦写进持久化 overlay 就不再重新生成，后续启动直接跳过。

### 更新后 apk 还显示旧版本

v1.2.7 之前创建的 VM 会遇到：文件其实已经是新的 Alpine 3.24，但包数据库在持久化 overlay 里还记着旧版本。在客户机跑一次 `apk upgrade` 恢复一致即可。v1.2.7+ 创建的 VM 不受影响。

---

## 13. 速查表

| 项目 | 值 |
|---|---|
| 下载地址 | github.com/ExTV/Podroid/releases |
| 架构 / 系统要求 | ARM64 only / Android 8 (API 26)+ |
| 默认凭据 | `root` / `podroid` |
| SSH | `ssh root@<phone-ip> -p 9922` |
| SFTP | `sftp -P 9922 root@<phone-ip>` |
| VNC / 音频端口 | 5900 / 4713（均仅绑定回环） |
| 存储大小 | 2 GB – 512 GB，只能增不能减 |
| 默认 RAM | 512 MB（建议手动调大） |
| 首次 / 后续启动 | 约 50–60 秒 / 10–15 秒 |
| 端口转发上限 | 2048 条；主机端口避开 ≥32768 |
| 需要重启 VM 的设置 | RAM、CPU 数、Backend、X11 DPI、USB 直通、Downloads sharing、存储大小 |
| 无需重启的设置 | 端口转发规则、SSH 开关 |

> **✅ 上手最快的一条路**
>
> 装 APK → 向导里把存储设成 32 GB、RAM 设成 2048 MB、打开 SSH → Start VM → 终端里先跑一遍 `adb` 那三条防幻影进程杀手的命令（重要！）→ 然后 `podman run --rm alpine echo hi` 验证。跑通之后再考虑装 Xfce 桌面和开 AVF。

---

*Podroid 使用教程 · 基于官方文档 v1.2.9 整理*
*Podroid 是 GPLv2 自由软件，仓库：[github.com/ExTV/Podroid](https://github.com/ExTV/Podroid)*
