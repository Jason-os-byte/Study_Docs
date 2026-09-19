# Linux 内核启动完整流程

> 平台：x86_64 ｜ 固件：BIOS / UEFI ｜ 引导器：GRUB2（BLS）｜ 早期用户空间：initramfs（dracut）｜ init：systemd
> 本文先给出**整体流程框架**，再逐阶段展开细节；每章末尾均说明“本阶段产出、交接给谁”，保证全程连贯。

---

# 第一章 整体流程框架

## 1.1 一张图看懂

```
        ┌──────────────┐
        │  电源 / 复位  │
        └──────┬───────┘
               ▼
┌──────────────────────────────────────────────────────┐
│ 阶段一 · 固件（BIOS / UEFI）                           │
│   自检 → 选择引导设备 → 加载并运行引导程序的第一小段      │
└──────────────────────────┬───────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────┐
│ 阶段二 · 引导加载器（GRUB2）                            │
│   接力加载自身 → 读配置 → 显示菜单 → 加载内核 + initramfs │
└──────────────────────────┬───────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────┐
│ 阶段三 · 内核（ring 0）                                 │
│   自解压 → 初始化子系统与驱动 → 执行第一个用户态程序       │
└──────────────────────────┬───────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────┐
│ 阶段四 · initramfs（PID 1 = /init）                     │
│   加载根所需驱动 → 挂载真实根 → switch_root             │
└──────────────────────────┬───────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────┐
│ 阶段五 · 用户空间（PID 1 = systemd）                    │
│   启动各服务 → 到达登录界面                             │
└──────────────────────────────────────────────────────┘
```

## 1.2 五个阶段一览

| 阶段 | 执行者 | 运行环境 | 核心任务 | 交接给 |
|---|---|---|---|---|
| 一 · 固件 | BIOS / UEFI | 实模式 | 自检、选设备、加载引导程序 | GRUB |
| 二 · 引导加载器 | GRUB2 | 实模式 / 保护模式 | 加载内核与 initramfs | 内核 |
| 三 · 内核 | Linux 内核 | ring 0 | 初始化系统、拉起 PID 1 | initramfs 的 `/init` |
| 四 · initramfs | `/init` | ring 3 | 挂载真实根 | 真实根的 `/sbin/init` |
| 五 · 用户空间 | systemd | ring 3 | 启动服务 | 登录 |

## 1.3 两条贯穿全程的主线

**主线一：控制权接力** —— 每一棒只负责把下一棒准备好并跳过去。

```
固件 ──► GRUB ──► 内核 ──► /init ──► systemd
```

**主线二：PID 1 的三次变身** —— 同一个进程（PID 1）先后被 `execve` 替换三次，是后三章的连接纽带。

```
kernel_init（内核线程）──execve──► /init（initramfs）──execve──► /sbin/init（systemd）
```

## 1.4 四个关键交接点

| 交接 | 从 → 到 | 交接内容 |
|---|---|---|
| ① | 固件 → GRUB | 控制权 + 引导设备 |
| ② | GRUB → 内核 | `boot_params`（内存图、命令行、initrd 地址）+ 内核入口 |
| ③ | 内核 → `/init` | 把 PID 1 变成用户态程序（ring 0 → ring 3） |
| ④ | `/init` → systemd | `switch_root` 换根，PID 1 执行真实根的 init |

> 记住这张框架与两条主线，下面五章只是逐段展开。

---

# 第二章 阶段一 · 固件（BIOS / UEFI）

**本阶段目标：完成自检，选定引导设备，加载并运行引导程序的第一小段。** 固件不认识文件系统，也不认识内核，它只做“搬运第一段代码”这件事。

## 2.1 上电与复位

CPU 复位后进入**实模式**，`CS:IP = F000:FFF0`，即物理地址 `0xFFFFFFF0`，指向主板固件。此处只有一条跳转指令，进入固件初始化代码。

## 2.2 先分清两个词

| 术语 | 是什么 | 回答的问题 |
|---|---|---|
| **引导设备**（boot device） | 硬件/介质，如 `/dev/sda`、U 盘、光驱 | 从哪块盘启动 |
| **引导程序**（bootloader） | 该设备上的软件，如 GRUB | 用什么代码启动系统 |

关系：固件从**引导设备**上把**引导程序**的第一小段读出来执行。

## 2.3 BIOS 路径

| 步骤 | 执行者 | 动作 |
|---|---|---|
| 1 | 固件 | POST 自检：初始化内存、总线、外设 |
| 2 | 固件 | 按启动顺序选择引导设备 |
| 3 | 固件 | 读设备**扇区 0（512 字节）**到内存 `0x7C00`，跳转执行 |

**为什么只读扇区 0、且其中引导代码只有 446 字节**

- BIOS 启动时**固定只读扇区 0**（512 字节扇区来自早期 IBM PC 与 BIOS Int 13h 的历史约定），不会自动读后面的扇区。
- 扇区 0 即 **MBR（主引导记录）**，格式固定：

  ```
  [ 446 字节引导代码 ][ 4×16 字节分区表 ][ 2 字节签名 0x55AA ]
    偏移 0               偏移 446            偏移 510
  ```

- 分区表钉在偏移 446、签名钉在偏移 510，留给引导代码的只有 `512 - 64 - 2 = 446` 字节，**在 MBR 约定内无法扩大**。

因此 BIOS 只能加载“第一小段”引导程序；它装不下“读文件系统 + 出菜单”，**剩余部分由引导程序自己接力**（见第三章）。

**BIOS 的局限**：实模式（只能寻址 1MB）、依赖 MBR 分区表、无安全启动——这正是 UEFI 出现的原因。

## 2.4 UEFI 路径

1. 固件从 NVRAM 的启动项读取 **ESP**（EFI System Partition，FAT32）上的 `.efi` 程序。
2. RHEL/CentOS 链条：`shimx64.efi` → `grubx64.efi`（`shim` 在 Secure Boot 下校验并加载 `grubx64.efi`）。
3. `grubx64.efi` 读 `grub.cfg`，进入菜单。

**UEFI 优势**：原生 64 位、支持 GPT、Secure Boot、图形输出；引导程序是 ESP 上的**普通文件**，摆脱了“第一个扇区”的限制。

## 2.5 本阶段产出与交接

- **产出**：引导程序第一段被载入内存并开始运行。
- **交接给**：GRUB（阶段二），控制权随跳转转移。

---

# 第三章 阶段二 · 引导加载器（GRUB2）

**本阶段目标：GRUB 完成自身加载，读取配置并显示菜单，把选中的内核与 initramfs 装入内存，按引导协议交给内核。** 前提：固件已把第一段 `boot.img` 载入并跳转。

## 3.1 GRUB 的加载模型（两段 + 模块）

固件只能加载第一小段，GRUB 必须自己接力。其代码/数据分布在三个位置：

| 位置 | 内容 | 大小 | 在分区内吗 |
|---|---|---|---|
| 扇区 0（MBR） | `boot.img`（第一段），只负责加载下一段 | 446 字节 | 否（裸扇区） |
| 扇区 1–2047（MBR 后、首个分区前的空隙） | `core.img`（第二段），含文件系统驱动 | 约 1 MB | 否（裸扇区） |
| 文件系统 `/boot/grub2/`（位于分区 `/dev/sda1`，挂载点 `/boot`） | `core.img` 读取的**模块与配置**：`*.mod`、`grub.cfg` | 按需 | 是 |

```
扇区 0          : boot.img         ← 第一段
扇区 1 .. 2047  : core.img         ← 第二段（横跨几十个扇区，作为一整块读入）
文件系统内       : 模块 + grub.cfg  ← 由 core.img 读取，不是“第三段”
```

流程：

1. `boot.img` 被固件执行。
2. `boot.img` 内置磁盘读例程，并记录了 `core.img` 的起始位置与长度（安装时写入），**一次性把整个 `core.img` 读入内存**再跳转——不是“逐扇区接力”。
3. `core.img` 借助文件系统驱动读 `/boot/grub2/`，加载所需模块与 `grub.cfg`。
4. 至此 GRUB 具备完整功能，显示菜单、加载内核。

> **关键结论**：GRUB2 是“**两段 + 模块**”，不是“N 个扇区 = N 段”；扇区只是存储单位，不是逻辑阶段。（GRUB Legacy 曾为 `stage1 → stage1.5 → stage2` 三段。）

## 3.2 配置来源

三个文件/目录协作，最终产物是 `grub.cfg`：

```
/etc/default/grub  +  /etc/grub.d/*  --(grub2-mkconfig)-->  /boot/grub2/grub.cfg  --(GRUB 开机读)-->  菜单
```

| 文件/目录 | 是什么 | 作用 | 内容 |
|---|---|---|---|
| `/boot/grub2/grub.cfg` | 成品（GRUB 主配置） | GRUB 开机唯一读取的文件 | GRUB 脚本命令 |
| `/etc/default/grub` | 人改的入口 | 设置全局行为 | 键值对 |
| `/etc/grub.d/*` | 生成脚本目录 | `grub2-mkconfig` 执行它们拼出 `grub.cfg` | 按编号排序的 shell 脚本 |

- **`/boot/grub2/grub.cfg`**（UEFI 为 `/boot/efi/EFI/<distro>/grub.cfg`）：GRUB 启动时唯一读取的文件，内容是 **GRUB 脚本语言**（类 shell，非 bash）的命令，如 `set timeout=5`、`insmod xfs`、`blscfg`、`menuentry { linux ...; initrd ...; }`。它由 `grub2-mkconfig` 自动生成，**不要直接编辑**。
- **`/etc/default/grub`**：给人改的键值对，如 `GRUB_TIMEOUT`、`GRUB_DEFAULT=saved`、`GRUB_CMDLINE_LINUX`、`GRUB_ENABLE_BLSCFG`。改后必须重跑 `grub2-mkconfig`。
- **`/etc/grub.d/*`**：`00_header`（头部/超时/默认）、`10_linux`（扫描 `/boot` 生成内核项；BLS 下放 `blscfg`）、`30_os-prober`（探测其他系统）、`40_custom`（自定义条目）。

## 3.3 GRUB 模块

GRUB2 模块化：`core.img` 只带“最小可用”能力，其余功能拆成 `.mod` 文件（`/boot/grub2/i386-pc/` 或 `x86_64-efi/`），用到才加载。本机 `/boot/grub2/i386-pc/` 有 **279 个 `.mod`**，而 `core.img` 仅 **34K**。

| 类别 | 模块示例 | 作用 |
|---|---|---|
| 文件系统驱动 | `ext2.mod`、`xfs.mod`、`fat.mod` | 读各文件系统上的内核/配置 |
| 分区表 | `part_msdos.mod`、`part_gpt.mod` | 识别 MBR/GPT 分区 |
| 卷/加密 | `lvm.mod`、`luks.mod` | 读 LVM、解锁加密卷 |
| 加载内核 | `linux.mod`、`chain.mod` | 按 Linux 引导协议加载内核 / 链式加载其他引导器 |
| 菜单与界面 | `normal.mod`、`gfxterm.mod` | normal 模式（菜单解释器）、图形终端 |
| BLS | `blscfg.mod` | 读 `/boot/loader/entries/*.conf` 动态建菜单 |

`grub.cfg` 中用 `insmod` 加载：`insmod xfs`、`insmod lvm`、`insmod blscfg`。

## 3.4 BLS（Boot Loader Specification）

BLS 是 freedesktop.org / systemd 定义的跨发行版标准；`systemd-boot` 原生使用，GRUB2 通过 `blscfg` 模块支持。RHEL 9 默认 `GRUB_ENABLE_BLSCFG=true`。

此时 `grub.cfg` 不含具体内核列表，只有 `blscfg`——**启动时扫描 `/boot/loader/entries/*.conf` 动态建菜单**。单条条目示例：

```
title   CentOS Stream (6.6.157) 9
linux   /vmlinuz-6.6.157
initrd  /initramfs-6.6.157.img
options root=/dev/mapper/cs-root ro rd.lvm.lv=cs/root rhgb quiet
```

默认项存于 `/boot/grub2/grubenv` 的 `saved_entry=<entry-id>`，由 `grubby --set-default` 维护。

## 3.5 GRUB 最终做的事

1. 解析菜单、超时、默认项。
2. 把 **vmlinuz**（`bzImage`，编译好的内核镜像）与 **initramfs**（cpio.gz）读入内存。
3. 把内核命令行（`options`）写入内核可读位置。
4. 按 **Linux/x86 引导协议**填充 `boot_params`（含 `setup_header`、`e820` 内存图、initrd 地址），跳转到内核入口。

> GRUB **不“理解”内核，也不运行内核**；它加载的是**编译好的内核镜像**，不是内核源码。

## 3.6 本阶段产出与交接

- **产出**：内核与 initramfs 已在内存中就位，`boot_params` 已填好。
- **交接给**：内核（阶段三），跳转到内核入口。

---

# 第四章 阶段三 · 内核（ring 0）

**本阶段目标：内核从入口自解压，建立子系统、初始化驱动，最后由内核把第一个用户态程序 `execve` 起来，完成 ring 0 → ring 3 的交接。** 前提：GRUB 已把内核与 initramfs 装入内存并跳转到内核入口。

## 4.1 内核阶段全景

内核启动不是一条直线：`start_kernel()` 建好核心子系统后，会分出**三个执行流**。

```
GRUB 跳入内核入口（setup 代码，实模式）
   │
   ▼
[1] setup + 解压器：实模式→保护模式→长模式，解压 vmlinux，KASLR
   │  startup_64 → x86_64_start_kernel → start_kernel()
   ▼
[2] start_kernel()：线性建立核心子系统（内存/调度/中断/控制台）
   │  末尾 rest_init() 创建 PID1、PID2；自身变为 PID0（idle）
   │
   ├─ PID 0  swapper      : idle 循环
   ├─ PID 2  kthreadd     : 内核线程之母
   └─ PID 1  kernel_init  : 继续启动 ↓
   ▼
[3] PID 1：do_basic_setup() → do_initcalls() 初始化驱动（含解压 initramfs）
   │
   ▼
[4] PID 1：execve 第一个用户态程序（= initramfs 的 /init）→ ring3
            （内核职责到此结束，转第五章）
```

> **关键衔接**：initcall **不是** `start_kernel()` 直接调用的，而是由 `start_kernel()` 创建的 **PID 1** 执行的。理解“先建线程、再由线程干活”这条链路，本章就顺了。

## 4.2 入口与自解压

`/boot/vmlinuz-<ver>` 即 `bzImage`：

```
[ setup 代码 ][ 压缩的 vmlinux ][ 解压器 ]
```

GRUB 跳入的是 **setup 代码**，不是 `start_kernel`：

1. **`arch/x86/boot/`（setup 代码）**：在**实模式**下收集硬件信息，然后切换到保护模式。
   - **内存（e820）**：调用 BIOS `Int 0x15, E820` 查询物理内存布局——哪些区间是可用的 RAM、哪些是保留区（ACPI、MMIO）。
   - **视频**：调用 BIOS `Int 0x10` 查询当前显示模式与屏幕信息（`screen_info`）。
   - **命令行**：整理引导器传来的启动参数（`root=...`、`ro`、`quiet` 等）。
   - 以上信息**填入 `boot_params`**（引导器/固件与内核之间的数据交接结构）。
   - **为何必须在此时做**：BIOS 中断只在实模式可用；一旦切到保护/长模式，就再也无法调用固件。
2. **`arch/x86/boot/compressed/`（解压器）**：把压缩的 `vmlinux` 解压到内存，建立恒等映射页表，进入长模式（64 位），可选做 KASLR 地址随机化。
3. 跳入解压后的 `vmlinux` 入口 `startup_64`（`arch/x86/kernel/head_64.S`）→ `x86_64_start_kernel()` → `start_kernel()`。

## 4.3 start_kernel()：建核心子系统并创建 PID 1/2

`init/main.c` 的 `start_kernel()` 是一段**线性执行**的 C 代码，按序建立“跑 C 代码的基础设施”：

| 顺序 | 动作 | 目的 |
|---|---|---|
| 1 | `setup_arch()` | 解析 e820/ACPI、内存与页表、NUMA |
| 2 | `trap_init()` / irq | 异常、中断描述符 |
| 3 | `mm_init()` | 页分配器(buddy)、slab/slub、vmalloc |
| 4 | `sched_init()` | 调度器 |
| 5 | `rcu_init()` / timer | RCU、时钟中断 |
| 6 | `console_init()` | 早期控制台，`printk` 可见 |
| 7 | `rest_init()` | 创建 `kernel_init`(PID 1) 与 `kthreadd`(PID 2) |

执行到末尾调用 `rest_init()`，它做三件事：

1. 创建内核线程 **PID 1 `kernel_init`** —— 未来的用户空间起点；
2. 创建内核线程 **PID 2 `kthreadd`** —— 所有内核线程的父进程；
3. 当前执行流变为 **PID 0（swapper/idle）**，进入 idle 循环。

`rest_init()` 之后 `start_kernel()` **永不返回**，后续启动工作由 PID 1 接手。

## 4.4 PID 1 初始化系统：initcall

PID 1（`kernel_init`）接着运行 `kernel_init_freeable()` → `do_basic_setup()` → `do_initcalls()`，这才是各子系统/驱动的初始化入口：

```
early → core → postcore → arch → subsys → fs → rootfs → device → late
```

顺序含义：内存/调度 → 总线/驱动模型 → 文件系统 → 设备，最后收尾；驱动探测多在 `device` 及更晚。此阶段还完成两件关键事：

- `populate_rootfs()`（`rootfs` 级）：把 initramfs 解压成 `rootfs`（内存文件系统）。
- 初始化存储/文件系统驱动，为后续挂载做准备。

initcalls 跑完，PID 1 才具备“执行 init”的条件。

## 4.5 内核的最后动作：execve 第一个用户态程序

initcalls 结束后，`kernel_init` 执行用户空间起点，这也是内核的**最后一个动作**：

1. 选择程序：有 initramfs 且含 `/init` → 选它；否则选 `/sbin/init`、`/etc/init`、`/bin/init`、`/bin/sh`（可用 `init=` 覆盖）。
2. `execve` 该程序，**ring 0 → ring 3**，PID 1 变为用户态程序。

> **与第五章的分工（不是重复，是交接）**：内核只负责“把 `/init` 执行起来”，**不运行 `/init` 的逻辑**。有 initramfs 的系统上，这个被执行的程序就是 initramfs 的 `/init`；它接下来做什么（挂载真实根、`switch_root`）是第五章的内容。内核到此交棒。

## 4.6 本阶段产出与交接

- **产出**：PID 1 已存在并完成 initcalls；initramfs 已解压为 `rootfs`；驱动就绪。
- **交接给**：initramfs 的 `/init`（阶段四）——内核以 `execve` 将其拉起。

---

# 第五章 阶段四 · initramfs（PID 1 = /init）

**本阶段目标：用临时根里的驱动和工具，找到并挂载真实根，然后切过去。** 前提：上一阶段末尾，内核已 `execve` 了 initramfs 的 `/init`——现在 PID 1 就是 `/init`，本章从它开始执行讲起。

## 5.1 为什么需要 initramfs

真实根可能位于 LVM、RAID、dm-crypt、iSCSI、NVMe 等之上，挂载它需要对应驱动和用户态工具。把这些全编进内核会臃肿；于是先用一个**临时根**（initramfs）加载驱动、挂载真实根。若根就在简单分区上，也可以不用 initramfs。

## 5.2 dracut 生成什么

`dracut` 生成 `/boot/initramfs-<ver>.img`（cpio.gz），内含：

- 精简的 `/init` 程序（PID 1 在这里继续）；
- 挂载根所需的内核模块；
- `udev`、LVM/crypt 等工具；
- dracut 的启动逻辑。

## 5.3 /init 做什么

```
/init (PID 1)
  → 加载存储/文件系统驱动，启动 udev
  → 按 root= / rd.lvm.lv= 等参数定位并挂载真实根（加密卷先解锁）
  → switch_root：chroot 到真实根，丢弃临时根
  → execve /sbin/init（真实根上的 init）   ← 转第六章
```

`switch_root` 是本章关键动作：**把 PID 1 的执行环境从内存根换成磁盘根**，PID 1 随之 `execve` 真实根的 init。

## 5.4 本阶段产出与交接

- **产出**：真实根已挂载，PID 1 的执行环境已切换到磁盘根。
- **交接给**：真实根的 `/sbin/init`（阶段五），PID 1 `execve` 之。

---

# 第六章 阶段五 · 用户空间（PID 1 = systemd）

**本阶段目标：真实根的 init（通常是 systemd）作为 PID 1 拉起所有服务，直到出现登录。** 前提：PID 1 已 `switch_root` 并执行真实根的 init。

## 6.1 systemd 的角色

`/sbin/init` 通常是 `systemd`。PID 1 经 `switch_root` 后 `execve` 它，systemd 成为 PID 1，负责整个用户空间。

## 6.2 启动流程

1. 读取 `/etc/systemd/system/`、`/usr/lib/systemd/system/` 的 unit。
2. 按依赖图**并行**启动：`sysinit.target` → `basic.target` → `multi-user.target`（或 `graphical.target`）。
3. 挂载文件系统、启动 udev、网络、日志、getty 等。
4. 到达 `default.target`，出现登录界面/提示符。

systemd 与内核的接口：内核提供 sysfs/procfs/cgroup/udev 事件，systemd 据此编排服务。

## 6.3 本阶段产出

- **产出**：系统服务就绪，出现登录，启动流程结束。

---

# 第七章 完整时间线（贯穿五阶段）

```
[电源]
  │ 复位向量 0xFFFFFFF0
  ▼
[阶段一 固件]  POST → 选引导设备 → 读扇区 0 / ESP
  │ 交接①：控制权
  ▼
[阶段二 GRUB2]  boot.img → core.img → 模块+grub.cfg → 菜单
  │ 交接②：加载 vmlinuz + initramfs，填 boot_params，跳转
  ▼
[阶段三 内核]  setup(收集硬件信息) → 解压 vmlinux → start_kernel
  │ → initcalls（初始化驱动、解压 initramfs）→ PID1 execve /init
  │ 交接③：ring0 → ring3
  ▼
[阶段四 initramfs]  /init 加载驱动 → 挂载真实根 → switch_root
  │ 交接④：PID1 execve /sbin/init
  ▼
[阶段五 用户空间]  systemd → target 依赖图 → 登录
```

---

# 第八章 关键对象与文件

| 名称 | 位置 | 说明 |
|---|---|---|
| `bzImage` / `vmlinuz` | `/boot/vmlinuz-<ver>` | 压缩内核 + setup + 解压器 |
| `initramfs` | `/boot/initramfs-<ver>.img` | 临时根，cpio.gz |
| 内核命令行 | `/proc/cmdline`、BLS `options` | `root=`、`rd.lvm.lv=`、`ro`、`quiet`、`init=`、`rdinit=` |
| BLS 条目 | `/boot/loader/entries/*.conf` | `title/linux/initrd/options` |
| GRUB 引导程序 | 扇区 0 `boot.img` + 空隙 `core.img` + `/boot/grub2/` 模块 | 三段式存放 |
| GRUB 配置 | `/boot/grub2/grub.cfg`、`/etc/default/grub`、`/etc/grub.d/*` | 成品 / 人改入口 / 生成脚本 |
| `boot_params` | 内核内存中的交接结构 | 内存图、视频信息、命令行、initrd 地址 |
| `System.map` / `config` | `/boot/System.map-<ver>`、`/boot/config-<ver>` | 符号表 / 构建配置 |
| PID 1 | `kernel_init` → `/init` → `/sbin/init` | 用户空间起点 |

---

# 第九章 观察与调试

```bash
cat /proc/cmdline                 # 本次启动的内核命令行
dmesg                             # 内核日志（含 initcall、驱动探测）
journalctl -b                     # 本次启动完整日志（内核 + 用户空间）
journalctl -b -k                  # 仅内核
systemd-analyze                   # 启动耗时分解
systemd-analyze blame             # 各服务耗时
```

内核命令行调试参数（在 GRUB 菜单按 `e` 临时添加）：

| 参数 | 作用 |
|---|---|
| `initcall_debug` | 打印每个 initcall 及耗时 |
| `earlyprintk=serial,ttyS0,115200` | 极早期串口输出 |
| `debug` / `ignore_loglevel` | 提高日志等级 |
| `init=/bin/sh` | 跳过 systemd，直接进 shell（救援） |
| `rd.break` | 在 initramfs 阶段中断（救援） |
| `rd.shell` | initramfs 失败时进入 shell |
| `nokaslr` | 关闭内核地址随机化 |

---

# 第十章 通用机制与本机特定

## 10.1 BLS 的性质

- **全称**：Boot Loader Specification。
- **性质**：freedesktop.org / systemd 定义的跨发行版标准，非本机特有。
- **谁在用**：`systemd-boot` 原生使用；GRUB2 通过 `blscfg` 模块支持。
- **是否启用看配置**：RHEL / Fedora / CentOS 9 默认 `GRUB_ENABLE_BLSCFG=true`。

## 10.2 对照表

| 环节 | 本机情况 | 通用性 |
|---|---|---|
| 架构 | x86_64 | 架构相关：复位向量、实模式/长模式仅限 x86 |
| 固件 | BIOS（无 `/sys/firmware/efi`） | BIOS/UEFI 二选一；UEFI 路径为 ESP + shim + grubx64 |
| 引导器 | GRUB2 | 常见但非唯一（systemd-boot、syslinux…） |
| BLS | 启用 | 标准通用，是否启用依发行版 |
| initramfs 生成器 | dracut | RHEL/Fedora/SUSE 用 dracut；Debian 系用 initramfs-tools |
| 根/交换 | LVM：`cs-root` / `cs-swap` | 本机布局；通用概念是 `root=` + 挂载它所需的驱动 |
| init 系统 | systemd | 主流但非唯一（sysvinit、OpenRC…） |
| 内核内部流程 | 解压、`start_kernel`、initcall、PID 1 | 通用（仅架构细节除外） |

## 10.3 使用注意

- 文中命令与路径（`/boot/grub2/grub.cfg`、`/boot/loader/entries`、`dracut`、`systemd-analyze`）基于 RHEL 9 系；换发行版需替换对应工具（如 Debian 的 `update-initramfs`、`update-grub`）。
- **阶段划分与内核内部流程（第四章）是通用的**；差异集中在“固件类型、引导器、initramfs 生成器、根分区布局、init 系统”五项。

---

# 结语

一句话概括全程：

**固件**把控制权交给 **GRUB**；GRUB 把**内核与 initramfs** 按引导协议摆进内存并跳转；**内核**自解压、初始化子系统与驱动，把 **PID 1** 拉起为 initramfs 的 **`/init`**；`/init` 加载驱动、挂载真实根后 **`switch_root`**，让 PID 1 执行真实根的 **systemd**；systemd 拉起服务，系统可用。
