# minikube


## 零. 官方文档

- [minikube安装文档:](https://kubernetes.io/zh/docs/tasks/tools/install-minikube/) 
    - https://kubernetes.io/zh/docs/tasks/tools/install-minikube/
- [minikube-github:](https://github.com/kubernetes/minikube)
    - https://github.com/kubernetes/minikube

## 一. 环境信息

- ubuntu20.04 最小化安装

- OS信息
```bash
~$ screenfetch 
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

## 二. 安装kubectl

?>在安装minikube之前，要确保kubectl已经安装

- 下载最新版本的kubectl
```bash
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
```

- 下载特定版本的kubectl(Optional)
```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.19.0/bin/linux/amd64/kubectl
```

- 标记 kubectl 文件为可执行

```bash
chmod +x ./kubectl
```

- 将文件放到 PATH 路径下

```bash
sudo mv ./kubectl /usr/local/bin/kubectl
```
- 查看安装的版本

```bash
kubectl version --client
```

## 三. 安装minikube

!> 前置条件:安装了kubectl

- 下载minikube并设置可执行权限

```bash
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \
  && chmod +x minikube
```

- 将minikube添加到PATH
```bash
sudo mv minikube /usr/local/bin/
```

- 确认安装

!>使用`--image-repository`指定minikube使用国内repo

```bash
minikube start --vm-driver=none --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers
```


- 启动minikube

```bash
~$ sudo minikube start --vm-driver=none --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers
😄  minikube v1.13.1 on Ubuntu 20.04 (vbox/amd64)
✨  Using the none driver based on existing profile

🧯  The requested memory allocation of 1987MiB does not leave room for system overhead (total system memory: 1987MiB). You may face stability issues.
💡  Suggestion: Start minikube with less memory allocated: 'minikube start --memory=1987mb'

👍  Starting control plane node minikube in cluster minikube
🔄  Restarting existing none bare metal machine for "minikube" ...
ℹ️  OS release is Ubuntu 20.04.1 LTS
🐳  Preparing Kubernetes v1.19.2 on Docker 19.03.13 ...
    ▪ kubelet.resolv-conf=/run/systemd/resolve/resolv.conf
    > kubectl.sha256: 65 B / 65 B [--------------------------] 100.00% ? p/s 0s
    > kubelet.sha256: 65 B / 65 B [--------------------------] 100.00% ? p/s 0s
    > kubeadm.sha256: 65 B / 65 B [--------------------------] 100.00% ? p/s 0s
    > kubectl: 41.01 MiB / 41.01 MiB [---------------] 100.00% 2.84 MiB p/s 15s
    > kubeadm: 37.30 MiB / 37.30 MiB [---------------] 100.00% 2.63 MiB p/s 15s
    > kubelet: 104.88 MiB / 104.88 MiB [-------------] 100.00% 4.14 MiB p/s 25s



🤹  Configuring local host environment ...

❗  The 'none' driver is designed for experts who need to integrate with an existing VM
💡  Most users should use the newer 'docker' driver instead, which does not require root!
📘  For more information, see: https://minikube.sigs.k8s.io/docs/reference/drivers/none/

❗  kubectl and minikube configuration will be stored in /root
❗  To use kubectl or minikube commands as your own user, you may need to relocate them. For example, to overwrite your own settings, run:

    ▪ sudo mv /root/.kube /root/.minikube $HOME
    ▪ sudo chown -R $USER $HOME/.kube $HOME/.minikube

💡  This can also be done automatically by setting the env var CHANGE_MINIKUBE_NONE_USER=true
🔎  Verifying Kubernetes components...
🌟  Enabled addons: default-storageclass, storage-provisioner
🏄  Done! kubectl is now configured to use "minikube" by default

```
