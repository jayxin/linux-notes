# MySQL

## MySQL 忘记密码

### MySQL 8

进入 `skip-grant-tables` 模式:

```bash
su - root
mysqld_safe --skip-grant-tables &

mysql -u root

# in mysqld_safe
UPDATE mysql.user SET authentication_string=null WHERE User='root';
FLUSH PRIVILEGES;
exit;

mysql -u root
ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'yourpasswd';
```

若不在 `skip-grant-tables` 模式且可以进入 `mysql`:

```bash
ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'yourpasswd';
```
