# K8S-Deploy


部署 Kubernetes
=============

阿里云有提供一套可用的 k8s，如果自己部署的话就不需要再依赖于阿里云相关文档中所要求的环境。

准备工作
----

1.  运行 deb/rpm 兼容操作系统的一台或多台机器，例如 Ubuntu 或 CentOS
2.  每台机器 2GB 或更多内存
3.  master 上的 CPU 要求 2 核或以上
4.  集群中所有机器（公用或专用网络都可以）的网络都是可连通的

安装 kubeadm
----------

### 准备工作

*   操作系统为 Ubuntu 16.04+
    
*   禁用 Swap。为了让 kubelet 正常工作你必须禁用 swap。
    

#### 安装 Docker

在你的每一台机器上安装 Docker。建议使用的版本号为 17.03。而 1.11， 1.12 和 1.13 是可以正常使用的。17.06+ 可以用，但是 Kubernetes 团队还没有测试和验证。

如果你已经安装了要求的 Docker 版本，你可以进入下一部分了。如果没有，请用下面的命令安装 Docker。

    apt-get update
    apt-get install -y docker.io
    

或者安装 Docker CE 17.03

    apt-get update
    apt-get install -y apt-transport-https ca-certificates curl software-properties-common
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
    add-apt-repository "deb https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") $(lsb_release -cs) stable"
    apt-get update && apt-get install -y docker-ce=$(apt-cache madison docker-ce | grep 17.03 | head -1 | awk '{print $3}')
    

这里是 [Docker 官方的安装指南](https://docs.docker.com/engine/installation/)

### 安装 kubeadm, kubectl, kubelet

    版本号：1.10.3-00
    

*   kubeadm: 引导启动 k8s 集群的命令工具。
*   kubelet: 在群集中的所有计算机上运行的组件, 并用来执行如启动 pods 和 containers 等操作。
*   kubectl: 用于操作运行中的集群的命令行工具。

    sudo apt-get update && sudo apt-get install -y apt-transport-https
    curl -s http://packages.faasx.com/google/apt/doc/apt-key.gpg | sudo apt-key add -
    sudo cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
    deb http://mirrors.ustc.edu.cn/kubernetes/apt/ kubernetes-xenial main
    EOF
    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    

> apt-key 官方地址为：[https://packages.cloud.google.com/apt/doc/apt-key.gpg。](https://packages.cloud.google.com/apt/doc/apt-key.gpg。)
> 
> apt安装包地址使用了中科大的镜像，官方地址为：[http://apt.kubernetes.io/。](http://apt.kubernetes.io/。)

从该链接 [https://raw.githubusercontent.com/EagleChen/kubernetes\_init/master/kube\_apt\_key.gpg](https://raw.githubusercontent.com/EagleChen/kubernetes_init/master/kube_apt_key.gpg) 下载 kube\_apt\_key.gpg 到当前工作目录下，并按如下指令添加

`$ cat kube_apt_key.gpg | sudo apt-key add -`

显示 OK 即表示成功

### 下载 k8s 镜像

    docker pull anjia0532/kube-proxy-amd64:v1.10.3 \
    && docker pull anjia0532/kube-controller-manager-amd64:v1.10.3 \
    && docker pull anjia0532/kube-scheduler-amd64:v1.10.3 \
    && docker pull anjia0532/kube-apiserver-amd64:v1.10.3 \
    && docker pull anjia0532/k8s-dns-dnsmasq-nanny-amd64:1.14.8 \
    && docker pull anjia0532/k8s-dns-sidecar-amd64:1.14.8 \
    && docker pull anjia0532/k8s-dns-kube-dns-amd64:1.14.8 \
    && docker pull anjia0532/pause-amd64:3.1 \
    && docker pull quay.io/coreos/flannel:v0.10.0-amd64
    

### 修改 image tag

    docker tag anjia0532/kube-proxy-amd64:v1.10.3 k8s.gcr.io/kube-proxy-amd64:v1.10.3 \
    && docker tag anjia0532/kube-controller-manager-amd64:v1.10.3 k8s.gcr.io/kube-controller-manager-amd64:v1.10.3 \
    && docker tag anjia0532/kube-scheduler-amd64:v1.10.3 k8s.gcr.io/kube-scheduler-amd64:v1.10.3 \
    && docker tag anjia0532/kube-apiserver-amd64:v1.10.3 k8s.gcr.io/kube-apiserver-amd64:v1.10.3 \
    && docker tag anjia0532/k8s-dns-dnsmasq-nanny-amd64:1.14.8 k8s.gcr.io/k8s-dns-dnsmasq-nanny-amd64:1.14.8 \
    && docker tag anjia0532/k8s-dns-sidecar-amd64:1.14.8 k8s.gcr.io/k8s-dns-sidecar-amd64:1.14.8 \
    && docker tag anjia0532/k8s-dns-kube-dns-amd64:1.14.8 k8s.gcr.io/k8s-dns-kube-dns-amd64:1.14.8 \
    && docker tag anjia0532/pause-amd64:3.1 k8s.gcr.io/pause-amd64:3.1
    

**需要下载 `crictl`**

*   USER 与 IP 修改为自己的

    $ wget https://github.com/kubernetes-incubator/cri-tools/releases/download/v1.0.0-beta.1/crictl-v1.0.0-beta.1-linux-amd64.tar.gz
    $ tar xvf crictl-v1.0.0-beta.1-linux-amd64.tar.gz
    $ sudo chown root:root crictl
    $ sudo cp crictl /usr/bin
    $ scp crictl USER@IP:/home/USER
    
    $ sudo chown root:root crictl
    $ sudo mv crictl /usr/bin/
    

**上面的步骤需要在所有机器上安装**

### 初始化 master 节点

    $ sudo kubeadm init --apiserver-advertise-address=10.66.180.159 --pod-network-cidr=10.244.0.0/16 --feature-gates=CoreDNS=true --kubernetes-version=v1.10.3
    

**输出**

    [init] Using Kubernetes version: v1.10.3
    [init] Using Authorization modes: [Node RBAC]
    [preflight] Running pre-flight checks.
        [WARNING SystemVerification]: docker version is greater than the most recently validated version. Docker version: 18.03.1-ce. Max validated version: 17.03
    [preflight] Starting the kubelet service
    [certificates] Generated ca certificate and key.
    [certificates] Generated apiserver certificate and key.
    [certificates] apiserver serving cert is signed for DNS names [org2 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.66.180.159]
    [certificates] Generated apiserver-kubelet-client certificate and key.
    [certificates] Generated etcd/ca certificate and key.
    [certificates] Generated etcd/server certificate and key.
    [certificates] etcd/server serving cert is signed for DNS names [localhost] and IPs [127.0.0.1]
    [certificates] Generated etcd/peer certificate and key.
    [certificates] etcd/peer serving cert is signed for DNS names [org2] and IPs [10.66.180.159]
    [certificates] Generated etcd/healthcheck-client certificate and key.
    [certificates] Generated apiserver-etcd-client certificate and key.
    [certificates] Generated sa key and public key.
    [certificates] Generated front-proxy-ca certificate and key.
    [certificates] Generated front-proxy-client certificate and key.
    [certificates] Valid certificates and keys now exist in "/etc/kubernetes/pki"
    [kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"
    [kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"
    [kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/controller-manager.conf"
    [kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/scheduler.conf"
    [controlplane] Wrote Static Pod manifest for component kube-apiserver to "/etc/kubernetes/manifests/kube-apiserver.yaml"
    [controlplane] Wrote Static Pod manifest for component kube-controller-manager to "/etc/kubernetes/manifests/kube-controller-manager.yaml"
    [controlplane] Wrote Static Pod manifest for component kube-scheduler to "/etc/kubernetes/manifests/kube-scheduler.yaml"
    [etcd] Wrote Static Pod manifest for a local etcd instance to "/etc/kubernetes/manifests/etcd.yaml"
    [init] Waiting for the kubelet to boot up the control plane as Static Pods from directory "/etc/kubernetes/manifests".
    [init] This might take a minute or longer if the control plane images have to be pulled.
    [apiclient] All control plane components are healthy after 20.001731 seconds
    [uploadconfig] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
    [markmaster] Will mark node org2 as master by adding a label and a taint
    [markmaster] Master org2 tainted and labelled with key/value: node-role.kubernetes.io/master=""
    [bootstraptoken] Using token: yof8me.re7zttbijtwmdavm
    [bootstraptoken] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
    [bootstraptoken] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
    [bootstraptoken] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
    [bootstraptoken] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
    [addons] Applied essential addon: CoreDNS
    [addons] Applied essential addon: kube-proxy
    
    Your Kubernetes master has initialized successfully!
    
    To start using your cluster, you need to run the following as a regular user:
    
      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config
    
    You should now deploy a pod network to the cluster.
    Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
      https://kubernetes.io/docs/concepts/cluster-administration/addons/
    
    You can now join any number of machines by running the following on each node
    as root:
    
      kubeadm join 10.66.180.159:6443 --token yof8me.re7zttbijtwmdavm --discovery-token-ca-cert-hash sha256:507756057bf6982624fa07bbbbdab0855d555c809ec185aac10de95a00c77e2b
    

如果需要在非 root 用户使用 `kubectl`, 可以执行以下命令（也是 `kubeadm init` 输出的一部分）

    $ mkdir -p $HOME/.kube
    $ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    $ sudo chown $(id -u):$(id -g) $HOME/.kube/config
    

如果是 root 用户的话，你可以运行以下命令

    export KUBECONFIG=/etc/kubernetes/admin.conf
    

kubeadm init 输出的 token 用于 master 和加入节点间的身份认证，token 是机密的，需要保证它的安全，因为拥有此标记的人都可以随意向集群中添加节点。你也可以使用 `kubeadm` 命令列出，创建，删除 Token，有关详细信息, 请参阅[官方引用文档](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-token/)。

**注意：kubeadm 加节点，还会在找 /var/run/dockershim.sock 文件时会报错，得加参数–ignore-preflight-errors=cri**

### 加入集群

    token 需要换成自己初始化时生成的。
    

    kubeadm join 10.66.180.159:6443 --token yof8me.re7zttbijtwmdavm --discovery-token-ca-cert-hash sha256:507756057bf6982624fa07bbbbdab0855d555c809ec185aac10de95a00c77e2b --ignore-preflight-errors=cri
    

* * *

### 常用命令

#### kubeadm 常用命令

*   [kubeadm init](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/)
*   [kubeadm join](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/)
*   [kubeadm upgrade](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-upgrade/)
*   [kubeadm config](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-config/)
*   [kubeadm reset](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/)
*   [kubeadm token](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-token/)
*   [kubeadm version](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-version/)
*   [kubeadm alpha](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-alpha/)

#### kubectl 常用命令

*   `kubectl get node --show-labels`
*   `kubectl get nodes`

* * *

### 参考文档

*   [Google Container Registry(gcr.io) 中国可用镜像(长期维护)](https://anjia0532.github.io/2017/11/15/gcr-io-image-mirror/)
*   [使用 kubeadm 搭建 Kubernetes(1.10.2) 集群（国内环境）](https://www.cnblogs.com/RainingNight/p/using-kubeadm-to-create-a-cluster.html)
*   [ubuntu下使用 kubeadm 安装部署 kubernetes v1.10](https://blog.csdn.net/qq_37423198/article/details/79762687)
*   [使用 kubeadm 创建集群–官方文档](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)
*   [kubeadm 设置工具参考指南](https://k8smeetup.github.io/docs/admin/kubeadm/)
