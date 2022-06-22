---
title: "Install the OSM CLI"
description: "This section describes installing and using the `osm` CLI."
type: docs
weight: 1
---

## Prerequisites

- Kubernetes cluster running Kubernetes {{< param min_k8s_version >}} or greater

## Set up the OSM CLI

### From the Binary Releases

Download platform specific compressed package from the [Releases page](https://github.com/openservicemesh/osm/releases).
Unpack the `osm` binary and add it to `$PATH` to get started.

#### Linux and macOS

In a bash-based shell on Linux/macOS or [Windows Subsystem for Linux](https://docs.microsoft.com/windows/wsl/about), use `curl` to download the OSM release and then extract with `tar` as follows:

```console
# Specify the OSM version that will be leveraged throughout these instructions
OSM_VERSION={{< param osm_version >}}

# Linux curl command only
curl -sL "https://github.com/openservicemesh/osm/releases/download/$OSM_VERSION/osm-$OSM_VERSION-linux-amd64.tar.gz" | tar -vxzf -

# macOS curl command only
curl -sL "https://github.com/openservicemesh/osm/releases/download/$OSM_VERSION/osm-$OSM_VERSION-darwin-amd64.tar.gz" | tar -vxzf -
```

The `osm` client binary runs on your client machine and allows you to manage OSM in your Kubernetes cluster. Use the following commands to install the OSM `osm` client binary in a bash-based shell on Linux or [Windows Subsystem for Linux](https://docs.microsoft.com/windows/wsl/about). These commands copy the `osm` client binary to the standard user program location in your `PATH`.

```console
sudo mv ./linux-amd64/osm /usr/local/bin/osm
```

For **macOS** use the following commands:

```console
sudo mv ./darwin-amd64/osm /usr/local/bin/osm
```

You can verify the `osm` client library has been correctly added to your path and its version number with the following command.

```console
osm version
```

#### Windows

In a PowerShell-based shell on Windows, use `Invoke-WebRequest` to download the OSM release and then extract with `Expand-Archive` as follows:

```console
# Specify the OSM version that will be leveraged throughout these instructions
$OSM_VERSION="{{< param osm_version >}}"

[Net.ServicePointManager]::SecurityProtocol = "tls12"
$ProgressPreference = 'SilentlyContinue'; Invoke-WebRequest -URI "https://github.com/openservicemesh/osm/releases/download/$OSM_VERSION/osm-$OSM_VERSION-windows-amd64.zip" -OutFile "osm-$OSM_VERSION.zip"
Expand-Archive -Path "osm-$OSM_VERSION.zip" -DestinationPath .
```

The `osm` client binary runs on your client machine and allows you to manage the OSM controller in your Kubernetes cluster. Use the following commands to install the OSM `osm` client binary in a PowerShell-based shell on Windows. These commands copy the `osm` client binary to an OSM folder and then make it available both immediately (in current shell) and permanently (across shell restarts) via your `PATH`. You don't need elevated (Admin) privileges to run these commands and you don't need to restart your shell.

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

You can verify the `osm` client library has been correctly added to your path and its version number with the following command.

```console
osm version
```

### From Source (Linux, MacOS)

Building OSM from source requires more steps but is the best way to test the latest changes and useful in a development environment.

You must have a working [Go](https://golang.org/doc/install) environment.

```console
git clone git@github.com:openservicemesh/osm.git
cd osm
make build-osm
```

`make build-osm` will fetch any required dependencies, compile `osm` and place it in `bin/osm`. Add `bin/osm` to `$PATH` so you can easily use `osm`.

## Install OSM

### OSM Configuration

By default, the control plane components are installed into a Kubernetes Namespace called `osm-system` and the control plane is given a unique identifier attribute `mesh-name` defaulted to `osm`.
During installation, the Namespace and mesh-name can be configured through flags when using the `osm` CLI or by editing the values file when using the `helm` CLI.

The `mesh-name` is a unique identifier assigned to an osm-controller instance during install to identify and manage a mesh instance.

The `mesh-name` should follow [RFC 1123](https://tools.ietf.org/html/rfc1123) DNS Label constraints. The `mesh-name` must:

- contain at most 63 characters
- contain only lowercase alphanumeric characters or '-'
- start with an alphanumeric character
- end with an alphanumeric character

### Using the OSM CLI

Use the `osm` CLI to install the OSM control plane on to a Kubernetes cluster.

Run `osm install`.

```console
# Install osm control plane components
osm install
OSM installed successfully in namespace [osm-system] with mesh name [osm]
```

Run `osm install --help` for more options.
