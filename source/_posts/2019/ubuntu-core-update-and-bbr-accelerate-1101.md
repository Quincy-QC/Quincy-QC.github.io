---
title: "Ubuntu内核升级与开启BBR加速"
catalog: true
toc_nav_num: true
date: 2019-11-01 16:01:20
subtitle: "BBR Accelerate"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- Tech

---

# Introduction

BBR（新的TCP拥塞控制算法Bottleneck Bandwidth and RTT）是个好东西。

众所周知，Ubuntu开启BBR的前提是内核必须高于或等于4.9，所以想要使用它需要先看看我们的内核是否在4.9或以上。

查看命令：`uname -a`

如果内核版本已经在4.9或以上，那么就可以跳过内核升级这一步骤。如何在4.9以下，那么就需要更新一下内核了。（很遗憾，GCE官方默认搭载的镜像，内核是4.4版本的，所以默认情况下都是需要升级的）

# Ubuntu内核升级

升级过程比较简单，先确定系统是32位还是64位，可用`getconf LONG_BIT`查看。

确认系统之后，需要下载必要的升级程序包，可以在[kernel.ubuntu.com](http://kernel.ubuntu.com/~kernel-ppa/mainline/)查找最新的程序包，根据自己的需要使用`wget`命令来下载服务器。

比如我的服务器是64位，安装4.10.2的内核：

```
sudo wget http://kernel.ubuntu.com/~kernel-ppa/mainline/v4.10.2/linux-image-4.10.2-041002-generic_4.10.2-041002.201703120131_amd64.deb
（拥有root权限的话可以去掉命令前面的“sudo”）
```

然后切换到下载目录，执行命令升级：

```
sudo dpkg -i linux-image-4.10.2-041002-generic_4.10.2-041002.201703120131_amd64.deb
```

最后执行命令更新grub引导装入程序：

```
sudo update-grub
```

一旦各方面完成，就可以重启服务器。系统重启后，可以执行`uname -a`查看内核版本，确保更新成功。

# 开启TCP BBR

修改系统变量：

```
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
```

如果执行以上命令显示拒绝访问，可以尝试以下命令：

```
sudo bash -c 'echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf'
sudo bash -c 'echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf'
```

保存生效：

```
sysctl -p
```

执行：

```
sysctl net.ipv4.tcp_available_congestion_control
```

如果返回结果，那么就表示RRB开启成功了：

```
net.ipv4.tcp_available_congestion_control = bbr cubic reno
```

也可以执行`lsmod | grep bbr`来检测BBR是否真的开启成功。

至此，我们的服务器BBR就开启成功了，赶紧试一试吧。