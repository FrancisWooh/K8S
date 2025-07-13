# **Chap 1：K8S Deployment**



### What Is Kubernetes?

Kubernetes, also known as K8s, is an open source system for automating deployment, scaling, and management of 

containerized applications.

#### Why We Need Kubernetes

Development of containerized application is the trend of the industry.  How to manage the various containers in production system in a stable and resilient manner and deploy them in an agile and scalable format in the production environment has become more and more a challenge.  K8S aims the MANO of the container applications. 

**Self-Healing**:   Once certain container application corrupts, K8S can bring up a new application instantly.

**Resilient Scalable:**  K8S can adjust the number of container applications based on the load dynamically.  

**Load Balance:**  K8S can balance the incoming traffic among all application containers.

**Version Management:**  K8S provides instant switch over among different application versions.

#### Development of Deployment

![image-20241029104522606](C:\Users\Pre-Installed User\AppData\Roaming\Typora\typora-user-images\image-20241029104522606.png)

#### K8S Cluster Roles

Kubernetes can be deployed in multi nodes to form the cluster.  There are two roles of the cluster members:

**Master:**   Cluster Management,  control plane of the cluster

**Node:**      Also called worker node.  Prepare the compute environment to run the pod

#### K8S Cluster Type

One Master + Multi Node

Multi Master + Multi Node

![image-20241029135346121](C:\Users\Pre-Installed User\AppData\Roaming\Typora\typora-user-images\image-20241029135346121.png)



#### K8S Cluster Component

**Control Plane Components**:

Manage the overall state of the cluster:

- [kube-apiserver](https://kubernetes.io/docs/concepts/architecture/#kube-apiserver)

  The core component server that exposes the Kubernetes HTTP API

- [etcd](https://kubernetes.io/docs/concepts/architecture/#etcd)

  Consistent and highly-available key value store for all API server data

- [kube-scheduler](https://kubernetes.io/docs/concepts/architecture/#kube-scheduler)

  Looks for Pods not yet bound to a node, and assigns each Pod to a suitable node.

- [kube-controller-manager](https://kubernetes.io/docs/concepts/architecture/#kube-controller-manager)

  Runs [controllers](https://kubernetes.io/docs/concepts/architecture/controller/) to implement Kubernetes API behavior.

**Node Components**:

Run on every node, maintaining running pods and providing the Kubernetes runtime environment:

- [kubelet](https://kubernetes.io/docs/concepts/architecture/#kubelet)

  Ensures that Pods are running, including their containers.

- [kube-proxy](https://kubernetes.io/docs/concepts/architecture/#kube-proxy) (optional)

  Maintains network rules on nodes to implement [Services](https://kubernetes.io/docs/concepts/services-networking/service/).

- [Container runtime](https://kubernetes.io/docs/concepts/architecture/#container-runtime)

  Software responsible for running containers. Read [Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/) to learn more.

  ![image-20241029140837998](C:\Users\Pre-Installed User\AppData\Roaming\Typora\typora-user-images\image-20241029140837998.png)

#### How Kubelet Provision POD



![image-20241029143130058](C:\Users\Pre-Installed User\AppData\Roaming\Typora\typora-user-images\image-20241029143130058.png)

CRI :  A standard of API definition on how to use different container runtimes in Kubernetes

**Control Plane Components**:

[The differences between Docker, containerd, CRI-O and runc | by Vineet Kumar | Medium](https://vineetcic.medium.com/the-differences-between-docker-containerd-cri-o-and-runc-a93ae4c9fdac)



#### Other Alternatives

​	RedHat  OpenShift

​	AWS   EKS

​	Docker Swarm



### VM Setup

#### Server Specification



| Seq  | HostName | IP            | Role   | OS        | HW Spec        |
| ---- | -------- | ------------- | ------ | --------- | -------------- |
| 1    | master01 | 192.168.1.128 | Master | Rocky 8.9 | 2CPU/4GMem/40G |
| 2    | master02 | 192.168.1.129 | Master | Rocky 8.9 | 2CPU/4GMem/40G |
| 3    | worker01 | 192.168.1.130 | Worker | Rocky 8.9 | 2CPU/4GMem/40G |
| 4    |          | 192.168.1.100 | vip    |           |                |

| Seq  | HostName | Component           | Remark            |
| ---- | -------- | ------------------- | ----------------- |
| 1    | master01 | haproxy、keepalived | keepalived Master |
| 2    | master02 | haproxy、keepalived | keepalived Slave  |



![image-20241028100910114](C:\Users\Pre-Installed User\AppData\Roaming\Typora\typora-user-images\image-20241028100910114.png)



#### Configure IP Address

Execute in all servers（mandatory）

~~~powershell
# cd /etc/sysconfig/network-scripts/
[root@localhost network-scripts]# more ifcfg-ens33 
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPADDR=192.168.0.128
MASK=255.255.255.0
GATEWAY=192.168.0.1
DNS1=8.8.8.8
NAME=ens33
UUID=ba18daa7-3a45-45cb-b0e0-52364df36c6e
DEVICE=ens33
ONBOOT=yes
~~~



### Host Setup

#### Host Name Configuration

The cluster is composed by 3 servers, 2 are master, which is working in active/active mode with a virtual IP in the control plan.

The other is worker nodes, which works in the traffic plan to host various pods

~~~powershell
master01
# hostnamectl set-hostname master01
~~~

~~~powershell
master02
# hostnamectl set-hostname master02
~~~

~~~powershell

~~~

~~~powershell
worker1 node
# hostnamectl set-hostname worker01

~~~

#### System Initial Setup

Execute in all servers（mandatory）

> 

~~~powershell
# yum install lrzsz
......
upload the system initialization script
[root@master01 ~]# chmod +x sysconfigure_en.sh 
[root@master01 ~]# ./sysconfigure_en.sh 
~~~

#### Resolve the hostname and IP

Execute in all servers（mandatory）

> 

~~~powershell
# vim /etc/hosts
......
192.168.0.128 master01
192.168.0.129 master02
192.168.0.130 worker01
~~~

#### SSH-Key 

> We can generate the ssh key and share within the cluster to login each other without password（not mandatory）

~~~powershell
# ssh-keygen
# for i in 128 129 130; do ssh-copy-id 192.168.1.$i; done
# ssh master01
# ssh master02
# ssh worker01

~~~

Enable bridge-nf-call-iptables feature

> NAT bridge is used in kubernetes. Enable the bridge feature in all servers

~~~powershell
# cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
~~~

~~~powershell
Enable IPVE route forwarding, br-netfileter is mandatroy
# modprobe br_netfilter && lsmod | grep br_netfilter
br_netfilter           22256  0
bridge                151336  1 br_netfilter
~~~

~~~powershell
Effect the configuration change
# sysctl -p /etc/sysctl.d/k8s.conf
~~~

#### Configure ipvs

> Ip Virtual Server balances the traffic in Transport Layer.

~~~powershell
Install package
# yum -y install ipset ipvsadm
~~~

~~~powershell
Enable ipvs in alogrithm modules
# cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack
EOF


~~~

~~~powershell
Authorize and enable the alogirthm
# chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack
~~~



#### Disable SAWP

> It is required to disable the swap during k8s installation

~~~powershell
Disable swap permanently
# ansible k8s -m shell -a "sed -i '11 s/^/#/' /etc/fstab"
# cat /etc/fstab
......

# /dev/mapper/centos-swap swap                    swap    defaults        0 0


~~~

```powershell
Reboot to effect the change
# reboot
```

```powershell
Verify the OS core
# uname -r
```

```powershell
Verify swap is off
# free -h
```

### Docker Setup

#### Docker Preparation

> Execute in all servers

```powershell
Install the repository		----No need
# yum install -y yum-utils && yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

yum install -y yum-utils && yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

yum list docker-ce --showduplicates | sort -r    ---check available versions
yum install docker-ce-20.10.20 docker-ce-cli-20.10.20 containerd.io --allowerasing   
---Install 20.10.20

yum remove docker-buildx-plugin.x86_64		---in case of lib conflict failure

https://www.cnblogs.com/lifuqiang/articles/17293679.html
https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-rocky-linux-8
https://github.com/docker/docker-install/issues/230
```

~~~powershell
Install docker package
# yum -y install docker-ce

dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

dnf install docker-ce docker-ce-cli containerd.io --allowerasing 

https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-rocky-linux-8
https://github.com/docker/docker-install/issues/230
~~~



#### Configure Cgroup Driver

>  Control Group assigns the process resources（CGroup）

~~~powershell
Modify /etc/docker/daemon.json
 mkdir /etc/docker
 cat > /etc/docker/daemon.json <<EOF
{
        "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
~~~

~~~powershell
Configure auto start
# systemctl enable docker ; systemctl start docker
~~~



### HA Setup

#### Install HAProxy keepalived



![image-20241029141358008](C:\Users\Pre-Installed User\AppData\Roaming\Typora\typora-user-images\image-20241029141358008.png)



Execute the following steps only in master01 and master02

~~~powershell
master01
[root@master01 ~]# yum -y install haproxy keepalived
~~~

~~~powershell
master02
[root@master02 ~]# yum -y install haproxy keepalived 
~~~



#### HAProxy configuration



~~~powershell
[root@master01 ~]# vim /etc/haproxy/haproxy.cfg

...

backend k8s-master      
  mode tcp
  option tcplog
  option tcp-check
  balance roundrobin
  default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
  server master01   192.168.1.215:6443  check   
  server master02   192.168.1.216:6443  check   
 
In case of haproxy startup failure, execute the following command to resolve. 
 setsebool -P nis_enabled 1

~~~

~~~powershell

[root@master01 ~]# systemctl start haproxy
[root@master01 ~]# systemctl enable haproxy 
[root@master01 ~]# systemctl status haproxy
~~~

~~~powershell

[root@master01 ~]# scp /etc/haproxy/haproxy.cfg master02:/etc/haproxy/haproxy.cfg
~~~

~~~powershell

[root@master02 ~]# systemctl start haproxy
[root@master02 ~]# systemctl enable haproxy 
[root@master02 ~]# systemctl status haproxy
~~~



#### Keepalived Configuration

~~~powershell
[root@master01 ~]# vim /etc/keepalived/keepalived.conf
~~~



Upload the check_apiserver.sh monitor script

~~~powershell
[root@master01 ~]# cat /etc/keepalived/check_apiserver.sh
~~~

~~~powershell
[root@master01 ~]# chmod +x /etc/keepalived/check_apiserver.sh
~~~

~~~powershell

[root@master01 ~]# scp /etc/keepalived/keepalived.conf master02:/etc/keepalived/

[root@master01 ~]# scp /etc/keepalived/check_apiserver.sh master02:/etc/keepalived/
~~~



Modify configuration files:

~~~powershell
[root@master02 ~]# cat /etc/keepalived/keepalived.conf



vrrp_instance VI_1 {
    state BACKUP    
    interface ens32 
    mcast_src_ip 192.168.0.11 
    virtual_router_id 51
    priority 99 
...
~~~

> master01、master02 Autostart keepalived

~~~powershell
[root@master01 ~]# systemctl enable keepalived --now
[root@master01 ~]# systemctl status keepalived

[root@master02 ~]# systemctl enable keepalived --now
[root@master02 ~]# systemctl status keepalived
~~~



#### Verify cluster HA

~~~powershell
[root@master01 ~]# ip a s ens33
~~~



~~~powershell

[root@master01 ~]# ss -anput | grep ":16443"
tcp    LISTEN     0      2000   127.0.0.1:16443                 *:*                   users:(("haproxy",pid=2983,fd=6))
tcp    LISTEN     0      2000      *:16443                 *:*                   users:(("haproxy",pid=2983,fd=5))
~~~



~~~powershell
[root@master02 ~]# ss -anput | grep ":16443"
tcp    LISTEN     0      2000   127.0.0.1:16443                 *:*                   users:(("haproxy",pid=2974,fd=6))
tcp    LISTEN     0      2000      *:16443                 *:*                   users:(("haproxy",pid=2974,fd=5))
~~~



**********



### K8S Setup

#### k8s Cluster Deployment

Most popular ways to deploy the K8s

- kubeadm mode：kubeadm is a cluster tool that fast the deployment procedures

- Binary installation：Download all necessary packages and install manually

  

[Creating a cluster with kubeadm | Kubernetes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

#### Cluster Version Description

|          | kubeadm                  | kubelet                  | kubectl                  |
| -------- | ------------------------ | ------------------------ | ------------------------ |
| Version  | 1.23.0                   | 1.23.0                   | 1.23.0                   |
| Node     | All nodes in the cluster | All nodes in the cluster | All nodes in the cluster |
| Function | Cluster Initialization   | Agent for worker nodes   | Kube AIP CLI tool        |



#### kubernetes YUM prepration



#### Google Yum

~~~powershell
cat > /etc/yum.repos.d/k8s.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
~~~



#### Ali Yum

>

~~~powershell
cat > /etc/yum.repos.d/k8s.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
~~~

**************************2nd snapshot

#### Cluster Software Installation

> Execute in all nodes

~~~powershell
Install specified version
# yum install -y --nogpgcheck kubeadm-1.23.0-0  kubelet-1.23.0-0 kubectl-1.23.0-0
~~~



#### Configure Kubelet

>

~~~powershell
# vim /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--cgroup-driver=systemd"
~~~

~~~powershell

# systemctl enable kubelet
~~~



#### Cluster Initialization

> **master01 Only

~~~powershell
[root@master01 ~]# vim kubeadm-config.yaml

...

kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.0.10       ---vip
  bindPort: 6443

...
~~~

~~~powershell
Cluster Initialization
[root@master01 ~]# kubeadm init --config /root/kubeadm-config.yaml --upload-certs
~~~

```powershell
You can reset the nodes in case of error
kubeadm  reset
```

[root@master01 ~]# kubeadm init --config /root/kubeadm-config.yaml --upload-certs
[init] Using Kubernetes version: v1.23.0
[preflight] Running pre-flight checks
	[WARNING SystemVerification]: this Docker version is not on the list of validated versions: 26.1.3. Latest validated version: 20.10
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR NumCPU]: the number of available CPUs 1 is less than the required 2
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher
[root@master01 ~]# poweroff

#### Prepare the environment configuration

~~~powershell
[root@master01 ~]# mkdir -p $HOME/.kube
[root@master01 ~]# cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[root@master01 ~]# chown $(id -u):$(id -g) $HOME/.kube/config
[root@master01 ~]# export KUBECONFIG=/etc/kubernetes/admin.conf
~~~



#### Master node joins the cluster



Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 192.168.1.214:16443 --token 7t2weq.bjbawausm0jaxury \
	--discovery-token-ca-cert-hash sha256:66637f1d904092fb3be664e8b5b1a2344294317eb9ff4704dd07ed01b8fafb4f \
	--control-plane --certificate-key 96c6214650f22ea2a52488ac7040a9290615d7512c916549d2acde251a34c1f2

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.1.214:16443 --token 7t2weq.bjbawausm0jaxury \
	--discovery-token-ca-cert-hash sha256:66637f1d904092fb3be664e8b5b1a2344294317eb9ff4704dd07ed01b8fafb4f 



![1686893925351](D:\1 Customer\MEC\ShareSession\2nd\day01 笔记.accets\1686811830446.png)



#### Wroker nodes join the cluster





If token expires, regenerate the token

```powershell

[root@master01 ~]# kubeadm token generate
c05nt0.xn6zb1k40g82k0gm


[root@master01 ~]#kubeadm token create c05nt0.xn6zb1k40g82k0gm --print-join-command --ttl=0 
```



#### Cluster Network Preparation



#### Calico Installation

> Execute only in Master01

https://projectcalico.docs.tigera.io/about/about-calico



~~~powershell
Create resource file
[root@master01 ~]# kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/tigera-operator.yaml
~~~

~~~powershell
Deploy the resource file
[root@master01 ~]# wget https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/custom-resources.yaml
~~~

~~~powershell
Modify the cidr ip
[root@master01 ~]# vim custom-resources.yaml
......
 11     ipPools:
 12     - blockSize: 26
 13       cidr: 10.244.0.0/16          192.168.1.240/28
 14       encapsulation: VXLANCrossSubnet
......
~~~

~~~powershell
Apply the resource file
[root@master01 ~]# kubectl create -f custom-resources.yaml
~~~

~~~powershell
check the name space
[root@master01 ~]# kubectl get ns
[root@master01 ~]# kubectl get pods -n calico-system
~~~

>

~~~powershell
Check Taint
[root@master01 ~]# kubectl describe nodes  master01 | grep Taints
Taints:             node-role.kubernetes.io/master:NoSchedule


~~~

```powershell
Delete the taint
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
kubectl taint nodes --all node-role.kubernetes.io/master-
```

~~~powershell
Check the pod status
[root@master01 ~]# kubectl get pods -n calico-system
NAME                                      READY   STATUS    RESTARTS   AGE
calico-kube-controllers-666bb9949-dzp68   1/1     Running   0          11m
calico-node-jhcf4                         1/1     Running   4          11m
calico-typha-68b96d8d9c-7qfq7             1/1     Running   2          11m
~~~

~~~powershell
Check the system pod status
[root@master01 ~]# kubectl get pods -n kube-system
NAME                               READY   STATUS    RESTARTS   AGE
coredns-558bd4d5db-4jbdv           1/1     Running   0          113m
coredns-558bd4d5db-pw5x5           1/1     Running   0          113m
etcd-master01                      1/1     Running   0          113m
kube-apiserver-master01            1/1     Running   0          113m
kube-controller-manager-master01   1/1     Running   4          113m
kube-proxy-kbx4z                   1/1     Running   0          113m
kube-scheduler-master01            1/1     Running   3          113m
~~~



#### Cluster Verification

~~~powershell
Check nodes
[root@master01 ~]# kubectl get nodes
NAME       STATUS   ROLES                  AGE   VERSION
master01   Ready    control-plane,master   25m   v1.23.0
master02   Ready    control-plane,master   25m   v1.23.0
~~~

~~~powershell
[root@master01 ~]# kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health":"true"}
~~~



~~~powershell
[root@master01 ~]# kubectl get pods -n kube-system
NAME                               READY   STATUS    RESTARTS   AGE
coredns-558bd4d5db-smp62           1/1     Running   0          13m
coredns-558bd4d5db-zcmp5           1/1     Running   0          13m
etcd-master01                      1/1     Running   0          14m
etcd-master02                      1/1     Running   0          3m10s
etcd-master03                      1/1     Running   0          115s
kube-apiserver-master01            1/1     Running   0          14m
kube-apiserver-master02            1/1     Running   0          3m13s
kube-apiserver-master03            1/1     Running   0          116s
kube-controller-manager-master01   1/1     Running   1          13m
kube-controller-manager-master02   1/1     Running   0          3m13s
kube-controller-manager-master03   1/1     Running   0          116s
kube-proxy-629zl                   1/1     Running   0          2m17s
kube-proxy-85qn8                   1/1     Running   0          3m15s
kube-proxy-fhqzt                   1/1     Running   0          13m
kube-proxy-jdxbd                   1/1     Running   0          3m40s
kube-proxy-ks97x                   1/1     Running   0          4m3s
kube-scheduler-master01            1/1     Running   1          13m
kube-scheduler-master02            1/1     Running   0          3m13s
kube-scheduler-master03            1/1     Running   0          115s

~~~

~~~powershell
[root@master01 ~]# kubectl get pod -n calico-system
NAME                                      READY   STATUS    RESTARTS   AGE
calico-kube-controllers-666bb9949-4z77k   1/1     Running   0          10m
calico-node-b5wjv                         1/1     Running   0          10m
calico-node-d427l                         1/1     Running   0          4m45s
calico-node-jkq7f                         1/1     Running   0          2m59s
calico-node-wtjnm                         1/1     Running   0          4m22s
calico-node-xxh2p                         1/1     Running   0          3m57s
calico-typha-7cd9d6445b-5zcg5             1/1     Running   0          2m54s
calico-typha-7cd9d6445b-b5d4j             1/1     Running   0          10m
calico-typha-7cd9d6445b-z44kp             1/1     Running   1          4m17s
~~~

### Kuboard Setup

~~~powershell
Check nodes
kubectl apply -f https://addons.kuboard.cn/kuboard/kuboard-v3.yaml
~~~

~~~powershell
[root@master01 ~]# kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health":"true"}
~~~

[安装 Kuboard v3 - kubernetes | Kuboard](https://kuboard.cn/install/v3/install-in-k8s.html#安装)
