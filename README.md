# 本分支用于记录本人给自己的Wyse 5070瘦客户机安装配置fedora 43 Server
## 机器配置
主机：Wyse 5070 Thin Client <br>
CPU：Intel(R) Celeron(R) J4105 (4) @ 2.50 GHz <br>
GPU：Intel UHD Graphics 600 @ 0.75 GHz <br>
RAM：SK Hynix 4G<br>

## 目的
给Wyse 5070瘦客户机安装fedora43 Server，配置Knock敲门，用于节点服务，流量转发，反向代理。

## 开始
### 下载fedora43 Server
fedora官方下载地址（会重定向到镜像站）：https://download.fedoraproject.org/pub/fedora/linux/releases/43/Server/x86_64/iso/Fedora-Server-dvd-x86_64-43-1.6.iso <br>
fedora官方下载地址（BitTorrent）：https://fedoraproject.org/torrents/43/ <br>
### 刷写fedora43 Server
本人使用Ventoy，直接把镜像放进去刷好Ventoy的移动硬盘里就行<br>
Ventoy项目地址：https://github.com/ventoy/Ventoy<br>
选择移动硬盘启动后，跟着操作指引安装系统就行，fedora的安装已经足够宝宝巴士了。<br>
### 系统配置
#### 更换镜像源
使用一键脚本更换软件源，选择中科大镜像并进行软件源更新
```
bash <(curl -sSL https://linuxmirrors.cn/main.sh)
```
紧接着用一键脚本更换docker源
```
bash <(curl -sSL https://linuxmirrors.cn/docker.sh)
```
#### 移除Web Console里的root用户限制
先安装编辑器，我两个都装但是我用nano
```
sudo dnf install nano vim
```
接着打开/etc/cockpit/disallowed-users
```
sudo nano /etc/cockpit/disallowed-users
```
用#号将root注释掉，ctrl+x退出并保存即可
### 软件安装与配置
#### 安装Easytier
拉取easytier二进制程序
```
cd /opt/easytier
wget https://github.com/EasyTier/EasyTier/releases/latest/easytier-linux-x86_64-v2.6.4.zip
```
部署web服务端
```
./easytier-web-embed \
    --api-server-port 11211 \
    --api-host "http://127.0.0.1:11211" \
    --config-server-port 22020 \
    --config-server-protocol udp
```


#### 安装Dpanel来管理docker容器
使用官方的一键脚本
```
curl -sSL https://dpanel.cc/quick.sh | bash
```
运行后跟随指引安装就行。
#### 安装openlist用于管理网盘映射
```
mkdir -p /etc/openlist
docker run --user $(id -u):$(id -g) -d --restart=unless-stopped -v /etc/openlist:/opt/openlist/data -p 5244:5244 -e UMASK=022 --name="openlist" openlistteam/openlist:latest
```
反正只用来映射网盘，直接照抄官方配置<br>
部署完之后直接用命令修改默认密码
```
docker exec -it openlist ./openlist admin set 123456
```
然后网页登录openlist进去修改密码
#### 配置网盘
115网盘使用115开放平台接入：
https://doc.oplist.org.cn/guide/drivers/115_open <br>
天翼云盘：https://doc.oplist.org.cn/guide/drivers/189 <br>
#### 安装Knock敲门
使用官方最小Docker Compose最小部署教程
```
mkdir -p /opt/fn-knock-docker
cd /opt/fn-knock-docker
```
准备.env文件
```
sudo nano .env
```
```
FN_KNOCK_IMAGE=kcilnk/fn-knock:latest
TZ=Asia/Shanghai
ADMIN_VIEW_PORT=7991
BACKEND_PORT=7998
AUTH_PORT=7997
GO_BACKEND_PORT=7996
GO_REPROXY_PORT=7999
FN_KNOCK_DOCKER_IPV4_SUBNET=172.30.0.0/16
FN_KNOCK_DOCKER_IPV6_SUBNET=fd42:fb33:7f7a:100::/64
DOCKER_ADMIN_TRUSTED_PROXY_CIDRS=
DOCKER_DISCOVER_LAN_IP=
```
准备compose文件
```
nano docker-compose.yml
```
```
services:
  fn-knock:
    image: ${FN_KNOCK_IMAGE}
    restart: unless-stopped
    environment:
      TZ: ${TZ:-Asia/Shanghai}
      FN_KNOCK_RUNTIME_TARGET: docker
      REDIS_HOST: redis
      REDIS_PORT: 6379
      FN_KNOCK_DATA_DIR: /var/lib/fn-knock
      FN_KNOCK_GATEWAY_CONFIG_DIR: /usr/local/etc/fn-knock
      ADMIN_VIEW_PORT: ${ADMIN_VIEW_PORT:-7991}
      BACKEND_PORT: ${BACKEND_PORT:-7998}
      AUTH_PORT: ${AUTH_PORT:-7997}
      GO_BACKEND_PORT: ${GO_BACKEND_PORT:-7996}
      GO_REPROXY_PORT: ${GO_REPROXY_PORT:-7999}
      DOCKER_ADMIN_TRUSTED_PROXY_CIDRS: ${DOCKER_ADMIN_TRUSTED_PROXY_CIDRS:-}
      DOCKER_DISCOVER_LAN_IP: ${DOCKER_DISCOVER_LAN_IP:-}
      DDNS_HOST_IF_INET6_PATH: /host/proc/net/if_inet6
      ADMIN_VIEW_HOST: 0.0.0.0
      BACKEND_HOST: 127.0.0.1
    ports:
      - "${ADMIN_VIEW_PORT:-7991}:${ADMIN_VIEW_PORT:-7991}"
      - "${GO_REPROXY_PORT:-7999}:${GO_REPROXY_PORT:-7999}"
    networks:
      - fn_knock_net
    volumes:
      - fn_knock_data:/var/lib/fn-knock
      - fn_knock_gateway:/usr/local/etc/fn-knock
      - /proc/1/net:/host/proc/net:ro
    depends_on:
      redis:
        condition: service_healthy
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -fsS http://127.0.0.1:${ADMIN_VIEW_PORT:-7991}/api/admin/healthz || exit 1",
        ]
      interval: 10s
      timeout: 5s
      retries: 12
      start_period: 20s

  redis:
    image: redis:7-bookworm
    restart: unless-stopped
    environment:
      TZ: ${TZ:-Asia/Shanghai}
    command: ["redis-server", "--appendonly", "yes"]
    networks:
      - fn_knock_net
    volumes:
      - fn_knock_redis:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 20

volumes:
  fn_knock_data:
  fn_knock_gateway:
  fn_knock_redis:

networks:
  fn_knock_net:
    enable_ipv6: true
    ipam:
      config:
        - subnet: ${FN_KNOCK_DOCKER_IPV4_SUBNET:-172.30.0.0/16}
        - subnet: ${FN_KNOCK_DOCKER_IPV6_SUBNET:-fd42:fb33:7f7a:100::/64}
```


启动
```
docker compose pull
docker compose up -d
docker compose ps
docker compose logs -f fn-knock
```






## 本篇涉及的项目：<br>
https://github.com/donknap/dpanel <br>
https://github.com/OpenListTeam/OpenList <br>
https://github.com/kci-lnk/fn-knock-turborepo <br>



