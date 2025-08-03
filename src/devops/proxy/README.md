<!-- vim-markdown-toc GFM -->

* [Proxy](#proxy)
    * [Use proxy](#use-proxy)
        * [Environment variable](#environment-variable)
        * [proxychains-ng](#proxychains-ng)
            * [Install](#install)
            * [Config](#config)
            * [Usage](#usage)
        * [curl](#curl)
        * [git](#git)
        * [ssh](#ssh)
    * [Setup proxy](#setup-proxy)
        * [Setup clash using docker](#setup-clash-using-docker)
        * [Setup clash using docker-compose](#setup-clash-using-docker-compose)

<!-- vim-markdown-toc -->

# Proxy

## Use proxy

假设本地已配置好代理:

- `7890` 端口对应 http 代理协议
- `7891` 端口对应 socks5 代理协议

### Environment variable

```sh
http_addr='http://127.0.0.1:7890'
socks5_addr='socks5://127.0.0.1:7891'

export HTTP_PROXY=$http_addr
export HTTPS_PROXY=$http_addr
export ALL_PROXY=$http_addr
# or
export HTTP_PROXY=$http_addr
export HTTPS_PROXY=$socks5_addr
export ALL_PROXY=$socks5_addr
```

```sh
function set_proxy() {
  local proxy_addr=${1:-"socks5://127.0.0.1:7891"}

  HTTP_PROXY="$proxy_addr"
  HTTPS_PROXY="$proxy_addr"
  ALL_PROXY="$proxy_addr"

  export HTTP_PROXY HTTPS_PROXY ALL_PROXY
}

function unset_proxy() {
  unset HTTP_PROXY HTTPS_PROXY ALL_PROXY
}
```

### proxychains-ng

这种方式只能代理基于 TCP 协议的请求, 因此也就无法代理 ping 命令(ICMP).

#### Install

```sh
sudo apt update
sudo apt install proxychains-ng
```

#### Config

```sh
vim /etc/proxychains/proxychains.conf
```

```conf
dynamic_chain
#strict_chain

[ProxyList]
socks5  127.0.0.1 7891
# or
#http 127.0.0.1 7890
```

#### Usage

```sh
proxychains4 firefox

proxychains4 wget "https://www.google.com"
```

### curl

```sh
proxy_server=socks5://127.0.0.1:7891
curl -x $proxy_server -Iv https://www.google.com
unset proxy_server
```

### git

```sh
function set_git_proxy() {
    local proxy_addr=${1:-"socks5://127.0.0.1:7891"}

    git config --local http.proxy "${proxy_addr}"
    git config --local https.proxy "${proxy_addr}"
}

function unset_git_proxy() {
    git config --local --unset http.proxy
    git config --local --unset https.proxy
}
```

### ssh

```sh
ssh -o ProxyCommand='nc -X 5 -x 127.0.0.1:7891 %h %p' root@google.com
```

## Setup proxy

### Setup clash using docker

```sh
# Prepare configuration
mkdir -p ~/clash
cp ./config.yaml ~/clash/ # 自行提供 config.yaml

# Running Clash
docker run --name clash \
  -p 5090:9090 -p 5890:7890 -p 5891:7891 \
  -v ~/clash/config.yaml:/root/.config/clash/config.yaml -d \
  dreamacro/clash

# Running Web-UI
docker run --name clash-ui -p 5080:80 -d haishanh/yacd

# visit http://<host_ip>:5080
# add http://<host_ip>:5090

# Use environment variables
#export HTTP_PROXY='http://127.0.0.1:5890'
#export HTTPS_PROXY='http://127.0.0.1:5890'
#export ALL_PROXY='socks5://127.0.0.1:5891'
```

### Setup clash using docker-compose

`docker-compose.yml`

```yaml
# docker-compose.yml
# version: '3.7'
services:
  clash-server:
    image: dreamacro/clash
    container_name: clash
    restart: unless-stopped
    ports:
      - "9090:9090"
      - "7890:7890"
      - "7891:7891"
    volumes:
      - ~/clash/config.yaml:/root/.config/clash/config.yaml
    environment:
      - TZ=Asia/Shanghai

  clash-ui:
    image: haishanh/yacd
    container_name: clash-ui
    restart: unless-stopped
    ports:
      - 7080:80
    environment:
      - TZ=Asia/Shanghai
```

Run docker-compose:

```sh
# Change to the directory where the docker-compose.yml is located.
docker-compose up -d
```
