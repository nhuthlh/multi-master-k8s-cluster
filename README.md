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
