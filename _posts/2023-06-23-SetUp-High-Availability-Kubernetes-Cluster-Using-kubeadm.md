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
1. Turn Off firewall in all master and worker nodes

   systemctl stop firewalld

   systemctl disable firewalld

2. Turn Off Selinux in all master and worker nodes

   sed -i 's/enforcing/disabled/' /etc/selinux/config  # permanent

   setenforce 0  # temporary

3. Turn Off swap in all master and worker nodes

   swapoff -a  # temporary

   sed -ri 's/.*swap.*/#&/' /etc/fstab    # permanent







