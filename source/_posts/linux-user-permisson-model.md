---
title: linux_user_permisson_model
date: 2025-11-08 17:23:25
tags:
---
Linux 不是只给“人类用户”设计的，它也是系统和守护进程的运行环境。为了**安全隔离、权限管理、进程控制**，会创建大量**系统用户**（service accounts）。

这些用户不是给人登录用的，而是**专门用来运行服务**。

示例：

-   `root`：拥有全部权限
    
-   `yxz`：你这个真实用户
    
-   `docker`：用于控制 `docker.sock`、管理 docker daemon 权限
    
-   `mysql`：专门运行 MySQL 服务
    
-   `www-data`：用于 nginx / apache 等 Web 服务
    
-   `systemd-network` / `sshd` / `messagebus` …
    

这些用户通常设置为：

-   无密码
    
-   没有 home 目录
    
-   无法登录 shell
    

只负责**运行特定服务**。

---

## 为什么要这么做？

### 1) 最小权限原则（Principle of Least Privilege）

每个服务只拥有它**运行所需的最小权限**，而不是 root。

例子：  
如果 MySQL 进程直接用 root 身份跑，数据库漏洞 = 系统沦陷。  
但用 `mysql` 用户运行，则数据库权限被隔离出系统权限。

避免：

-   被黑客利用提权
    
-   服务故障破坏整个系统文件
    
-   不同服务之间互相干扰
    

---

### 2) 安全隔离

每个服务只能操作自己的文件目录。

### 3) 权限模型

Linux 权限模型 = 用户 (user) + 组 (group) + 文件权限 (rwx)

服务单独成为用户与用户组，对应文件权限：

```kotlin
-rw-------  mysql mysql   /var/lib/mysql/innodb_data
srw-rw----  root docker   /var/run/docker.sock
drwxr-xr-x  www-data www-data /var/www
```

---

### 4) 守护进程的传统

Unix 设计哲学：

> Everything is a process

每个服务（数据库、Web 服务器、容器引擎）都是一个进程 → 就需要独立账户。

---
## 核心概念

Linux 使用**用户/用户组/其他人**三层权限模型（Discretionary Access Control, DAC）：

| 主体（谁） | 描述 |
| --- | --- |
| owner (u) | 文件的拥有者（单一用户） |
| group (g) | 文件所属用户组（多人共享权限） |
| others (o) | 其他所有用户（系统剩下的人） |

操作权限（能做什么）：

| 符号 | 含义 | 文件 | 目录 |
| --- | --- | --- | --- |
| `r` | read | 读取文件内容 | 列出目录内容 |
| `w` | write | 修改文件内容 | 在目录内创建/删除/改名 |
| `x` | execute | 执行二进制文件或脚本 | 进入目录、访问目录内文件 |

额外特殊位：

| 符号 | 名称 | 作用 |
| --- | --- | --- |
| `s` | setuid/setgid | 以文件 owner 或 group 身份运行程序 |
| `t` | sticky bit | 目录中只有文件 owner 可删除（如 `/tmp`） |

---

### 权限字段格式

示例：

```diff
-rwxr-x---
```

结构说明：

| 位置 | 代表 | 示例 |
| --- | --- | --- |
| 1 | 文件类型 (`- d l s`) | `-` 普通文件 |
| 2–4 | owner 权限 | `rwx` |
| 5–7 | group 权限 | `r-x` |
| 8–10 | others 权限 | `---` |

文件类型常见标识：

| 字符 | 类型 |
| --- | --- |
| `-` | 普通文件 |
| `d` | 目录 |
| `l` | 链接 |
| `s` | socket |
| `p` | 管道 |
| `c` | 字符设备 |
| `b` | 块设备 |

---
### 常用命令

| 命令 | 含义 |
| --- | --- |
| `ls -l` | 查看权限 |
| `chmod` | 修改权限 |
| `chown` | 修改 owner |
| `chgrp` | 修改 group |
| `umask` | 默认权限掩码 |
| `setfacl / getfacl` | ACL 细粒度权限控制 |

---
