---
title: LearnedKV(RocksDB+Learned-Index)测试记录
date: 2025-10-31 18:30:33
tags:
    - LSM
    - RocksDB
    - Learned-Index
    - SSD
categories: Study
---

# 说明

## 使用Docker

现在要做LearnedKV论文的测试。
希望在一个独立的环境配置RocksDB，所以想到用Docker。
让AI创建了一个Dockerfile：

```Dockerfile
FROM ubuntu:20.04

ENV DEBIAN_FRONTEND=noninteractive

# 环境配置基于LearnedKV文档

RUN apt-get update && apt-get install -y sudo libhiredis-dev \
build-essential git cmake pkg-config curl wget unzip python3 python3-pip \
libtbb-dev libsnappy-dev libbz2-dev liblz4-dev libzstd-dev \
libleveldb-dev libgmp-dev libeigen3-dev \
libaio-dev uuid-dev zlib1g-dev \
&& rm -rf /var/lib/apt/lists/*

RUN pip3 install --no-cache-dir numpy pandas matplotlib seaborn

# 安装RocksDB (在/opt文件夹)

WORKDIR /opt
RUN git clone https://github.com/facebook/rocksdb.git && \
cd rocksdb && \
git fetch --all --tags && \
git checkout v9.3.1 && \
make -j"$(nproc)" shared_lib && \
    make install -j"$(nproc)"

# 运行时能找到动态库

ENV LD_LIBRARY_PATH=/usr/local/lib

# 将 /usr/local/lib 加入动态链接缓存（可选但更稳妥）

RUN echo "/usr/local/lib" > /etc/ld.so.conf.d/local.conf && ldconfig

# 工作目录

WORKDIR /work
```

## 镜像相关

为了解决拉取镜像超时问题，修改了`/etc/docker/daemon.json`文件。

可使用`docker info`查看docker信息。
使用`docker images`或`docker images | grep learnedkv`获取镜像信息。
使用`docker system df -v`查看镜像空间使用。
使用 `docker ps -a` 命令看一下所有容器的状态

## 构建命令

使用`docker build -t learnedkv:ubuntu20.04 .`

## 运行容器

使用：
```
docker run --name learnedkv-run -it --rm \
-v ~/learnedkv-lab/workdir:/work \
learnedkv:ubuntu20.04 \
/bin/bash
```

注意这会关闭之前的容器，没有保存在宿主机的文件会丢失，包括自己安装的软件。

平时要下班或释放资源，推荐`docker stop learnedkv`，让进程优雅退出，容器进入 Exited 状态.
如果容器主进程只有一个shell，也可以输入`exit`来退出。

如果只想重启（保留改动）：

`docker start -ai learnedkv`

删除镜像：用`docker rmi ${image-id}`，id用`docker images`看，取前几位就行

## 补充：vscode远程连接容器

在“已通过SSH连上的远端VS Code”里 Attach 到容器
前提：已经用 VS Code Remote-SSH 连接到了 fpga01。
步骤：

在连接到 fpga01 的 VS Code 窗口里安装扩展：
Dev Containers（ms-vscode-remote.remote-containers）

可选：Docker 扩展（便于查看容器列表）

打开命令面板（Ctrl+Shift+P），选择：
Dev Containers: Attach to Running Container...

选择你在 fpga01 上正在运行的那个容器（名字或ID）。

VS Code 会在 fpga01 上将当前窗口“附加到容器”，你将获得：
容器内文件浏览器、终端、调试等完整体验

扩展会在容器内安装对应的 VS Code Server

若要在“另一个窗口”打开：在命令面板里选择 “Attach to Running Container in New Window”。

## 进入容器（安装LearnedKV）

进入容器后，在 /work 下操作，数据与结果会同步到宿主 ~/learnedkv-lab/workdir。

以下以仓库 https://github.com/yima77/LearnedKV_Artifact 为准，给出容器内建议命令（假定 /work 是挂载目录）：

获取仓库并进入
cd /work
git clone https://github.com/yima77/LearnedKV_Artifact.git
cd LearnedKV_Artifact

RocksDB
你已在镜像里安装 v9.3.1，可跳过仓库内的编译步骤。若脚本里强依赖本地路径的 rocksdb（例如子模块路径），按其 README 调整。
通常系统级 /usr/local/lib 和 /usr/local/include 已满足。

数据集 YCSB-C
/work的内容是宿主机共享的。我把LearnedKV的东西都放work文件夹了，所以相当于宿主机能随便访问，其中的datasets也是

mkdir -p /work/datasets
mkdir datasets 
cd YCSB-C
apt-get update && apt-get install -y libtbb-dev && rm -rf /var/lib/apt/lists/*
make
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib
sh generate_datasets.sh
确认 datasets 目录里有生成数据：ls -lh ../datasets 或仓库要求的默认路径

learned-LSM
cd ../learned-LSM
apt-get update && apt-get install -y libsnappy-dev libbz2-dev liblz4-dev libzstd-dev libleveldb-dev libgmp-dev libeigen3-dev && rm -rf /var/lib/apt/lists/*
pip3 install --no-cache-dir numpy pandas matplotlib seaborn
mkdir -p db

运行脚本：

sh phase_throughput.sh

还有其他测试脚本

具体解释见LearnedKV的learned-LSM文件夹的README.md。

注：这个测试跑挺慢的，比如3M的测试得跑半小时多，把datasets全测完总共可能需要两三个小时。可以只跑1M以内的

关于空间占用，我跑一遍吞吐测试发现db中生成的vlog特别大，有13GB，其他文件倒还好，估计源码没有处理好db的回收。可自行删除db。

## 代码修改

源码中，test.cpp使用的头文件是learnedKV，而非learnedKV_PLR+，虽然根据论文后者应该是优于前者的。
