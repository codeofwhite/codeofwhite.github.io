---
title: "解决 Docker Desktop K8S 启动时间长的问题"
date: 2025-11-07
categories: ["bigdata"]  
tags: ["wsl", "docker desktop", "k8s"]
draft: false
---

Windows 上 k8s 启动卡在 Starting 的一个解决方案。

<!--more-->

# 1 挂个梯子

这个不用多说魔法一下就行

# 2 自己先把 k8s 的镜像拉下来
```bash
# 你的 K8s 版本
$K8S_VERSION="v1.32.4"
# 对应依赖镜像版本（自己查一下自己什么版本）
$ETCD_VERSION="3.5.16-0"
$COREDNS_VERSION="v1.11.1"
$PAUSE_VERSION="3.9"

# 从阿里云拉取镜像
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:$K8S_VERSION
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:$K8S_VERSION
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:$K8S_VERSION
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:$K8S_VERSION
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:$PAUSE_VERSION
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:$ETCD_VERSION
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:$COREDNS_VERSION
```

## 给镜像打默认标签
```bash
# 给镜像打上 K8s 启动时默认查找的标签（registry.k8s.io 前缀）
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:$K8S_VERSION registry.k8s.io/kube-apiserver:$K8S_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:$K8S_VERSION registry.k8s.io/kube-controller-manager:$K8S_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:$K8S_VERSION registry.k8s.io/kube-scheduler:$K8S_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:$K8S_VERSION registry.k8s.io/kube-proxy:$K8S_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:$PAUSE_VERSION registry.k8s.io/pause:$PAUSE_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:$ETCD_VERSION registry.k8s.io/etcd:$ETCD_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:$COREDNS_VERSION registry.k8s.io/coredns/coredns:$COREDNS_VERSION
```

## 最后一步
1. 重启 docker desktop
2. 启动 k8s
3. 大功告成
