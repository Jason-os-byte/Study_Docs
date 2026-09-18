# Linux 内核 RPM 编译、打包、安装与清理指南

适用：CentOS Stream 9 / RHEL 9，x86_64；示例源码 `linux-6.6.157`。

两条路线：

- **专业方式**（第 3 节）：`rpmdev-setuptree` 标准 rpmbuild，产出全套子包（`kernel` / `kernel-devel` / `kernel-headers`）及源码包，可复现、可分发。
- **快速方式**（第 5 节）：源码树内 `make binrpm-pkg`，仅主包，适合单机快速验证。

含安装、跨机分发与彻底卸载。

---

## 1. 依赖

```bash
dnf groupinstall -y "Development Tools"
dnf install -y ncurses-devel bison flex elfutils-libelf-devel openssl-devel \
               dwarves rpm-build rpmdevtools bc perl python3 rsync wget tar xz
```

## 2. 源码与配置

```bash
cd /root/project/kernel
tar -xf linux-6.6.157.tar.xz

# 生成供 rpmbuild 使用的源码包（顶层目录改为 linux）；第 3 节专业方式用
mkdir -p ~/rpmbuild/SOURCES
tar -czf ~/rpmbuild/SOURCES/linux.tar.gz \
    --transform 's,^linux-6.6.157,linux,' linux-6.6.157

cd linux-6.6.157
cp /boot/config-5.14.0-745.el9.x86_64 .config     # 以本机自带内核配置为基线
```

### 2.1 必改项

```bash
scripts/config --set-str SYSTEM_TRUSTED_KEYS ""
scripts/config --set-str SYSTEM_REVOCATION_KEYS ""
```

原因：RHEL/CentOS 的 `.config` 里这两个值指向 `certs/rhel.pem` 等**红帽私有证书**，而 kernel.org 的 vanilla 源码中不存在这些文件；不改，编译到 `certs/` 阶段会直接报 `No rule to make target 'certs/rhel.pem'`。置空即不嵌入这些内置信任密钥，不影响功能，只去掉红帽的签名信任链。

### 2.2 调试信息开关（默认关闭，避免构建体积膨胀约 3 倍）

`CONFIG_DEBUG_INFO` 是无提示符的隐藏符号，由 DWARF choice 反向 `select`，必须关 choice 本身：

```bash
scripts/config -e DEBUG_INFO_NONE \
               -d DEBUG_INFO_DWARF_TOOLCHAIN_DEFAULT \
               -d DEBUG_INFO_DWARF4 -d DEBUG_INFO_DWARF5 \
               -d DEBUG_INFO_BTF -d GDB_SCRIPTS
```

可选：

```bash
scripts/config -d MODULE_SIG        # 关闭模块签名，省去 sign-file 与临时文件
make olddefconfig
# 可选：仅保留当前已加载的模块（4300+ -> 数百），大幅省空间/时间
# make localmodconfig && make olddefconfig
```

`make localmodconfig` 会读取 `lsmod`（本机当前实际加载的模块），把**已加载的置为 `m`/`y`，未加载的全部置 `n`**，配置里只开启本机在用的驱动，构建体积与时间骤降。代价是换硬件/换机器可能缺驱动，**通用内核不要用**。

反过来，**需要 `kernel-debuginfo`（crash/gdb/perf 调试）时**：

```bash
# 1) 配置里开启调试信息：选一个 DWARF choice、关掉 NONE
scripts/config -d DEBUG_INFO_NONE -e DEBUG_INFO_DWARF_TOOLCHAIN_DEFAULT
make olddefconfig
grep CONFIG_DEBUG_INFO .config        # 应出现 CONFIG_DEBUG_INFO=y

# 2) 生成的 spec 默认用这两行关闭了 debuginfo 抽取，删掉它们后再打包
sed -i '/^%define debug_package %{nil}/d; /^%define __spec_install_post /d' \
    ~/rpmbuild/SPECS/kernel.spec
rpmbuild -ba ~/rpmbuild/SPECS/kernel.spec

# 3) 安装
dnf install -y ~/rpmbuild/RPMS/x86_64/kernel-debuginfo-*.rpm
```

代价：体积/编译时间大幅上升（仅 BUILD 就可能 15G+），务必先确认磁盘。若之后重新生成 spec，这两行会回来，需再删一次。

### 2.3 验证（必须，否则不要开始编译）

```bash
grep -E 'CONFIG_DEBUG_INFO|CONFIG_MODULE_SIG' .config
# 期望：
#   CONFIG_DEBUG_INFO_NONE=y
#   # CONFIG_DEBUG_INFO is not set        （隐藏符号，通常整行不存在）
#   # CONFIG_MODULE_SIG is not set
```

---

## 3. 专业方式：标准 rpmbuild 打包全套子包

内核 spec 由 `scripts/package/mkspec` 基于 `scripts/package/kernel.spec` 生成，其 `%prep` 使用 `%setup -q -n linux`，因此 `SOURCES` 中源码包顶层目录名必须为 `linux`。

标准 rpmbuild **不加** `--without devel`，默认 `with_devel=1`，会产出 `kernel`、`kernel-devel`、`kernel-headers` 三个二进制包；用 `-ba` 另出源码包。

### 3.1 建立目录与准备 SOURCES

```bash
rpmdev-setuptree     # 生成 ~/rpmbuild/{BUILD,BUILDROOT,RPMS,SOURCES,SPECS,SRPMS}

# linux.tar.gz 已在第 2 节生成；这里放配置与补丁
cp /root/project/kernel/linux-6.6.157/.config ~/rpmbuild/SOURCES/config
: > ~/rpmbuild/SOURCES/diff.patch
```

> `: > ~/rpmbuild/SOURCES/diff.patch`：`:` 是 shell 空操作命令，配合 `>` 重定向，作用是**创建/清空**该文件为空。spec 的 `%prep` 会执行 `patch -p1 < diff.patch`，空补丁合法（退出码 0），故用空文件占位；若你确实改动了源码，把改动导出成补丁写入此文件即可。

### 3.2 生成 spec

顶层 Makefile 只把 `%pkg` / `%src-pkg` 目标转发给 `scripts/Makefile.package`，**没有 `kernel.spec` 目标**，所以直接 `make kernel.spec` 会报 `没有规则可制作目标"kernel.spec"`。正确做法是调用该 Makefile 并传入所需变量：

```bash
cd /root/project/kernel/linux-6.6.157
export ARCH=x86 srctree="$PWD" KERNELRELEASE="$(make -s kernelrelease)"
make -f scripts/Makefile.package kernel.spec
cp kernel.spec ~/rpmbuild/SPECS/kernel.spec
```

验证（应含 `with_devel ... 1`、`KERNELRELEASE 6.6.157`）：

```bash
grep -E 'with_devel|KERNELRELEASE|ARCH|pkg_release' ~/rpmbuild/SPECS/kernel.spec
```

> 说明：`srctree` / `ARCH` / `KERNELRELEASE` 是 `scripts/package/mkspec` 读取的环境变量，必须显式导出；`KERNELRELEASE` 由 `make -s kernelrelease` 得到（依赖第 2 节已 `make olddefconfig`）。

### 3.3 打包

```bash
# -bb 仅二进制包；-ba 同时产出源码包（推荐，便于目标机重建/留档）
rpmbuild -ba ~/rpmbuild/SPECS/kernel.spec
```

### 3.4 产物

| 文件 | 内容 |
|---|---|
| `~/rpmbuild/RPMS/x86_64/kernel-6.6.157-1.x86_64.rpm` | 内核本体（vmlinuz + 全部模块 + System.map + config） |
| `~/rpmbuild/RPMS/x86_64/kernel-devel-6.6.157-1.x86_64.rpm` | 编译外部模块所需的头文件与构建树 |
| `~/rpmbuild/RPMS/x86_64/kernel-headers-6.6.157-1.x86_64.rpm` | 用户态 UAPI 头（`/usr/include`） |
| `~/rpmbuild/SRPMS/kernel-6.6.157-1.src.rpm` | 源码包（`-ba` 时产出） |

> 本 spec 设了 `%define debug_package %{nil}`，不产 `kernel-debuginfo`。需要调试符号须先启用 `CONFIG_DEBUG_INFO` 并删除该行，体积会显著增大。

---

## 4. 内核包家族与安装选择（专业视角）

发行版把内核拆成 `kernel-core` / `kernel-modules` / `kernel-modules-core` / `kernel` 元包；本 build 的 mkspec 生成的是**单包 `kernel`**（全部模块都在内），不要照搬发行版包名。

| 包 | 包含内容 | 作用 | 何时安装 |
|---|---|---|---|
| `kernel` | `/lib/modules/<ver>/`（`vmlinuz`、全部 `.ko` 模块及 `modules.*` 索引）；`/boot/vmlinuz-<ver>`、`System.map-<ver>`、`config-<ver>`；`%post` 经 `kernel-install` 生成的 `/boot/initramfs-<ver>.img` 与 BLS 条目 | 内核本体 | 运行该内核：**必装** |
| `kernel-devel` | `/usr/src/kernels/<ver>/`（编译外部模块所需的内核头文件、`Makefile`/`Kconfig`、`Module.symvers` 等）；`/lib/modules/<ver>/build` 符号链接 | 编译外部模块（DKMS/驱动） | 为该内核开发/编译模块：装 |
| `kernel-headers` | `/usr/include/` 下的用户态 UAPI 头（`linux/`、`asm/`、`asm-generic/`、`drm/`、`sound/` 等） | 编译用户态程序（glibc 等） | 需要系统 UAPI 头跟随 6.6：装；否则保留发行版自带 |
| `kernel-debuginfo` | `/usr/lib/debug/lib/modules/<ver>/` 下的调试符号（`vmlinux`、各 `.ko` 的 `.debug`）；部分发行版另拆 `kernel-debuginfo-common-<arch>` | crash/gdb/perf 调试内核 | 需要调试内核：装（体积最大） |
| `kernel-<ver>.src.rpm` | `linux.tar.gz` + `config` + `diff.patch` + `kernel.spec` | 源码包，可在目标机 `rpmbuild --rebuild` 复现 | 不是“安装”，用于留档/跨机重建 |

要点：

- **基于内核开发模块，需要的是 `kernel-devel`，不是 `kernel-headers`。**
- `kernel-headers` 带无版本 `Obsoletes: kernel-headers` 且 `%files` 打包整个 `/usr/include`，安装会**顶替发行版同名包**；与发行版 glibc/工具链混用有风险，按需选择。

### 4.1 全套安装

```bash
cd ~/rpmbuild/RPMS/x86_64
dnf install -y ./kernel-6.6.157-1.x86_64.rpm          # 必装
dnf install -y ./kernel-devel-6.6.157-1.x86_64.rpm    # 需要开发/编译模块时装
# dnf install -y ./kernel-headers-6.6.157-1.x86_64.rpm  # 仅当要让系统 UAPI 头跟随 6.6
```

### 4.2 引导（initramfs / BLS / grub）

CentOS 9 的 `/usr/bin/kernel-install`（systemd-udev 提供）会在 rpm `%post` 中自动完成：

- 生成 `/boot/initramfs-6.6.157.img`
- 写入 `/boot/loader/entries/<machine-id>-6.6.157.conf`
- 复制 `/boot/vmlinuz-6.6.157`、`System.map-6.6.157`、`config-6.6.157`

若目标机无 `kernel-install` 或未自动生成，手动补齐：

```bash
# 1) 生成该内核的 initramfs：内核启动到挂载真正根分区前所需的最小驱动/脚本环境（LVM、dm、文件系统、virtio 等）
dracut -f --kver 6.6.157

# 2) 在 BLS 里新建引导条目 /boot/loader/entries/<machine-id>-6.6.157.conf；
#    --copy-default 复制当前默认条目的内核参数（root=、resume= 等），免手写
grubby --add-kernel=/boot/vmlinuz-6.6.157 \
       --initrd=/boot/initramfs-6.6.157.img \
       --title="CentOS Stream (6.6.157) 9" --copy-default

# 3) 重新生成 GRUB 主配置，使菜单/默认项与当前条目一致
grub2-mkconfig -o /boot/grub2/grub.cfg
```

`grubby --add-kernel` 各参数含义：

| 参数 | 含义 |
|---|---|
| `--add-kernel=/boot/vmlinuz-6.6.157` | 新建一个引导条目，指定用这个内核镜像 |
| `--initrd=/boot/initramfs-6.6.157.img` | 该条目使用的 initramfs |
| `--title="..."` | GRUB 菜单里显示的名字 |
| `--copy-default` | 复制当前默认条目的启动参数（`root=`、`resume=`、`rd.lvm.lv=`、`crashkernel=` 等），不用手写 |

作用：把磁盘上的内核登记成一条**可启动的菜单项**（BLS 下写入 `/boot/loader/entries/<machine-id>-6.6.157.conf`）。没有这一步，内核文件虽然存在，但 GRUB 菜单里看不到、也启动不了。

一句话总结：`grubby --add-kernel` 把内核镜像（`linux`）+ initramfs（`initrd`）+ 菜单名（`title`）+ 启动参数（`options`，由 `--copy-default` 从当前默认项复制）组成一条引导条目，写成 `/boot/loader/entries/<machine-id>-6.6.157.conf`；GRUB 启动时经 `blscfg` 读到它，菜单里就会出现这个新内核，选中即可正确启动。

注意：这一步只让它“出现在菜单里”。要让它**默认**启动，还需下一节的 `grubby --set-default`——即“能被选择启动”由 `--add-kernel` 负责，“默认选它”由 `--set-default` 负责。

### 4.3 设置默认并验证

```bash
grubby --set-default=/boot/vmlinuz-6.6.157
grubby --info=ALL | grep -E '^(index|kernel|title)'
reboot
uname -r
```

---

## 5. 快速方式：源码树内 `make binrpm-pkg`（仅主包）

适合单机快速出包；隐含 `--without devel`，只产 `kernel` 与 `kernel-headers`，**不产 `kernel-devel`**。

```bash
cd /root/project/kernel/linux-6.6.157
make -j$(nproc) binrpm-pkg
# 产物：rpmbuild/RPMS/x86_64/{kernel,kernel-headers}-6.6.157-1.x86_64.rpm
```

安装、引导同第 4 节；需要 `kernel-devel` 请用第 3 节专业方式。

> 快捷法（复用标准 rpmbuild 输出目录）：
> ```bash
> rpmdev-setuptree
> make binrpm-pkg RPMOPTS="--define '_topdir $HOME/rpmbuild'"
> ```

---

## 6. 分发到其他目标机

前提：与构建机**架构一致**（x86_64），目标为 RPM 系发行版（RHEL/CentOS/Fedora）。

```bash
# 分发运行包 + 开发包（按需）
scp kernel-6.6.157-1.x86_64.rpm [kernel-devel-6.6.157-1.x86_64.rpm] root@target:/tmp/
# 目标机执行
dnf install -y /tmp/kernel-6.6.157-1.x86_64.rpm
dnf install -y /tmp/kernel-devel-6.6.157-1.x86_64.rpm     # 需要开发模块时
# 无 kernel-install 时补：
dracut -f --kver 6.6.157
grubby --add-kernel=/boot/vmlinuz-6.6.157 --initrd=/boot/initramfs-6.6.157.img \
       --title="Linux 6.6.157" --copy-default
grub2-mkconfig -o /boot/grub2/grub.cfg
grubby --set-default=/boot/vmlinuz-6.6.157
```

也可分发源码包在目标机重建（目标机需装齐第 1 节依赖）：

```bash
rpmbuild --rebuild kernel-6.6.157-1.src.rpm
```

注意：

- **Secure Boot**：本方式产出内核未签名，开启 Secure Boot 的机器无法启动，需关闭 Secure Boot 或自行签名。
- **模块签名**：若未关 `MODULE_SIG`，模块由自动生成的 `certs/signing_key.pem` 签名，内核内嵌对应公钥可正常加载；跨机分发建议固定密钥或关闭签名。
- `kernel-headers` 会顶替目标机 UAPI 头，非必要不分发；需要目标机开发模块时，分发 `kernel-devel`。

---

## 7. 卸载内核与彻底清理

### 7.1 RPM 安装的内核（自带内核、本指南产出的内核）

一个内核由多个包组成，**用包名卸载，不要手工删文件**：

```bash
# 查看
rpm -qa | grep -E '^kernel(-core|-modules|-modules-core|-devel|-headers)?-' | sort

# 卸载（元包会带出 core/modules/modules-core）
dnf remove -y kernel-5.14.0-480.el9
```

`dnf remove` 会自动删除 `/boot/*-<ver>`、`/lib/modules/<ver>`、注销 rpm 记录，并经 `%preun` 调用 `kernel-install remove` 删除 BLS 条目。

### 7.2 非 RPM（手工 `make install`）安装的内核

RPM 库中查不到，必须手动清理三处：

```bash
VER=6.1.150
rm -rf /lib/modules/$VER
rm -f  /boot/vmlinuz-$VER /boot/System.map-$VER /boot/config-$VER \
       /boot/initramfs-$VER.img /boot/initramfs-${VER}kdump.img
rm -f  /boot/loader/entries/*-$VER.conf
grub2-mkconfig -o /boot/grub2/grub.cfg
```

### 7.3 残留检查清单

```bash
# 引导条目
ls /boot/loader/entries/

# 失效条目（引用的 vmlinuz 不存在）
for f in /boot/loader/entries/*.conf; do k=$(grep -oP '^linux \K/.*' "$f"); \
  [ -e "/boot$k" ] || echo "STALE: $f"; done

# 模块目录：rpm 未登记却存在的 /lib/modules/<ver> 即手工内核残留
ls /lib/modules/

# 常见可回收项
dnf remove -y kernel-debuginfo-* kernel-debuginfo-common-*    # 通常体积最大
dnf remove -y kernel-devel-<无对应内核的版本>
dnf clean all && rm -rf /var/cache/PackageKit/* /var/cache/dnf/*
rm -rf /usr/src/debug/kernel-*
```

### 7.4 删除顺序建议

1. `uname -r` 确认当前运行内核不是要删的目标；
2. 至少保留一个可启动内核 + rescue 条目；
3. 先删 RPM 内核，再删手工内核；
4. 收尾 `grubby --set-default` + `grub2-mkconfig`。

---

## 8. 常见问题

| 现象 | 原因 | 处理 |
|---|---|---|
| `make olddefconfig` 后 `CONFIG_DEBUG_INFO=y` 又出现 | `DEBUG_INFO` 为隐藏符号，被 `DEBUG_INFO_DWARF_*` choice `select` | 关 choice：`-e DEBUG_INFO_NONE -d DEBUG_INFO_DWARF_*` |
| `No space left on device`（`%install` / `sign-file`） | 磁盘或 inode 满；DEBUG_INFO 未关；模块过多 | 关 DEBUG_INFO、`localmodconfig`，用 `df -h` / `df -i` 核查 |
| `Failed build dependencies` | 第 1 节依赖未装齐 | 按第 1 节补齐，或 `rpmbuild --nodeps`（不推荐） |
| `make kernel.spec` 报 `没有规则可制作目标"kernel.spec"` | 顶层 Makefile 无此目标，只有 `%pkg`/`%src-pkg` 转发 | 按 3.2 用 `make -f scripts/Makefile.package ... kernel.spec` |
| `It's not recommended to have unversioned Obsoletes: kernel-headers` | spec 固有警告 | 忽略；安装该 `kernel-headers` 会顶替发行版同名包，按 4 节按需选择 |
| `已安装的软件包 ... 已不可用`（reinstall 失败） | 仓库已移除该旧版本 | 改装仓库最新版本，或从 vault 获取对应 rpm |
