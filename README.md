# v2board-docker


| FPM | Caddy-FPM | XrayR |
|---|-----|---|
| [![GitHub Actions Workflow Status](https://img.shields.io/github/actions/workflow/status/waynelone/v2board-docker/fpm-docker-image.yml)](https://github.com/WayneLone/v2board-docker/actions/workflows/fpm-docker-image.yml) | [![GitHub Actions Workflow Status](https://img.shields.io/github/actions/workflow/status/waynelone/v2board-docker/caddy-fpm-docker-image.yml)](https://github.com/WayneLone/v2board-docker/actions/workflows/caddy-fpm-docker-image.yml) | [![GitHub Actions Workflow Status](https://img.shields.io/github/actions/workflow/status/waynelone/v2board-docker/xrayr-docker-image.yml)](https://github.com/WayneLone/v2board-docker/actions/workflows/xrayr-docker-image.yml) |

鉴于找到的 docker 镜像都像开盲盒一样，于是就自己搞了一份。镜像是基于 [v2board](https://github.com/v2board/v2board) 构建的，镜像内部使用 supervisor 作为主进程，启动了 crontab、队列、fpm。

## 前言

在启动 v2board 前，请先启动 redis 和 mysql。docker compose 如下：

```yaml
services:

  redis:
    image: redis:7.2.4
    container_name: redis
    restart: unless-stopped
    command: redis-server /etc/redis/redis.conf
    volumes:
      - ~/redis/data:/data
      - ~/redis/redis.conf:/etc/redis/redis.conf

  mysql:
    image: mysql:8.3.0
    container_name: mysql
    restart: unless-stopped
    environment:
      - MYSQL_ROOT_PASSWORD=demo123.
    volumes:
      - ~/mysql/conf:/etc/mysql/conf.d
      - ~/mysql/data:/var/lib/mysql
```

这里将 redis 和 mysql 相关配置和数据映射到外部。其中的 redis 的配置文件可以参考: <https://github.com/redis/redis> 中的 redis.conf 文件（下载并修改其中的内容）。

### 初始化 mysql 数据库

首先进入到 mysql 实例当中

```shell
# 进入容器实例
docker compose exec -it mysql bash
# 登录 mysql cli
mysql -u root -p
```

创建数据库和用户

```shell
-- 创建数据库
create database v2board;
-- 创建用户并指定密码
create user 'v2board'@'%' identified by 'v2board123.';
-- 把 v2board 数据库所有的权限给到 v2board 用户
grant all on v2board.* to 'v2board'@'%';
-- 刷新权限 使其生效
flush privileges;
```

## fpm-alpine

此镜不包含 web server，需要使用 nginx 或者 caddy 作为前置 web server，并使用 fast-cgi 通信。

nginx 请参考以下配置：

```nginx
server {
    server_name demo.com;
    index index.php index.html;
    root /var/www/html/v2board/public;

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass v2board:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }

    location / {
        try_files $uri $uri/ /index.php?$query_string;
        gzip_static on;
    }
}
```

docker compose 调整部分：

```yaml
  v2board:
    image: vkarz/v2board:fpm-alpine-1.7.4
    container_name: v2board
    restart: unless-stopped
    volumes:
      - ~/nginx/php/html/v2board:/var/www/html/v2board
    depends_on:
      - redis
      - mysql

  nginx:
    image: nginx:alpine
    container_name: nginx
    restart: always
    ports:
      - "80:80"
      - "443:443"
    environment:
      - TZ=Asia/Shanghai
    volumes:
      - ~/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ~/nginx/vhost:/etc/nginx/conf.d
      - ~/nginx/ssl:/etc/nginx/ssl
      - ~/nginx/php:/var/www
```

这里映射了 nginx 的主要配置文件及配置文件目录、证书等，请根据实际情况调整。

需要注意的点：

由于使用 fpm 进行通信，这就要求 nginx 也要能访问到实际的 php 文件，所以要把 v2board 容器实例里的源码映射出去。

`fastcgi_param SCRIPT_FILENAME` 这一行的作用是告诉 fastcgi 要到哪里找到该文件。因此 `SCRIPT_FILENAME` 后面的路径一定要能在 v2board 容器实例中找到。

nginx 配置文件中使用了 `$document_root` 变量，该变量依赖于 `root` 指令指定的路径，而 `root` 指令后面的路径正好是 v2board 容器实例中 php 源码所在的目录（这里 nginx 和 v2board 容器使用相同的路径）。如果 nginx 容器实例 php 源码和 v2board 容器实例 php 源码不在同一路径，则替换 `$document_root` 变量为 `/var/www/html/v2board/public `

## caddy-fpm-alpine

该镜像包含 caddy 作为 web server。可以参考以下内容（或上面提到的 caddy 文档）：

```nginx
demo.com {
  root * /var/www/html/v2board/public
  php_fastcgi 127.0.0.1:9000 {
    root /var/www/html/v2board/public
  }
  file_server
  encode gzip
}
```

caddy 请参阅文档：

[php_fastcgi (Caddyfile directive) &mdash; Caddy Documentation](https://caddyserver.com/docs/caddyfile/directives/php_fastcgi)

docker compose 调整部分：

```yaml
  v2board:
    image: vkarz/v2board:caddy-fpm-alpine-1.7.4
    container_name: v2board
    restart: unless-stopped
    volumes:
      - ~/v2board/Caddyfile:/etc/caddy/Caddyfile
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - redis
      - mysql
```

## 初始化 v2board

在初始化之前，最好检查一下容器的日志，确保队列服务成功启动了

```shell
docker compose logs v2board
```

如果显示启动失败，则尝试映射出一份 `.env.example` 文件到宿主机上，编辑 redis 部分。然后重启 v2board 实例，再次确认容器情况。

```shell
docker compose restart v2board
docker compose logs v2board
```

初始化 v2board

```shell
# 进入容器实例
docker compose exec -it v2board /bin/sh
# 初始化 v2board
# 数据库用户名和密码就是前面设置的
php artisan v2board:install
```
