---
title: 第一次用 Docker 镜像部署静态站
date: 2026-07-06T00:00:00.000+00:00
lang: zh
duration: 8min
author: 沈佳棋
---

## 背景

这次要部署的是一个开源项目生成出来的静态网站。项目本身没有后端服务，最终产物就是一批 HTML、CSS、JS 和文章文件。

我以前部署静态站，脑子里默认会想到两种方式：

1. 把文件直接丢到 nginx 的静态目录。
2. 找个平台，比如 Vercel、GitHub Pages。

这次想换一种方式：在本地把静态站打成 Docker 镜像，传到自己的服务器，再用服务器上的 nginx 反代过去。

这样做有一个好处：服务器不需要知道项目怎么构建，也不需要安装 Python、Node 或其他依赖。它只负责加载镜像、启动容器。以后更新网站时，也是重新打一个镜像包传过去。

## 服务器已有环境

我的服务器上已经有一套 nginx 入口服务，所有项目都放在：

```text
/home/deploy
```

nginx 配置放在：

```text
/home/deploy/nginx/conf.d
```

现有结构是一个 nginx 容器负责对外暴露 80 和 443，其他项目容器接入同一个 Docker 网络，再由 nginx 通过容器名反向代理。

这篇文章里的服务器地址、用户名、域名都做了脱敏，下面统一用这些占位符：

```text
<server_user>      服务器用户名
<server_host>      服务器 IP 或域名
<domain>           对外访问域名
<app_network>      nginx 和业务容器共用的 Docker 网络
```

## 第一步：本地构建静态 nginx 镜像

静态站目录是：

```text
site/
```

先进入这个目录：

```bash
cd /path/to/project/site
```

然后执行：

```bash
docker buildx build \
  --platform linux/amd64 \
  -t tech-blog-hub-static:latest \
  -f - . <<'EOF'
FROM nginx:alpine
COPY . /usr/share/nginx/html/
EOF
```

这里最容易卡住的是：

```text
/usr/share/nginx/html/
```

这个路径属于 Docker 镜像内部。构建镜像时不需要在本地电脑或服务器宿主机上手动创建它。

`nginx:alpine` 官方镜像默认会从容器内部的 `/usr/share/nginx/html/` 读取静态文件。构建镜像时，命令里的 `COPY . /usr/share/nginx/html/` 会把当前 `site/` 目录下的文件复制进去。

加上 `--platform linux/amd64` 是因为我的本地电脑和服务器架构可能不一样。本地用 Mac 构建时，显式指定服务器常见的 Linux x86_64 架构，后面少一些奇怪的问题。

构建完成后看一下镜像：

```bash
docker images tech-blog-hub-static
```

## 第二步：本地试跑镜像

镜像构建完，先在本机跑一下，再传服务器：

```bash
docker run --rm -p 8080:80 tech-blog-hub-static:latest
```

浏览器打开：

```text
http://127.0.0.1:8080
```

如果页面正常，说明镜像里的 nginx 能读到静态文件。

这个命令没有加 `-d`，所以容器会占着当前终端。测试完直接按：

```text
Ctrl + C
```

因为命令里带了 `--rm`，容器停止后会自动删除，不需要再手动清理。

如果已经在后台跑起来了，可以用下面的方式停掉：

```bash
docker ps
docker stop <container_id>
```

## 第三步：导出镜像包

确认本地能访问后，把镜像导出成一个压缩包：

```bash
docker save tech-blog-hub-static:latest | gzip > tech-blog-hub-static-latest.tar.gz
```

看一下文件大小：

```bash
ls -lh tech-blog-hub-static-latest.tar.gz
```

这里有个小误区。我一开始也会顺口把它叫成 zip 包。这个文件是 Docker 镜像包，传到服务器后应该用 `docker load` 加载，普通的 `unzip` 处理不了它。

## 第四步：传到服务器

把镜像包传到服务器：

```bash
scp tech-blog-hub-static-latest.tar.gz \
  <server_user>@<server_host>:/home/deploy/
```

登录服务器：

```bash
ssh <server_user>@<server_host>
cd /home/deploy
```

加载镜像：

```bash
docker load -i tech-blog-hub-static-latest.tar.gz
```

确认镜像已经进入服务器 Docker：

```bash
docker images tech-blog-hub-static
```

能看到 `tech-blog-hub-static:latest` 就可以继续。

## 第五步：用 docker compose 启动容器

在服务器上创建项目目录：

```bash
cd /home/deploy
mkdir -p tech-blog-hub
cd tech-blog-hub
```

创建 `docker-compose.yml`：

```yaml
services:
  static:
    image: tech-blog-hub-static:latest
    container_name: tech-blog-hub-static
    restart: always
    networks:
      - app-network

networks:
  app-network:
    external: true
```

这里的 `app-network` 要替换成自己服务器上 nginx 容器正在使用的 Docker 网络。

可以先查一下：

```bash
docker network ls
```

或者看 nginx 的 compose 文件：

```bash
cat /home/deploy/nginx/docker-compose.yml
```

启动容器：

```bash
docker compose up -d
```

确认容器已经起来：

```bash
docker ps | grep tech-blog-hub-static
```

## 第六步：接入 nginx

打开 nginx 配置文件：

```bash
vim /home/deploy/nginx/conf.d/<site>.conf
```

在 HTTPS 的 `server` 块里加上：

```nginx
location = /tech-blog-hub {
    return 301 /tech-blog-hub/;
}

location /tech-blog-hub/ {
    proxy_pass http://tech-blog-hub-static/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}
```

前面那个精确匹配的 redirect 是为了处理少一个尾斜杠的访问：

```text
https://<domain>/tech-blog-hub
```

让它跳到：

```text
https://<domain>/tech-blog-hub/
```

后面的 `proxy_pass http://tech-blog-hub-static/;` 会把请求转给同一个 Docker 网络里的静态站容器。

改完先检查 nginx 配置：

```bash
docker exec nginx nginx -t
```

看到配置测试成功后，再 reload：

```bash
docker exec nginx nginx -s reload
```

最后访问：

```text
https://<domain>/tech-blog-hub/
```

## 这次踩到的几个点

第一，`/usr/share/nginx/html/` 是容器内部路径。构建镜像时把本地 `site/` 复制进去，服务器宿主机上不需要创建这个目录。

第二，`docker buildx build` 之后的产物是本地 Docker 镜像。它不会出现在当前文件夹里，要用 `docker images` 查看。

第三，`docker save` 之后得到的 `.tar.gz` 是镜像包。传到服务器后用 `docker load -i`，不要按普通 zip 去解压。

第四，nginx 反代到容器名的前提是两个容器在同一个 Docker 网络里。nginx 容器和静态站容器如果不在同一个网络，`http://tech-blog-hub-static/` 这个名字解析不到。

第五，先在本地用 `docker run --rm -p 8080:80` 测一下，能省掉很多服务器上的来回排查。

## 最后保留的部署流程

本地构建：

```bash
cd /path/to/project/site

docker buildx build \
  --platform linux/amd64 \
  -t tech-blog-hub-static:latest \
  -f - . <<'EOF'
FROM nginx:alpine
COPY . /usr/share/nginx/html/
EOF
```

本地验证：

```bash
docker run --rm -p 8080:80 tech-blog-hub-static:latest
```

导出镜像：

```bash
docker save tech-blog-hub-static:latest | gzip > tech-blog-hub-static-latest.tar.gz
```

上传服务器：

```bash
scp tech-blog-hub-static-latest.tar.gz \
  <server_user>@<server_host>:/home/deploy/
```

服务器加载：

```bash
cd /home/deploy
docker load -i tech-blog-hub-static-latest.tar.gz
docker images tech-blog-hub-static
```

服务器启动：

```bash
cd /home/deploy/tech-blog-hub
docker compose up -d
docker ps | grep tech-blog-hub-static
```

nginx 检查并重载：

```bash
docker exec nginx nginx -t
docker exec nginx nginx -s reload
```

这套流程跑通以后，我对 Docker 部署静态站的理解清楚了很多：项目文件、镜像内部文件、服务器宿主机文件，这三层要分开看。只要这三层不混，部署过程就没有想象中那么玄学。
