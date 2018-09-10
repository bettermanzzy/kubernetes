## kubeadm快速构建kubernetes集群（总结回顾）
构建环境：Ubuntu 16.04  国内局域网（无法直接访问外网）

参考文档：kubernetes官方网站的[kubeadm安装文档](https://kubernetes.io/docs/setup/independent/install-kubeadm/)
以及[利用kubeadm创建集群](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)这两个文档

搭建需要至少3台服务器主机，一台作为master，两台为slave节点。
|  Node |IP             |主机名        |
|-------|---------------|-------------|
|Master |192.168.88.151 |zhou-k8s-0003|
|slave  |192.168.88.249|zhou-k8s-0001 |
|slave  |192.168.88.160 |zhou-k8s-0002|

### 配置网络环境（翻墙） 

>使用shadowsock服务器，http代理,连接外部网络。  
>通过ss代理，本次连接外网主机IP为192.168.88.229，三台服务器，直接export连网主机配置，连接外部网络
>
>
	export http_proxy=http://192.168.88.229:8118
	export https_proxy=http://192.168.88.229:8118 
	export no_proxy="localhost,localaddress,192.168.88.229,本机IP"

### 安装docker，并配置docker网络代理
>更新apt-get 安装docker
>
	apt-get update && apt-get install docker.io

>配置docker 网络代理，docker需要去外网pull kubernetes相关镜像
>
	mkdir -p /etc/systemd/system/docker.service.d/http-proxy-config
	systemctl daemon docker
	systemctl restart docker

### 安装kubeadm,kubectl,kubelet 并配置kubelet参数
>从google cloud下载包，安装kubeadm，kubectl,kubelet
>
	apt-get update && apt-get install -y apt-transport-https curl
	curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
	cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
	deb http://apt.kubernetes.io/ kubernetes-xenial main
	EOF
	apt-get update
	apt-get install -y kubelet kubeadm kubectl
	apt-mark hold kubelet kubeadm kubectl

>配置kubectl参数
>
	vim /etc/default/kubelet
	KUBELET_KUBEADM_EXTRA_ARGS=--cgroup-driver=cgroupfs

### 初始化master节点的kubeadm
>基本命令 `kubeadm init`  
>一般初始化集master构建集群的步骤如下：
>
>1. 下载kubeadm镜像  `kubeadm config images pull`  
>2. 提前选取pod网络类型，确定--pod-network-cidr取值  
>3. `kubeadm init --podnetwork-cidr=192.168.0.0/16` (本文选取Calio pod network)  
  如果初始化failed，且错误显示为port in use 则先重置在执行命令 `kubeadm reset`
>4. 出现successful 留意 kubeadm join --token --discovery-token-ca-cert-hash（后面会用到）
>5. root用户执行 `export KUBECONFIG=/etc/kubernetes/admin.conf`

>本文执行初始化命令： 
>	
	kubeadm config images pull
	kubeadm init --ignore-preflight-errors=all --pod-network-cidr=192.168.0.0/16
	export KUBECONFIG=/etc/kubernetes/admin.conf

### master节点安装pod-network

>基本命令：kubectl apply -f <add-on.yaml>

>本文试验安装Calio pod network
>	
	kubectl apply -f https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/hosted/rbac-kdd.yaml
	kubectl apply -f https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml

>获取pod-network网络状态信息  
>	
	kubectl get pod -n kube-system -o wide


### 添加节点到kubernetes集群 
>基本命令：kubeadm join --token --discovery-token-ca-cert-hash  
>node加入master kubernetes集群前，需要关闭代理，并且最好重置kubeadm  
>	
	unset http_proxy
	unset https_proxy
	kubeadm reset
	kubeadm join 192.168.88.151:8118 --token xxxx --discovery-token-ca-cert-hash xxxx
	
### 查看/删除集群节点
>master 查看集群节点状态   `kubectl get nodes`  
>删除集群节点,首先需要排空节点
>	
	kubectl drain <node-name> --delete-local-data --force --ignore-daemonsets
 	kubectl delete node <node-name>