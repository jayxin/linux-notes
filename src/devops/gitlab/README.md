# GitLab

## 常用命令

<details>
<summary>GitLab 常用命令</summary>

```bash
# 启动所有 gitlab 组件
gitlab-ctl start

# 停止所有 gitlab 组件
gitlab-ctl stop

# 重启所有 gitlab 组件
gitlab-ctl restart

# 查看服务状态
gitlab-ctl status

# 修改配置
vim /etc/gitlab/gitlab.rb

# 重新编译 gitlab 的配置
gitlab-ctl reconfigure

# 查看配置
gitlab-ctl show-config

# 检查 gitlab
gitlab-rake gitlab:check SANITIZE=true --trace

# 查看日志
# 日志地址 /var/log/gitlab
# 服务地址 /var/opt/gitlab
# Nginx 配置路径 /var/opt/gitlab/nginx/conf/nginx.conf
gitlab-ctl tail
gitlab-ctl tail nginx/gitlab_access.log

# 查看 gitlab 版本
cat /opt/gitlab/embedded/service/gitlab-rails/VERSION
```

</details>

## 重置账号密码

[官方文档](https://docs.gitlab.com/ee/security/reset_user_password.html)

### 使用 Rake 任务

```bash
gitlab-rake "gitlab:password:reset"
```

### 使用 Rails 控制台

```bash
# 进入交互模式
gitlab-rails console -e production

# 定位到 gitlab 数据库中 Users 表中的一个用户
user = User.where(id:1).first
# or by user name
user = User.find_by_username 'root'
# or by email
user = User.find_by(email: 'user@example.com')
# or all
User.all

# 密码不小于 8 个字符
user.password=12345678
# 确认
user.password_confirmation=12345678

# 保存
user.save!

exit
```

## 备份和恢复

两台服务器上的 gitlab 必须版本一致，才可以进行恢复。

### 备份

```bash
# 生成备份文件
/usr/bin/gitlab-rake gitlab:backup:create
```

生成的备份文件在 `/var/opt/gitlab/backups` 目录。

可通过修改配置文件 `/etc/gitlab/gitlab.rb` 中的 `gitlab_rails['backup_path']`
字段调整备份目录。

### 恢复

```bash
# 停止 unicorn 和 sidekiq，保证数据库没有新的连接，不会有写数据情况
gitlab-ctl stop unicorn
gitlab-ctl stop sidekiq

# 进入备份目录进行恢复，1751390965_2023_04_01_14.10.0 为备份文件的时间以及 gitlab 版本号
cd /var/opt/gitlab/backups
gitlab-rake gitlab:backup:restore BACKUP=1751390965_2023_04_01_14.10.0

# 启动 unicorn 和 sidekiq
gitlab-ctl start unicorn
gitlab-ctl start sidekiq
```
