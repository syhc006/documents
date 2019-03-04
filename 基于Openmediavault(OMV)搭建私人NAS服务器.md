#  基于Debian操作系统安装OMV
- 安装Debian9系统(使用最小镜像进行安装)-[地址](https://www.debian.org/CD/netinst/#netinst-stable)
- 添加Debian软件仓库-[参考](https://openmediavault.readthedocs.io/en/latest/installation/on_debian.html)

```
cat <<EOF >> /etc/apt/sources.list.d/openmediavault.list
deb http://packages.openmediavault.org/public arrakis main
# deb http://downloads.sourceforge.net/project/openmediavault/packages arrakis main
## Uncomment the following line to add software from the proposed repository.
# deb http://packages.openmediavault.org/public arrakis-proposed main
# deb http://downloads.sourceforge.net/project/openmediavault/packages arrakis-proposed main
## This software is not part of OpenMediaVault, but is offered by third-party
## developers as a service to OpenMediaVault users.
# deb http://packages.openmediavault.org/public arrakis partner
# deb http://downloads.sourceforge.net/project/openmediavault/packages arrakis partner
EOF
```
- 安装OMV4-[参考](https://openmediavault.readthedocs.io/en/latest/installation/on_debian.html)

```
export LANG=C
export DEBIAN_FRONTEND=noninteractive
export APT_LISTCHANGES_FRONTEND=none
apt-get update
apt-get --allow-unauthenticated install openmediavault-keyring
apt-get update
apt-get --yes --auto-remove --show-upgraded \
    --allow-downgrades --allow-change-held-packages \
    --no-install-recommends \
    --option Dpkg::Options::="--force-confdef" \
    --option DPkg::Options::="--force-confold" \
    install postfix openmediavault
# Initialize the system and database.
omv-initsystem
```
- 安装OMV4-plugin-[参考](http://omv-extras.org/joomla/index.php/guides)

```
wget http://omv-extras.org/openmediavault-omvextrasorg_latest_all4.deb
dpkg -i openmediavault-omvextrasorg_latest_all4.deb 
apt-get -f inall
apt-get -f install
apt update
```

# 基于OMV安装Virtual Box
- 安装linux-headers-4.9.0-8-amd64(uname -r查看4.9.0.8)

```
apt-get install linux-headers-4.9.0-8-amd64
```
- 在OMV插件中搜索Virtual Box并安装

# 基于OMV安装Docker

- 在OMV插件中搜索Docker并安装

# 基于FRP实现内网穿透-[参考](https://www.jianshu.com/p/e8e26bcc6fe6)

- 云主机安装FRP服务器端
云主机可以是阿里云、腾讯云和京东云等。主要需要一个公网IP

```
wget https://github.com/fatedier/frp/releases/download/v0.20.0/frp_0.20.0_linux_amd64.tar.gz
tar -zxvf frp_0.20.0_linux_amd64.tar.gz
rm -f frpc
rm -f frpc.ini
```

- 修改云主机FRP服务端配置frps.ini

```
[common]
bind_addr = 0.0.0.0
bind_port = 7000
vhost_http_port = 80
vhost_https_port = 443

dashboard_user = admin
dashboard_pwd = admin
dashboard_port = 7500
auth_token = 123
```
- 运行FRP服务端

```
./frps -c frps.ini
```


- 内网NAS安装FRP客户端

```
wget https://github.com/fatedier/frp/releases/download/v0.20.0/frp_0.20.0_linux_amd64.tar.gz
tar -zxvf frp_0.20.0_linux_amd64.tar.gz
rm -f frps
rm -f frps.ini
```

- 修改内网NAS主机FRP客户端配置frpc.ini

```
[common]
server_addr = 云主机公网IP
server_port = 7000
auth_token = 123

[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 6000

[nas-admin]
type = http
local_ip = 内网NAS主机IP
local_port = 80
custom_domains = 自定义域名(可以是花生壳的域名，需要将此域名关联到云主机的公网IP)

[smb]
type = tcp
local_ip = 内网NAS主机IP
local_port = 445
remote_port = 6001
```
- 运行FRP客户端

```
./frpc -c frpc.ini
```
# FRP服务端docker镜像制作-[参考](https://www.cnblogs.com/thyong/p/8509040.html)
- dockerfile

```
FROM ubuntu
MAINTAINER sunyu <sunyu-1987@qq.com>

ARG FRP_VERSION=0.20.0

RUN apt update \
    && apt install -y wget

WORKDIR /tmp
RUN set -x \
    && wget https://github.com/fatedier/frp/releases/download/v${FRP_VERSION}/frp_${FRP_VERSION}_linux_amd64.tar.gz \
    && tar -zxf frp_${FRP_VERSION}_linux_amd64.tar.gz \
    && mv frp_${FRP_VERSION}_linux_amd64 /var/frp \
    && mkdir -p /var/frp/conf \
    && apt remove -y wget \
    && apt autoremove -y \
    && rm -rf /var/lib/apt/lists/*

COPY conf/frps.ini /var/frp/conf/frps.ini

VOLUME /var/frp/conf

WORKDIR /var/frp
ENTRYPOINT ./frps -c ./conf/frps.ini
```
- 配置frps.ini
- 制作镜像
```
docker build -t frp-server .
```
- 启动FRP服务端

```
docker run -d --name frp-server --net=host frp-server
```
# FRP客户端docker镜像制作
- dockerfile

```
FROM ubuntu
MAINTAINER sunyu <sunyu-1987@qq.com>

ARG FRP_VERSION=0.20.0

RUN apt update \
    && apt install -y wget

WORKDIR /tmp
RUN set -x \
    && wget https://github.com/fatedier/frp/releases/download/v${FRP_VERSION}/frp_${FRP_VERSION}_linux_amd64.tar.gz \
    && tar -zxf frp_${FRP_VERSION}_linux_amd64.tar.gz \
    && mv frp_${FRP_VERSION}_linux_amd64 /var/frp \
    && mkdir -p /var/frp/conf \
    && apt remove -y wget \
    && apt autoremove -y \
    && rm -rf /var/lib/apt/lists/*

COPY conf/frpc.ini /var/frp/conf/frpc.ini

VOLUME /var/frp/conf

WORKDIR /var/frp
ENTRYPOINT ./frpc -c ./conf/frpc.ini
```
- 配置frpc.ini
- 制作镜像
```
docker build -t frp-client .
```
- 启动FRP服务端

```
docker run -d --name frp-client --net=host frp-client
```
