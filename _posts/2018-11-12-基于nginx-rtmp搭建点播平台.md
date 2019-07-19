---
layout: post
title: "基于nginx-rtmp搭建点播平台"
date: 2018-11-12
description: "基于nginx-rtmp搭建点播平台"
tag: 博客 
---

## 序言
RTMP（Real Time Message Protocol，实时信息传输协议）协议属于应用层协议，其靠底层的TCP来保证信息传输的可靠性。它由Adobe公司提出，用来解决多媒体数据传输流的多路复用（Multiplexing）和分包（packetizing）的问题。nginx-rtmp是由俄罗斯人开发的NGINX模块，该模块完善了NGINX对视频的支持，并且实现了对HLS的支持。

本次实验在滴滴云上完成，基于nginx-rtmp搭建一个点播平台。

## 准备

工具名称 | 描述
--------|---------
操作系统| CentOS Linux release 7.4.1708
nginx | release-1.15.0
nginx-rtmp-module | 1.2.1
VLC | 播放器

## 搭建流程
- 安装依赖库

```
sudo yum install git gcc make pcre-devel openssl-devel
```

- Build nginx with nginx-rtmp

```
sudo ./auto/configure --with-http_ssl_module --add-module=../nginx-rtmp-module-1.2.1
sudo make
sudo make install
```

- Start nginx Server

```
sudo /usr/local/nginx/sbin/nginx
```

- 新建放置视频文件的目录

```
sudo mkdir /nginxData/mp4
sudo chmod -R 777 /nginxData/mp4
```
由于nginx的子进程属于nobody（权限极低），所以本次实验将视频文件的所有权限都放开

- 移动stat.xsl文件

将nginx-rtmp源码中的stat.xsl文件复制到nginxData目录中，并将其权限改为664
```
sudo cp /home/dc2-user/nginx-rtmp-module-1.2.1/stat.xsl /nginxData/stat.xsl
sudo chmod 644 /nginxData/stat.xsl
```

## 配置详解
nginx的所有配置都在其conf目录下（也就是安装完成后的/usr/local/nginx/conf中）,最主要的配置文件nginx.conf文件具体配置样本文件见nginx.conf.md文件

### 配置文件
```
#user  nobody;
worker_processes  1;
error_log  logs/error.log debug;

events {
    worker_connections  1024;
}

http {
    ...

    server {
        listen       80;
        server_name  localhost;

        location /stat {
            rtmp_stat all;
            rtmp_stat_stylesheet stat.xsl;
        }

        location /stat.xsl {
            # you can move stat.xsl to a different location
            root /nginxData/;
        }

        location /control {
            rtmp_control all;
        }

        ...
    }
}

rtmp {
    server {
        listen 1935;
        ping 30s;

        application vod {
            play /nginxData/mp4;
        }
    }
}
```

## 使用
### 查看状态
通过访问http://<domain>/stat，可以查看点播平台的状态

### 访问视频
- 在/nginxData/mp4目录下放置一个名为test.mp4的视频文件
- 在本地通过VLC播放该视频, 点击VLC播放器file -> open network, 填写地址 rtmp://<ip>:<port>/vod/test.mp4