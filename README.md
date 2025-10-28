K3s 二进制文件集成了运行生产级、符合 CNCF 标准的 Kubernetes 集群所需的全部组件，包括 containerd、runc、kubelet 等。在本文中，我们将介绍 containerd 如何与 OCI runtime 通信，并探讨如何在 K3s 中添加另一个容器运行时 —— **Sysbox**，以及它在运行系统级 Pod 时的应用场景。

## Containerd 与 Runc 的关系

首先，让我们简要了解一下 containerd 是如何与 runc 协作的。**containerd** 是一个常驻的守护进程，主要负责以下任务：

* **镜像管理**：从镜像仓库拉取并存储镜像。
* **容器管理**：管理容器生命周期（创建、启动、停止、删除）。
* **快照管理**：通过 snapshotter 管理容器文件系统层。
* **运行时管理**：将容器创建的任务委托给兼容 OCI 的运行时（如 runc）。

当你在 Kubernetes 中创建 Pod 时，**kubelet** 会通过 containerd 实现的 **CRI 插件** 向 containerd 发送请求，要求创建 Pod sandbox（`RunPodSandbox`）并创建容器（`CreateContainer`）。containerd 随后会启动一个 **shim 进程**，该进程在 containerd 与 OCI runtime（如 runc）之间充当中间层。

shim 的存在有一个关键作用：即便 containerd 守护进程崩溃或重启，容器依然可以继续运行。

在启动过程中，shim 会生成 OCI runtime 所需的 bundle（包括 `config.json` 和 rootfs 路径），然后调用 **runc** 二进制文件。runc 读取 `config.json`，配置容器的 namespace 和 cgroup，最后启动容器进程。

**runc** 是直接与 Linux 内核交互的组件——它负责配置 cgroup、namespace、seccomp、权限和挂载。当容器创建完成后，runc 就会退出，shim 则继续负责容器生命周期与 I/O 的管理。

![](https://raw.githubusercontent.com/kingsd041/picture/main/202510281103222.png)

## Sysbox Runtime 简介

**[Sysbox](https://github.com "Sysbox")** 是由 Nestybox 创建的开源新一代容器运行时。与传统的 runc 不同，Sysbox 专为运行 “系统容器（system containers）” 而设计。它利用 **Linux 用户命名空间（user namespaces）** 以及其他内核特性，为容器提供更接近轻量虚拟机的行为。

这意味着，你可以在一个 Pod 中运行 **Docker、Systemd、containerd，甚至 K3s 本身**——而无需使用特权模式（privileged mode）。

简而言之，Sysbox 填补了应用容器与虚拟机之间的空白。它的典型应用场景包括：

* 在 Kubernetes 中运行 Kubernetes（K8s-in-K8s）。
* 构建需要完整操作系统环境的 CI/CD 流水线。
* 提供具备虚拟机隔离级别但速度接近容器的开发环境。

> 目前，Sysbox 官方仅支持 **CRI-O**。
> CRI-O 原生支持 Linux 用户命名空间，这是 Sysbox 运行的基础。
> 虽然 containerd 自 v2.0 起也加入了 user namespace 支持，但此前 sysbox-runc 存在一个 bug，导致与 containerd 不兼容。

## Sysbox-runc 与 Containerd 的集成

经过排查这个问题之后，我们发现问题的根源在于 containerd 无法执行 sysbox-runc 的一个子命令 `features`，导致出现以下错误：

```
level=debug msg="failed to introspect features of runtime \"sysbox-runc\"" error="failed to unmarshal Features (*anypb.Any): type with url : not found"
```

由于无法正确检测到 Sysbox 的功能特性，containerd 认为 sysbox-runc 不支持 user namespace，进而导致容器创建失败。该问题现已在 **sysbox-runc** 仓库中修复，使得 containerd 能够正确地与 sysbox-runc 协作运行。

## 在 K3s 中运行 Sysbox-runc

若要在 K3s 中使用 sysbox-runc，你需要首先拥有一个已运行的 K3s 集群。然后安装最新版本的 Sysbox。但由于 containerd 支持修复尚未合并到 Sysbox 主仓库（仅在 sysbox-runc 中），因此需要从源码构建最新版。

### 1. 安装 Docker

在构建 Sysbox 之前，请确保系统中已安装 Docker。

### 2. 克隆仓库并准备代码

```
git clone --recursive https://github.com/nestybox/sysbox.git

cd sysbox/sysbox-runc
git pull origin main
cd ..

make IMAGE_BASE_DISTRO=ubuntu IMAGE_BASE_RELEASE=jammy sysbox-static
```

构建完成后，你可以将生成的二进制文件复制到 `/usr/bin`，或者直接在同一台运行 containerd 的机器上执行：

```
make install
```

### 3. 启动 Sysbox

```
sysbox
```

### 4. 创建 Sysbox RuntimeClass

```
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: sysbox-runc
handler: sysbox-runc
```

### 5. 在 containerd 中添加 Sysbox 运行时配置

创建文件 `/var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl`：

```
[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.sysbox-runc]
  runtime_type = "io.containerd.runc.v2"

[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.sysbox-runc.options]
  SystemdCgroup = false
  BinaryName="/usr/bin/sysbox-runc"
```

### 6. 使用 Sysbox 运行 Pod

```
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu
spec:
  runtimeClassName: sysbox-runc
  hostUsers: false
  containers:
    - name: ubuntu2204
      image: ubuntu:22.04
      command: ["sleep", "40000000000"]
  restartPolicy: Never
```

## 总结

**Sysbox** 为 Kubernetes 带来了新的能力：它允许在容器中运行系统级工作负载，具备强隔离性，同时不依赖特权模式。结合 **K3s** 使用后，它可以实现以下创新场景：

* 在 Kubernetes 中运行 Kubernetes（如 **[K3k 虚拟集群](https://github.com "K3k"):[FlowerCloud机场订阅官网](https://hanlianfangzhi.com)**）。
* 为开发者提供安全、轻量的类似于虚拟机的沙箱。
* 在 Pod 中运行系统守护进程或嵌套的容器引擎。

虽然目前 Sysbox 官方仅支持 **CRI-O**，但随着 **sysbox-runc** 的修复，它已经可以与 **containerd** 正常配合使用，从而实现与 K3s 的集成。

这也标志着容器生态的发展方向正在从传统的“应用容器”，逐步迈向更加灵活、具备系统级能力的“系统容器”。

如果你正在使用 K3s，并希望在 Pod 中探索系统级工作负载，Sysbox 无疑是一个值得尝试的方案，它让你在保持原生 Kubernetes 工作流的同时，获得接近虚拟机的隔离体验。
