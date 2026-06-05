---
categories: Linux
tags: sysctl 配置内核参数
---

## 简单介绍

sysctl命令用于运行时配置内核参数，这些参数位于/proc/sys目录下。sysctl配置与显示在/proc/sys目录中的内核参数

这里贴一个简介，网上的各个文档都很清楚了~

```bash
NAME
       sysctl - configure kernel parameters at runtime

SYNOPSIS
       sysctl [options] [variable[=value]] [...]
       sysctl -p [file or regexp] [...]
DESCRIPTION
       sysctl  is  used  to modify kernel parameters at runtime.  The parameters available are
       those listed under /proc/sys/.  Procfs is required for sysctl support  in  Linux.   You
       can use sysctl to both read and write sysctl data.
sysctl用于在运行时修改内核参数。可用的参数有那些列在/proc/sys/下的。Procfs是Linux中支持sysctl所必需的。你
可以使用sysctl读取和写入sysctl数据。
--system
              Load settings from all system configuration files.
              /run/sysctl.d/*.conf
              /etc/sysctl.d/*.conf
              /usr/local/lib/sysctl.d/*.conf
              /usr/lib/sysctl.d/*.conf
              /lib/sysctl.d/*.conf
              /etc/sysctl.conf
命令格式：

    sysctl [-n] [-e] -w variable=value

    sysctl [-n] [-e] -p <filename> (default /etc/sysctl.conf)

    sysctl [-n] [-e] -a

    常用参数的意义：

    -w   临时改变某个指定参数的值，如

        sysctl -w net.ipv4.ip_forward=1

    -a   显示所有的系统参数

    -p   从指定的文件加载系统参数，如不指定即从/etc/sysctl.conf中加载
```


## 注意事项（坑点）


### 1. `sysctl -p` 部分失败时，配置仍可能已生效

执行：

```bash
sysctl -p
```

时，如果配置文件中某一项加载失败：

* 命令会返回失败
* 但并不是“全部回滚”
* 实际上其它配置项仍然会继续尝试加载

例如：

```conf
net.ipv4.ip_local_reserved_ports = 50053
kernel.not_exist = 1
```

执行后：

* `kernel.not_exist` 会报错
* 但 `net.ipv4.ip_local_reserved_ports = 50053` 已经成功生效

因此：

> `sysctl -p` 返回失败，并不代表所有配置都失败。

---

### 2. 删除配置项后再`sysctl -p`不会自动恢复默认值

例如：

原配置：

```conf
net.ipv4.ip_local_reserved_ports = 50053
```

执行：

```bash
sysctl -p
```

后参数已经生效。

随后你删除这一行，再次执行：

```bash
sysctl -p
```

结果：

* `net.ipv4.ip_local_reserved_ports` 不会自动恢复默认值
* 当前运行中的值仍然保持为 `50053`

原因是：

> `sysctl -p` 只会“加载存在的配置项”，不会主动重置已经生效但配置文件中被删除的操作。

如果需要恢复默认值，需要：

* 手动设置回默认值
* 或重启系统