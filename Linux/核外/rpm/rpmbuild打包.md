# rpmbuild 打包
目标是把 rpmbuild 的核心概念、标准流程、常见命令、服务类软件打包方式和排错方法串成一份可直接上手的教程。

## 1. 什么是 RPM 和 rpmbuild
+ **RPM**：Linux 下常见的软件包格式与底层包管理机制。
+ **rpm 命令**：负责安装、卸载、查询、校验本地 `.rpm` 包。
+ **dnf/yum**：更上层的包管理工具，负责仓库管理、依赖解析，并最终调用 rpm。
+ **rpmbuild**：用来把源码、脚本、配置文件、systemd 服务文件等内容构建成 RPM 包。

可以简单理解为：

```latex
dnf/yum 负责“拿包并解决依赖”
rpm     负责“安装和管理包”
rpmbuild 负责“制作包”
```

## 2. RPM 打包
### 2.1 rpm 包包含哪些东西？
一个 RPM 包通常包含：

+ 可执行文件，如 `/usr/bin/*`
+ 配置文件，如 `/etc/*`
+ 服务文件，如 `/usr/lib/systemd/system/*.service`
+ 文档、license、man 手册
+ 安装/升级/卸载脚本
+ 依赖关系、架构信息、校验信息等元数据
+ 源码 rpm 包包含源码

最终**哪些文件会进 RPM 包，不是看你复制了什么，而是看 **`%files`** 里声明了什么**。

+ 在 `%install` 里放进 `%{buildroot}`，但没写到 `%files`：不会被打进包
+ 写到了 `%files`，但实际文件不存在：打包会失败

### 2.2 rpm 包打包流程
<font style="color:rgb(15, 17, 21);">写 </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">foo.spec</font>`<font style="color:rgb(15, 17, 21);"> → 源码放入 </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">SOURCES/</font>`<font style="color:rgb(15, 17, 21);"> → 执行 </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">rpmbuild -ba foo.spec</font>`<font style="color:rgb(15, 17, 21);"> → 得到 </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">foo-1.0-1.src.rpm</font>`<font style="color:rgb(15, 17, 21);"> 和 </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">foo-1.0-1.x86_64.rpm</font>`

### 2.3 rpm 包分类
| 分类 | 名称 | 内容与用途 |
| --- | --- | --- |
| 源码包 | <font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">package-name-<version>-<release>.src.rpm</font> | <font style="color:rgb(15, 17, 21);">含源码、补丁、spec 文件，用于编译生成二进制 RPM</font> |
| 主包 | <font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">package-name-<version>-<release>.<arch>.rpm</font> | <font style="color:rgb(15, 17, 21);">软件运行时必需的核心可执行文件、配置和基础库</font> |
| 开发包 | <font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">package-name-devel-<version>-<release>.<arch>.rpm</font> | <font style="color:rgb(15, 17, 21);">头文件、静态库、pkgconfig，用于编译依赖此软件的程序</font> |
| <font style="color:rgb(15, 17, 21);">调试包</font> | <font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">package-name-debuginfo-<version>-<release>.<arch>.rpm</font> | <font style="color:rgb(15, 17, 21);">调试符号表，用于 gdb 调试、崩溃分析</font> |
| <font style="color:rgb(15, 17, 21);">文档包</font> | <font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">package-name-doc-<version>-<release>.<arch>.rpm</font> | <font style="color:rgb(15, 17, 21);">说明文档、手册、示例</font> |
| <font style="color:rgb(15, 17, 21);">库包</font> | <font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">package-name-libs-<version>-<release>.<arch>.rpm</font> | <font style="color:rgb(15, 17, 21);">共享库.so ,供多个程序使用</font> |
| <font style="color:rgb(15, 17, 21);">静态库包</font> | <font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">package-name-static-<version>-<release>.<arch>.rpm</font> | <font style="color:rgb(15, 17, 21);">静态库.a，用于静态链接</font> |
| <font style="color:rgb(15, 17, 21);">工具包</font> | <font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">package-name-tools-<version>-<release>.<arch>.rpm</font> | <font style="color:rgb(15, 17, 21);">辅助命令行工具或脚本</font> |
| <font style="color:rgb(15, 17, 21);">内核模块包</font> | <font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">package-name-kmod-<version>-<release>.<arch>.rpm</font> | <font style="color:rgb(15, 17, 21);">内核模块，需要与内核版本匹配</font> |


## 3. rpm 与 dnf/yum 的关系
```latex
用户
  ↓
dnf / yum      ← 自动处理仓库和依赖
  ↓
rpm            ← 执行本地包安装、查询、卸载
  ↓
文件系统 + RPM 数据库
```

### 什么时候优先用哪个
+ **安装本地包做测试**：`rpm -ivh` 或 `dnf install ./xxx.rpm`
+ **正式部署，且希望自动补依赖**：优先 `dnf install`
+ **查看包内容、脚本、元数据**：用 `rpm -q*` 系列命令

### <font style="color:rgb(15, 17, 21);">rpm 与 dnf 的区别</font>
| <font style="color:rgb(15, 17, 21);">对比项</font> | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">rpm</font>` | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">dnf</font>` |
| --- | --- | --- |
| <font style="color:rgb(15, 17, 21);">依赖处理</font> | **<font style="color:rgb(15, 17, 21);">不自动解决依赖</font>**<font style="color:rgb(15, 17, 21);">，缺依赖直接报错</font> | **<font style="color:rgb(15, 17, 21);">自动解决依赖</font>**<font style="color:rgb(15, 17, 21);">，从仓库下载安装</font> |
| <font style="color:rgb(15, 17, 21);">仓库支持</font> | <font style="color:rgb(15, 17, 21);">不支持仓库</font> | <font style="color:rgb(15, 17, 21);">支持仓库，可从网络安装</font> |
| <font style="color:rgb(15, 17, 21);">安装本地包</font> | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">rpm -ivh xxx.rpm</font>` | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">dnf install xxx.rpm</font>` |
| <font style="color:rgb(15, 17, 21);">卸载</font> | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">rpm -e 包名</font>` | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">dnf remove 包名</font>` |
| <font style="color:rgb(15, 17, 21);">查询</font> | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">rpm -q</font>`<br/><font style="color:rgb(15, 17, 21);">、</font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">rpm -ql</font>` | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">dnf info</font>`<br/><font style="color:rgb(15, 17, 21);">、</font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">dnf list</font>` |
| <font style="color:rgb(15, 17, 21);">升级</font> | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">rpm -Uvh</font>` | `<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">dnf upgrade</font>` |
| <font style="color:rgb(15, 17, 21);">底层关系</font> | <font style="color:rgb(15, 17, 21);">底层工具</font> | <font style="color:rgb(15, 17, 21);">上层工具，最终调用 rpm 完成安装</font> |


+ `**<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">rpm</font>**`<font style="color:rgb(15, 17, 21);">：底层工具，只管装/卸/查，</font>**<font style="color:rgb(15, 17, 21);">不管依赖</font>**<font style="color:rgb(15, 17, 21);">。</font>
+ `**<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">dnf</font>**`<font style="color:rgb(15, 17, 21);">：上层工具，</font>**<font style="color:rgb(15, 17, 21);">自动处理依赖</font>**<font style="color:rgb(15, 17, 21);">，优先用它。</font>
+ **<font style="color:rgb(15, 17, 21);">二进制 RPM</font>**<font style="color:rgb(15, 17, 21);">：可直接装。</font>
+ **<font style="color:rgb(15, 17, 21);">源码 RPM（.src.rpm）</font>**<font style="color:rgb(15, 17, 21);">：不能直接装，只能用来重建二进制包。</font>

## 4. 安装打包工具
在 RPM 系发行版上安装：

```bash
sudo dnf install -y rpm-build rpmdevtools rpmlint
```

适用于：

+ RHEL
+ CentOS Stream
+ Fedora
+ openEuler
+ Rocky Linux
+ AlmaLinux
+ 部分麒麟/统信 RPM 环境

> 说明：`rpmdevtools` 并不是所有 Debian/Ubuntu 环境都适用，本文教程以 RPM 系系统为主。
>

## 5. 初始化 rpmbuild 目录结构
执行：

```bash
rpmdev-setuptree
```

默认会在当前用户家目录下生成：

```latex
~/rpmbuild/
├── BUILD/        # 解压源码、执行编译的工作目录
├── BUILDROOT/    # 模拟安装根目录
├── RPMS/         # 生成的二进制 RPM 包
├── SOURCES/      # 源码包、补丁、配置源文件
├── SPECS/        # .spec 文件
└── SRPMS/        # 生成的源码 RPM 包
```

后续最常打交道的是：

+ `SOURCES/`
+ `SPECS/`
+ `RPMS/`
+ `BUILDROOT/`

## 6. `%{buildroot}` 是什么
`%{buildroot}` 是 **RPM 构建时的虚拟根目录**，用来模拟真实系统的 `/`。

你在 `%install` 阶段**不能直接把文件安装到真实系统路径**，而是必须先装进 `%{buildroot}`，最后再由 rpmbuild 打包。

例如：

```plain
%install
mkdir -p %{buildroot}/usr/bin
install -m 0755 myapp %{buildroot}/usr/bin/
```

这表示：

+ 构建阶段先把 `myapp` 放到临时根目录里
+ 最终用户安装 RPM 后，它才会出现在真实系统的 `/usr/bin/myapp`

### 为什么一定要这样做
+ 避免构建过程污染当前系统
+ 可以重复构建和检查包内容
+ 能清楚区分“构建机环境”和“安装目标环境”

### 常见错误
错误写法：

```plain
install -m 0755 myapp /usr/bin/
```

正确写法：

```plain
install -m 0755 myapp %{buildroot}/usr/bin/
```

## 7. `.spec` 文件是什么
`.spec` 是 RPM 打包的核心控制文件。它定义：

+ 包名、版本、依赖、描述
+ 源码从哪里来
+ 怎么解压、怎么编译、怎么安装
+ 最终打哪些文件进包
+ 安装和卸载时需要执行哪些脚本

可以把它理解成 RPM 的“打包配方”。

## 8. `.spec` 文件基本结构
一个最小化的 `.spec` 文件通常长这样：

```plain
Name:           hello-kernel
Version:        1.0
Release:        1%{?dist}
Summary:        A dummy package for learning rpmbuild
License:        GPL-3.0-or-later
Source0:        %{name}-%{version}.tar.gz
BuildArch:      noarch

%description
A dummy package for learning rpmbuild.

%prep
%setup -q

%build
: 

%install
mkdir -p %{buildroot}/usr/share/hello
printf 'Hello from kernel!\n' > %{buildroot}/usr/share/hello/message

%files
/usr/share/hello/message

%changelog
* Sun May 25 2026 Your Name <you@example.com> - 1.0-1
- Initial package
```

## 9. `.spec` 中最重要的字段
### 9.1 头部字段
| 字段 | 作用 |
| --- | --- |
| `Name` | 包名 |
| `Version` | 软件版本 |
| `Release` | 打包发布号 |
| `Summary` | 简短描述 |
| `License` | 许可证 |
| `URL` | 项目主页，可选 |
| `Source0` | 源码压缩包 |
| `BuildRequires` | 构建依赖 |
| `Requires` | 运行依赖 |
| `BuildArch` | 架构，如 `x86_64`、`noarch` |


### 9.2 常见区段
| 区段 | 作用 |
| --- | --- |
| `%description` | 详细描述 |
| `%prep` | 解压源码、打补丁 |
| `%build` | 编译 |
| `%install` | 安装到 `%{buildroot}` |
| `%files` | 声明最终进包的文件 |
| `%pre` | 安装前脚本 |
| `%post` | 安装后脚本 |
| `%preun` | 卸载前脚本 |
| `%postun` | 卸载后脚本 |
| `%changelog` | 变更记录 |


## 10. rpmbuild 的标准构建流程
rpmbuild 一般遵循下面这条主线：

```latex
准备源码 → %prep → %build → %install → %files 校验 → 生成 RPM
```

### 10.1 `%prep`
负责准备源码，一般会解压 `Source0`：

```plain
%prep
%setup -q -n %{name}-%{version}
```

其中：

+ `-q`：安静模式
+ `-n`：指定解压后的顶层目录名

`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">%setup -q</font>`<font style="color:rgb(15, 17, 21);"> </font><font style="color:rgb(15, 17, 21);">默认做两件事：</font>

1. <font style="color:rgb(15, 17, 21);">解压</font><font style="color:rgb(15, 17, 21);"> </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">Source0</font>`<font style="color:rgb(15, 17, 21);"> </font><font style="color:rgb(15, 17, 21);">指定的那个压缩包到</font><font style="color:rgb(15, 17, 21);"> </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">BUILD/</font>`<font style="color:rgb(15, 17, 21);"> </font><font style="color:rgb(15, 17, 21);">目录。</font>
2. <font style="color:rgb(15, 17, 21);">解压后，自动</font><font style="color:rgb(15, 17, 21);"> </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">cd</font>`<font style="color:rgb(15, 17, 21);"> </font><font style="color:rgb(15, 17, 21);">进入</font><font style="color:rgb(15, 17, 21);"> </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">%{name}-%{version}</font>`<font style="color:rgb(15, 17, 21);"> </font><font style="color:rgb(15, 17, 21);">目录。</font>

<font style="color:rgb(15, 17, 21);">所以：</font>

+ **<font style="color:rgb(15, 17, 21);">压缩包名</font>**<font style="color:rgb(15, 17, 21);"> </font><font style="color:rgb(15, 17, 21);">← 由</font><font style="color:rgb(15, 17, 21);"> </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">Source0</font>`<font style="color:rgb(15, 17, 21);"> </font><font style="color:rgb(15, 17, 21);">决定。</font>
+ **<font style="color:rgb(15, 17, 21);">解压后进入的目录名</font>**<font style="color:rgb(15, 17, 21);"> ← 默认由 </font>`<font style="color:rgb(15, 17, 21);background-color:rgb(235, 238, 242);">%{name}-%{version}</font>`<font style="color:rgb(15, 17, 21);"> 决定。</font>

<font style="color:rgb(15, 17, 21);">如果解压后压缩包的顶层目录名称和 %{name}-%{version}  不一致，则可以通过指定 -n 参数来指定解压后的顶层目录名称。</font>

### 10.2 `%build`
如果需要编译，就在这里执行编译命令：

```plain
%build
gcc -g -O2 -Wall -o myprogram src/myprogram.c
```

如果是不需要编译的纯脚本/纯配置包，也可以留空或写：

```plain
%build
:
```

### 10.3 `%install`
把产物安装到 `%{buildroot}`：

```plain
%install
mkdir -p %{buildroot}%{_bindir}
install -m 0755 myprogram %{buildroot}%{_bindir}/
```

### 10.4 `%files`
列出最终打包的路径：

```plain
%files
%{_bindir}/myprogram
```

## 11. 可以编译，也可以不编译
这是很多人最容易混淆的点。

**rpmbuild 只负责执行 spec 里的步骤，并不强制要求你一定要编译。**

### 11.1 场景一：源码编译型打包
适合：

+ C/C++ 程序
+ 需要针对目标平台构建的软件
+ 从源码生成二进制

示例：

```plain
%build
make %{?_smp_mflags}

%install
make install DESTDIR=%{buildroot}
```

### 11.2 场景二：预编译/脚本型打包
适合：

+ shell/python/perl 脚本
+ 配置文件包
+ 已提前在 CI 中产出的二进制
+ 闭源程序分发

示例：

```plain
%build
:

%install
mkdir -p %{buildroot}%{_bindir}
install -m 0755 myapp %{buildroot}%{_bindir}/
```

## 12. 最小可运行示例：打一个最简单的 RPM 包
### 12.1 准备源码目录
```bash
cd ~
mkdir -p hello-kernel-1.0
echo 'Hello from kernel!' > hello-kernel-1.0/message
```

### 12.2 制作源码包
```bash
# 制作压缩包
tar -czf ~/rpmbuild/SOURCES/hello-kernel-1.0.tar.gz -C ~/hello-kernel-1.0 .  

# 查看压缩包的内容
tar -tzf hello-kernel-1.0.tar.gz
```

注意这里的压缩包名和顶层目录名要匹配：

+ 压缩包：`hello-kernel-1.0.tar.gz`
+ 顶层目录：`hello-kernel-1.0/`

### 12.3 编写 spec 文件
保存到 `~/rpmbuild/SPECS/hello-kernel.spec`：

```plain
Name:           hello-kernel
Version:        1.0
Release:        1%{?dist}
Summary:        A dummy package for learning rpmbuild
License:        GPL-3.0-or-later
Source0:        %{name}-%{version}.tar.gz
BuildArch:      noarch

%description
A dummy package for learning rpmbuild.

%prep
%setup -q

%build
:

%install
mkdir -p %{buildroot}/usr/share/hello
install -m 0644 message %{buildroot}/usr/share/hello/message

%files
/usr/share/hello/message

%changelog
* Sun May 25 2026 Your Name <you@example.com> - 1.0-1
- Initial package
```

### 12.4 开始构建
```bash
rpmbuild -bb ~/rpmbuild/SPECS/hello-kernel.spec
```

### 12.5 成功后产物位置
```bash
~/rpmbuild/RPMS/noarch/hello-kernel-1.0-1.noarch.rpm
```

> 修正说明：如果 `BuildArch: noarch`，最终包名通常是 `noarch.rpm`，不是 `x86_64.rpm`。
>

### 12.6 安装验证
```bash
sudo dnf install -y ~/rpmbuild/RPMS/noarch/hello-kernel-1.0-1.noarch.rpm
cat /usr/share/hello/message
```

## 13. 常用 rpmbuild 命令
### 13.1 构建二进制包
```bash
rpmbuild -bb your-package.spec
```

### 13.2 构建源码包
```bash
rpmbuild -bs your-package.spec
```

### 13.3 同时构建源码包和二进制包
```bash
rpmbuild -ba your-package.spec
```

### 13.4 查看已安装包信息
```bash
rpm -qi your-package-name  //  查看软件包的元数据信息，描述信息
rpm -ql your-package-name   // 查询软件包中所有安装到系统上的文件的路径
  rpm -q --scripts your-package-name  // 查看这个 RPM 包在安装、升级、卸载时执行的脚本
```

### 13.5 查询某个文件属于哪个包
```bash
rpm -qf /path/to/file
```

### 13.6 查看 rpm 包内容但不安装
```bash
rpm -qpl your-package.rpm
rpm -qpi your-package.rpm

/**
-q	query	进入查询模式
-p	package file	指定操作对象是 .rpm 文件，而不是已安装的包名
-l	list	列出包内包含的文件
-i	info	显示包的详细信息
*/
```

## 14. 常用宏
| 宏 | 含义 | 常见值 |
| --- | --- | --- |
| `%{_topdir}` | rpmbuild 根目录 | `~/rpmbuild` |
| `%{_sourcedir}` | 源文件目录 | `~/rpmbuild/SOURCES` |
| `%{_specdir}` | spec 目录 | `~/rpmbuild/SPECS` |
| `%{_builddir}` | 编译目录 | `~/rpmbuild/BUILD` |
| `%{_buildrootdir}` | buildroot 根目录 | `~/rpmbuild/BUILDROOT` |
| `%{buildroot}` | 当前包的临时安装根目录 | 动态生成 |
| `%{_bindir}` | 可执行文件目录 | `/usr/bin` |
| `%{_sbindir}` | 管理命令目录 | `/usr/sbin` |
| `%{_unitdir}` | systemd unit 目录 | 通常是 `/usr/lib/systemd/system` |
| `%{_sysconfdir}` | 配置目录 | `/etc` |
| `%{_datadir}` | 数据目录 | `/usr/share` |


建议优先写宏，不要硬编码路径。

例如：

```plain
install -m 0755 myapp %{buildroot}%{_bindir}/
install -m 0644 myapp.service %{buildroot}%{_unitdir}/
```

## 15. RPM 安装过程到底发生了什么
执行下面命令时：

```bash
sudo rpm -ivh pkg.rpm
```

或：

```bash
sudo dnf install ./pkg.rpm
```

大致流程如下：

```latex
1. 读取 RPM 头部信息
2. 检查架构、依赖、签名
3. 执行 %pre
4. 解包并写入文件系统
5. 处理 %config 配置文件策略
6. 更新 RPM 数据库
7. 执行 %post
```

如果是卸载：

```latex
%preun → 删除文件 → %postun
```

## 16. 安装/升级/卸载脚本怎么写
服务类软件经常会用到脚本区段。

### 16.1 安装后脚本 `%post`
```plain
%post
if [ $1 -eq 1 ]; then
    systemctl daemon-reload
    systemctl enable myapp.service >/dev/null 2>&1 || :
    systemctl start myapp.service >/dev/null 2>&1 || :
fi
```

### 16.2 卸载后脚本 `%postun`
```plain
%postun
if [ $1 -eq 0 ]; then
    systemctl stop myapp.service >/dev/null 2>&1 || :
    systemctl disable myapp.service >/dev/null 2>&1 || :
    systemctl daemon-reload
fi
```

### 16.3 `$1` 的常见含义
不同脚本阶段对 `$1` 的语义会有差异，但在实际打包里，最常见判断方式是：

+ `$1 -eq 1`：通常表示安装后首次配置场景
+ `$1 -eq 0`：通常表示彻底卸载场景

实际项目里如果脚本逻辑比较敏感，建议结合发行版文档和测试结果确认。

## 17. 配置文件与日志文件要区别处理
### 17.1 配置文件
如果文件允许用户改动，建议使用：

```plain
%files
%config(noreplace) /etc/myapp/myapp.conf
```

这样升级时：

+ 用户改过的配置尽量保留
+ 不会被新包直接覆盖

### 17.2 日志文件
日志文件一般**不建议直接作为普通静态文件打进包**，更常见做法是：

+ 由程序自己创建
+ 或通过 `tmpfiles.d`、logrotate、首次启动逻辑创建

如果只是教学演示，也可以在 `%post` 中用 `touch` 创建：

```plain
%post
install -d -m 0755 /var/log/myapp >/dev/null 2>&1 || :
touch /var/log/myapp/myapp.log || :
chmod 0644 /var/log/myapp/myapp.log || :
```

> 说明：生产环境更推荐让应用本身或日志系统接管日志文件，而不是在 `%files` 里长期维护一个空日志文件。
>

## 18. 实战示例：把一个 systemd 服务打成 RPM
下面用你文档里的 `sdet-monitor` 作为完整示例。

目标：

+ 安装一个监控程序 `det-monitor`
+ 安装对应的 systemd 服务
+ 安装后自动启用并启动服务
+ 卸载时自动停止并禁用服务

## 19. 项目目录建议
先准备项目目录：

```bash
mkdir -p ~/det-monitor/{src,systemd}
cd ~/det-monitor
```

目录结构建议如下：

```latex
det-monitor/
├── src/
│   └── det-monitor.c
└── systemd/
    └── det-monitor.service
```

> 修正说明：源码压缩包里通常不必再额外塞一个 `rpm/` 目录，只要最终把 spec 复制到 `~/rpmbuild/SPECS/` 即可。
>

## 20. 编写 systemd 服务文件
文件：`systemd/det-monitor.service`

```properties
[Unit]
Description=Monitor for det command execution
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/det-monitor
Restart=always
RestartSec=5
User=root
Group=root
StandardOutput=journal
StandardError=journal
Nice=19
IOSchedulingClass=idle

[Install]
WantedBy=multi-user.target
```

### 说明
+ `Restart=always`：异常退出后自动重启
+ `RestartSec=5`：5 秒后重启
+ `Nice=19`：降低 CPU 调度优先级
+ `IOSchedulingClass=idle`：尽量减少 I/O 干扰

如果程序不必须以 root 运行，生产环境建议尽量使用专用低权限用户。

## 21. 编写服务类软件的 spec 文件
文件：`~/rpmbuild/SPECS/det-monitor.spec`

```plain
Name:           det-monitor
Version:        1.0
Release:        1%{?dist}
Summary:        Monitor for execution of det command
License:        GPL-3.0-or-later
Source0:        %{name}-%{version}.tar.gz
BuildRequires:  gcc, systemd
Requires(post): systemd
Requires(preun): systemd
Requires(postun): systemd

%description
This service monitors the execution of the det command.

%prep
%setup -q -n %{name}-%{version}

%build
gcc -g -O2 -Wall -o det-monitor src/det-monitor.c

%install
rm -rf %{buildroot}
install -d %{buildroot}%{_bindir}
install -d %{buildroot}%{_unitdir}
install -m 0755 det-monitor %{buildroot}%{_bindir}/det-monitor
install -m 0644 systemd/det-monitor.service %{buildroot}%{_unitdir}/det-monitor.service

%files
%{_bindir}/det-monitor
%{_unitdir}/det-monitor.service

%post
if [ $1 -eq 1 ]; then
    systemctl daemon-reload >/dev/null 2>&1 || :
    systemctl enable det-monitor.service >/dev/null 2>&1 || :
    systemctl start det-monitor.service >/dev/null 2>&1 || :
fi

%preun
if [ $1 -eq 0 ]; then
    systemctl stop det-monitor.service >/dev/null 2>&1 || :
    systemctl disable det-monitor.service >/dev/null 2>&1 || :
fi

%postun
systemctl daemon-reload >/dev/null 2>&1 || :

%changelog
* Sun May 25 2026 Your Name <you@example.com> - 1.0-1
- Initial package
```

## 22. 制作标准源码包
源码包必须满足两个要求：

1. 压缩包文件名与 `Source0` 对应
2. 解压后的顶层目录名与 `%setup` 期望一致

在项目目录执行：

```bash
cd ~/det-monitor
mkdir -p det-monitor-1.0
cp -a src systemd det-monitor-1.0/
tar -czf ~/rpmbuild/SOURCES/det-monitor-1.0.tar.gz det-monitor-1.0
rm -rf det-monitor-1.0
```

也可以先检查压缩包内容：

```bash
tar -tf ~/rpmbuild/SOURCES/det-monitor-1.0.tar.gz | head
```

你应该看到类似：

```latex
det-monitor-1.0/
det-monitor-1.0/src/
det-monitor-1.0/src/det-monitor.c
det-monitor-1.0/systemd/
det-monitor-1.0/systemd/det-monitor.service
```

## 23. 构建 RPM 包
执行：

```bash
rpmbuild -ba ~/rpmbuild/SPECS/det-monitor.spec
```

成功后通常会生成：

```latex
~/rpmbuild/RPMS/x86_64/det-monitor-1.0-1.x86_64.rpm
~/rpmbuild/SRPMS/det-monitor-1.0-1.src.rpm
```

> 注意：这里是 `x86_64` 还是 `aarch64`，取决于你的构建机架构以及包本身是否声明为 `noarch`。
>

## 24. 安装与验证
### 24.1 安装
```bash
sudo dnf install -y ~/rpmbuild/RPMS/*/det-monitor-1.0-1.*.rpm
```

也可以用：

```bash
sudo rpm -ivh ~/rpmbuild/RPMS/*/det-monitor-1.0-1.*.rpm
```

### 24.2 检查服务状态
```bash
systemctl status det-monitor.service
systemctl is-enabled det-monitor.service
```

预期：

+ 状态为 `active (running)` 或至少已成功启动
+ `is-enabled` 输出 `enabled`

### 24.3 查看包内容
```bash
rpm -ql det-monitor
rpm -q --scripts det-monitor
```

## 25. 卸载与验证
卸载：

```bash
sudo rpm -e det-monitor
```

或：

```bash
sudo dnf remove -y det-monitor
```

验证：

```bash
rpm -q det-monitor
systemctl status det-monitor.service
ls -l /usr/bin/det-monitor
ls -l /usr/lib/systemd/system/det-monitor.service
```

预期：

+ 包查询不到
+ 服务单元文件已移除
+ 可执行文件已移除

## 26. 常见错误与排查方法
### 26.1 `File not found` 或 `%files` 报错
原因通常是：

+ `%install` 没把文件放进 `%{buildroot}`
+ `%files` 路径写错
+ 目标文件名和实际文件名不一致

检查思路：

```bash
find ~/rpmbuild/BUILDROOT -type f | sort
```

### 26.2 `%prep` 阶段解压失败
常见原因：

+ `Source0` 文件名不匹配
+ 压缩包顶层目录名与 `%setup -n ...` 不一致

### 26.3 构建依赖缺失
比如：

```latex
error: Failed build dependencies: gcc is needed by xxx
```

处理：

```bash
sudo dnf install -y gcc make systemd-rpm-macros
```

### 26.4 服务安装后没有启动
检查：

```bash
systemctl status det-monitor.service
journalctl -u det-monitor.service -xe
rpm -q --scripts det-monitor
```

### 26.5 日志或配置文件被覆盖
检查是否正确使用：

+ `%config(noreplace)`
+ 应用自身日志目录策略
+ 是否误把运行时文件直接打进 `%files`

## 27. 调试技巧
### 27.1 检查 spec 质量
```bash
rpmlint ~/rpmbuild/SPECS/det-monitor.spec
rpmlint ~/rpmbuild/RPMS/*/det-monitor-*.rpm
```

### 27.2 查看构建目录
```bash
ls -l ~/rpmbuild/BUILD/
ls -l ~/rpmbuild/BUILDROOT/
```

### 27.3 查看最终 buildroot 内容
```bash
tree ~/rpmbuild/BUILDROOT
```

### 27.4 查看包内文件列表
```bash
rpm -qpl ~/rpmbuild/RPMS/*/det-monitor-*.rpm
```

### 27.5 查看安装/卸载脚本
```bash
rpm -qp --scripts ~/rpmbuild/RPMS/*/det-monitor-*.rpm
```

## 28. 最佳实践
### 28.1 压缩包命名规范
推荐：

```latex
<name>-<version>.tar.gz
```

例如：

```latex
det-monitor-1.0.tar.gz
```

### 28.2 顶层目录名保持一致
例如压缩包内应是：

```latex
det-monitor-1.0/
```

### 28.3 能用宏就用宏
不要硬编码：

```plain
/usr/bin
/usr/lib/systemd/system
```

推荐：

```plain
%{_bindir}
%{_unitdir}
```

### 28.4 不要在 `%install` 中改真实系统
所有写入都应进入 `%{buildroot}`。

### 28.5 区分构建依赖和运行依赖
+ `BuildRequires`：编译时依赖
+ `Requires`：安装运行时依赖

### 28.6 尽量不用 root 做构建
构建阶段通常应在普通用户下执行，安装阶段再使用 `sudo`。

### 28.7 服务包优先考虑 systemd 生命周期
至少要考虑：

+ 安装后 `daemon-reload`
+ 首次安装后 `enable/start`
+ 卸载前 `stop/disable`
+ 卸载后 `daemon-reload`

## 29. 一份从零到打包成功的最短操作清单
如果你只想快速上手，按下面执行即可。

### 29.1 初始化环境
```bash
sudo dnf install -y rpm-build rpmdevtools rpmlint
rpmdev-setuptree
```

### 29.2 准备源码包
```bash
mkdir -p ~/demo-1.0
echo 'hello rpm' > ~/demo-1.0/README
tar -czf ~/rpmbuild/SOURCES/demo-1.0.tar.gz -C ~ demo-1.0
```

### 29.3 写 spec
```plain
Name:           demo
Version:        1.0
Release:        1%{?dist}
Summary:        Simple demo rpm
License:        MIT
Source0:        %{name}-%{version}.tar.gz
BuildArch:      noarch

%description
Simple demo rpm.

%prep
%setup -q

%build
:

%install
mkdir -p %{buildroot}/usr/share/demo
install -m 0644 README %{buildroot}/usr/share/demo/README

%files
/usr/share/demo/README
```

保存到：

```bash
~/rpmbuild/SPECS/demo.spec
```

### 29.4 构建
```bash
rpmbuild -bb ~/rpmbuild/SPECS/demo.spec
```

### 29.5 安装验证
```bash
sudo dnf install -y ~/rpmbuild/RPMS/noarch/demo-1.0-1.noarch.rpm
rpm -ql demo
```

## 30. 总结
你可以把 RPM 打包记成三句话：

1. **源码、脚本、服务文件先准备好，放进 **`SOURCES/`
2. **用 **`.spec`** 明确描述“怎么解压、怎么编译、怎么安装、打哪些文件”**
3. **用 **`rpmbuild`** 构建，再用 **`rpm/dnf`** 安装验证**

真正掌握 rpmbuild，关键不是死记命令，而是理解下面这条链路：

```latex
Source0 → %prep → %build → %install → %{buildroot} → %files → RPM 包 → rpm/dnf 安装
```

