---
layout:     post
title:      如何修改镜像root密码
date:       2018-07-11
summary:    如何更改Openstack镜像的root用户密码
categories: OpenStack Image
---

从公共镜像库下载的虚拟机镜像，通常都使用证书注入的方式进行安全验证，例如[Centos](https://cloud.centos.org/centos/7/images/)提供的镜像均不提供固定的用户名与密码认证登陆.

使用这些镜像实例化的虚拟机只能通过固定的用户与证书远程SSH访问实例。如果在实例化出现问题，特别是网络配置未设置成功时，往往需要通过后端VNC登陆。此时通常都需要使用root用户进行问题定位，但由于不清楚root用户密码无法登陆。

其实通过简单的virt工具可以简单的修改镜像root的密码：
'''
    virt-customize -a CentOS-7-x86_64-GenericCloud.qcow2 --root-password password:{new_password}
'''
这样使用新的镜像就有root密码了。

本地Linux能够使用virt-customize命令：需要安装libvirt，libguestfs-tools包，同时需要libvirtd服务启动。
'''
    yum install -y libvirt libguestfs-tools
    systemctl start libvirtd
'''
