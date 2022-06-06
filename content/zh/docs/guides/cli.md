---
title: "安装 OSM CLI"
description: "这个章节描述了安装和使用 `osm` CLI."
type: docs
weight: 1
---

## 先决条件

- Kubernetes 集群，运行 Kubernetes {{< param min_k8s_version >}} 或者更高

## 搭建 OSM CLI

### 从二进制发布版开始

从[发布页](https://github.com/openservicemesh/osm/releases)下载平台指定的压缩包
解压 `osm` 二进制文件，然后添加它到 `$PATH` 来开启。

#### Linux 和 macOS

在 Linux/macOS 或者 [Windows Linux 子系统 (WSL)](https://docs.microsoft.com/windows/wsl/about) 上的基于 bash 的 shell 环境里，使用 `curl` 来下载 OSM 发布版，然后按照如下方式解压 `tar` 包。

```console
# Specify the OSM version that will be leveraged throughout these instructions
OSM_VERSION={{< param osm_version >}}

# Linux curl command only
curl -sL "https://github.com/openservicemesh/osm/releases/download/$OSM_VERSION/osm-$OSM_VERSION-linux-amd64.tar.gz" | tar -vxzf -

# macOS curl command only
curl -sL "https://github.com/openservicemesh/osm/releases/download/$OSM_VERSION/osm-$OSM_VERSION-darwin-amd64.tar.gz" | tar -vxzf -
```

`osm` 客户端二进制程序运行在您的客户端机器上，并且允许您在您的 Kubernetes 集群里管理 OSM。使用下面的命令在 Linux 或者 [Windows Linux 子系统 (WSL)](https://docs.microsoft.com/windows/wsl/about) 上基于 bash 的 shell 里面来安装 OSM `osm` 客户端二进制程序。这些命令复制 `osm` 客户端二进制程序到您 `PATH` 下面的标准用户程序位置里。


```console
sudo mv ./linux-amd64/osm /usr/local/bin/osm
```

对于 **macOS**，使用下面的命令：

```console
sudo mv ./darwin-amd64/osm /usr/local/bin/osm
```

您可以通过下面的命令来验证那些已经正确地添加到您环境的 `osm` 客户端库和它们的版本号。

```console
osm version
```

#### Windows

在 Windows 上基于 PowerShell 的 shell 里面，使用 `Invoke-WebRequest` 来下载 OSM 发布版，然后用 `Expand-Archive` 来解压，就像下面这样：

```console
# Specify the OSM version that will be leveraged throughout these instructions
$OSM_VERSION="{{< param osm_version >}}"

[Net.ServicePointManager]::SecurityProtocol = "tls12"
$ProgressPreference = 'SilentlyContinue'; Invoke-WebRequest -URI "https://github.com/openservicemesh/osm/releases/download/$OSM_VERSION/osm-$OSM_VERSION-windows-amd64.zip" -OutFile "osm-$OSM_VERSION.zip"
Expand-Archive -Path "osm-$OSM_VERSION.zip" -DestinationPath .
```

`osm` 客户端二进制程序运行在您的客户端机器上，并且允许您在您的 Kubernetes 集群里来管理 OSM 控制器。使用下面的命令来安装 OSM `osm` 客户端二进制程序，这个需要在 Windows 上的基于 PowerShell 的 shell 环境来完成。这些命令复制 `osm` 客户端二进制程序到一个 OSM 文件夹，然后通过您的 `PATH` 环境变量，使即刻（在当前 shell）和持久（跨 shell 重新启动）环境下 OSM 都有效。您不需要提升（管理员）权限来运行这些命令，你也不需要重启您的 shell。

```console
# Copy osm.exe to C:\OSM
New-Item -ItemType Directory -Force -Path "C:\OSM"
Move-Item -Path .\windows-amd64\osm.exe -Destination "C:\OSM\"

# Add C:\OSM to PATH.
# Make the new PATH permanently available for the current User
$USER_PATH = [environment]::GetEnvironmentVariable("PATH", "User") + ";C:\OSM\"
[environment]::SetEnvironmentVariable("PATH", $USER_PATH, "User")
# Make the new PATH immediately available in the current shell
$env:PATH += ";C:\OSM\"
```

您可以通过下面的命令，来验证已经被正确添加到您的环境里的 `osm` 客户端库和它们的版本号。

```console
osm version
```

### 从源码 (Linux, macOS)

从源码来构建 OSM 需要更多的步骤，但是这是最好的方式用来在一个开发环境里测试最近的变更和有用的东西。

您必须有一个工作的 [Go](https://golang.org/doc/install) 环境。

```console
$ git clone git@github.com:openservicemesh/osm.git
$ cd osm
$ make build-osm
```

`make build-osm` 将拉取任何被需要的依赖，编译 `osm`，然后把它放置于 `bin/osm`。添加 `bin/osm` 到 `$PATH`，这样您就可以轻松地使用 `osm` 了。

## 安装 OSM

### OSM 配置

默认的，控制平面组件被安装到一个 Kubernetes 命名空间，被称之为 `osm-system`，然后这个控制平面被给与一个唯一的标识符属性 `mesh-name`，默认是 `osm`。
安装期间，这个命名空间和 mesh-name 能够被配置，当使用 `osm` CLI 时，通过标志来进行，或者当使用 `helm` CLI 时，通过编辑值文件来进行。

在安装到标识并且管理一个网格实例期间，`mesh-name` 是一个唯一的标识符，其被指派给一个 OSM 控制器实例。

`mesh-name` 应该遵循 [RFC 1123](https://tools.ietf.org/html/rfc1123) DNS 标签限定。其必须：

- 至多包含 63 个字符
- 只能包含小写字母数字字符或者 '-'
- 以一个字母数字字符开始
- 以一个字母数字字符结束

### 使用 OSM CLI

使用 `osm` CLI 来安装 OSM 控制平面到一个 Kubernetes 集群。

运行 `osm install`。

```console
# Install osm control plane components
$ osm install
OSM installed successfully in namespace [osm-system] with mesh name [osm]
```

运行 `osm install --help` 来了解更多选项。
