---
layout: ubuntu
title: ubuntu 24.04 安装K8s.md
date: 2024-07-06 23:46:04
categories: devops
tags: [k8s,运维]
---

# Ubuntu 24.04  LTS 安装K8s

设备信息

2c4g vps *3 【均为国外vps】

系统版本 ubuntu 24.04 lts



### (一)初始化系统环境变量

1. 安装vim

````shell
apt-get update 
apt-get install apt-transport-https ca-certificates curl gpg
apt-get install vim -y
````

2. 开启转发

```shell
tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

```

3. 内没模块加载配置

```shell
tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

```

配置完后使用 sysctl --system 使上述配置生效

4. 防火墙开启端口

```shell
【Master 开启防火墙端口】
ufw allow 6443
ufw allow 2379
ufw allow 2380
ufw allow 10250
ufw allow 16443

【若使用flannel网络插件，则额外开放此端口】
ufw allow 8285

```



### (二)安装docker

1. 配置docker源并安装

```shell
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
 
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

apt-get update
apt-get install containerd.io
```

2. 修改containerd 配置文件

```shell
mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

[plugins."io.containerd.grpc.v1.cri".containerd]
  snapshotter = "overlayfs"
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
  
# 24.04仅需要修改SystemdCgroup =true 即可，其他默认
```

3. 启动容器

```shell
systemctl restart containerd
systemctl enable containerd
systemctl status containerd
```

### (三)安装kubeadm、kubelet、kubectl

1. 配置源并安装

```shell
mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

apt-get update
apt-get install kubelet kubeadm kubectl -Y

systemctl enable --now kubelet
```

2.  master 节点生成连接信息

```shell
kubeadm init --apiserver-advertise-address=AAA.BBB.CCC.DDD --service-cidr=10.92.0.0/12 --pod-network-cidr=172.16.0.0/16 --ignore-preflight-errors=all

```

3. slave 根据master 节点生成的信息join至主节点

```shell
kubeadm AAA.BBB.CCC.DDD:6443 --token tj03rj.azwccu8al36fd2vw --discovery-token-ca-cert-hash sha256:xx

```

4. 验证节点状态

```shell
# 验证子节点状态
kubectl get nodes
```

若出现 NotReady

则安装网络插件即可

```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
