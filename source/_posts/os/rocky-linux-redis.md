---
title: Rocky Linux 10.x 安装 Redis (Remi 源) 指南
categories: 操作系统
keywords: 'linux,systemd'
description: Linux Init Service 和 Systemd
cover: >-
  https://graph.linganmin.cn/230202/45c4be216e488ead47b3373c4a04187f?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/230202/45c4be216e488ead47b3373c4a04187f?x-oss-process=image/format,webp/quality,q_60
abbrlink: 49a0b438
---

# Rocky Linux 10.x 安装 Redis (Remi 源) 指南

本指南详细说明了如何通过 Remi 仓库在 Rocky Linux 10 上安装稳定版或最新版的 Redis。

## 1. 准备工作：安装基础仓库
在安装 Redis 之前，必须先安装 EPEL (Extra Packages for Enterprise Linux) 和 Remi 的发布包，因为 Remi 依赖于 EPEL 提供的基础库。

```bash
# 安装 EPEL 源
sudo dnf install -y epel-release

# 安装 Remi 10 源
sudo dnf install -y https://rpms.remirepo.net/enterprise/remi-release-10.rpm
```

## 2. 核心步骤：处理模块化过滤
Rocky 10 默认可能会过滤掉第三方源中的同名软件包。我们需要先清理缓存，并明确启用 Remi 提供的 Redis 模块流。

### 2.1 清理并刷新缓存
```bash
sudo dnf clean all
sudo dnf makecache
```

### 2.2 启用 Redis 模块
查看可用的 Redis 版本：
```bash
sudo dnf module list redis --enablerepo=remi
```

选择你需要的版本（例如最新的 **7.2** 或 **7.4**），执行以下命令启用（此处以 `remi-7.2` 为例）：
```bash
sudo dnf module enable redis:remi-7.2 -y
```

## 3. 执行安装
启用模块后，系统将能够识别并下载 Remi 源中的 Redis 包。

```bash
sudo dnf install -y redis
```

## 4. 服务管理
安装完成后，需要手动启动服务并设置开机自启。

* **启动 Redis 服务：**
    ```bash
    sudo systemctl start redis
    ```
* **设置开机自启：**
    ```bash
    sudo systemctl enable redis
    ```
* **查看服务状态：**
    ```bash
    sudo systemctl status redis
    ```

## 5. 验证安装
使用 Redis 自带的客户端工具检查版本信息：

```bash
redis-cli --version
# 或者进入客户端输入 ping
redis-cli
> ping
# 正常应返回 PONG
```

---

## 常见问题与提示 (Tips)

### 远程访问配置
默认情况下，Redis 仅监听 `127.0.0.1`（本地回环）。如需远程连接，请编辑配置文件：
1.  打开文件：`sudo vi /etc/redis/redis.conf`
2.  修改 `bind 127.0.0.1` 为 `bind 0.0.0.0`（注意安全风险，建议配合防火墙）。
3.  修改 `protected-mode yes` 为 `protected-mode no`。
4.  **重启服务：** `sudo systemctl restart redis`

### 防火墙规则
如果开启了远程访问，记得开放 6379 端口：
```bash
sudo firewall-cmd --permanent --add-port=6379/tcp
sudo firewall-cmd --reload
```

### 为什么不能直接 `dnf install redis`？
在 Rocky 10 中，如果官方库（AppStream）里没有该软件，或者版本较低，DNF 会因为保护机制忽略第三方源的包。通过 `dnf module enable` 可以强制告诉系统：“我信任并要使用这个源提供的特定版本”。