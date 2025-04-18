---
layout: post
title: K8S集群创建
date: 2023-08-15
tags: ["caclio","k8s","容器"]
---

# 1.准备工作

<table>
<thead>
<tr>
<th>ip</th>
<th>节点</th>
</tr>
</thead>
<tbody>
<tr>
<td>192.168.44.137</td>
<td>master</td>
</tr>
<tr>
<td>192.168.44.136</td>
<td>slave-1</td>
</tr>
<tr>
<td>192.168.44.134</td>
<td>slave-2</td>
</tr>
</tbody>
</table>
<!--more-->

系统我使用的是centos 7，三台机器，安装前需要升级下内核，不然会出现一些莫名其妙的问题，内核升级步骤：

    #导入 elrepo 仓库
    rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
    yum install https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm
    #查看可用内核
    yum --disablerepo="*" --enablerepo="elrepo-kernel" list available
    #安装最新内核
    yum --enablerepo=elrepo-kernel install kernel-ml kernel-ml-devel kernel-ml-tools
    #查看系统可用内核
    awk -F\' '$1=="menuentry " {print i++ " : " $2}' /boot/grub2/grub.cfg
    #查看内核启动顺序
    grub2-editenv list
    #修改内核启动项
    grub2-set-default 0
    grub2-mkconfig -o /boot/grub2/grub.cfg
    #重启
    reboot

接着做下关防火墙，时间同步等操作

    systemctl stop firewalld
    yum update
    swapoff -a 

    yum install -y chrony;
    systemctl start chronyd;
    systemctl enable chronyd

将桥接的ipv4流量传递到iptables链

    modprobe br_netfilter   ##生成bridge相关内核参数
    cat > /etc/sysctl.d/k8s.conf << EOF
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    net.ipv4.ip_forward = 1
    EOF
    sysctl --system # 生效

这些做完就可以开始正式搭建了。

# 2.搭建步骤

## 2.1 安装containerd

    yum install -y yum-utils  
    yum-config-manager \
        --add-repo \
        https://download.docker.com/linux/centos/docker-ce.repo
    # 安装containerd
    yum install containerd.io -y
    # 启动服务
    systemctl enable containerd
    systemctl start containerd
    # 生成默认配置
    containerd  config default > /etc/containerd/config.toml
    # 修改配置
    vi  /etc/containerd/config.toml
    sandbox_image = "registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.8"   # 修改为阿里云镜像地址
    SystemdCgroup = true    
    # 重启containerd服务
    systemctl restart containerd

## 2.2 配置k8s仓库

    cat <<EOF > /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
    EOF

## 2.3 安装kubeadm和kubelet

    yum install -y kubelet-1.25.4 kubeadm-1.25.4 kubectl-1.25.4
    systemctl start kubelet.service
    systemctl enable kubelet.service
    #设置crictl连接 containerd
    crictl config --set runtime-endpoint=unix:///run/containerd/containerd.sock

## 2.4 初始化

初始化需要在master节点上做

    kubeadm init --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers --apiserver-advertise-address=192.168.44.137 --kubernetes-version=v1.25.4  --service-cidr=10.15.0.0/16  --pod-network-cidr=10.18.0.0/16

如果创建成功会输出这样

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

    kubeadm join 192.168.44.137:6443 --token u529o4.invnj3s6anxekg79 \
            --discovery-token-ca-cert-hash sha256:27b967c444cf3f4a45fedae24ed886663a1dc2cd6ceae03930fcbda491ec5ece

    # 上面这条命令就是如果需要将node节点加入到集群需要执行的命令，这个token有效期为24小时，如果过期，可以使用下面命令获取
    # kubeadm token create --print-join-command

接着配置下kubectl

      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config

## 2.5 加入节点

也就是在其他两个节点上执行下这段命令

    kubeadm join 192.168.44.137:6443 --token u529o4.invnj3s6anxekg79 \
            --discovery-token-ca-cert-hash sha256:27b967c444cf3f4a45fedae24ed886663a1dc2cd6ceae03930fcbda491ec5ece

然后再master上把更改kubectl的配置拷贝到salve节点上,让slave节点也能用kubectl命令

    scp -r /root/.kube root@192.168.44.136:/root/.kube 

到这一步已经能够看到节点信息了

    kubectl get nodes

    NAME             STATUS   ROLES           AGE    VERSION
    192.168.44.134   Ready    <none>          73m    v1.25.4
    192.168.44.136   Ready    <none>          97m    v1.25.4
    192.168.44.137   Ready    control-plane   103m   v1.25.4

## 2.6 安装calico

    # 下载calico
    curl https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml -O
    # 修改Pod ⽹络

    vim calico.yaml
    # - name: CALICO_IPV4POOL_CIDR
    # value: "192.168.0.0/16"
    # 修改为：
    - name: CALICO_IPV4POOL_CIDR
      value: "10.18.0.0/16"

    # 部署
    kubectl apply -f calico.yaml

    # kubectl get pods -n kube-system

    NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
    kube-system   calico-kube-controllers-74677b4c5f-gjhm9   1/1     Running   0          45m
    kube-system   calico-node-4cwbb                          1/1     Running   0          35m
    kube-system   calico-node-mwk7q                          1/1     Running   0          45m
    kube-system   calico-node-vhzhz                          1/1     Running   0          45m
    kube-system   coredns-7f8cbcb969-z29fg                   1/1     Running   0          79m
    kube-system   coredns-7f8cbcb969-zqm7f                   1/1     Running   0          79m
    kube-system   etcd-192.168.44.137                        1/1     Running   0          80m
    kube-system   kube-apiserver-192.168.44.137              1/1     Running   0          80m
    kube-system   kube-controller-manager-192.168.44.137     1/1     Running   0          80m
    kube-system   kube-proxy-648wr                           1/1     Running   0          74m
    kube-system   kube-proxy-sb2b6                           1/1     Running   0          5m35s
    kube-system   kube-proxy-x52pl                           1/1     Running   0          79m
    kube-system   kube-scheduler-192.168.44.137              1/1     Running   0          80m

如果出现有节点没跑起来,排查方法,一个是`kuectl describe pod xxx -n kube-system` 关注下最下面的events,还有个是`kubectl logs xxx` 根据这两个的信息判断到底是什么问题,我遇到的问题以下两个:

*   创建calico一直不成功,报/sys/fs/bpf目录创建不成功,原因是内核不支持,升级了下内核ok
*   salve-2上kube-proxy和calico一直没起来,原因是拉不到镜像,修改了下crictl的仓库地址,ok解决

    vim /etc/containerd/config.toml
    [plugins."io.containerd.grpc.v1.cri".registry]
          [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
             #默认仓库为docker.io
            [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
                ##修改为国内镜像
              endpoint = ["https://registry-1.docker.io"]