RPM 包构建常见有两种方式：
- rpmbuild
- fpm

下面讲解的是通过 rpmbuild 构建。

# 1. rpmbuild 环境
执行如下命令安装 rpmbuild 和 rpmdevtools:
```
#yum install rpmbuild
#yum install rpmdevtools
```

rpm 的版本 <=4.4.x，rpmbuild 工具其默认的工作路径是 `/usr/src/redhat` 因为权限的问题，普通用户不能制作 rpm 包，制作 rpm 软件包时必须切换到 root 身份才可以。

rpm 从 4.5.x 版本开始，将 rpmbuid 的默认工作路径移动到用户家目录下的 rpmbuild 目录里，即 `$HOME/rpmbuild`，并且推荐用户在制作 rpm 软件包时尽量不要以 root 身份进行操作。

rpmbuild 默认工作路径的确定，通常由在 `/usr/lib/rpm/macros` 这个文件里的一个叫做 `%_topdir` 的宏变量来定义。如果用户想更改这个目录名，rpm官方并不推荐直接更改这个目录，而是在用户家目录下建立一个名为 `.rpmmacros` 的隐藏文件然后在里面重新定义 `%_topdir`，指向一个新的目录名。这样就可以满足某些用户的差异化需求了。通常情况下 `.rpmmacros` 文件里一般只有一行内容，比如：
```
%_topdir    $HOME/myrpmbuildenv 
```

在 `%_topdir` 目录下一般需要建立 6 个目录.

| 目录名 | 说明 | macros 中的宏名 |
| --- | --- | --- |
| BUILD | 编译 rpm 包的临时目录 | %_builddir |
| BUILDROOT | 编译后生成的软件临时安装目录 | $_buildrootdir |
| RPMS | 最终生成的可安装 rpm 包的所在目录 | %_rpmdir |
| SOURCES | 所有源代码和补丁文件的存放目录 | %_sourcedir |
| SPECS | 存放 spec 文件的目录 | %_specdir |
| SRPMS | 软件最终的 rpm 源码格式存放路径 | %_srcrpmdir |

如果有安装 rpmdevtools，可以使用 rpmdev-setuptree 命令在当前用户 home/rpmbuild 目录里自动建立上述目录。

## rpmbuild 命令选项
- `-bp xxx.spec`：制作到 %prep 段
- `-bc xxx.spec`：制作到 %build 段
- `-bi xxx.spec`：执行 spec 文件的 "%install" 阶段 (在执行了 %prep 和 %build 阶段之后)。这通常等价于执行了一次 “make install”
- `-bb xxx.spec`：制作二进制包（在执行了 %prep 和 %build 阶段之后)
- `-bs xxx.spec`：仅制作源码包
- `-bl xxx.spec`：从 spec 文件宏扩展 %files 段，检查并且验证每个文件是否存在
- `-ba xxx.spec`：表示既制作二进制包又制作 src 格式包（在执行了 %prep 和 %build 阶段之后)

# 2. spec 文件
> liunx 环境可以使用 vi 来编辑 spec 文件，windows 环境可以可以使用 sublime 编辑 spec 文件。 需要注意的是，绝对不能使用记事本来编辑，这是因为 windows 的默认换行符是 \r\n，而 liunx 的换行符是 \n。如果 spec 文件中包含多余的 \r 会导致 rpm 包创建失败。

当打包目录建立好之后，将所有用于生成 rpm 包的源代码、shell 脚本、配置文件都拷贝到 SOURCES 目录里，注意通常情况下源码的压缩格式都为 `*.tar.gz` 格式。然后，将最重要的 SPEC 文件，命名格式一般是 `"软件名-版本.spec"` 的形式，将其拷贝到 SPECS 目录下，切换到该目录下执行：
```
rpmbuild -bb 软件名-版本.spec 
```

如果系统有 rpmdevtools 工具，可以用 `rpmdev-newspec -o Name-version.spec` 命令来生成 SPEC 文件的模板，然后进行修改.

`rpmbuild -ba xxx.spec` 时，RPMBUILD 执行流程：
- 读取并解析 `xxx.spec` 文件
- 运行 `%prep` 部分来将源代码解包到一个临时目录，并应用所有补丁文件
- 运行 `%build` 部分来编译代码
- 运行 `%install` 部分来将代码安装到构建机器的目录中
- 读取 `%file` 部分的文件列表，收集文件并创建二进制和 RPM 文件
- 运行 `%clean` 部分来除去临时构建目录

执行流程图如下：  
![image](https://www.linuxidc.com/upload/2016_09/160906201819201.jpg)

## 2.1 spec 语法
### 2.1.1 宏
spec 支持通过 %define 定义宏。如：
```
%define _hardened_build 1
```
后面可通过 %{_hardened_build} 或 %_hardened_build 来引用该宏。

### 2.1.2 定义包的信息字段
#### Summary
软件包的摘要信息。摘要包含一行介绍包的文字，不要超过 50 个字符，例如：
```
Summary:	Fast, scalable and extensible HTTP/1.1 compliant caching proxy server
```

#### Name
软件包的名字，包的名称不能含有空白字符，比如空格、Tab和回车等。后面可以通过 %{name} 引用。如：`name: trafficserver`。

#### Version
软件包的版本号，版本号用于做版本的比较，rpm 的版本比较算法很复杂，最好使用统一的一种版本命名方法，比如2.1.0，要定义包的版本号。后面可通过 %{version} 引用。如：`version: 7.1.4`。

注：包的版本号中不能使用 `-`，因为 rpm 使用它来分隔 name、version、release。

#### Release
发布序列号，表明第几次打包，后面可通过 %{release} 引用。

release number 在特定版本的第一次 build 时应该以 1 开始，之后每次 build 加 1，比如：`Release: 1`。

#### License
软件的授权方式。

#### Group
软件包的分组。

#### URL
软件的主页。

#### Vendor
发行商或打包组织的信息，如 RedFlag Co,Ltd。

#### Disstribution
发行版标识。

#### Build Arch
指编译的目标处理器架构，noarch 标识不指定，但通常都是以 /usr/lib/rpm/marcros 中的内容为默认值.

#### ExcludeArch
使用 ExcludeArch 来表示不构建这些平台的 rpm 包，如：
```
ExcludeArch:	%{arm} s390 s390x
```

#### ExclusiveArch
指定只 build 这些平台的 rpm 包，如：
```
ExclusiveArch: i386 x86-64
```

#### ExcludeOs 和 ExclusiveOs
这两个指令用于指定特定的操作系统，比如:
```
ExcludeOs: windows
```
表示不 build windows 的包。

#### Prefix
- Prefix: %{_prefix}。这个主要是为了解决今后安装 rpm 包时，并不一定把软件安装到 rpm 中打包的目录的情况。这样，必须在这里定义该标识，并在编写 %install 脚本的时候引用，才能实现 rpm 安装时重新指定位置的功能。
- Prefix: %{_sysconfdir}。这个原因和上面的一样，但由于 %{_prefix} 指 /usr，而对于其他的文件，例如 /etc下的配置文件，则需要用 `%{_sysconfdir}`标识。

#### Source
如 %{name}-%{version}.tar.gz。源代码包的名称（默认时 rpmbuild 回到 SOURCES 目录中去找），这里的 name 和 version 即为上面定义的 Name 和 Version。若后面还有其他的源代码包则为 Source1，Source2 等等。后面可通过 %{source0}, %{source1} 等引用。

此外，Source 指定的路径可以是网络路径。如 `http://192.168.208.97/packages/%{name}-%{version}-src.tar.gz`

#### Patch
补丁源码，可使用 Patch1，Patch2 等标识多个补丁，后面可通过 %patch0 或 %{patch0} 引用。

#### BuildRequires
在本机编译 rpm 包时需要的辅助工具，以逗号分隔。例如，要求编译软件包时，gcc 的版本至少为 4.4.2，则可以写成 gcc >= 4.2.2。

#### Requires
编译好的 rpm 软件在其他机器上安装时，需要依赖的其他软件包，可以用 >= 或 <= 表示大于或小于某一特定版本。如 `libpng-devel >= 1.0.20 zlib `。注：">=" 号两边需用空格隔开，而不同软件名称也用空格分开.

还有例如 `PreReq`、`Requires(pre)`、`Requires(post)`、`Requires(preun)`、`Requires(postun)`、`BuildRequires` 等都是针对不同阶段的依赖指定.

#### Provides
指明本软件一些特定的功能，以便其他 rpm 识别。

#### Packager
打包者的信息。

#### %description
描述允许更详细的介绍，描述支持少量的格式化，空行用于分割段落，以空白开头（比如空格或者 Tab）的行通常会以等宽字体显示。例如：
```
%description
Apache Traffic Server is a fast, scalable and extensible HTTP/1.1 compliant
caching proxy server.
```

## 2.2 spec 主体（各阶段）
#### spec 文件阶段
| 阶段 | 读取的目录 | 写入的目录 | 具体动作 |
| ---  | ---       | ---       | ---     |
| %prep| %_sourcedir| %_builddir | 读取位于 %_sourcedir 目录的源代码和 patch 。之后，解压源代码至 %_builddir 的子目录并应用所有 patch。|
| %build| %_builddir|%_builddir|编译位于 %_builddir 构建目录下的文件。通过执行类似 ./configure && make 的命令实现。|
| %install|%_builddir|%_buildrootdir|读取位于 %_builddir 构建目录下的文件并将其安装至 %_buildrootdir 目录。这些文件就是用户安装 RPM 后，最终得到的文件。注意一个奇怪的地方: 最终安装目录 不是 构建目录。通过执行类似 make install 的命令实现。|
|%check|%_builddir|%_builddir|检查软件是否正常运行。通过执行类似 make test 的命令实现。很多软件包都不需要此步。|
|bin|%_buildrootdir|%_rpmdir|读取位于 %_buildrootdir 最终安装目录下的文件，以便最终在 %_rpmdir 目录下创建 RPM 包。在该目录下，不同架构的 RPM 包会分别保存至不同子目录， noarch 目录保存适用于所有架构的 RPM 包。这些 RPM 文件就是用户最终安装的 RPM 包。|
|src|%_sourcedir|%_srcrpmdir|创建源码 RPM 包（简称 SRPM，以.src.rpm 作为后缀名），并保存至 %_srcrpmdir 目录。SRPM 包通常用于审核和升级软件包。|

#### 宏路径
```
%{_sysconfdir}        /etc
%{_prefix}            /usr
%{_exec_prefix}       %{_prefix}
%{_bindir}            %{_exec_prefix}/bin
%{_libdir}            %{_exec_prefix}/%{_lib}
%{_libexecdir}        %{_exec_prefix}/libexec
%{_sbindir}           %{_exec_prefix}/sbin
%{_sharedstatedir}    /var/lib
%{_datarootdir}       %{_prefix}/share
%{_datadir}           %{_datarootdir}
%{_includedir}        %{_prefix}/include
%{_infodir}           /usr/share/info
%{_mandir}            /usr/share/man
%{_localstatedir}     /var
%{_initddir}          %{_sysconfdir}/rc.d/init.d
%{_var}               /var
%{_tmppath}           %{_var}/tmp
%{_usr}               /usr
%{_usrsrc}            %{_usr}/src
%{_lib}               lib (lib64 on 64bit multilib systems)
%{_docdir}            %{_datadir}/doc
%{buildroot}          %{_buildrootdir}/%{name}-%{version}-%{release}.%{_arch}
$RPM_BUILD_ROOT       %{buildroot}
```


### %prep
该阶段通常执行将源代码包解压到一个临时目录（通常为 BUILD）以及打补丁（如果有的话）等操作。解压时最常见到的一句话是：
```
%setup -q
```
该指令成功执行的前提是位于 SOURCES 目录下的源码包必须是 `name-version.tar.gz | name-version.tar.bz2` 的格式，并且它还会完成后续阶段目录的切换和设置。如果在该阶段没有执行该指令，则后面的每个阶段都需要手动去改变相应的目录。

### %build
该阶段通常执行 configure 和 make 操作，如果有些软件需要最先执行 bootstrap 之类的，可以放在 configure 之前来做。该阶段最常见的两条指令：
```
%configure
make %{?_smp_mflags}
```
它将自动将软件安装时的路径自动设置成如下约定：
- 可执行程序 `/usr/bin`
- 依赖的动态库 `/usr/lib` 或者 `/usr/lib64` 视操作系统版本而定
- 二次开发的头文件 `/usr/include`
- 文档及手册 `/usr/share/man`

`%configure` 是个宏常量，会自动将 prefix 设置成 `/usr`。另外，这个宏还可以接受额外的参数，如果某些软件有某些高级特性需要开启，可以通过给 `%configure` 宏传参数来开启。如果不用 `%configure` 这个宏的话，就需要完全手动指定 configure 时的配置参数了。同样地，我们也可以给 `make` 传递额外的参数，例如：
```
make %{?_smp_mflags} CFLAGS="" …
```

### %install
这个阶段就是执行 `make install` 操作。这个阶段会在 `%_buildrootdir` 目录里建好目录结构，然后将需要打包到 `rpm` 软件包里的文件从 `%_builddir`里拷贝到 `%_buildrootdir` 里对应的目录里。这个阶段最常见的两条指令是
```
rm -rf %{buildroot}
make DESTDIR=%{buildroot} install
```
如果软件有配置文件或者额外的启动脚本之类，就要手动用 `copy` 命令或者 `install` 命令将其也拷贝到 `%{buildroot}` 相应的目录里。用 `copy` 命令时如果目录不存在要手动建立，不然也会报错，所以推荐用 `install` 命令。 

### %check
### %clean
编译完成后一些清理工作，主要包括对 `%{buildroot}` 目录的清空(当然这不是必须的)，通常执行诸如 `make clean` 之类的命令。 


### %post
### %pre
### %preun
### %postun

### %files
指定安装时需要安装的文件列表，可以指定文件、目录，也可以使用通配符，如：
```
%file
/usr/local/sbin/tairserver # 一个可指定文件
/usr/local/sbin/taircfgsvr
/usr/local/etc/*.conf # 使用通配符
/usr/local/lib # 使用目录
```

### %changelog