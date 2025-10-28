---
title: Docker架构LNMP并部署Discuz!
date: 2025-10-27 14:31:09
tags: 基础技术
toc: true
comments: true
description: 这篇文章选择我的毕业设计作为主题，用做博客的开篇，也纪念着我大学时代的结束。
---

这篇文章选择我的毕业设计作为主题，用做博客的开篇，也纪念着我大学时代的结束。
现在我们开始：

*本篇文章基于Ubuntu24.04系统作为基础*

# 1.事前准备
## 1.1安装docker
随着容器技术的兴起，传统架构面对的环境复杂、难以迁移，且容易因依赖冲突导致“开发环境正常，生产环境失效”等问题都能通过容器化架构完美解决；
故而，本篇文章我们采用docker容器来进行LNMP环境的架构。
首先我们完成docker容器的安装：
```
#更新系统包索引
sudo apt update
sudo apt upgrade -y

#​​添加Docker官方GPG密钥​
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

#添加Docker APT仓库​
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

#安装Docker引擎​
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

这样我们的docker环境就安装完成了
可以运行`docker --version`进行测试，预期输出：类似于`Docker version 24.0.7, build afdd53b`

## 1.2基础目录认知
首先我们对于正式架构前需要有一个目录认知，以便后续我们进行部署及管理：
```
discuz-docker/
├── mysql/                          # MySQL 数据目录
├── nginx/                          #nginx文件目录
├── php/                            #php文件目录
└── www/                           # 网站根目录（Discuz 程序）
```


# 2.LNMP架构

## 2.1docker网络搭建
只需创建一下即可
```
docker network create discuz-network
```
docker网络搭建完成后，后续仅需在启动容器时将其加入docker网络，容器间即可自动建立连接，取缔了已过时的--link

## 2.2MySQL
对于搭建而言，关于mysql的部署较为简单
首先从Docker的公共仓库 Dockerhub 下载 MySQL 镜像：
```
docker pull mysql
```
然后启动数据库：
```
docker run --name discuz-mysql --network discuz-network -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql
```
这样一来就完成了MySQL的安装，后续我们再在php-fpm容器中使用。

*补充说明*
上述命令中：
- `run`：创建一个新的容器
- `--name`：指定容器的名称（如果留空，docker会自动分配一个名称）
- `--network`：加入docker网络，后跟相应docker网络名称
- `-p`：导出容器端口到本地服务器，格式：-p <local-port>:<container-port>。在本例中，我们映射容器的80端口到本地服务器的80端口。
- `-e`:设置容器内部环境变量
- `-d`：后台启动。
- `mysql`：是拉取的镜像名称，代表启动的是该镜像


## 2.3Nginx
首先从Docker的公共仓库 Dockerhub 下载 Nginx 镜像：
```
docker pull nginx
```
这个命令会下载容器需要的所有部件，并且Docker会对它们进行缓存，所以在运行容器的时候，不需要每次都下载容器镜像。

默认情况下，Docker nginx服务器的HTML路径（网站根目录）在容器`/usr/share/nginx/html`目录下，现在需要把这个目录映射到本地服务器的`~/www/html`目录。在上面命令的基础上加上`-v`参数，具体如下：
```
docker run --name discuz-nginx -p 80:80 -v /home/program/discuz-docker/www/:/usr/share/nginx/html -d nginx
```
`-v`的参数格式如下
`<local-volumes>:<container-volumes>`
/Users/xiaohao/Desktop/www 左边是本机项目地址 : 右边是nginx容器的项目地址 /usr/share/nginx/html

在/Users/xiaohao/Desktop/www 下新建 index.html ，也就是测试用的页面文件，这个文件无需配置
在浏览器上访问 http://localhost/或者本机ip地址/或者127.0.0.1 ，刷新一下就可以看到新的内容了。

接下来配置 Nginx
```
cd /home/program/discuz-docker/nginx/
mkdir config
cd config
touch default.conf
docker cp discuz-nginx:/etc/nginx/conf.d/default.conf default.conf
```
再加一个`-v`参数，把本地的配置文件映射到容器上，再重启容器：
```
docker rm -f discuz-nginx
docker run --name discuz-nginx --network discuz-network -p 80:80 \
> -v /home/program/discuz-docker/www/:/usr/share/nginx/html \
> -v /home/program/discuz-docker/nginx/config/default.conf:/etc/nginx/conf.d/default.conf \
> -d nginx
```
*注意*
如果配置文件有修改，需要重启容器生效：
```
docker restart discuz-nginx
```
这样就可以直接在本地修改配置文件了。

## 2.3PHP
首先下载php-fpm镜像：
由于dockerhub默认的php-fpm为8.1.1版本，其中没有内置mysqli扩展，所以我们需要自己构建dockerfile来打包一个我们自己的php-fpm
直接在项目根目录下构建dokcerfile：
```
cd /home/program/discuz-docker
touch dockerfile
```
内容如下
```
FROM php:8.1-fpm

# 更换为国内软件源
RUN sed -i 's/deb.debian.org/mirrors.aliyun.com/g' /etc/apt/sources.list && \
    sed -i 's/security.debian.org/mirrors.aliyun.com/g' /etc/apt/sources.list

# 安装系统依赖
RUN apt-get update && apt-get install -y \
    libpng-dev \
    libjpeg-dev \
    libfreetype6-dev \
    libzip-dev \
    libonig-dev \
    && rm -rf /var/lib/apt/lists/*

# 安装 PHP 扩展（包括 mysqli）
RUN docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install \
    mysqli \
    pdo \
    pdo_mysql \
    gd \
    mbstring \
    zip \
    opcache

# 设置工作目录
WORKDIR /var/www/html

# 复制项目文件
COPY . .

# 设置权限
RUN chown -R www-data:www-data /var/www/html

# 暴露端口
EXPOSE 9000

CMD ["php-fpm"]
```
具体配置如下：
```
docker build -t my-php-fpm .    #根据Dockerfile创建自定义镜像
docker run --name discuz-php --network discuz-network -p 9000:9000 -d my-php-fpm
cd /home/program/discuz-docker/php
touch www.conf
#复制www.conf到本地的www/config中
docker cp discuz-php:/usr/local/etc/php-fpm.d/www.conf www.conf  
#查看容器的  containerID
docker ps -a   
#进入到正在运行的php-fpm容器中
docker exec -it containerID /bin/bash
#进入到src中会看到2个压缩包
root@007a1318b5ea:/var/www/html# cd /usr/src/
root@007a1318b5ea:/usr/src# ls -ll
total 11460
-rw-r--r-- 1 root root 11728680 Dec 21  2021 php.tar.xz
-rw-r--r-- 1 root root      833 Dec 21  2021 php.tar.xz.asc
root@007a1318b5ea:/usr/src# 
#解压tar.xz文件：先 xz -d xxx.tar.xz 将 xxx.tar.xz解压成 xxx.tar 然后，再用 tar xvf xxx.tar来解包。
root@007a1318b5ea:/usr/src# xz -d php.tar.xz
root@007a1318b5ea:/usr/src# ls -ll
total 148288
-rw-r--r-- 1 root root 151838720 Dec 21  2021 php.tar
-rw-r--r-- 1 root root       833 Dec 21  2021 php.tar.xz.asc
root@007a1318b5ea:/usr/src#tar xvf php.tar
root@007a1318b5ea:/usr/src# ls -ll
total 148292
drwxr-xr-x 16 1000 1000      4096 Dec 15  2021 php-8.1.1
-rw-r--r--  1 root root 151838720 Dec 21  2021 php.tar
-rw-r--r--  1 root root       833 Dec 21  2021 php.tar.xz.asc
root@007a1318b5ea:/usr/src# exit;
#回到宿主机的config文件夹中，把容器里面的php.ini复制下来
touch php.ini
docker cp discuz-php:/usr/src/php-8.1.1/php.ini-production php.ini
```
在本地服务器修改 php.ini 的内容，设置cgi.fix_pathinfo=0（要先删除前面的;注释符）：
```
cgi.fix_pathinfo=0
```
然后重新启动容器：
```
docker rm -f discuz-php
docker run --name discuz-php --network discuz-network \
-v /home/program/discuz-docker/www/:/usr/share/nginx/html  \
-v /home/program/discuz-docker/php/www.conf:/usr/local/etc/php-fpm.d/www.conf \
-v /home/program/discuz-docker/php/php.ini:/usr/local/etc/php-8.1.1/php.ini \
-d my-php-fpm
```

## Nginx容器支持FPM
打开 nginx 的配置文件
```
vi /home/program/discuz-docker/nginx/config/default.conf
```
然后修改内容如下：
```
server {
    listen       80;
    server_name  localhost;
    root /usr/share/nginx/html;
    index index.php index.html index.htm;

    #charset koi8-r;
    #access_log  /var/log/nginx/log/host.access.log  main;

    location / {
        #root   /usr/share/nginx/html;
        #index  index.php index.html index.htm;
    try_files $uri $uri/ =404;
    }

    error_page  404  /404.html;
    location = /40x.html {
        root    /user/share/nginx/html;     
    }

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    location ~ \.php$ {
        fastcgi_pass   discuz-php:9000;
        fastcgi_index  index.php;
        #fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include        fastcgi_params;
    }
    
    # Discuz静态文件缓存
    location ~* \.(js|css|png|jpg|jpeg|gif|ico)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    location ~ /\.ht {
        deny  all;
    }
}
```
最后重新启动nginx容器：
```
docker rm -f discuz-nginx
docker run --name discuz-nginx -p 80:80 --network discuz-network \
-v /home/program/discuz-docker/www:/usr/share/nginx/html \
-v //home/program/discuz-docker/nginx/conf.d/default.conf:/etc/nginx/conf.d/default.conf \
-d nginx
```

这样一来我们的LNMP架构就完成了！