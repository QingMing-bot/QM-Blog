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
## 1.1 安装docker
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

## 1.2 基础目录认知
首先我们对于正式架构前需要有一个目录认知，以便后续我们进行部署及管理：
```
discuz-docker/
├── mysql/                          # MySQL 数据目录
├── nginx/                          #nginx文件目录
├── php/                            #php文件目录
└── www/                           # 网站根目录（Discuz 程序）
```


# 2.LNMP架构

## 2.1 docker网络搭建
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


## 2.3 Nginx
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

## 2.3 PHP
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

## 2.4 Nginx容器支持FPM
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

# 3.安装Discuz！论坛
使用weget下载最新版Discuz！安装包
```
weget -P /home/program/discuz-docker/www https://foruda.gitee.com/attach_file/1756976263314096137/discuz_x3.5_sc_utf8_20250901.zip?token=2c73766cc44e3159933cc411cf1e7afb&ts=1761133475&attname=Discuz_X3.5_SC_UTF8_20250901.zip 
```
## 3.1 解压
```
# 进入项目目录
cd /home/program/discuz-docker/www/

# 解压Discuz安装包
unzip Discuz_X3.5_SC_UTF8_20250901.zip

# 查看解压后的文件结构
ls -la
```

## 3.2 部署Discuz文件
```
# 将Discuz文件复制到网站根目录
cp -r upload/* ./

# 设置文件权限（非常重要！）
chmod -R 777 config/ data/ uc_server/ uc_client/

# 检查文件是否部署成功
ls -la | grep -E "(index.php|admin.php|install)"
```

## 3.3 检查Nginx配置
确保您的`config/default.conf`配置文件支持Discuz。如果需要，可以这样配置：
```
# 编辑Nginx配置
vim config/default.conf
```
确保配置包含以下内容：​​
```
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html index.htm index.php;
    
    location / {
        try_files $uri $uri/ /index.php?$args;
    }
    
    location ~ \.php$ {
        fastcgi_pass discuz-php:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
    
    # Discuz静态文件缓存
    location ~* \.(js|css|png|jpg|jpeg|gif|ico)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

## 3.4 重启服务使配置生效
```
# 重启Nginx容器
docker restart discuz-nginx

# 检查容器状态
docker ps

# 查看Nginx日志确认无错误
docker logs discuz-nginx
```

## 3.5 创建Discuz数据库
```
# 进入MySQL容器
docker exec -it discuz-mysql mysql -uroot -p

# 在MySQL中执行以下命令（将your_password替换为实际密码）：
CREATE DATABASE discuz DEFAULT CHARSET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'discuz_user'@'%' IDENTIFIED BY '123456';
GRANT ALL PRIVILEGES ON discuz.* TO 'discuz_user'@'%';
FLUSH PRIVILEGES;
EXIT;
```

## 3.6 通过Web界面安装Discuz
1.​打开浏览器访问​​：http://您的服务器IP/install/

2.​​按照安装向导操作​​：

设置数据库信息：
数据库服务器：discuz-mysql
数据库名：discuz
数据库用户名：discuz_user
数据库密码：123456 或 你自己设置的密码
表前缀：pre_（默认即可）

3.​​设置管理员账户​​：
论坛名称：您的论坛名称
管理员账号/密码/邮箱

## 3.7 安装后的操作
```
# 删除安装目录（安全考虑）
rm -rf install/

# 再次检查文件权限
chmod -R 755 ./
chmod -R 777 data/ config/ uc_server/data/ uc_client/data/
```

# 4.配置HTTPS
## 4.1 SSL 证书申请
***首先需要确保具有可解析的域名***
第一步：安装certbot
```
apt update && apt install -y certbot
```
第二步：申请Let`s Encrypt证书
```
certbot certonly --standalone -d 你的域名
```
根据提示完成证书申领
完成后你的域名存在于/etc/letsencrypt/live/你的域名/ 路径

## 4.2 配置nginx
```
vim /home/program/discuz-docker/nginx/conf.d/default.conf
```
确保配置如下
```
# HTTP 重定向到 HTTPS
server {
    listen 80;
    server_name 你的域名;
    return 301 https://$host$request_uri;
    # 你也可以选择强制重定向到某端口，如3333
    # return 301 https://$host:3333$request_uri;
}

# HTTPS 服务器配置
server {
    listen 443 ssl http2;
    # 如果你选择指定端口，则这里需要为指定端口提供https
    # listen 3333 ssl http2;
    server_name 你的域名;

    ssl_certificate /etc/letsencrypt/live/你的域名/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/你的域名/privkey.pem;
    # ... 
}
```

## 4.3 重新启动nginx容器
```
# 停止旧容器
docker rm -f discuz-nginx

# 使用正确的卷挂载启动新容器
docker run -d --name discuz-nginx \
  --network discuz-network \
  -p 80:80 \
  -p 443:443 \
  -v /etc/letsencrypt:/etc/letsencrypt:ro \
  -v /home/program/discuz-docker/www:/usr/share/nginx/html:ro \
  -v /home/program/discuz-docker/nginx/conf.d:/etc/nginx/conf.d:ro \
  nginx
```

## 4.4 最后在Discuz！后台配置
- 访问 https://你的域名/admin.php
- 进入 ​​全局 → 站点信息​​
- 修改 ​​网站 URL​​ 为：https://hui.lch.ink/
- 保存设置

这样一来完整的LNMP架构及Discuz！论坛就部署完毕了，感谢阅读！

**心潮伏涌意难平，几心悲喜似流萤**