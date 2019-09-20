---  
layout: post                          # 表明是博文  
title: "通过nginx来实现TCP转发https"           # 博文的标题  
subtitle: 通过nginx来实现TCP转发https    # 博文的小标题  
date: 2019-09-20                      # 博文的发表日期，此日期决定主页上博文的先后顺序  
author: "jano X"                       # 博文的作者  
header-img: "img/post-bg-cka.jpg"     # 博文页面上端的背景图片  
header-mask: "0.1"                    # 博文页面上端的背景图片的亮度，数值越大越黑暗  
catalog: True                         # 开启catalog，将在博文侧边展示博文的结构  
tags:                                 # 本篇博文的标签，该标签将在archive页面中对博文进行分类  
  - git  
  - github  
  - vscode  
  - markdown  
  - jekyll  
  - 图床  
---  
# 通过nginx来实现TCP转发https
## 需求背景
项目的主域名为`test.example.com`，在跨地区访问或国家防火墙封锁等网络问题，会出现无法直接访问的问题。
因此通过在海外某地区采购一台云服务器来部署nginx做代理。最终用户的访问方式会有以下两种方式：

1. 用户能正常访问业务服务器
用户 -> 真实服务器(test.example.com,10.10.10.10)

2. 用户访问异常,则会在app请求备用线路域名(app切换域名需由研发实现)用户 -> 代理节点(mlp-00-test.example.com,135.10.10.10) -> 真实web服务器(test.example.com,10.10.10.10)

> 两种方式都能访问,由业务APP自行选择线路

## 解决方式
1. 设置一系列备用域名`mlp-[0-9][0-9]-test.example.com` 并将其解析到代理节点
2. 代理节点部署nginx,开启tcp转发,将请求转发到后端真实服务器(test.example.com)
3. 真实服务器需配置备用域名作为`server_name`,若开启https，还需证书支持备用域名的解析

## 实现细节
- 代理节点安装nginx

nginx 需安装模块`stream`及`ngx_stream_ssl_preread_module `ngx_stream_ssl_preread_module 模块（1.11.5）允许从 ClientHello 消息中提取信息而不终止SSL / TLS，例如，通过 SNI 请求的服务器名称或 ALPN 中公布的协议。默认情况下不构建此模块，
应使用 `--with-stream_ssl_preread_module` 配置参数启用它。

```bash
/opt/nginx/sbin/nginx  -V
nginx version: nginx/1.12.2
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-36) (GCC) 
built with OpenSSL 1.0.2k-fips  26 Jan 2017
TLS SNI support enabled
configure arguments: --prefix=/opt/nginx --with-http_ssl_module --with-pcre=/tmp/pcre-8.42 --with-stream --with-stream_ssl_preread_module
```

nginx.conf 配置stream的支持:
```conf
stream{

    log_format  main  '$remote_addr - [$time_local] $connection '
                      '$status $proxy_protocol_addr $server_addr ';
    access_log  logs/access.log  main;
    resolver 8.8.8.8;
    resolver_timeout 60s;
    variables_hash_bucket_size 512;

    include stream/*.conf;
}
```
stream/proxy.conf 如下:
```conf
#$ssl_preread_server_name #通过SNI请求的服务器名称
map $ssl_preread_server_name $real_server {
    ~^mlp-[0-9][0-9]-test.example.com  apollo;
    test.example.com  apollo;
    default vipay_default;
}

upstream apollo {
    server 10.10.10.10:443;
}

upstream vipay_default {
    server 10.10.10.11:443;
    server 10.10.10.12:443;
}


server {
    listen 443;
    ssl_preread on;  #允许在预读阶段从ClientHello消息中提取信息
    resolver 8.8.8.8;
    proxy_pass $real_server;
    proxy_connect_timeout 5s;
    proxy_timeout 15s;
    error_log /log/nginx/stream_ssl_preread.log info;
}
```

- 真实服务器配置支持备用域名请求

新增配置如下：
```conf
server {
            listen 443;
            server_name    test.example.com ~^mlp-[0-9][0-9]-test.example.com;
            access_log  /log/nginx/access_test.example.com.log main;
            error_log  /log/nginx/error_test.example.com.log;
            ssl on;
            ssl_certificate /opt/nginx/ssl/_.vipaylife.com/server.pem;
            ssl_certificate_key     /opt/nginx/ssl/_.vipaylife.com/server.key;
            ssl_session_timeout 5m;
            ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
            ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
            ssl_prefer_server_ciphers on;
		...
	    }
```

> 需注意SSL证书也要支持备用域名的请求，这里使用泛解析证书

- 将备用域名的DNS解析指向代理节点

现在可以将备用域名`mlp-00-test.example.com` 指向代理节点了
解析后进行测试
```bash
curl https://mlp-00-test.example.com/
```

## 备用域名命名规则
```
mlp-[0-9][0-9]-selfdefine.domain 
```
> mlp: multi line proxy 编号从00-99，分别代表100个代理节点

## 将主域名的解析切到代理节点
之前的配置，需依赖app自主选择主或备用域名进行请求。如果有需要也可以将**主域名的DNS解析直接指向备用域名**
例如将域名`test.example.com` DNS 解析到代理节点后.所有用户的请求路由如下：

用户 -> 代理节点(test.example.com,135.10.10.10) ->真实服务器(10.10.10.10)
