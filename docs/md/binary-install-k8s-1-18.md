# kubernetes 1.18 二进制安装(单master)

## 一. OS和软件信息

- 操作系统: centos7.8 64 bit 最小化安装
- Docker: 19-ce
- kubernets: 1.18
- etcd: 3.4

## 二. 服务器规划

|  角色   | IP  | 组件 |
|  ----  | ----  | ---- |
| k8s-master1  | 10.0.2.26 | kube-apiserver,kube-controller-manager, kube-scheduler, etcd |
| k8s-node1  | 10.0.2.27 | kubelet, kube-proxy, docker, etcd |
| k8s-node2  | 10.0.2.28 | kubelet, kube-proxy, docker, etcd |

## 三. 操作系统初始化配置

!> hosts文件请根据自己实际情况修改

```bash
# 安装基础软件
yum install epel-release -y
yum install wget jq vim -y

# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭selinux
sed -i 's/enforcing/disabled' /etc/selinux/config  # 永久
setenforce 0 # 临时

# 关闭swap
sed -ri 's/.*swap.*/#&/' /etc/fstab  # 永久
swapoff -a # 临时

# 在master上添加hosts
cat >> /etc/hosts << EOF
10.0.2.26 k8s-master
10.0.2.27 k8s-node1
10.0.2.28 k8s-node2
EOF

# 将桥接的IPv4流量传递到iptables的链
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system # 生效

# 时间同步
yum install ntpdate -y
ntpdate time.windows.com
```

## 四. 部署etcd集群

Etcd是一个分布式键值存储系统，kubernetes使用Etcd进行数据存储，所以先准备一个Ectd数据库，为解决Etcd单点故障，应采用集群方式部署，这里使用3台组建集群，可容忍1台机器故障，当然你也可以使用5台组建集群，可容忍2台机器故障

!> 注： 为了节省机器，这里与k8s节点机器复用.也可以独立于k8s集群之外部署，只要apiserver能连接到就行。

| 节点名称 | IP |
| ---- | ---- |
| etcd-1 | 10.0.2.26 |
| etcd-2 | 10.0.2.27 |
| etcd-3 | 10.0.2.28 |

### 4-1. 准备cfssl证书生成工具
cfssl 是一个开源证书管理工具，使用json文件生成证书，相比openssl更方便使用.<br>
找任意一台服务器操作，这里使用Master节点。

```
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64
mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo
```

### 4-2. 生成Etcd证书

#### 4-2-1. 自签证书颁发机构（CA）

创建工作目录

```bash
mkdir -p ~/TLS/{etcd,k8s}
cd ~/TLS/etcd
```
自签CA:

```bash
cat > ca-config.json << EOF
{
    "signing":{
        "default":{
            "expiry":"87600h"
        },
        "profiles":{
            "www":{
                "expiry":"87600h",
                "usages":[
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
EOF

cat > ca-csr.json << EOF
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing"
        }
     ]
}
EOF
```

生成证书：
```bash
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -

ls *pem
ca-key.pem ca.pem
```

#### 4-2-2. 使用自签证书CA颁发Etcd HTTPS证书

创建证书申请文件：

!> 注： hosts字段中IP为所有Etcd节点的集群内部通信IP，一个都不能少！ 为了后期扩容方便，可以多写几个预留IP
```bash
cat > server-csr.json << EOF
{
    "CN": "etcd",
    "hosts": [
        "10.0.2.26",
        "10.0.2.27",
        "10.0.2.28"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing"
        }
    ]
}
EOF
```

生成证书：

```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server

ls  server*pem
server-key.pem  server.pem
```

### 4-3. 从Github下载二进制文件

下载地址：

```
cd ~
wget https://github.com/etcd-io/etcd/releases/download/v3.4.13/etcd-v3.4.13-linux-amd64.tar.gz
```

### 4-4. 部署Etcd集群

以下在节点1上操作，为简化操作，将节点1生成的文件拷贝到节点2和节点3

#### 4-4-1. 创建工作目录并解压二进制包

```bash
mkdir /opt/etcd/{bin,cfg,ssl} -p
tar -zxvf etcd-v3.4.13-linux-amd64.tar.gz
mv etcd-v3.4.13-linux-amd64/{etcd,etcdctl} /opt/etcd/bin/
```

#### 4-4-2. 创建etcd配置文件
```bash
# etcd-1
cat > /opt/etcd/cfg/etcd.conf << EOF
#[Member]
ETCD_NAME="etcd-1"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://10.0.2.26:2380"
ETCD_LISTEN_CLIENT_URLS="https://10.0.2.26:2379"
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.0.2.26:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://10.0.2.26:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://10.0.2.26:2380,etcd-2=https://10.0.2.27:2380,etcd-3=https://10.0.2.28:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF
```
```bash
# etcd-2
cat > /opt/etcd/cfg/etcd.conf << EOF
#[Member]
ETCD_NAME="etcd-2"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://10.0.2.27:2380"
ETCD_LISTEN_CLIENT_URLS="https://10.0.2.27:2379"
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.0.2.27:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://10.0.2.27:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://10.0.2.26:2380,etcd-2=https://10.0.2.27:2380,etcd-3=https://10.0.2.28:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF

```

```bash
# etcd-3
cat > /opt/etcd/cfg/etcd.conf << EOF
#[Member]
ETCD_NAME="etcd-3"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://10.0.2.28:2380"
ETCD_LISTEN_CLIENT_URLS="https://10.0.2.27:2379"
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.0.2.28:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://10.0.2.28:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://10.0.2.26:2380,etcd-2=https://10.0.2.27:2380,etcd-3=https://10.0.2.28:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF

```

#### 4-4-3. systemd管理etcd

!> 各节点都执行

```bash
cat > /usr/lib/systemd/system/etcd.service << EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
[Service]
Type=notify
EnvironmentFile=/opt/etcd/cfg/etcd.conf
ExecStart=/opt/etcd/bin/etcd \
--cert-file=/opt/etcd/ssl/server.pem \
--key-file=/opt/etcd/ssl/server-key.pem \
--peer-cert-file=/opt/etcd/ssl/server.pem \
--peer-key-file=/opt/etcd/ssl/server-key.pem \
--trusted-ca-file=/opt/etcd/ssl/ca.pem \
--peer-trusted-ca-file=/opt/etcd/ssl/ca.pem \
--logger=zap
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF
```

#### 4-4-4. 拷贝刚才生成的证书到各节点

```bash
cp ~/TLS/etcd/ca*pem ~/TLS/etcd/server*pem /opt/etcd/ssl/

scp ~/TLS/etcd/ca*pem ~/TLS/etcd/server*pem 10.0.2.27:/opt/etcd/ssl/

scp ~/TLS/etcd/ca*pem ~/TLS/etcd/server*pem 10.0.2.28:/opt/etcd/ssl/

```

#### 4-4-5. 启动etcd并设置开机启动

```bash
systemctl daemon-reload
systemctl start etcd
systemctl enable etcd
```

#### 4-4-6. 查看集群状态

```bash
ETCDCTL_API=3 /opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem \
--key=/opt/etcd/ssl/server-key.pem --endpoints="https://10.0.2.26:2379,https://10.0.2.27:2379,https://10.0.2.28:2379" \
endpoint health
```

## 五. 安装Docker

!> 每个节点都需要安装

### 5-1. 下载安装包
```bash
cd ~/
wget https://download.docker.com/linux/static/stable/x86_64/docker-19.03.13.tgz
```
在所有节点安装.这里采用的是二进制方式安装

### 5-1. 解压二进制包

```bash
tar -zxvf docker-19.03.13.tgz
mv docker/* /usr/bin/
```

### 5-2. systemd管理docker

```bash
cat > /usr/lib/systemd/system/docker.service << EOF
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target
[Service]
Type=notify
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
# Uncomment TasksMax if your systemd version supports it.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
LimitSTACK=104857600
LimitSIGPENDING=600000
# Only systemd 226 and above support this version.
#TasksMax=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
# kill only the docker process, not all processes in the cgroup
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s
[Install]
WantedBy=multi-user.target
EOF
```

### 5-3. 创建Docker配置文件

```bash
mkdir /etc/docker
cat >/etc/docker/daemon.json << EOF
{
  "registry-mirrors": [
    "https://registry.docker-cn.com",
    "http://hub-mirror.c.163.com",
    "https://docker.mirrors.ustc.edu.cn"
  ]
}
EOF
```
- registry-mirrors配置国内仓库

### 5-4. 启动并设置开机启动

```bash
systemctl daemon-reload
systemctl start docker
systemctl enable docker
```

## 6. 部署K8s Master Node

### 6-1. 生成kube-apiserver证书

#### 6-1-1. 自签证书颁发机构（CA）

!> k8s master上操作。本示例 10.0.2.26

```bash
cd ~/TLS/k8s/
cat > ca-config.json << EOF
{
    "signing":{
        "default":{
            "expiry":"87600h"
        },
        "profiles":{
            "kubernetes":{
                "expiry":"87600h",
                "usages":[
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
EOF
cat > ca-csr.json << EOF
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing",
            "O": "k8s",
            "OU": "System"
        }
     ]
}
EOF
```

生成证书：

```bash
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -

ls *pem
ca-key.pem ca.pem
```

#### 6-1-2. 使用自签CA签发kube-apiserver HTTPS证书

!> hosts 字段中IP为所有master/LB/VIP IP,一个都不能少！ 为了方便以后扩容，可以多写几个预留的IP。

创建证书申请文件：
```bash
cat > server-csr.json << EOF
{
    "CN": "kubernetes",
    "hosts": [
        "127.0.0.1",
        "10.0.2.26",
        "10.0.2.27",
        "10.0.2.28",
        "kubernetes",
        "kubernetes.default",
        "kubernetes.default.svc",
        "kubernetes.default.svc.cluster",
        "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

生成证书：
```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server

ls  server*pem
server-key.pem  server.pem

```

### 6.2 从Github下载二进制文件