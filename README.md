## 基于docker建设家庭影院
### 简介
本文主要描述使用docker部署一套由三个组件组成的家庭影院。
### 技术选型
- 文件下载（aria2）
- 文件管理（elfinder）
- DLNA服务器（MiniDLNA）
### 实现步骤
#### 安装并准备docker
本文以Ubuntu18.04-server作为基础说明。

```shell
# 以下脚本均在root用户下执行
# step 1: 安装必要的一些系统工具
apt-get update
apt-get -y install apt-transport-https ca-certificates curl software-properties-common
# step 2: 安装GPG证书
-fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# Step 3: 写入软件源信息
add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# Step 4: 更新并安装 Docker-CE
apt-get -y update
apt-get -y install docker-ce
```
	
#### 应用安装准备

```shell
## 创建用户
useradd -M -s /usr/sbin/nologin www-data
## 记录用户uid与gid，在compose中对应PUID与PGID
cat /etc/passwd |grep www-data |cut -d : -f 3,4
## 创建目录
APP_DATA=/mnt/data01
mkdir -p ${APP_DATA}/media/audio
mkdir -p ${APP_DATA}/media/video
mkdir -p ${APP_DATA}/media/image
mkdir -p ${APP_DATA}/media/download
mkdir -p ${APP_DATA}/aria2/conf
mkdir -p ${APP_DATA}/aria2/config
chown www-data:www-data -R ${APP_DATA}/media
chown www-data:www-data -R ${APP_DATA}/aria2
```

#### compose文件
```yaml
version: '3.7'

volumes:
  app:
    driver: local
  data:
    driver: local
    driver_opts:
      o: bind
      type: none
      device: /mnt/data01/media
  aria2_conf:
    driver: local
    driver_opts:
      o: bind
      type: none
      device: /mnt/data01/aria2/conf
  aria2_config:
    driver: local
    driver_opts:
      o: bind
      type: none
      device: /mnt/data01/aria2/config
  aria2_download:
    driver: local
    driver_opts:
      o: bind
      type: none
      device: /mnt/data01/media/download

networks:
  hostnet:
    external: true
    name: host

services:
  elfinder:
    image: ruediste/elfinder
    environment:
      TZ: Asia/Shanghai
    ports:
      - 8097:80
    volumes:
      - data:/data 
      - /etc/localtime:/etc/localtime:ro
  aria2:
    image: fanningert/aria2-with-ariang
    environment:
      PGID: 33
      PUID: 33
      TZ: Asia/Shanghai
    ports:
      - 6880:80
      - 6800:6800
    volumes:
      - aria2_conf:/conf
      - aria2_config:/config
      - aria2_download:/download
      - aria2_download:/finished
      - /etc/localtime:/etc/localtime:ro
  dlna:
    image: vladgh/minidlna
    networks:
      - hostnet
    environment:
      MINIDLNA_MEDIA_DIR: /media/download
      MINIDLNA_MEDIA_DIR_1: A,/media/audio
      MINIDLNA_MEDIA_DIR_2: V,/media/video
      MINIDLNA_MEDIA_DIR_3: P,/media/image
      MINIDLNA_FRIENDLY_NAME: MiniDLNA
      MINIDLNA_INOTIFY: "yes"
      MINIDLNA_NOTIFY_INTERVAL: 3
      TZ: Asia/Shanghai
    deploy:
      mode: replicated
      replicas: 1
    volumes:
      - data:/media
      - /etc/localtime:/etc/localtime:ro
```
#### 安装stack
```shell
docker swarm init
## 将compose文件保存到/tmp/docker-compose.yml
## 部署stack
docker stack deploy --compose-file /tmp/docker-compose.yml storage
```
#### have fun
| 应用 | 端口 |
|-----|-----|
|elfinder| 8097|
|aria2|6880|
|minidlna|8200|
#### notice
aria2的rpc密码貌似没法通过环境变量注入进去，该密码默认值为：`YOUR_SECRET_CODE`
所在位置：aria2_config卷下的aria2.conf:rpc-secret
#### 坑总结
1. 文件权限是一个很大的坑，如果没有统一elfinder和aria2的用户的话，将会导致一些文件因为没有权限导致的异常。
2. dlna需要使用host网络才能正确的被发现，其中有一个问题就是，使用host网络之后将不能指定端口，如果在compose文件中指定的话，将会导致容器无法拉起。
3. 容器内的时间需要通过同步/etc/localtime卷进行同步，为了以防抽风，还要把TZ变量注入进去。
