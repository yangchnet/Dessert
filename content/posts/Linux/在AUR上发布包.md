---
author: "李昌"
title: "在AUR上发布包"
date: "2023-02-10"
tags: ["Linux", "arch"]
categories: ["Linux"]
ShowToc: true
TocOpen: true
---

# 如何在AUR上发布包

## 1. 准备`PKGBUILD` 文件

> https://wiki.archlinux.org/title/PKGBUILD
> https://wiki.archlinux.org/title/Creating_packages

想要打包一个可以在AUR上发布的package， 需要使用`makepkg`工具，`makepkg`在运行时会寻找当前工作目录下的`PKGBUILD`文件。

找到之后，会从`PKGBUILD`配置的`source`去获取源码并根据`PKGBUILD`的配置去进行编译，最终产生二进制文件以及一些元信息文件。

如果你使用的是arch系的系统，可以在`/usr/share/pacman/`目录下找到一些PKGBUILD示例文件。例如：
```bash
$ cat /usr/share/pacman/PKGBUILD.proto
# This is an example PKGBUILD file. Use this as a start to creating your own,
# and remove these comments. For more information, see 'man PKGBUILD'.
# NOTE: Please fill out the license field for your package! If it is unknown,
# then please put 'unknown'.

# Maintainer: Your Name <youremail@domain.com>
pkgname=NAME
pkgver=VERSION
pkgrel=1
epoch=
pkgdesc=""
arch=()
url=""
license=('GPL')
groups=()
depends=()
makedepends=()
checkdepends=()
optdepends=()
provides=()
conflicts=()
replaces=()
backup=()
options=()
install=
changelog=
source=("$pkgname-$pkgver.tar.gz"
        "$pkgname-$pkgver.patch")
noextract=()
md5sums=()
validpgpkeys=()

prepare() {
        cd "$pkgname-$pkgver"
        patch -p1 -i "$srcdir/$pkgname-$pkgver.patch"
}

build() {
        cd "$pkgname-$pkgver"
        ./configure --prefix=/usr
        make
}

check() {
        cd "$pkgname-$pkgver"
        make -k check
}

package() {
        cd "$pkgname-$pkgver"
        make DESTDIR="$pkgdir/" install
}

```

文件类型是proto，不过这跟grpc的protobuf应该关系不大。。

### 1.1 `PKGBUILD`中的常见变量

PKGBUILD中有很多字段，下面介绍一些常用的：
- `pkgname`: 包的名称
- `pkgver`: 包的版本
- `pkgrel`: 版本号，应该为一个正整数，将显示在最终打包的文件名中
- `pkgdesc`: 包的说明，建议小于80个字符
- `arch`: 支持的架构
- `url`: 软件官网地址
- `license`: 证书
- `depends`: 依赖的包
- `source`: 软件源地址，可以是文件路径、HTTP、FTP、Git地址
- `sha1sums`: source中包含的文件的sha1校验值,这对于git仓库来说并不是必须的

### 1.2 `PKGBUILD`中的内建变量

除此之外，`PKGBUILD`还有一些内建的变量, 通过`$variable`的方式使用：
- `srcdir`: 指向 makepkg 提取或符号链接源数组中所有文件的目录，可简单理解为makepkg运行的目录
- `pkgdir`: 指向 makepkg 打包安装包的目录，它成为构建包的根目录

### 1.3 `PKGBUILD`的内建函数

当使用`makepkg`构建包时，将调用`PKGBUILD`中存在的以下五个函数：

**注意: `package()`是必须的**

1. `prepare()`
这个函数在包被提取后， 在`pkgver()`和`build()`运行。调用这个函数，将运行用于准备生成源的命令。

2. `pkgver()`

`pkgver()`在包被拉取、解压以及`prepare()`运行之后运行。你可以在这一阶段调整包的版本

需要注意的是，在`pkgver()`中的输出、sed修改，都会被认为是在操作`$pkgver`变量

3. `build()`
此函数使用 Bash 语法中的常见 shell 命令自动编译软件并创建一个名为`pkg`的安装软件的目录。这允许 makepkg 打包文件，而无需筛选文件系统。

`build()`函数的第一步是切换到解压缩源码包时创建的目录。makepkg 会在执行`build()`之前将`$srcdir`设为当前目录。因此，在大多数情况下，如`/usr/share/pacman/PKGBUILD.proto`中建议的那样，第一个命令将如下所示：
```bash
cd "$pkgname-$pkgver"
```

然后，你可以在下面列出编译软件所需的命令，比如`make build`等

4. `check()`
调用`make check`以及其他任何检查逻辑的地方。

5. `package()`
最后一步是将编译的文件放在一个目录中，`makepkg`可以在其中检索这些文件并创建包。默认情况下，这是`pkg`文件夹：一个“fakeroot”环境，即一个假的根目录。这个`pkg`目录将复制软件安装目录的根目录层次结构。例如，如果你要安装一个文件到`/usr/bin`，则应将其放在`$pkgdir/usr/bin`中，

### 1.4 检查`PKGBUILD`的语法
使用`namcap`进行检查

```bash
$ namcap PKGBUILD
```

## 2. 实战进行打包

### 2.1 打包本地二进制文件

准备一个可执行文件`hello`, 该文件的作用是输出"hello aur"，存放在workspace目录中：
```bash
mkdir workspace && cd workspace
mv /path/to/hello .
```

计算出`hello`文件的md5:
```bash
$ md5sum hello
2acd771608bd1552ae27795a864d87ac hello
```

在当前目录下创建`PKGBUILD`文件，内容如下:
```
# Maintainer: Crest Lee <crest@selefra.io>
pkgname=hello
pkgver=0.1
pkgrel=1
pkgdesc='say hello to everyone'
arch=('any')
license=('GPL3')
source=("$srcdir/hello")
md5sums=('2acd771608bd1552ae27795a864d87ac')

package() {
        cd $srcdir
        mkdir -p $pkgdir/usr/bin
        install hello $pkgdir/usr/bin/$pkgname
}
```

然后执行：
```bash
$ makepkg

==> 正在创建软件包：hello 0.1-1 (2023年02月10日 星期五 18时27分27秒)
==> 正在检查运行时依赖关系...
==> 正在检查编译时依赖关系==> 获取源代码...
  -> 找到 hello
==> 正在验证 source 文件，使用md5sums...
    hello ... 通过==> 正在释放源码...
==> 正在删除现存的 $pkgdir/ 目录...
==> 正在进入 fakeroot 环境...
==> 正在开始 package()...
==> 正在清理安装...
  -> 正在删除 libtool 文件...
  -> 正在清除不打算要的文件...
  -> 正在移除静态库文件...
  -> 正在从二进制文件和库中清除不需要的系统符号...
  -> 正在压缩 man 及 info 文档...
==> 正在检查打包问题...
==> 正在构建软件包"hello"...
  -> 正在生成 .PKGINFO 文件...
  -> 正在生成 .BUILDINFO 文件...
  -> 正在生成 .MTREE 文件...
  -> 正在压缩软件包...
==> 正在离开 fakeroot 环境。==> 完成创建：hello 0.1-1 (2023年02月10日 星期五 18时27分28秒)
```

构建完成后，workspace的目录结构为：
```
.
├── hello
├── hello-0.1-1-any.pkg.tar.zst
├── pkg
│   └── hello
│       ├── .BUILDINFO
│       ├── .MTREE
│       ├── .PKGINFO
│       └── usr
│           └── bin
│               └── hello
├── PKGBUILD
└── src
    └── hello -> /path/to/hello
```

可以看到，生成了一些元信息文件。打包结果为：hello-0.1-1-any.pkg.tar.zst

安装它：

```bash
$ sudo pacman -U hello-0.1-1-any.pkg.tar.zst
```

测试：
```bash
$ hello
hello world

$ which hello
/usr/bin/hello
```

卸载这个包：
```bash
$ sudo pacman -R hello
```

完成


### 2.2 打包git仓库

准备一个git仓库：https://github.com/yangchnet/hello-aur

这是一个golang语言编写的简单程序，输出
```
hello, aur. Version: <version>
```

因此，在本例中，你可能需要安装golang。

新建一个工作目录：
```bash
mkdir workspace && cd workspace
```

创建一个`PKGBUILD`文件，内容如下：

```PKGBUILD
# Maintainer: yangchent 1048887414@qq.com
_pkgname=hello-aur
pkgname="$_pkgname"-git
pkgver=1
pkgrel=1
pkgdesc="say hello to aur in your computer"
arch=("any")
license=('GPL')
source=("git+https://github.com/yangchnet/$_pkgname.git")
md5sums=("SKIP")

pkgver() {
    cd "$_pkgname"
    printf "r%s.%s" "$(git rev-list --count HEAD)" "$(git rev-parse --short=8 HEAD)"
    sed -i 's/{{.version}}/'$pkgver'/' main.go
}

build() {
    cd "$_pkgname"
    go build -o hello-aur main.go
}

check() {
    cd "$_pkgname"
    if [ -f "hello-aur" ];then
        echo "检查通过"
    else
        exit 1
    fi
}

package() {
    cd "$_pkgname"
    mkdir -p $pkgdir/usr/bin
    mv hello-aur $pkgdir/usr/bin/hello-aur
}
```

这里跟构建本地二进制文件有点不同，例如
- pkgname必须以”-git“结尾

- source必须符合'project_name::git+https://project_url#branch=project_branch'的格式

- 对于md5或sha1的验证可以通过配置"SKIP"略过，因为git本身已经进行了验证

需要参考：https://wiki.archlinux.org/title/VCS_package_guidelines#top-page
https://bbs.archlinux.org/viewtopic.php?id=223966

构建：
```bash
$ makepkg
```

安装：
```bash
$ sudo pacman -U hello-aur-git-r1.9f29fc45-1-any.pkg.tar.zst
```

测试：
```bash
$ hello-aur
hello, aur. Version: 1
```
这里需要注意，看一下pkgver()函数，我们在最后将代码内部{{.Version}}替换为由git commit id组成的版本号，但并没有生效，仍然是初始的1，这可能表明：`$pkgver`是在`pkgver()`函数结束后才被赋值的。

卸载:
```bash
sudo pacman -R hello-aur-git # 注意pkgname多了"-git"后缀
```

## 3. 发布

TODO


## References

https://wiki.archlinux.org/title/Arch_User_Repository#top-page

https://wiki.archlinux.org/title/Makepkg

https://wiki.archlinux.org/title/PKGBUILD

https://wiki.archlinux.org/title/Creating_packages

https://wiki.archlinux.org/title/VCS_package_guidelines#top-page