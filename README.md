# multi-master-k8s-cluster

https://github.com/hub-kubernetes/kubeadm-multi-master-setup

2 machines for master, ubuntu 20.04+

2 machines for worker, ubuntu 20.04+

1 machine for loadbalancer, ubuntu 20.04+

All machines must be accessible on the network. For cloud users - single VPC for all machines

sudo privilege

ssh access from loadbalancer node to all machines (master & worker).

ssh access can be given to any account. ssh through root is not mandatory


# Load Balancer

Update your repository and your system

sudo apt-get update && sudo apt-get upgrade -y

Install haproxy

sudo apt-get install haproxy -y

Edit haproxy configuration

vi /etc/haproxy/haproxy.cfg

Add the below lines to create a frontend configuration for loadbalancer -

frontend fe-apiserver
   bind 0.0.0.0:6443
   mode tcp
   option tcplog
   default_backend be-apiserver
   
Add the below lines to create a backend configuration for master1 and master2 nodes at port 6443. Note : 6443 is the default port of kube-apiserver

backend be-apiserver
   mode tcp
   option tcplog
   option tcp-check
   balance roundrobin
   default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100

       server master1 172.31.38.35:6443 check
       server master2 172.31.34.171:6443 check

Here - master1 and master2 are the names of the master nodes and 172.31.38.35 and 172.31.34.171 are the corresponding internal IP addresses.

Restart and Verify haproxy

systemctl restart haproxy

systemctl status haproxy

Ensure haproxy is in running status.

Run nc command as below -

nc -v localhost 6443

Connection to localhost 6443 port [tcp/*] succeeded!


# setup node
```
#Create configuration file for containerd:
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

#Load modules:
sudo modprobe overlay
sudo modprobe br_netfilter

#Set system configurations for Kubernetes networking:
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

#Apply new settings:
sudo sysctl --system

#Install containerd:
sudo apt-get update && sudo apt-get install -y containerd

#Create default configuration file for containerd:
sudo mkdir -p /etc/containerd

#Generate default containerd configuration and save to the newly created default file:
sudo containerd config default | sudo tee /etc/containerd/config.toml

#Restart containerd to ensure new configuration file usage:
sudo systemctl restart containerd

#Verify that containerd is running:
sudo systemctl status containerd

#Disable swap:
sudo swapoff -a

#Install dependency packages:
sudo apt-get update && sudo apt-get install -y apt-transport-https curl

#Download and add GPG key:
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

#Add Kubernetes to repository list:
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

#Update package listings:
sudo apt-get update

#Install Kubernetes packages (Note: If you get a dpkg lock message, just wait a minute or two before trying the command again):
sudo apt-get install -y kubelet=1.24.0-00 kubeadm=1.24.0-00 kubectl=1.24.0-00

#Turn off automatic updates:
sudo apt-mark hold kubelet kubeadm kubectl
```




# Install kubeadm,kubelet and docker on master and worker nodes

In this step we will install kubelet and kubeadm on the below nodes

master1

master2

worker1

worker2

The below steps will be performed on all the below nodes.

Log in to all the 4 machines as described above

Switch as root - sudo -i

Update the repositories

apt-get update

Turn off swap

swapoff -a 

## Install kubeadm and kubelet

apt-get update && apt-get install -y apt-transport-https curl

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -

cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

apt-get update

apt-get install -y kubelet kubeadm

apt-mark hold kubelet kubeadm 

## Install container runtime - docker

sudo apt-get update

sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io -y
   
Configure kubeadm to bootstrap the cluster
   
We will start off by initializing only one master node. For this demo, we will use master1 to initialize our first control plane.

Log in to master1
   
Switch to root account - sudo -i
   
Execute the below command to initialize the cluster -
   
kubeadm init --control-plane-endpoint "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT" --upload-certs --pod-network-cidr=192.168.0.0/16 
Here, LOAD_BALANCER_DNS is the IP address or the dns name of the loadbalancer. I will use the dns name of the server, i.e. loadbalancer as the LOAD_BALANCER_DNS. In case your DNS name is not resolvable across your network, you can use the IP address for the same.

The LOAD_BALANCER_PORT is the front end configuration port defined in HAPROXY configuration. For this demo, we have kept the port as 6443.

The command effectively becomes -

kubeadm init --control-plane-endpoint "loadbalancer:6443" --upload-certs --pod-network-cidr=192.168.0.0/16 
image

Your output should look like below -

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join loadbalancer:6443 --token cnslau.kd5fjt96jeuzymzb \
    --discovery-token-ca-cert-hash sha256:871ab3f050bc9790c977daee9e44cf52e15ee37ab9834567333b939458a5bfb5 \
    --control-plane --certificate-key 824d9a0e173a810416b4bca7038fb33b616108c17abcbc5eaef8651f11e3d146

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use 
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join loadbalancer:6443 --token cnslau.kd5fjt96jeuzymzb \
    --discovery-token-ca-cert-hash sha256:871ab3f050bc9790c977daee9e44cf52e15ee37ab9834567333b939458a5bfb5 
The output consists of 3 major tasks -

Setup kubeconfig using -
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Setup new control plane (master) using
  kubeadm join loadbalancer:6443 --token cnslau.kd5fjt96jeuzymzb \
    --discovery-token-ca-cert-hash sha256:871ab3f050bc9790c977daee9e44cf52e15ee37ab9834567333b939458a5bfb5 \
    --control-plane --certificate-key 824d9a0e173a810416b4bca7038fb33b616108c17abcbc5eaef8651f11e3d146

Join worker node using
kubeadm join loadbalancer:6443 --token cnslau.kd5fjt96jeuzymzb \
    --discovery-token-ca-cert-hash sha256:871ab3f050bc9790c977daee9e44cf52e15ee37ab9834567333b939458a5bfb5 
NOTE

Your output will be different than what is provided here. While performing the rest of the demo, ensure that you are executing the command provided by your output and dont copy and paste from here.

Save the output in some secure file for future use.

Log in to master2
Switch to root - sudo -i
Check the command provided by the output of master1
You can now use the below command to add another node to the control plane -

kubeadm join loadbalancer:6443 --token cnslau.kd5fjt96jeuzymzb \
    --discovery-token-ca-cert-hash sha256:871ab3f050bc9790c977daee9e44cf52e15ee37ab9834567333b939458a5bfb5 \
    --control-plane --certificate-key 824d9a0e173a810416b4bca7038fb33b616108c17abcbc5eaef8651f11e3d146

Execute the kubeadm join command for control plane on master2
image

Your output should look like -

 This node has joined the cluster and a new control plane instance was created:

* Certificate signing request was sent to apiserver and approval was received.
* The Kubelet was informed of the new secure connection details.
* Control plane (master) label and taint were applied to the new node.
* The Kubernetes control plane instances scaled up.
* A new etcd member was added to the local/stacked etcd cluster.

Now that we have initialized both the masters - we can now work on bootstrapping the worker nodes.

Log in to worker1 and worker2
Switch to root on both the machines -  sudo -i
Check the output given by the init command on master1 to join worker node -
kubeadm join loadbalancer:6443 --token cnslau.kd5fjt96jeuzymzb \
    --discovery-token-ca-cert-hash sha256:871ab3f050bc9790c977daee9e44cf52e15ee37ab9834567333b939458a5bfb5 
Execute the above command on both the nodes -

Your output should look like -

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Configure kubeconfig on loadbalancer node
Now that we have configured the master and the worker nodes, its now time to configure Kubeconfig (.kube) on the loadbalancer node. It is completely up to you if you want to use the loadbalancer node to setup kubeconfig. kubeconfig can also be setup externally on a separate machine which has access to loadbalancer node. For the purpose of this demo we will use loadbalancer node to host kubeconfig and kubectl.

Log in to loadbalancer node
Switch to root - sudo -i
Create a directory - .kube at $HOME of root
mkdir -p $HOME/.kube
SCP configuration file from any one master node to loadbalancer node
scp master1:/etc/kubernetes/admin.conf $HOME/.kube/config

NOTE

If you havent setup ssh connection, you can manually download the file /etc/kubernetes/admin.conf from any one of the master and upload it to $HOME/.kube location on the loadbalancer node. Ensure that you change the file name as just config on the loadbalancer node.

Provide appropriate ownership to the copied file
chown $(id -u):$(id -g) $HOME/.kube/config
image

Install kubectl binary
snap install kubectl --classic
Verify the cluster
kubectl get nodes 

NAME      STATUS     ROLES    AGE     VERSION
master1   NotReady   master   21m     v1.16.2
master2   NotReady   master   15m     v1.16.2
worker1   NotReady   <none>   9m17s   v1.16.2
worker2   NotReady   <none>   9m25s   v1.16.2

Install CNI and complete installation
From the loadbalancer node execute -

kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml

This installs CNI component to your cluster.

You can now verify your HA cluster using -

kubectl get nodes 

NAME      STATUS   ROLES    AGE   VERSION
master1   Ready    master   22m   v1.16.2
master2   Ready    master   17m   v1.16.2
worker1   Ready    <none>   10m   v1.16.2
worker2   Ready    <none>   10m   v1.16.2

This concludes the demo to create a multi master cluster using kubeadm utility.
