<!-- vim-markdown-toc GFM -->

* [Cloudflare](#cloudflare)
    * [使用 Cloudflare 进行内网穿透](#使用-cloudflare-进行内网穿透)
        * [HTTP](#http)
            * [使用随机地址](#使用随机地址)
            * [使用自定义域名](#使用自定义域名)
        * [SSH](#ssh)

<!-- vim-markdown-toc -->

# Cloudflare

## 使用 Cloudflare 进行内网穿透

- [官网](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
- [cloudflared](https://github.com/cloudflare/cloudflared/releases)

### HTTP

在局域网启动 HTTP 服务器:

```sh
python3 -m http.server 8080 --bind 127.0.0.1
```

#### 使用随机地址

```sh
cloudflared tunnel --url 127.0.0.1:8080
```

找到命令行输出的以 `trycloudflare.com` 结尾链接,
访问即可.

#### 使用自定义域名

确保有自己的域名且已经添加到 Cloudflare 进行托管.

```sh
# 登录 cloudflare, 需要在浏览器中进行授权
# 登录后会在 ~/.cloudflared 目录下生成授权文件
cloudflared tunnel login

# 创建隧道, <tunnel-name> 任意
# 会在 ~/.cloudflared 目录下生成隧道的配置文件
#cloudflared tunnel create <tunnel-name>
cloudflared tunnel create http

# 更新 DNS 记录
# 将隧道解析到自己的域名上, <subdomain> 是想要创建的子域名,
# cloudflared 会添加 CNAME 记录
#cloudflared tunnel route dns <tunnel-name> <subdomain>
cloudflared tunnel route dns http http.example.org

# 运行隧道
#cloudflared tunnel --url localhost:8080 run <tunnel-name>
cloudflared tunnel --url localhost:8080 run http

# 访问 https://http.example.org 即可访问到内网的服务器
```

### SSH

确保有自己的域名且已经添加到 Cloudflare 进行托管.

在 SSH 服务端:

```sh
cloudflared tunnel login

cloudflared tunnel create ssh

cloudflared tunnel route dns ssh ssh.example.org

cloudflared tunnel --url ssh://localhost:22 run ssh
```

在 SSH 客户端:

`~/.ssh/config`:

```conf
Host nat-server
  HostName ssh.example.org
  ProxyCommand cloudflared access ssh --hostname %h
  User root
  IdentityFile ~/.ssh/id_rsa
```

登录到内网的 SSH 服务器:

```sh
ssh nat-server
```
