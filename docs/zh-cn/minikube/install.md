# minikube

## 一、参考资料

!>官方资料是进步的最佳实践

- [minikube安装文档:](https://kubernetes.io/zh/docs/tasks/tools/install-minikube/) 
    - https://kubernetes.io/zh/docs/tasks/tools/install-minikube/
- [minikube-github:](https://github.com/kubernetes/minikube)
    - https://github.com/kubernetes/minikube

## 二、ubuntu20.04 安装kubectl

```bash
bourne@vm-10-0-2-100:~$ screenfetch 
                          ./+o+-       bourne@vm-10-0-2-100
                  yyyyy- -yyyyyy+      OS: Ubuntu 20.04 focal
               ://+//////-yyyyyyo      Kernel: x86_64 Linux 5.4.0-48-generic
           .++ .:/++++++/-.+sss/`      Uptime: 59m
         .:++o:  /++++++++/:--:/-      Packages: 631
        o:+o+:++.`..```.-/oo+++++/     Shell: bash 5.0.17
       .:+o:+o/.          `+sssoo+/    Disk: 5.5G / 12G (52%)
  .++/+:+oo+o:`             /sssooo.   CPU: Intel Core i7-8565U @ 1.992GHz
 /+++//+:`oo+o               /::--:.   GPU: VMware SVGA II Adapter
 \+/+o+++`o++o               ++////.   RAM: 414MiB / 1987MiB
  .++.o+++oo+:`             /dddhhh.  
       .+.o+oo:.          `oddhhhh+   
        \+.++o+o``-````.:ohdhhhhh+    
         `:o+++ `ohhhhhhhhyo++os:     
           .o:`.syhhhhhhh/.oo++o`     
               /osyyyyyyo++ooo+++/    
                   ````` +oo+++o\:    
                          `oo++. 
```

### 2.1 下载最新版本的kubectl
```bash
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
```

### 2.2 下载特定版本的kubectl(Optional)
```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.19.0/bin/linux/amd64/kubectl
```

### 2.3 标记 kubectl 文件为可执行
```bash
chmod +x ./kubectl
```

### 2.4 将文件放到 PATH 路径下
```bash
sudo mv ./kubectl /usr/local/bin/kubectl
```

### 2.5 测试你所安装的版本是最新的
```bash
kubectl version --client
```

## 三、ubuntu20.04 安装minikube

!> 前置条件:安装了kubectl

### 3.1 二进制包安装minikube

```bash
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \
  && chmod +x minikube
```

### 3.2 将minikube添加到PATH
```bash
sudo mv minikube /usr/local/bin/
```

### 3.3 确认安装

- --vm-driver类型 https://kubernetes.io/zh/docs/setup/learning-environment/minikube/#specifying-the-vm-driver

?> 

```bash
minikube start --vm-driver=virtualbox
# 或者在需要时
minikube start --vm-driver=virtualbox --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers
```
