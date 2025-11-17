# Install GitLab

GitLab 是 Ruby 开发的自托管 Git 仓库的项目.
分为社区版(CE, Community Edition) 和企业版(EE, Enterprise Edition).

- 官网(https://gitlab.com/)
- 版本: v14.10.0

## Ubuntu

<details>
<summary>Prepare</summary>

```bash
apt update
apt install curl openssh-server ca-certificates tzdata perl -y
# postfix 用于发送通知邮件
apt install -y postfix
```

</details>

<details>
<summary>Install</summary>

```bash
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.deb.sh | sudo bash
# 安装企业版 gitlab
apt install gitlab-ee
# 查看可用的版本
sudo apt-cache madison gitlab-ee

# 修改配置
vim /etc/gitlab/gitlab.rb
# 需要修改如 `external_url` 等字段

# 重新编译配置文件
gitlab-ctl reconfigure

# 重启
gitlab-restart

# 查看默认 root 密码
cat /etc/gitlab/initial_root_password
```

</details>

## CentOS 7

<details>
<summary>Prepare</summary>

```bash
yum install curl policycoreutils-python openssh-server -y
systemctl start sshd
systemctl enable sshd

# 配置 postfix
yum install postfix
systemctl enable postfix
vim /etc/postfix/main.cf
# 修改 `inet_interfaces = localhost` 为 `inet_interfaces = all`
systemctl start postfix
```

</details>

<details>
<summary>Install</summary>

```bash
# Community Edition
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
#yum --showduplicates list gitlab-ce
EXTERNAL_URL="http://127.0.0.1:8099" yum install -y gitlab-ce

# 使用浏览器访问 web ui, 账号为 root, 密码如下命令查看
cat /etc/gitlab/initial_root_password
```

</details>
