# 一.kubernetes集群部署、配置和验证

## 1.安装要求

在开始之前。部署kubernetes集群机器需要满足以下几个条件：

- 一台或多台机器，操作系统Centos7.x86.x64
- 硬件配置：2GB或更多RAM。2个CPU或更多CPU，硬盘30GB或更多
- 集群中所有机器之间网络互通
- 可以访问外网，需要拉取镜像
- 禁止swap分区

## 2.学习目标

1. 在所有节点上安装docker和kubeadm
2. 部署kubernetes Master
3. 部署容器网络插件
4. 部署kubernetes Node，将节点加入kubernetes集群中
5. 部署Dashboard web页面，可视化插件按kubernetes资源

## 3.准备环境

单master集群

![1603974117645](C:\Users\pc\AppData\Roaming\Typora\typora-user-images\1603974117645.png)

| 角色       | IP            |
| ---------- | ------------- |
| k8s-master | 192.168.4.171 |
| k8s-node1  | 192.168.4.172 |
| k8s-node2  | 192.168.4.173 |

~~~shell
关闭防火墙：
#systemctl stop firewalld.service 
#systemctl disables firewalld

关闭selinux：
#sed -i 's/enforcing/disabled/' /etc/selinux/config    永久

#setenforce 0                                                   临时

关闭swap：
#swapoff -a          临时
#vim /etc/fstab     永久禁用swap

在master添加hosts
#cat >> /etc/hosts << EOF
192.168.4.171 k8s-master
192.168.4.172 k8s-node1
192.168.4.173 k8s-node2
EOF

将桥接的ipv4流量传到iptables的链
#cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables =1
net.bridge.bridge-nf-call-iptables =1
EOF

#sysctl --system      生效

时间同步：
#yum -y install ntpdate
#ntpdate time.windows.com
~~~



## 4.所有节点安装docker/kubeadm/kubelet

kubernetes默认CRI（容器运行时）为Docker，因此先安装Docker

### 4.1安装docker

~~~shell
#wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
#yum -y install docker-ce-18.06.1.ce-3.el7
#cat > /etc/docker/daemon.json << EOF
{
"registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"]
}
EOF
#systemctl enable docker && systemctl start docker
#docker info
~~~

### 4.2.添加阿里云yum软件源

~~~shell
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
~~~

### 4.3.安装kubelet、kubeadm、kubectl

由于版本更新频繁，这里指定版本部署

~~~shell
# yum -y install kubelet-1.18.0 kubeadm-1.18.0 kubectl-1.18.0
# systemctl enable kubelet.service 
~~~

## 5.部署kubernetes Master，  在192.168.4.171（master）执行

~~~shell
# kubeadm init \
> --apiserver-advertise-address=192.168.4.171 \    #内网地址    
> --image-repository registry.aliyuncs.com/google_containers \ #仓库
> --kubernetes-version v1.18.0 \            #版本
> --service-cidr=10.96.0.0/12 \       #ip地址随便不能与现有的网络冲突
> --pod-network-cidr=10.244.0.0/16     #pod网段
执行完命令解析
1.[init] Using Kubernetes version: v1.18.0
[preflight] Running pre-flight checks
检查当前环境是否满足，例如配置、swap
2.下载部署组件所需的镜像'kubeadm config images pull'
3.[kubelet-start]启动kubelet并准备配置文件
4.[certs]生成apiserver、etcd证书放在"/etc/kubernetes/pki“目录下
5.[kubeconfig]生成kubeconfig文件，连接apiserver信息，证书信息"/etc/kubernetes"
6.[control-plane]静态pod启动apiserver和controller-manager、etcd
7.[upload-config]存储kubelet配置文件到configmap中
8.[mark-control-plane]给master节点打污点
9. [bootstrap-token] 配置bootstrap tls，为kubelet自动颁发证书
10. [addons]，安装插件kube-proxy（负责pod的网络），CoreDNS(为k8s集群中做内部解析)
~~~

~~~shell
 #mkdir -p $HOME/.kube
 #sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
 #sudo chown $(id -u):$(id -g) $HOME/.kube/config
~~~

## 6.kubectl命令工具配置，安装网络插件flannel（master）

~~~shell
[root@k8s-master ~]# kubectl get nodes       #获取节点信息
NAME         STATUS     ROLES    AGE   VERSION
k8s-master   NotReady   master   55m   v1.18.0 #状态Notready，没有拉起
k8s-node1    NotReady   <none>   29m   v1.18.0
k8s-node2    NotReady   <none>   28m   v1.18.0
[root@k8s-master ~]#wget https://github.com/489557835/k8s/blob/main/kube-flannel.yaml 下载安装插件
[在master上操作]上传kube-flannel.yaml,并执行:
kubectl apply -f kube-flannel.yaml
kubectl get pods -n kube-system
[必须全部运行起来,否则有问题.]
[root@k8s-master1 ~]# kubectl get pods -n kube-system
NAME                                  READY   STATUS    RESTARTS   AGE
coredns-7ff77c879f-5dq4s              1/1     Running   0          13m
coredns-7ff77c879f-v68pc              1/1     Running   0          13m
etcd-k8s-master1                      1/1     Running   0          13m
kube-apiserver-k8s-master1            1/1     Running   0          13m
kube-controller-manager-k8s-master1   1/1     Running   0          13m
kube-flannel-ds-amd64-2ktxw           1/1     Running   0          3m45s
kube-flannel-ds-amd64-fd2cb           1/1     Running   0          3m45s
kube-flannel-ds-amd64-hb2zr           1/1     Running   0          3m45s
kube-proxy-4vt8f                      1/1     Running   0          13m
kube-proxy-5nv5t                      1/1     Running   0          12m
kube-proxy-9fgzh                      1/1     Running   0          12m
kube-scheduler-k8s-master1            1/1     Running   0          13m


[root@k8s-master1 ~]# kubectl get nodes
NAME          STATUS   ROLES    AGE   VERSION
k8s-master1   Ready    master   14m   v1.18.0
k8s-node1     Ready    <none>   12m   v1.18.0
k8s-node2     Ready    <none>   12m   v1.18.0
~~~



## 7.将node1，node2执行加入master集群

~~~shell
在要加入的节点种执行以下命令来加入:
#kubeadm join 192.168.4.171:6443 --token 48jz8v.u60l34tmrk9jolpz \
    --discovery-token-ca-cert-hash sha256:160f781d243fe58db6ba48c604cf7837ab4e4b5d3f95f2e36077b6d79617a7be 
#这个配置在安装master的时候有过提示,请注意首先要配置cni网络插件
#加入成功后,master节点检测:
#如果没有执行成功，node可以执行kubeadm set可以清空当前集群的环境
~~~

## 8.token创建和查询

~~~shell
默认token会保存24消失,过期后就不可用,如果需要重新建立token,可在master节点使用以下命令重新生成:

kubeadm token create
kubeadm token list
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
结果:
3d847b858ed649244b4110d4d60ffd57f43856f42ca9c22e12ca33946673ccb4


新token加入集群方法:
kubeadm join 192.168.4.171:6443 --discovery-token nuja6n.o3jrhsffiqs9swnu --discovery-token-ca-cert-hash 3d847b858ed649244b4110d4d60ffd57f43856f42ca9c22e12ca33946673ccb4
~~~

## 9.安装dashboard界面

~~~shell
#wget https://github.com/489557835/k8s/blob/main/dashboard.yaml
#kubectl apply -f dashboard.yaml
[root@k8s-master ~]# kubectl get svc -n kubernetes-dashboard
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
dashboard-metrics-scraper   ClusterIP   10.103.252.20    <none>        8000/TCP        2m11s
kubernetes-dashboard        NodePort    10.103.204.253   <none>        443:30001/TCP   2m12s
~~~

## 10.访问测试

 集群任意一个角色访问30001端口都可以访问到dashboard页面. 

![1604042781011](C:\Users\pc\AppData\Roaming\Typora\typora-user-images\1604042781011.png)

![1604042908068](C:\Users\pc\AppData\Roaming\Typora\typora-user-images\1604042908068.png)

## 11.获取dashboard token, 也就是创建service account并绑定默认cluster-admin管理员集群角色

~~~shell
# kubectl create serviceaccount dashboard-admin -n kubernetes-dashboard
# kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:dashboard-admin
# kubectl describe secrets -n kubernetes-dashboard $(kubectl -n kubernetes-dashboard get secret | awk '/dashboard-admin/{print $1}')

将复制的token 填写到 上图中的 token选项,并选择token登录
~~~

![1604043196784](C:\Users\pc\AppData\Roaming\Typora\typora-user-images\1604043196784.png)

## 12.集群dashboard证书不可信任，问题处理 [kuberadm部署的解决方案]

~~~shell
1.删除默认的secret,使用自签证书创建新的secret
kubectl delete secret kubernetes-dashboard-certs -n kubernetes-dashboard
kubectl create secret generic kubernetes-dashboard-certs \
--from-file=/etc/kubernetes/pki/apiserver.key --from-file=/etc/kubernetes/pki/apiserver.crt -n kubernetes-dashboard

使用二进制部署的这里的证书需要根据自己当时存储的路径进行修改即可.

2. 证书配置后需要修改dashboard.yaml文件,重新构建dashboard
wget https://github.com/489557835/k8s/blob/main/recommended.yaml
vim recommended.yaml
找到: kind: Deployment,找到这里之后再次查找 args 看到这两行:
- --auto-generate-certificates
- --namespace=kubernetes-dashboard

改为[中间插入两行证书地址]:
- --auto-generate-certificates
- --tls-key-file=apiserver.key
- --tls-cert-file=apiserver.crt
- --namespace=kubernetes-dashboard

[已修改的,可直接使用:wget https://github.com/489557835/k8s/blob/main/dashboard.yaml

3. 修改完毕后重新应用 recommended.yaml
kubectl apply -f recommended.yaml

应用后,可以看到触发了一次滚动更新,然后重新打开浏览器发现证书已经正常显示,不会提示不安全了.
[root@k8s-master1 ~]# kubectl get pods -n kubernetes-dashboard
NAME                                         READY   STATUS    RESTARTS   AGE
dashboard-metrics-scraper-694557449d-r9h5r   1/1     Running   0          2d1h
kubernetes-dashboard-5d8766c7cc-trdsv        1/1     Running   0          93s   <---滚动更新.

4. 查看新的访问端口:
 kubectl get svc -n kubernetes-dashboard
#[root@k8s-master ~]# kubectl get svc -n kubernetes-dashboard
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
dashboard-metrics-scraper   ClusterIP   10.103.252.20    <none>        8000/TCP        4h7m
kubernetes-dashboard        NodePort    10.103.204.253   <none>        443:30055/TCP   4h7m


5. 使用谷歌浏览器打开会发现已经可以打开了
   #1.注意,如果你忘记了登录token,重新生成
   # kubectl create serviceaccount dashboard-admin -n kubernetes-dashboard
   # kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:dashboard-admin
  
   
   #2.还可以查询以前的token 进行登录
    kubectl describe secrets -n kubernetes-dashboard $(kubectl -n kubernetes-dashboard get secret | awk '/dashboard-admin/{print $1}')
~~~

## 二.k8s的两种网络方案与多种工作模式[flannel与calico]

### 1.flannel：

flannel有三种工作模式：

1. vxlan（隧道方案）
2. host-gw（路由方案）
3. udp（在用户态实现的数据封装解封，性能差已经弃用）

~~~shell
#vxlan模式会在当前服务器中创建一个cni0的网桥,和flannel.1隧道端点. 这个隧道端点会对数据包进行再次封装.然后flannel会把数据包传输到目标节点中.同时它也会在本地创建几个路由表.(可以通过命令 ip route 查看到) 
[root@k8s-master ~]# ip route
default via 192.168.4.2 dev eth0 proto dhcp metric 100 
10.244.1.0/24 via 10.244.1.0 dev flannel.1 onlink 
10.244.2.0/24 via 10.244.2.0 dev flannel.1 onlink 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 
192.168.4.0/24 dev eth0 proto kernel scope link src 192.168.4.171 metric 100 
~~~

### 1.2 Flannel网络部署与卸载:

~~~shell
1. 安装flannel网络:
#wget https://github.com/489557835/k8s/blob/main/kube-flannel.yaml
#kubectl apply -f kube-flannel.yaml

2.验证网络:
#创建一个应用
#kubectl create deployment nginx --image=nginx
#kubectl expose deployment nginx --port=80 --target-port=80 --type=NodePort

3.检查测试：
[root@k8s-master ~]# kubectl get pods -o wide
NAME                   READY   STATUS    RESTARTS   AGE     IP           NODE        NOMINATED NODE   READINESS GATES
web-5dcb957ccc-vg2gb   1/1     Running   0          5h43m   10.244.1.4   k8s-node1   <none>

4. ping下ip
#ping 10.244.1.4
PING 10.244.1.4 (10.244.1.4) 56(84) bytes of data.
64 bytes from 10.244.1.4: icmp_seq=1 ttl=63 time=0.865 ms
64 bytes from 10.244.1.4: icmp_seq=2 ttl=63 time=0.549 ms

5.卸载flannel网络：
#kubectl delete -f kube-flannel.yaml  （master执行）

6.全部机器执行
#ip link del cni0      
#ip link del flannel.1

7.查看网桥是否删除,ping下主机或者ifcong查看下
[root@k8s-master ~]# ping 10.244.1.4
PING 10.244.1.4 (10.244.1.4) 56(84) bytes of data.
^C
--- 10.244.1.4 ping statistics ---
15 packets transmitted, 0 received, 100% packet loss, time 14014ms
~~~

### 2.Calico网咯部署与卸载

calico有2种中作模式:

1.ipip(隧道方案) 

2.bgp(路由方案)

```
注意: 公有云可能会对路由方案造成影响,并且有的云主机会禁止路由(bgp)方案,所以有些云厂商是禁止此实现方式的,因为他会写入路由表,这样可能会影响到厂商现有网络.
```

> 路由方案: 对现有网络有一定的要求,但是他的性能最好,它是直接的路由转发模式,他不会经过数据包封装再封装,没有网络消耗.此方案优先选择,但是也要看厂商是否支持. 它会要求,二层网络可达
> 隧道方案: 对现有网络要求不高,它只需要三层通信正常基本都可以通信.

### 

~~~shell
1.#calico网络插件下载:
官方地址:
#wget https://docs.projectcalico.org/manifests/calico.yaml
个人地址：
wget https://github.com/489557835/k8s/blob/main/calico.yaml

注意：安装calico网络插件 需要卸载flannel网络插件

1.默认网段修改:
找到以下内容:
# - name: CALICO_IPV4POOL_CIDR
#   value: "192.168.0.0/16"
改为安装kubernetes时初始化的网段:
- name: CALICO_IPV4POOL_CIDR
  value: "10.244.0.0/16"

2.#执行安装calico网络插件.
[root@k8s-master ~]# kubectl apply -f calico.yaml 
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
daemonset.apps/calico-node created
serviceaccount/calico-node created
deployment.apps/calico-kube-controllers created
serviceaccount/calico-kube-controllers created


3.卸载calico网络插件
[root@k8s-master1 ~]# kubectl delete -f calico.yaml
~~~

### 2.1查看状态

~~~shell
[root@k8s-master ~]# kubectl get pod -n kube-system -o wide
~~~

![1604066901435](C:\Users\pc\AppData\Roaming\Typora\typora-user-images\1604066901435.png)

~~~shell
#状态没有running起来，先检查下cni0网桥.flannel.1网桥是否删除。delete掉pod重建.

[root@k8s-master ~]# kubectl delete pod calico-node-xv6n5 -n kube-system
~~~

2.2验证与日志检查:

~~~shell
1.#应用创建:
#kubectl create deployment nginx --image=nginx
#kubectl expose deployment nginx --port=80 --target-port=80 --type=NodePort
2.[root@k8s-master ~]# kubectl get pods -o wide
NAME                    READY   STATUS    RESTARTS   AGE     IP               NODE        NOMINATED NODE   READINESS GATES
nginx-f89759699-7vskn   1/1     Running   0          40s     10.244.36.66     k8s-node1   <none>           <none>
3.[root@k8s-master ~]# curl -I 10.244.36.66
HTTP/1.1 200 OK
Server: nginx/1.19.3
Date: Fri, 30 Oct 2020 14:15:31 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 29 Sep 2020 14:12:31 GMT
Connection: keep-alive
ETag: "5f7340cf-264"
Accept-Ranges: bytes

4.[root@k8s-master ~]# kubectl logs nginx-f89759699-7vskn
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
10.244.235.192 - - [30/Oct/2020:14:15:31 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.29.0" "-
###访问没有问题,证明calico网络部署成功.
~~~

1.集群规模，flannel host-gw  集群规模小的100台用flannel
2.是否需要网络策略 支持calico，不支持flannel
3.现有网络有限制，列如主机写路由、BGP是否可以通信 
4.维护成本，flannel

# 三.验证集群：

### 1.能不能正常部署应用

### 2.集群网络是否正常

### 

~~~shell
[root@k8s-master ~]# kubectl create deployment web1 --image=nginx
[root@k8s-master ~]# kubectl expose deployment web1 --port=80 --target-port=80 --type=NodePort
[root@k8s-master ~]# kubectl get pod,svc
NAME                        READY   STATUS    RESTARTS   AGE
pod/nginx-f89759699-7vskn   1/1     Running   0          18m
pod/web-5dcb957ccc-95z76    1/1     Running   0          27m
pod/web1-7f87dfbd56-bpxp2   1/1     Running   0          105s

NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/k8s-status-checke   NodePort    10.99.253.229    <none>        80:32527/TCP   4h7m
service/kubernetes          ClusterIP   10.96.0.1        <none>        443/TCP        21h
service/nginx               NodePort    10.109.109.138   <none>        80:30408/TCP   18m
service/web1                NodePort    10.99.138.109    <none>        80:30572/TCP   55s
#每台主机ping
#ping 10.99.138.109
~~~

### 3.集群内部dns解析是否正常

~~~shell
[root@k8s-master ~]# kubectl get pods -n kube-system -o wide
~~~

![1604070323269](C:\Users\pc\AppData\Roaming\Typora\typora-user-images\1604070323269.png)

### 4.出现问题：由flannel切换网络 coredns， 有时会出现问题，解决方法是： 删除重建

~~~shell
1.导出yaml文件
#kubectl get deploy coredns -n kube-system -o yaml > coredns.yaml
2. 删除coredons
#kubectl delete -f coredns.yaml
3.检查是否删除
[root@k8s-master1 ~]# kubectl get pods -n kube-system
4.重建coredns
[root@k8s-master1 ~]# kubectl get pods -n kube-system
~~~

#### 4.1 日志查看，创建一个容器验证dns

~~~shell
[root@k8s-master ~]# kubectl logs coredns-7ff77c879f-69dpb -n kube-system
.:53
[INFO] plugin/reload: Running configuration MD5 = 4e235fcc3696966e76816bcd9034ebc7
CoreDNS-1.6.7
linux/amd64, go1.13.6, da7f65b

#k8s创建一个pod验证dns，指定一个版本，最新版的有问题.
[root@k8s-master1 ~]# kubectl run -it --rm --image=busybox:1.28.4 sh

/ # nslookup kubernetes
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local
#通过 nslookup来解析 kubernetes 能够出现解析,说明dns正常工作
~~~

