[Homebrew](http://brew.sh/) 是最初由 [Max Howell](https://mxcl.dev/) 用 Ruby 写的 OS X 软件管理系统，其代码开源在 [GitHub](https://github.com/Homebrew/brew/) 上，类似于Linux上[Redhat](http://en.wikipedia.org/wiki/Redhat)系的[yum](https://github.com/rpm-software-management/yum)、Debain系的[apt](https://github.com/Debian/apt)等包管理器。用了Homebrew再也不想手动安装软件了 ，在 Homebrew 的生态下，都只需要一条命令就可以安装、升级、卸载等操作，并且 Homebrew 会自动为你解决软件包的依赖问题。

但是[homebrew-core](https://github.com/Homebrew/homebrew-core/)中有很多包还没有，比如[iredis](https://github.com/laixintao/iredis)这个好用的客户端，所以想自己动手写一个homebrew安装包。

## 基本概念

在 Homebrew 的架构下，至少有 4 层概念

- Keg（酒桶）：安装好的脚本、软件等；
- Cellar（酒窖）：所有用 Homebrew 安装在本地的脚本、软件组成的集合；
- Formula（配方）：定义如何下载、编译和安装脚本或软件的 Ruby 脚本；
- Tap：一个包含若干 Formula 的 GitHub 专案。
- Cask（木桶）：MacOS GUI原生应用

我们平时使用 `brew install foobar` 安装软件时，实际上是从 `Homebrew/homebrew-core` 这个 Tap 中获取相应的 Formula，然后将 Keg 安装在 Cellar 中。`Homebrew/homebrew-core`安装时会直接从github克隆到本地，执行`brew update`就会更新仓库。但是，`Homebrew/homebrew-core` [不允许普通用户提交自己写的小众脚本、软件](https://docs.brew.sh/Acceptable-Formulae)。所以，我们需要建立一个新的 Tap，包含对应我们软件的 Formula，然后将 Keg 放入本地的 Cellar 中。

## 自己构建Formula

Homebrew 提供了一个[官方指南](https://docs.brew.sh/Formula-Cookbook)。不过，这个对于很多已经编译好的应用，这个指南显得比较复杂。所以，这篇文章介绍怎么对已编译好的应用创建 Homebrew

### 1. 创建 Formula

假设我们的软件已经有了一个发布版本，并且可供下载，比如 https://github.com/laixintao/iredis/releases/download/v1.9.1/iredis.tar.gz 。接下来，我们在终端里执行

```
brew create https://github.com/laixintao/iredis/releases/download/v1.9.1/iredis.tar.gz
```

这条命令会在 `/usr/local/Homebrew/Library/Taps/homebrew/homebrew-core/Formula/` 下创建一个 `.rb` 文件，其文件名取决于传给 `brew create` 的 URL。

> 比如，我们这里会创建 `/usr/local/Homebrew/Library/Taps/homebrew/homebrew-core/Formula/iredis.rb`。

```
# Documentation: https://docs.brew.sh/Formula-Cookbook
#                https://rubydoc.brew.sh/Formula
# PLEASE REMOVE ALL GENERATED COMMENTS BEFORE SUBMITTING YOUR PULL REQUEST!
class Iredis < Formula
  desc "Interactive Redis: A Terminal Client for Redis with AutoCompletion and Syntax Highlighting."
  homepage "https://iredis.io"
  url "https://github.com/laixintao/iredis/releases/download/v1.9.1/iredis.tar.gz"
  sha256 "ae551c7bb00acd6501463d7fd8d84eac90ca2b6cf85ad76471039660e648c09f"
  license "NOASSERTION"

  # depends_on "cmake" => :build

  def install
    # ENV.deparallelize  # if your formula fails when building in parallel
    # Remove unrecognized options if warned by configure
    system "./configure", "--disable-debug",
                          "--disable-dependency-tracking",
                          "--disable-silent-rules",
                          "--prefix=#{prefix}"
    # system "cmake", ".", *std_cmake_args
  end

  test do
    # `test do` will create, run in and delete a temporary directory.
    #
    # This test will fail and we won't accept that! For Homebrew/homebrew-core
    # this will need to be a test that verifies the functionality of the
    # software. Run the test with `brew test iredis`. Options passed
    # to `brew install` such as `--HEAD` also need to be provided to `brew test`.
    #
    # The installed folder is not in the path, so use the entire path to any
    # executables being tested: `system "#{bin}/program", "do", "something"`.
    system "#{bin}/iredis", "--help"
  end
end

```

头部的`desc`、`homepage`、`url`、`sha256`是这个包的几个描述字段

接下来是依赖项部分depends_on

由于 `iredis` 是已经编译好的文件，所以没有任何其他的依赖项，不需要在此填入任何内容。

接下来是安装部分

默认情况下，Homebrew 会并行地编译源代码来进行安装。不过，有些程序并不适用并行编译。此时可以将 `# ENV.deparallelize` 的注释解开。

后面的`./configure`和`cmake`是`*nix`标准的编译过程

由于我们要安装的 `iredis` 已经编译好，所以不需要考虑编译的过程。只需要调用 Homebrew 提供的 `bin.install` 就可以了。

最后的测试部分，检查我们的程序是否安装好，比如执行`iredis --help`命令检查，安装是否成功。

去掉不必要的注释，把软件版本tag抽出来，于是我们就得到了最后的 `iredis.rb`

```
HOMEBREW_IREDIS_VERSION = "v1.9.1".freeze
HOMEBREW_IREDIS_SHA = "ae551c7bb00acd6501463d7fd8d84eac90ca2b6cf85ad76471039660e648c09f".freeze

class Iredis < Formula
  desc "Interactive Redis: A Terminal Client for Redis with AutoCompletion and Syntax Highlighting."
  homepage "https://github.com/laixintao/iredis"
  sha256 HOMEBREW_IREDIS_SHA
  url "#{homepage}/releases/download/#{HOMEBREW_IREDIS_VERSION}/iredis.tar.gz"
  version HOMEBREW_IREDIS_VERSION

  def install
    bin.install "iredis"
  end

  test do
    system "#{bin}/iredis", "--help"
  end
end
```

保存后，你就可以执行 `brew search`和`brew info` 就能找到iredis了。

```
$ brew info iredis   
iredis: stable v1.9.1
Interactive Redis: A Terminal Client for Redis with AutoCompletion and Syntax Highlighting.
https://github.com/laixintao/iredis
Not installed
From: https://github.com/Homebrew/homebrew-core/blob/HEAD/Formula/iredis.rb
```

由于之前在 `brew create` 的过程中，已经下载了 `otfcc-mac64-0.2.3.tar.zx` 这一文件，所以 Homebrew 不会再次下载。执行完毕后，我们发现，`otfccdump` 和 `otfccbuild` 两个二进制程序已经被安装在了 `/usr/local/bin/` 目录下——这是 Homebrew 安装二进制的默认目录。

我们的 Formula 比较简单，不涉及 Homebrew 提供的更深的功能。如果有必要，你需要参考[官方指南](https://github.com/Homebrew/brew/blob/master/share/doc/homebrew/Formula-Cookbook.md)。

### 创建 Tap

至此我们已经创建了 Formula。根据这个 Formula，我们本地的 Homebrew 已经可以将 `otfcc` 安装到 `/usr/local/bin` 目录下了。然而，其他人暂时还不能获取、安装我们的软件。为此，我们需要创建自己的 Tap。

前面已经说过，Tap 实际上是 GitHub 网站上的一个专案。这个专案有些特殊，它的名字必须是 `homebrew-foobar` 的形式。也就是说，它必须有 `homebrew-` 这一前缀。`otfcc` 放在 GitHub 上的 `caryll` 组织中开发。于是我们可以在 `caryll` 中创建一个名为 `homebrew-tap` 的专案，然后将上述 `otfcc-mac64.rb` 纳入 `homebrew-tap` 这一专案的版本控制之下。

创建专案及纳入版本控制的过程是基本的 Git 操作，这里就不做展开了。创建好的专案在此：https://github.com/caryll/homebrew-tap，可供一观。

### 实际安装看看

现在，我们创建了自己的 Formula 和 Tap，并在 GitHub 上为 Tap 创建了一个公开的专案。那么，世界上所有的 Homebrew 用户就可以使用下列指令很方便地安装 `otfcc` 了。

```
brew tap caryll/tap && brew install otfcc-mac64
```

注意，这里我们首先调用了 `brew tap` 命令，将我们自建的 Tap 加入到本地 Homebrew 的搜索列表中。加入搜索列表的内容是 `caryll/tap`——相比 GitHub 上的专案名称，Homebrew 会自动在传入 `brew tap` 的地址里加上 `homebrew-` 这一前缀。而后，我们照常调用 `brew install` 命令，就能顺利地安装 `otfcc` 在 OS X 上了。