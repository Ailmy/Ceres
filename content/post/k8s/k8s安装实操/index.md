---
title: "K8s安装实操"
description: 
date: 2025-01-21T11:05:50+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
categories:
  - k8s
tags:
    - 学习笔记
---

## 1.网络配置
ip设置

## 2.系统配置
### 2.1关闭防火墙
```shell
systemctl stop firewalld
systemctl disable firewalld

```
### 2.2关闭selinux
```shell
setenforce 0
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
grubby --update-kernel ALL --args selinux=0
# 查看是否禁用，grubby --info DEFAULT
# 回滚内核层禁用操作，grubby --update-kernel ALL --remove-args selinux
```

### 2.3关闭swap
```shell
swapoff -a
sed -i 's:/dev/mapper/rl-swap:#/dev/mapper/rl-swap:g' /etc/fstab
```

### 2.4 设置时区
```shell
timedatectl set-timezone Asia/Shanghai
```


## 3 安装前准备
### 3.1 安装iptables
```shell
yum -y install iptables-services
systemctl start iptables
iptables -F
systemctl enable iptables
service iptables save
```

### 3.2 安装ipvs
```shell
yum -y install ipvsadm
```

### 3.3 开启路由转发
```shell
echo 'net.ipv4.ip_forward=1' >> /etc/sysctl.conf
sysctl -p
```

### 3.4 加载bridge
```shell
yum install -y epel-release
yum install -y bridge-utils

modprobe br_netfilter
echo 'br_netfilter' >> /etc/modules-load.d/bridge.conf
echo 'net.bridge.bridge-nf-call-iptables=1' >> /etc/sysctl.conf
echo 'net.bridge.bridge-nf-call-ip6tables=1' >> /etc/sysctl.conf
sysctl -p
```
### 3.5 安装docker
```shell
sudo dnf config-manager --add-repo=https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 安装 docker-ce
yum -y install docker-ce

# 配置 daemon.
cat > /etc/docker/daemon.json <<EOF
{
  "data-root": "/data/docker",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "100"
  },
  "insecure-registries": ["harbor.xinxainghf.com"],
  "registry-mirrors": ["https://kfp63jaj.mirror.aliyuncs.com"]
}
EOF

systemctl daemon-reload && systemctl restart docker && systemctl enable docker
```

可以发送到其他主机
```shell
scp 路径 root@主机名:路径 
```

发送文件夹
```shell
scp -r 路径 root@主机名:路径 
```

### 3.6 安装cri-docker

#### 3.6.1下载cri-docker去官网看文档
```shell

wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.9/cri-dockerd-0.3.9.amd64.tgz

tar -xf cri-dockerd-0.3.9.amd64.tgz
cp cri-dockerd/cri-dockerd /usr/bin/
chmod +x /usr/bin/cri-dockerd
```

#### 3.6.2配置 cri-docker 服务
```shell
cat <<"EOF" > /usr/lib/systemd/system/cri-docker.service
[Unit]
Description=CRI Interface for Docker Application Container Engine
Documentation=https://docs.mirantis.com
After=network-online.target firewalld.service docker.service
Wants=network-online.target
Requires=cri-docker.socket
[Service]
Type=notify
ExecStart=/usr/bin/cri-dockerd --network-plugin=cni --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.8
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always
StartLimitBurst=3
StartLimitInterval=60s
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
Delegate=yes
KillMode=process
[Install]
WantedBy=multi-user.target
EOF
```

> `ExecStart=/usr/bin/cri-dockerd` 为 `cri-dockerd` 的二进制文件路径，`--network-plugin=cni` 为指定网络插件为 `cni`，`--pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.8` 为指定 `pause` 容器的镜像地址。

### 3.7添加 cri-docker 套接字
```shell
cat <<"EOF" > /usr/lib/systemd/system/cri-docker.socket
[Unit]
Description=CRI Docker Socket for the API
PartOf=cri-docker.service
[Socket]
ListenStream=%t/cri-dockerd.sock
SocketMode=0660
SocketUser=root
SocketGroup=docker
[Install]
WantedBy=sockets.target
EOF
```

### 3.8启动cri-docker 对接服务
```shell
systemctl daemon-reload
systemctl enable cri-docker
systemctl start cri-docker
systemctl is-active cri-docker
```

## 4.安装k8s

### 4.1添加 kubeadm yum 源
```shell
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```

### 4.2安装 kubeadm 1.29 版本
```shell
yum install -y kubelet-1.29.0 kubectl-1.29.0 kubeadm-1.29.0
systemctl enable kubelet.service
```

### 4.3初始化 master 节点
```shell
kubeadm init --apiserver-advertise-address=192.168.66.11 --image-repository registry.aliyuncs.com/google_containers --kubernetes-version 1.29.2 --service-cidr=10.10.0.0/12 --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=all --cri-socket unix:///var/run/cri-dockerd.sock
```

> `--apiserver-advertise-address` 为指定 `apiserver` 的地址，`--image-repository` 为指定镜像仓库地址，`--kubernetes-version` 为指定 `kubernetes` 版本，`--service-cidr` 为指定 `service` 网段，`--pod-network-cidr` 为指定 `pod` 网段，`--ignore-preflight-errors=all` 为忽略所有检查，`--cri-socket` 为指定 `cri-dockerd` 的套接字地址，如果是使用docker这条命令必须加

 work token 过期后，重新申请
```shell
kubeadm token create --print-join-command
```

### worker 节点加入集群
```shell
# worker 加入
kubeadm join 192.168.10.11:6443 --token a6xh07.yg9wh2vru2grluwb         --discovery-token-ca-cert-hash sha256:7cd8499abae48c8403800152cc0f655ac704ea00ae30a549acd9bbac7b26dca4 --cri-socket unix:///var/run/cri-dockerd.sock
```

## 5.安装网络插件

https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises#install-calico-with-kubernetes-api-datastore-more-than-50-nodes

curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.3/manifests/calico-typha.yaml -o calico.yaml

CALICO_IPV4POOL_CIDR	指定为 pod 地址

 修改为 BGP 模式

 Enable IPIP

\- name: CALICO_IPV4POOL_IPIP

  value: "Always"  #改成Off
  
### 固定网卡(可选)
```shell
# 目标 IP 或域名可达
        - name: calico-node
          image: registry.geoway.com/calico/node:v3.19.1
          env:
            # Auto-detect the BGP IP address.
            - name: IP
              value: "autodetect"
            - name: IP_AUTODETECTION_METHOD
              value: "can-reach=www.google.com"
kubectl set env daemonset/calico-node -n kube-system IP_AUTODETECTION_METHOD=can-reach=www.google.com
# 匹配目标网卡
- name: calico-node
  image: registry.geoway.com/calico/node:v3.19.1
  env:
    # Auto-detect the BGP IP address.
    - name: IP
      value: "autodetect"
    - name: IP_AUTODETECTION_METHOD
      value: "interface=eth.*"
# 排除匹配网卡
- name: calico-node
  image: registry.geoway.com/calico/node:v3.19.1
  env:
    # Auto-detect the BGP IP address.
    - name: IP
      value: "autodetect"
    - name: IP_AUTODETECTION_METHOD
      value: "skip-interface=eth.*"
# CIDR
- name: calico-node
  image: registry.geoway.com/calico/node:v3.19.1
  env:
    # Auto-detect the BGP IP address.
    - name: IP
      value: "autodetect"
    - name: IP_AUTODETECTION_METHOD
      value: "cidr=192.168.200.0/24,172.15.0.0/24"
```

### 修改kube-proxy 模式为 ipvs(可选)
```shell
# kubectl edit configmap kube-proxy -n kube-system
mode: ipvs

kubectl delete pod -n kube-system -l k8s-app=kube-proxy
```

#### 额外补充
k8s init过程的命令解释
```shell
#使用的k8s版本
[init] Using Kubernetes version: v1.29.2
#运行预检查
[preflight] Running pre-flight checks
#拉取k8s集群所需的镜像
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
#也可以使用`kubeadm config images pull`提前拉取
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
#检测到容器运行时的沙箱镜像与kubeadm使用的不一致，建议使用`registry.aliyuncs.com/google_containers/pause:3.9`作为CRI沙箱镜像
W0123 16:27:44.874553    4304 checks.go:835] detected that the sandbox image "registry.aliyuncs.com/google_containers/pause:3.8" of the container runtime is inconsistent with that used by kubeadm. It is recommended that using "registry.aliyuncs.com/google_containers/pause:3.9" as the CRI sandbox image.
#使用证书目录
[certs] Using certificateDir folder "/etc/kubernetes/pki"
#证书创建
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8sm1 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.0.0.1 192.168.17.11]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [k8sm1 localhost] and IPs [192.168.17.11 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [k8sm1 localhost] and IPs [192.168.17.11 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key

#创建静态Pod清单
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "super-admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 5.503428 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node k8sm1 as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node k8sm1 as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: r1vfl2.8j9dn6xj9dhcb54a
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.17.11:6443 --token r1vfl2.8j9dn6xj9dhcb54a \
        --discovery-token-ca-cert-hash sha256:56e8a0a7338e0e7fedf92c4f970c265103925f476da437bddc2bf23962be57a8 
```
