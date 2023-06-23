## Setup High Available Kubernetes Cluster Using Kubeadm


Kubeadm is a tool which can help to build Kubernetes clusters. It facilitates the setup processes by the two command:

kube init: used to create a master node.

kube join: add a node(master node or worker node) to current cluster. 

Before I build my cluster, I prepared 7 virtual machines with CentOS 7 installed (CentOS 8 is no longer be maintained and CentOS is considered more stable than Ubuntu). I used 3 vm for master nodes and 4 vm for worker nodes.


The final goal of our setup is like the following picture:

![High Available Kubernetes Cluster](https://github.com/JZ0815/JZ0815.github.io/blob/main/images/myk8s.png?raw=true)

Here I have my master and worker nodes ip and host name below, if you are following my steps, please use ifconfig command to get your own ip.

192.168.31.137    master1

192.168.31.190    master2

192.168.31.172    master3

192.168.31.110    node1

192.168.31.148    node2

192.168.31.114    node3

192.168.31.109    node4

And I am going to use 192.168.31.158 as my virtual ip (in the middle box of the picture above).

#### Step One, Server Preparations
1 . Turn Off firewall in all master and worker nodes

   systemctl stop firewalld

   systemctl disable firewalld

2 . Turn Off Selinux in all master and worker nodes

   sed -i 's/enforcing/disabled/' /etc/selinux/config  # permanent

   setenforce 0  # temporary

3 . Turn Off swap in all master and worker nodes

   swapoff -a  # temporary

   sed -ri 's/.*swap.*/#&/' /etc/fstab    # permanent
4 . Set Hostname for all vm
   
   hostnamectl set-hostname <hostname>

   hostname

5 . We can also assign static ip to all vm, go to vi /etc/sysconfig/network-scripts/ and change the settings in ifcfg-ens33, below is a sample:

   TYPE="Ethernet"

   PROXY_METHOD="none"

   BROWSER_ONLY="no"

   BOOTPROTO="none"

   IPADDR=XXX.XXX.XXX.XXX

   PREFIX=24

   GATEWAY=XXX.XXX.XXX.XXX

   DNS1=192.168.31.137

   DNS2=8.8.8.8

   DNS3=8.8.4.4

   DEFROUTE="yes"

   IPV4_FAILURE_FATAL="no"

   IPV6INIT="no"

   IPV6_AUTOCONF="no"

   IPV6_DEFROUTE="no"

   IPV6_FAILURE_FATAL="no"

   IPV6_ADDR_GEN_MODE="stable-privacy"

   NAME="ens33"

   UUID="Your respective network UUID"

   DEVICE="ens33"

   ONBOOT="yes"

Run the command systemctl restart network to restart the network

6 . Edit /etc/hosts file in master nodes

   cat >> /etc/hosts << EOF

   192.168.31.158    master.k8s.io   k8s-vip

   192.168.31.137    master01.k8s.io master1

   192.168.31.190    master02.k8s.io master2

   192.168.31.172    master03.k8s.io master3

   192.168.31.110    node01.k8s.io   node1

   192.168.31.148    node02.k8s.io   node2

   192.168.31.114    node03.k8s.io   node3

   192.168.31.109    node04.k8s.io   node4

   EOF
7 . Update iptables

   cat > /etc/sysctl.d/k8s.conf << EOF

   net.bridge.bridge-nf-call-ip6tables = 1

   net.bridge.bridge-nf-call-iptables = 1

   EOF

   sysctl --system 
8 . Sync time in all master and worker nodes:
   
   yum install ntpdate -y

   ntpdate time.windows.com

#### Step Two, Intall keepalived in all Master Nodes
1 . Install keeplived package in all 3 master nodes

   yum install -y conntrack-tools libseccomp libtool-ltdl

   yum install -y keepalived

2 . Configure all 3 master nodes, run the following scripts in all 3 masters, in my settings, my vip is 192.168.31.158. Use yours when you are setting the values. 


   cat > /etc/keepalived/keepalived.conf <<EOF

   ! Configuration File for keepalived


   global_defs {

     router_id k8s

   }


   vrrp_script check_haproxy {

     script "killall -0 haproxy"

     interval 3

     weight -2

     fall 10

     rise 2

  }


  vrrp_instance VI_1 {

    state MASTER

    interface ens33

    virtual_router_id 51

    priority 250

    advert_int 1

    authentication {

    auth_type PASS

    auth_pass ceb1b3ec013d66163d6ab

  }

  virtual_ipaddress {

    192.168.31.158

  }

  track_script {

    check_haproxy

   }

  }

  EOF

3 . Start keeplived in all 3 masters, enable keepalived and check status,

   systemctl start keepalived.service

   systemctl enable keepalived.service

   systemctl status keepalived.service

#### Step Three, Setup Haproxy

We can also use Nginx here. Haproxy and Nginx are both open source software used for load balancing, reverse proxying, and web serving. The main difference between the two is that Haproxy is a proxy server specifically designed to handle high levels of traffic while Nginx can be used as either a proxy or web server depending on configuration.

1 . Install Haproxy

   yum install -y haproxy

2 . Configuration Haproxy, let's use port 16443 here. Run the following scripts in all 3 master nodes. Notice to put your own master ip here.


   cat > /etc/haproxy/haproxy.cfg << EOF

   global

   log         127.0.0.1 local2

     chroot      /var/lib/haproxy

     pidfile     /var/run/haproxy.pid

     maxconn     4000

     user        haproxy

     group       haproxy

     daemon 


    stats socket /var/lib/haproxy/stats

    defaults

    mode                    http

    log                     global

    option                  httplog

    option                  dontlognull

    option http-server-close

    option forwardfor       except 127.0.0.0/8

   option                  redispatch

   retries                 3

   timeout http-request    10s

   timeout queue           1m

   timeout connect         10s

   timeout client          1m

   timeout server          1m

   timeout http-keep-alive 10s

   timeout check           10s

   maxconn                 3000

   frontend kubernetes-apiserver

   mode                 tcp

   bind                 *:16443

   option               tcplog

   default_backend      kubernetes-apiserver

   backend kubernetes-apiserver

   mode        tcp

   balance     roundrobin

   server      master01.k8s.io   192.168.31.137:6443 check

   server      master02.k8s.io   192.168.31.190:6443 check

   server      master03.k8s.io   192.168.31.172:6443 check

  listen stats

  bind                 *:1080

  stats auth           admin:awesomePassword

  stats refresh        5s

  stats realm          HAProxy\ Statistics

  stats uri            /admin?stats

EOF

3 . Enable and Start Haproxy

   systemctl enable haproxy

   systemctl start haproxy

   systemctl status haproxy


#### Step Four, Install Docker/Containerd/Kubeadm/Kubelet at all Nodes
1. Install Docker (Notice Kubernetes has abandoned and in the latest version, we should install containerd, so step 1 is optional for latest k8s version).

  yum install -y yum-utils device-mapper-persistent-data lvm2

  yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

  yum install docker

  systemctl start docker

  systemctl enable docker

2 . Install Containerd

   yum install -y yum-utils

   yum install -y containerd.io

   mkdir -p /etc/containerd

   containerd config default|sudo tee /etc/containerd/config.toml

   systemctl enable containerd

   systemctl restart containerd

3 . Install kubeadm, kubelet and kubectl

   yum install -y kubelet kubeadm  kubectl

   systemctl enable kubelet

#### Step Five, Set up First Master Node

We need to set up on the master node which has vip. We can use the command

ip a s ens33

to  check which master node has vip.

For instance, my vip is at 192.168.31.137 (master1). So I ran the following scripts under master1.

mkdir /usr/local/kubernetes/manifests -p

cd /usr/local/kubernetes/manifests/

vi kubeadm-config.yaml

apiServer:

certSANs:

 - master1
 - master2
 - master3
 - master.k8s.io
 - 192.168.31.158
 - 192.168.31.137
 - 192.168.31.190
 - 192.168.31.172
 - 127.0.0.1

  extraArgs:

  authorization-mode: Node,RBAC

  timeoutForControlPlane: 4m0s

  apiVersion: kubeadm.k8s.io/v1be3

  certificatesDir: /etc/kubernetes/pki

  clusterName: kubernetes

  controlPlaneEndpoint: "master.k8s.io:16443"

  controllerManager: {}

  etcd:

  local:

  dataDir: /var/lib/etcd

  imageRepository: "registry.k8s.io"

  kind: ClusterConfiguration

  kubernetesVersion: v1.27.3

  networking:

  dnsDomain: cluster.local

  podSubnet: 10.244.0.0/16

  serviceSubnet: 10.1.0.0/16 

  scheduler: {}

Notice to check your k8s version and use your own, also put your own value at certSANs, also the latest k8s apiVersion should be match, I used kubeadm.k8s.io/v1be3 here.

  Now, congratulations! We are able to init our master1 by running the following command at master1:

  kubeadm init --config kubeadm-config.yaml

  After running the the command, we will see screen like below.

![High Available Kubernetes Cluster](https://github.com/JZ0815/JZ0815.github.io/blob/main/images/k8s-master1.png?raw=true)

  Run the command in the above picture from master1.

mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config

  After running, we can check the status by the following command.

  kubectl get nodes

  kubectl get pods -n kube-system

#### Step Six, Install CNI network plugin

We can install flannel or Calico. I installed flannel here.

mkdir flannel

cd flannel

wget -c https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

apply flannel network

kubectl apply -f kube-flannel.yml 

#### Step Seven, Setup Master2 and Master3

1. Copy certificates from Master1 to Master2 and Master3.  These command runs at Master1. 

ssh root@192.168.31.190 mkdir -p /etc/kubernetes/pki/etcd

scp /etc/kubernetes/admin.conf root@192.168.31.190:/etc/kubernetes

scp /etc/kubernetes/pki/{ca.*,sa.*,front-proxy-ca.*} root@192.168.31.190:/etc/kubernetes/pki

scp /etc/kubernetes/pki/etcd/ca.* root@192.168.31.190:/etc/kubernetes/pki/etcd



ssh root@192.168.31.172 mkdir -p /etc/kubernetes/pki/etcd

scp /etc/kubernetes/admin.conf root@192.168.31.172:/etc/kubernetes

scp /etc/kubernetes/pki/{ca.*,sa.*,front-proxy-ca.*} root@192.168.31.172:/etc/kubernetes/pki

scp /etc/kubernetes/pki/etcd/ca.* root@192.168.31.172:/etc/kubernetes/pki/etcd

2. Join master2 and master3

At master2 and master3 run the following command.


kubeadm join master.k8s.io:16443 --token t5758l.eca947obnf3hpred --discovery-token-ca-cert-hash sha256:5e54df6d4cf92d13241f483074dcd7a5fe21e55ac8cfcc17b932b5ed2079d9b7 --control-plane 

These command are copied from the picture I pasted above.   --control-plane  means we added master nodes.

#### Step Eight, Setup Node1, Nod2, Node3 and Node4

1. At the four worker nodes, run the command:


kubeadm join master.k8s.io:16443 --token t5758l.eca947obnf3hpred --discovery-token-ca-cert-hash sha256:5e54df6d4cf92d13241f483074dcd7a5fe21e55ac8cfcc17b932b5ed2079d9b7

Notice this command is the same as we add a master node except there is no --control-plan parameters here.

2. After joined the four worker nodes, apply flannel network again

kubectl apply -f kube-flannel.yml 

Now we have set up our high availability kubernetes cluster. We can check the status by running kubectl get nodes in master nodes

![High Available Kubernetes Cluster](https://github.com/JZ0815/JZ0815.github.io/blob/main/images/k8s-cluster.png?raw=true)

3. Validate our setup

We can deploy a Nginx pod to validate the installations

kubectl create deployment nginx --image=nginx

kubectl expose deployment nginx --port=80 --target-port=80 --type=NodePort

kubectl get pod,svc

Congratulations, we set up a high availability kubernetes cluster. Technologies such as virtual ip, load balanceing (haproxy) are envolved. The process is challenging but paid off.













