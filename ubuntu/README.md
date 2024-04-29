
# Ubuntu Packer 模板用于 MAAS

## 引言

此目录中的 Packer 模板用于创建 Ubuntu 镜像，以便与 MAAS 一起使用。

## 先决条件（创建镜像）

* 一台运行 Ubuntu 18.04+ 并能够运行 KVM 虚拟机的机器。
* qemu-utils, libnbd-bin, nbdkit 和 fuse2fs
* qemu-system
* ovmf
* cloud-image-utils
* parted
* [Packer](https://www.packer.io/intro/getting-started/install.html)，版本 1.7.0 或更新

## 部署镜像的需求

* [MAAS](https://maas.io) 3.0+
* [Curtin](https://launchpad.net/curtin) 21.0+

## ubuntu-cloudimg.pkr.hcl

此模板从官方 Ubuntu 云镜像构建一个 tgz 镜像。这将生成一个与 <https://images.maas.io/> 上的镜像非常接近的镜像。

### 构建镜像

要构建镜像，您需要提供一个脚本来定制模板：

```shell
packer init .
packer build -var customize_script=my-changes.sh -var ubuntu_series=jammy \
    -only='cloudimg.*' .
```

`my-changes.sh` 是您编写的脚本，用于在虚拟机内部定制镜像。例如，您可以使用 `apt-get` 安装软件包，调用 ansible，或者做任何您想要的事情。

使用 make 构建：

```shell
make custom-cloudimg.tar.gz SERIES=jammy
```

#### 从脚本访问外部文件

如果您想将文件放入或使用于镜像中，可以将这些文件放在 `http` 目录中。

您放置在那里的任何文件，都可以通过以下方式在脚本中访问：

```shell
wget http://${PACKER_HTTP_IP}:${PACKER_HTTP_PORT}:/my-file
```

### 安装内核

通常，MAAS 使用的镜像不包括内核。当在 MAAS 中部署机器时，会为该机器选择合适的内核，并将其安装在选定的镜像之上。

如果您确实想要强制镜像始终使用特定的内核，可以将内核包含在镜像中。

最简单的方法是使用 `kernel` 参数：

```shell
packer init .
packer build -var kernel=linux-lowlatency -var customize_script=my-changes.sh \
    -only='cloudimg.*' .
```

您也可以在 `my-changes.sh` 脚本中手动安装内核，但在这种情况下，您还需要将内核包的名称写入 `/curtin/CUSTOM_KERNEL`。这是为了确保 MAAS 在部署时不会安装另一个内核。

### 构建不同架构的镜像

默认情况下，镜像是为 amd64 架构生产的。如果您指定 `architecture` 参数，也可以为 arm64 构建：

```shell
packer init .
packer build -var architecture=arm64 -var customize_script=my-changes.sh \
    -only='cloudimg.*' .
```

## ubuntu-flat.pkr.hcl 和 ubuntu-lvm.pkr.hcl

这些模板使用 Ubuntu 服务器镜像将镜像安装到虚拟机中。它比使用云镜像需要更长的时间，但对某些用例可能很有用。

### 自定义镜像

您可以在 Ubuntu 安装期间或之后自定义镜像，然后再打包最终镜像。前者通过提供 [自动安装配置](https://ubuntu.com/server/docs/install/autoinstall)，编辑 _user-data-flat_ 和 _user-data-lvm_ 文件来完成。后者是通过 _install-custom-packages_ 脚本执行的。

### 使用代理构建镜像

Packer 模板从互联网下载 Ubuntu 网络安装程序。要告诉 Packer 使用代理，将 HTTP_PROXY 环境变量设置为您的代理服务器。或者，您可以重新定义 iso_url 为本地文件，将 iso_checksum_type 设置为 none 以禁用校验和，以及删除 iso_checksum_url。

### 构建镜像

您可以使用 Makefile 轻松构建镜像：

```shell
make custom-ubuntu-lvm.dd.gz
```

以构建带有 LVM 的原始镜像，或者，您可以构建一个 TGZ 镜像

```shell
make custom-ubuntu.tar.gz
```

您也可以手动运行 packer。您当前的工作目录必须在 packer-maas/ubuntu 中，这是此文件所在的位置。一旦进入 packer-maas/ubuntu，您可以使用以下命令生成镜像：

```shell
packer init .
PACKER_LOG=1 packer build -only=qemu.lvm .
```

或者

```shell
packer init .
PACKER_LOG=1 packer build -only=qemu.flat .
```

注意：ubuntu-lvm.pkr.hcl 和 ubuntu-flat.pkr.hcl 配置为以无头模式运行 Packer。将只看到 Packer 输出。如果您希望看到安装输出，请连接到 Packer 输出中给出的 VNC 端口，或者在 HCL2 文件中将 headless 的值更改为 false。

安装是非交互式的。请注意，安装将尝试连接到 QEMU 虚拟机上的 SSH，新构建的镜像正在那里启动。这是过程中的最后配置步骤。Packer 使用 SSH 来发现镜像实际上已经启动，因此可能会有多次失败尝试——超过 3-5 分钟——直到连接成功。这是 packer 的正常行为。

### Makefile 参数

#### PACKER_LOG

启用（1）或禁用（0）详细的 packer 日志。默认值设置为 0。

#### SERIES

指定要构建的 Ubuntu 系列。默认值设置为 Jammy。

#### ARCH

目标镜像架构。支持的值是 amd64（默认）和 arm64。

#### URL

托管给定系列 ISO 镜像的镜像的 URL 前缀。默认值设置为 http://releases.ubuntu.com。预期 ISO 镜像位于 URL/SERIES/ 下。

#### SUMS

校验和文件的文件名。默认值设置为 SHA256SUMS。

#### TIMEOUT

构建镜像时应用的超时时间。默认值设置为 1 小时。

### 默认用户名

默认用户名是 `ubuntu`

## 将镜像上传到 MAAS

TGZ 镜像

```shell
maas admin boot-resources create \
    name='custom/ubuntu-tgz' \
    title='Ubuntu Custom TGZ' \
    architecture='amd64/generic' \
    filetype='tgz' \
    content@=custom-cloudimg.tar.gz
```

LVM 原始镜像

```shell
maas admin boot-resources create \
    name='custom/ubuntu-raw' \
    title='Ubuntu Custom RAW' \
    architecture='amd64/generic' \
    filetype='ddgz' \
    content@=custom-ubuntu-lvm.dd.gz
``` 
```

# Ubuntu Packer Templates for MAAS

## Introduction

The Packer templates in this directory creates Ubuntu images for use with MAAS.

## Prerequisites (to create the image)

* A machine running Ubuntu 18.04+ with the ability to run KVM virtual machines.
* qemu-utils, libnbd-bin, nbdkit and fuse2fs
* qemu-system
* ovmf
* cloud-image-utils
* parted
* [Packer](https://www.packer.io/intro/getting-started/install.html), v1.7.0 or newer

## Requirements (to deploy the image)

* [MAAS](https://maas.io) 3.0+
* [Curtin](https://launchpad.net/curtin) 21.0+

## ubuntu-cloudimg.pkr.hcl

This template builds a tgz image from the official Ubuntu cloud images. This
results in an image that is very close to the ones that are on
<https://images.maas.io/>.

### Building the image

The build the image you give the template a script which has all the
customizations:

```shell
packer init .
packer build -var customize_script=my-changes.sh -var ubuntu_series=jammy \
    -only='cloudimg.*' .
```

`my-changes.sh` is a script you write which customizes the image from within
the VM. For example, you can install packages using `apt-get`, call out to
ansible, or whatever you want.

Building using make:

```shell
make custom-cloudimg.tar.gz SERIES=jammy
```

#### Accessing external files from you script

If you want to put or use some files in the image, you can put those in the `http` directory.

Whatever file you put there, you can access from within your script like this:

```shell
wget http://${PACKER_HTTP_IP}:${PACKER_HTTP_PORT}:/my-file
```

### Installing a kernel

Usually, images used by MAAS don't include a kernel. When a machine is deployed
in MAAS, the appropriate kernel is chosen for that machine and installed on top
of the chosen image.

If you do want to force an image to always use a specific kernel, you can
include it in the image.

The easiest way of doing this is to use the `kernel` parameter:

```shell
packer init .
packer build -var kernel=linux-lowlatency -var customize_script=my-changes.sh \
    -only='cloudimg.*' .
```

You can also install the kernel manually in your `my-changes.sh` script, but in
that case you also need to write the name of the kernel package to
`/curtin/CUSTOM_KERNEL`. This is to ensure that MAAS won't install another
kernel on deploy.

### Building different architectures

By default, images are produces for amd64. You can build for arm64 as well if
you specify the `architecture` parameter:

```shell
packer init .
packer build -var architecture=arm64 -var customize_script=my-changes.sh \
    -only='cloudimg.*' .
```

## ubuntu-flat.pkr.hcl and ubuntu-lvm.pkr.hcl

These templates use an Ubuntu server image to install the image to the VM. It
takes longer than using a cloud image, but can be useful for certain use cases.

### Customizing the Image

It is possible to customize the image either during the Ubuntu installation or afterwards, before packing the final image. The former is done by providing [autoinstall config](https://ubuntu.com/server/docs/install/autoinstall), editing the _user-data-flat_ and _user-data-lvm_ files. The latter is performed by the _install-custom-packages_ script.

### Building the image using a proxy

The Packer template downloads the Ubuntu net installer from the Internet. To tell Packer to use a proxy set the HTTP_PROXY environment variable to your proxy server. Alternatively you may redefine iso_url to a local file, set iso_checksum_type to none to disable checksuming, and remove iso_checksum_url.

### Building an image

You can easily build the image using the Makefile:

```shell
make custom-ubuntu-lvm.dd.gz
```

to build a raw image with LVM, alternatively, you can build a TGZ image

```shell
make custom-ubuntu.tar.gz
```

You can also manually run packer. Your current working directory must
be in packer-maas/ubuntu, where this file is located. Once in
packer-maas/ubuntu you can generate an image with:

```shell
packer init .
PACKER_LOG=1 packer build -only=qemu.lvm .
```

or

```shell
packer init .
PACKER_LOG=1 packer build -only=qemu.flat .
```

Note: ubuntu-lvm.pkr.hcl and ubuntu-flat.pkr.hcl are configured to run Packer in headless mode. Only Packer output will be seen. If you wish to see the installation output connect to the VNC port given in the Packer output or change the value of headless to false in the HCL2 file.

Installation is non-interactive.  Note that the installation will attempt an SSH connection to the QEMU VM where the newly-built image is being booted.  This is the final provisioning step in the process.  Packer uses SSH to discover that the image has, in fact, booted, so there may be a number of failed tries -- over 3-5 minutes -- until the connection is successful.  This is normal behavior for packer.

### Makefile Parameters

#### PACKER_LOG

Enable (1) or Disable (0) verbose packer logs. The default value is set to 0.

#### SERIES

Specify the Ubuntu Series to build. The default value is set to Jammy.

#### ARCH

Target image architecture. Supported values are amd64 (default) and arm64.

#### URL

The URL prefix for mirror that is hosting the ISO images for a given series. The default value is set to http://releases.ubuntu.com. ISO images are expected to be under URL/SERIES/.

#### SUMS

The file name for the checksums file. The default value is set to SHA256SUMS.

#### TIMEOUT

The timeout to apply when building the image. The default value is set to 1h.

### Default Username

The default username is ```ubuntu```

## Uploading images to MAAS

TGZ image

```shell
maas admin boot-resources create \
    name='custom/ubuntu-tgz' \
    title='Ubuntu Custom TGZ' \
    architecture='amd64/generic' \
    filetype='tgz' \
    content@=custom-cloudimg.tar.gz
```

LVM raw image

```shell
maas admin boot-resources create \
    name='custom/ubuntu-raw' \
    title='Ubuntu Custom RAW' \
    architecture='amd64/generic' \
    filetype='ddgz' \
    content@=custom-ubuntu-lvm.dd.gz
```
