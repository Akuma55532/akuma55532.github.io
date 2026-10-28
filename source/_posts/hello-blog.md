---
title: Hello Blog
tags:
  - blog
  - docker
date: 2025-08-18 10:22:55
---


## 前言

I've started my internship recently. My job involves helping the company review and rewrite some technical documents and tutorials. By now I've grown accustomed to writing articles with Markdown. I've realized that recording while studying is a really great way to help you remember your technical details. It also sparks your interest to explore the knowledge more deeply. So now I am going to develop it into a personal habit. I will try to write my technical blog in English.

In this blog, I want to record how I set up this blog site using Docker. It took me some time.

## Env build

First，we need have a server to deploy our hexo blog. Here I used the Ali's cloud which is free to colloage students for a year.
Now We need to setup docker. I am using the Ubuntu-22.04-LTS. Here is [offical tutorials site](https://docs.docker.com/engine/install/ubuntu/)

1. Set up Docker's apt repository.

~~~shell
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
~~~

有时候出现错误，可以使用阿里云版本

~~~shell
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
~~~


2. Install the Docker packages.

~~~shell
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
~~~

3. change the docker registry-mirrors.

~~~shell
sudo nano /etc/docker/daemon.json
~~~
write the registry-mirrors
~~~json
{
    "registry-mirrors": [
        "https://docker.m.daocloud.io",
        "https://docker.1panel.live",
        "https://hub.rat.dev"
    ]
}
~~~
restart docker
~~~shell
sudo service docker restart
~~~

4. Verify that the installation is successful by running the hello-world image:

~~~shell
sudo docker run hello-world
~~~

## build own image

Original image pulled from docker hub can not be used immeditly.
We need to build our own image.

First, pull the node and nginx images.
~~~shell
docker pull node:22-alpine
docker pull nginx:stable-alpine-perl
~~~
check image list
~~~shell
docker image list
~~~
now we need make a dir to build image.
create a Dockerfile
~~~docker
FROM node:22-alpine

WORKDIR /app

RUN apk add --no-cache git

RUN npm install -g hexo-cli

RUN mkdir /app/blog

COPY entrypoint.sh /app/entrypoint.sh
RUN chmod +x /app/entrypoint.sh

CMD ["tail", "-f", "/dev/null"]
~~~
create entrypoint.sh in the same dir.
~~~shell
#!/bin/sh
set -e

cd /app/blog

if [ ! -f package.json ]; then
  echo "正在初始化 Hexo 博客..."
  hexo init .
  npm install
  echo "初始化完成！"
else
  echo "检测到现有博客项目，跳过初始化"
fi

echo "安装或更新依赖..."
npm install
~~~
exec the command to build image
~~~shell
docker build -t node_withhexo .
~~~
now we can see our own image in `docker image list`

## Run docker

create docker-compose.yml in an empty dir.
~~~docker
services:
  node:
    image: node_withhexo:latest
    container_name: node_hexo
    volumes:
      - /home/hexo/blog:/app/blog
    working_dir: /app/blog
    command: sh -c "/app/entrypoint.sh && hexo generate && touch /app/blog/.generated && tail -f /dev/null"
    healthcheck:
      test: ["CMD", "test", "-f", "/app/blog/.generated"]
      interval: 5s
      timeout: 10s
      retries: 10
      start_period: 5m

  nginx:
    image: nginx:stable-alpine-perl
    container_name: nginx_hexo
    ports:
      - 8080:80
    volumes:
      - /home/hexo/blog/public:/usr/share/nginx/html
    depends_on:
      node:
        condition: service_healthy
~~~
be sure about your dir is the same as the example,and there must be nothing in the blog dir.