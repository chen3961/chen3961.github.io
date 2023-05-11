---
layout:     post
title:      Ansible任务中一些无法自动化执行的场景
date:       2018-08-02
summary:    Ansible脚本经常遇到的中无法自动执行的操作及解决办法
categories: Ansbile
---

使用Ansible的playbook可以方便的在同时多个环境中执行配置操作。Ansible无需再目标环境中配置客户端，插件丰富，使用起来比较简单便捷。

但是在运行Ansible playbook执行过程中，有一些任务却无法自动执行，导致自动执行的脚本因为等待用户输入或者超时失败，使得某些场景下“发射后不管”的体验不是很好。这里总结一些不能自动执行的操作机解决方法，以减少异常执行终端的干扰。

### 添加SSH Key Fingerprint

在ansible通过ssh首次访问目标主机时，及时配置了登陆私钥或者密码，仍然需要将目标主机的SSH Key Fingerprint添加到当前用户的.ssh/known_hosts文件中。如果这个文件中不存在这个Fingerprint的话，会弹出提示要求用户确认，在ansible的执行过程中也会弹出提示。

```shell
    The authenticity of host '10.0.1.6 (10.0.1.6)' can't be established.
    ECDSA key fingerprint is SHA256:1ZBHe9pc6lZTNZ+QCrTAyr8Y4MgYZX37NPLkZStY4H0.
    ECDSA key fingerprint is MD5:de:be:b3:b2:e7:63:44:f2:e7:28:53:2f:f7:0d:33:bd.
    Are you sure you want to continue connecting (yes/no)?
```

这种要求用户输入的场景直接破坏了自动执行。

目前没有发现可以自动导入的SSH key的命令，但是可以通过修改ansbile配置文件/etc/ansible/ansible.cfg，忽略host key的检查，这样就忽略这一步。使得程序在第一次链接也可以自动执行。
```shell
    #/etc/ansible/ansible.cfg
    # uncomment this to disable SSH key host checking
    host_key_checking = False
```

### 访问新的repo

如果playbook中存在yum操作，如果yum源中存在新的或者动态repo源，会通过gpg校验repo源的repodata.xml文件。在手动执行yum命令的过程中，会提示用户确认导入pgp文件，但是在ansible yum任务中无法配置自动导入文件，会导致访问repo源失败从而导致任务失败。

在ansible playbook中如何自动访问一个新的repo源的呢？

首先可以尝试通过rpm_key的方式在执行yum任务中先从目标路径导入pgp文件，但是当前ansible中rpm_key任务暂不支持使用代理，所以当需要从外网导入时可能会出现失败的场景。

另外一种方案是在执行yum任务前使用makecache先更新repo的源信息。

```yml
   - name: update repor cache
     command: yum -q makecache -y
```
添加这个命令在yum install 任务前就可以自动导入repo源的库，从而客服yum任务失败的场景。
