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

