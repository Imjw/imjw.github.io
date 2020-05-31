---
layout:     post
title:      "GitLab 搭建"
subtitle:   "搭建属于你的 Git 私服"
date:       2019-06-17 13:12:52
author:     "Jw"
header-img: "img/post/gitlab-logo.png"
catalog:    true
description: "GitLab 搭建"
tags:
    - 环境搭建
---

*本文示例采用的 GitLab 安装方式为 Omnibus package installation，类型为 GitLab-CE*

## 环境
- CentOS 7
- GitLab 11.7.3

## 安装

> 采用 Omnibus package installation 方式进行安装

- 安装配置必要依赖

```shell
sudo yum install -y curl policycoreutils-python openssh-server
sudo systemctl enable sshd
sudo systemctl start sshd
sudo firewall-cmd --permanent --add-service=http
sudo systemctl reload firewalld
```
- （非必需）安装 Postfix 发送通知电子邮件，若采用其他邮件发送方式，可在安装 GitLab 后自行配置

```shell
sudo yum install postfix
sudo systemctl enable postfix
sudo systemctl start postfix
```
- 添加 GitLab 软件包仓库并进行安装
> 具体版本名称可在 [GitLab Community Edition repository](https://packages.gitlab.com/gitlab/gitlab-ce) 进行查询

```shell
# 添加仓库
curl -s https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
# 进行安装
sudo yum install -y gitlab-ce-11.7.3-ce.0.el7.x86_64
```

## 配置 GitLab
配置文件路径：`/etc/gitlab/gitlab.rb`
- 配置外部访问 URL

```
external_url 'http://xx.xx.xx.xx'
```

> <font color="red">external_url 只能配置 ip 或者域名，不能带有端口，否则不能启动。</font>

- 配置邮箱，以网易163邮箱为例配置邮箱:

```
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.163.com"
gitlab_rails['smtp_port'] = 25
gitlab_rails['smtp_user_name'] = "xxxx@163.com"
gitlab_rails['smtp_password'] = "xxxxpassword"
gitlab_rails['smtp_domain'] = "163.com"
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = false
gitlab_rails['smtp_openssl_verify_mode'] = "peer"

gitlab_rails['gitlab_email_from'] = "xxxx@163.com"
user["git_user_email"] = "xxxx@163.com"
```

> 注意以上的 xxxx@163.com 代表用户名，即邮箱地址，而 xxxxpassword 不是邮箱的登陆密码而是网易邮箱的客户端授权密码, 在网易邮箱web页面的设置 -POP3/SMTP/IMAP- 客户端授权密码查看。


## 更新
> 将 GitLab 更新到指定版本，本示例采用手动下载更新包方式进行 GitLab 版本更新。([more](https://docs.gitlab.com/omnibus/update/README.html))  

<font color="red">注意：</font>  
*1. 下载更新包时需注意版本型号，`gitlab-ce-xxx.el7` 对应CentOS 7 版本，`gitlab-ce-xxx.el6` 对应CentOS 6 版本。*  
*2. 跨主版本更新需要逐级更新，不能直接升级，需要先升级到下个主版本的首个次版本。如 `8.13.4` -> `11.3.4`, 升级的过程： `8.13.4` -> `8.17.7` -> `9.5.10` -> `10.8.7` -> `11.3.4` 。具体跨版本升级请参考 [upgrade recommendations](https://docs.gitlab.com/ee/policy/maintenance.html#upgrade-recommendations)。*
*3. 无论是通过官网仓库更新还是手动下载安装包更新，在更新GitLab新版本之前都会自动备份GitLab数据库，可以通过在以下位置创建一个空文件来跳过此自动备份`/etc/gitlab/skip-auto-backup`:*

```shell
sudo touch /etc/gitlab/skip-auto-backup
```

前往 [GitLab Community Edition repository](https://packages.gitlab.com/gitlab/gitlab-ce) ，搜索所需版本安装包，下载 rpm 安装包，执行安装包进行更新。

```shell
# 本文示例为 11.7.3 版本
wget --content-disposition https://packages.gitlab.com/gitlab/gitlab-ce/packages/ol/7/gitlab-ce-11.7.3-ce.0.el7.x86_64.rpm/download.rpm
# 停止 GitLab 部分服务
gitlab-ctl stop unicorn
gitlab-ctl stop sidekiq
gitlab-ctl stop nginx
# 安装更新
rpm -Uvh gitlab-ce-XXX1.rpm
# 如果是跨版本更新，需要连续更新多个版本，继续执行上一步
rpm -Uvh gitlab-ce-XXX2.rpm
# 重启 GitLab 服务
gitlab-ctl restart
# 查看当前已安装版本
rpm -qa gitlab-ce
```

## 备份
> 对已有代码仓库进行备份。（[more](https://docs.gitlab.com/ee/raketasks/backup_restore.html#creating-a-backup-of-the-gitlab-system)）

执行以下命令进行备份：
```shell
sudo gitlab-rake gitlab:backup:create
```
随后生成备份文件目录，默认为：`/var/opt/gitlab/backups`,文件名字格式为 `EPOCH_YYYY_MM_DD_GitLab_version`，如： `1560739396_2019_06_17_11.7.3_gitlab_backup.tar`。

## 恢复备份
> 将已备份的仓库文件恢复到新的 GitLab 上。（[more](https://docs.gitlab.com/ee/raketasks/backup_restore.html#restore)）

**前提**
- 目标 GitLab 和生成备份文件的 GitLab 具有<font color="red">完全相同的版本和类型（CE / EE）</font>
- 目标 GitLab 至少跑过一次 `sudo gitlab-ctl reconfigure`
- GitLab 正在运行，如果没有，执行 `sudo gitlab-ctl start`

**步骤**
- 将备份 tar 文件放在 `gitlab.rb` 配置中描述的备份目录中 `gitlab_rails['backup_path']`，默认为 `/var/opt/gitlab/backups`，并将其权限改为 git 用户拥有：

```shell
sudo cp 1560739396_2019_06_17_11.7.3_gitlab_backup.tar /var/opt/gitlab/backups/
sudo chown git.git /var/opt/gitlab/backups/1560739396_2019_06_17_11.7.3_gitlab_backup.tar
```

- 停止连接数据库的进程，保持其他 GitLab 进程运行：

```shell
sudo gitlab-ctl stop unicorn
sudo gitlab-ctl stop sidekiq
# Verify
sudo gitlab-ctl status
```

- 还原备份，指定还原文件

```shell
# This command will overwrite the contents of your GitLab database!
sudo gitlab-rake gitlab:backup:restore BACKUP=1560739396_2019_06_17_11.7.3
```

- 重启 GitLab

```shell
sudo gitlab-ctl restart
```

<font color="red">Tip : 以上备份还原，不包含配置文件（存储的 GitLab 配置信息，数据库加密密钥，双重身份认证信息），以下文件需单独备份，然后复制到新的 GitLab 中：</font>

- `/etc/gitlab/gitlab-secrets.json`
- `/etc/gitlab/gitlab.rb`

## 参考
[GitLab 官方文档](https://docs.gitlab.com/)



