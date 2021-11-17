# Server-setup
# Tools used are
Nginx, Kubeadm, helm, Docker, Java, Redis, kafka/zookeeper, ELK, Mysql, Keepalive


# Kubernetes HA Cluster using kubeadm
**Cluster 1**
**VMs**  
Master Machine count 3 ip **192.168.1.195 192.168.1.197 192.168.1.199**
Worker Machine count 3 ip **192.168.1.201 192.168.1.203 192.168.1.205**
Load Balancer count 1 ip **192.168.1.211**
VIP **192.168.1.215** port **9292**

**Cluster 2**
**VMs**  
Master Machine count 3 ip **192.168.1.196 192.168.1.198 192.168.1.200**
Worker Machine count 3 ip **192.168.1.202 192.168.1.204 192.168.1.206**
Load Balancer count 1 ip **192.168.1.212**
VIP **192.168.1.215** port **9292**

# HA Control-Plane Installation -- Stacked ETCD
Same steps for both load balancer server 211 and 212 only master ip will be changed
Step 1 - LB Configuration

Install Nginx

$sudo yum install epel-release
$sudo yum install nginx
Disable Selinux

SELINUX=disabled in the cat /etc/selinux/config and reboot the server
Start Nginx

sudo systemctl start nginx
Nginx configuration.

Create directory for loadbalancing config.
mkdir -p /etc/nginx/tcp.conf.d/
Add this directory path to the nginx config file /etc/nginx/nginx.conf
vi /etc/nginx/nginx.conf
# including directory for tcp load balancing
include /etc/nginx/tcp.conf.d/*.conf;
create config for api-server loadbalancing
vi /etc/nginx/tcp.conf.d/apiserver.conf
stream {
        upstream apiserver_read {
             server 192.168.1.195:6443;                     #--> control plane node 1 ip and kube-api port
             server 192.168.1.197:6443;                     #--> control plane node 2 ip and kube-api port
             server 192.168.1.199:6443;                     #--> control plane node 2 ip and kube-api port
        }
        server {
                listen 6443;                               # --> port on which load balancer will listen
                proxy_pass apiserver_read;
        }
}
Reload the config
nginx -s reload
Test the proxy
yum install nc -y
nc -v LOAD_BALANCER_IP PORT
Add this directory path to the nginx config file /etc/nginx/conf.d/default.conf
# including directory for ssl and proxy pass
include /etc/nginx/conf.d/*.conf;
create config for ssl key and proxy pass default.conf
vim /etc/nginx/conf.d/default.conf
server {
        listen 192.168.1.215:9292 ssl;
               ssl_certificate /home/appuser/Documents/provided-key/root.crt;              # Path where root.crt is present 
               ssl_certificate_key /home/appuser/Documents/provided-key/hydlb.key;         # Path where hydlb.key is present
               ssl_session_cache shared:SSL:20m;
               ssl_session_timeout 10m;
        location /itc-frontend {
               proxy_pass http://192.168.1.199:30013/itc-frontend;                        #port "MASTER IP":30013 frontend is running 
               proxy_redirect http://192.168.1.199:30013/itc-frontend https://192.168.1.215:9292/itc-frontend;    #redirecting 30013 frontend to VIP:9292 
        }
}
A connection refused error is expected because the apiserver is not yet running. A timeout, however, means the load balancer cannot communicate with the control plane node
**Install Keepalived**
Prerequisites
1) Two servers running RedHat 8 one for the master node and one for the backup node.
2) A root password is configured on your server.

dnf install keepalived -y
Make changes in the keepalived.conf
vim /etc/keepalived/keepalived.conf
vrrp_script chk_haproxy {
        script "killall -0 haproxy"
        interval 2
        weight 2
}

vrrp_instance VI_1 {
        interface ens192
        state MASTER
        virtual_router_id 51
        priority 101
    advert_int 1

    unicast_src_ip 192.168.1.211
         unicast_peer {
                 192.168.1.212
         }
         authentication {
                 auth_type PASS
                 Auth_pass fail123@#!
         }
        virtual_ipaddress {
                192.168.1.215/32
        }
        track_script {
                chk_haproxy
        }
}

systemctl start keepalived
systemctl enable keepalived


# Install HA K8 cluster

**Run this on all k8 server**

#Run these on all your servers that will be part of the Kubernetes cluster

#Config firewall
sudo -i
  firewall-cmd --permanent --add-port=6443/tcp
  firewall-cmd --permanent --add-port=2379-2380/tcp
  firewall-cmd --permanent --add-port=10250/tcp
  firewall-cmd --permanent --add-port=10251/tcp
  firewall-cmd --permanent --add-port=10252/tcp
  firewall-cmd --permanent --add-port=10255/tcp 
  #Also opne dynaic ports 30000 to 32767 for "NodePort" access.
  firewall-cmd --permanent --add-port=30000-32767/tcp
  firewall-cmd --zone=trusted --permanent --add-source=192.168.0.0/24
  firewall-cmd --add-masquerade --permanent
  
  #Netfilter offers various functions and operations for packet filtering, network address translation, and port translation, which provide the functionality required for directing packets through a network 
  #modprobe - program to add and remove modules from the Linux Kernel
  modprobe br_netfilter
  systemctl restart firewalld
exit

**Example**
#Add both servers to hosts file
sudo nano /etc/hosts
192.168.0.20    kube-master
192.168.0.21    kube-node1

# Docker packages are not available anymore on CentOS 8 or RHEL 8 package repositories, so run following dnf command to enable Docker CE package repository.
sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

#Install Docker
sudo dnf install docker-ce --nobest -y --allowerasing

#Start and enable the Docker daemon
sudo systemctl enable --now docker

#Add your user to the docker group
sudo usermod -aG docker $USER

#logoof and log back in
exit
ssh YOUR_ID@NODE_YOU_WERE_WORKING_ON

#Veiry docker installed correctly
docker --version
docker run hello-world

#Now we can install Kubernetes on CentOS. First, we must create a new repository:
sudo nano /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg

#Install Kubernetes
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

#Modify kubelet file
sudo nano /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS= --runtime-cgroups=/systemd/system.slice --kubelet-cgroups=/systemd/system.slice

#Start the Kubernetes service
sudo systemctl enable --now kubelet

#Now we’re going to have to su to the root user and then create a new file (to help configure iptables):
sudo -i
  nano /etc/sysctl.d/k8s.conf
  net.bridge.bridge-nf-call-ip6tables = 1
  net.bridge.bridge-nf-call-iptables = 1

  #Load the new configuration
  sysctl --system
  
  #Disable swap
  sudo swapoff -a
  #Also premanently disable swap
  sudo nano /etc/fstab
      #/dev/mapper/cl-swap

  #Create a docker Daemon File  
  nano /etc/docker/daemon.json
  {
    "exec-opts": ["native.cgroupdriver=systemd"],
    "log-driver": "json-file",
    "log-opts": {
      "max-size": "100m"
    },
    "storage-driver": "overlay2",
    "storage-opts": [
      "overlay2.override_kernel_check=true"
    ]
  }   

  mkdir -p /etc/systemd/system/docker.service.d
  systemctl daemon-reload
  systemctl restart docker
exit


# sudo firewall-cmd --permanent --add-port=63/tcp
  # sudo firewall-cmd --permanent --add-port=63/udp
  # sudo firewall-cmd --permanent --add-port=67/tcp
  # sudo firewall-cmd --permanent --add-port=67/udp
  # sudo firewall-cmd --permanent --add-port=68/tcp
  # sudo firewall-cmd --permanent --add-port=68/udp
  # firewall-cmd --permanent --add-port=8472/udp
  # firewall-cmd --add-masquerade --permanent
  
**Run only on master**
kubeadm init --control-plane-endpoint="**Load Balancer IP:6443**" --upload-certs --apiserver-advertise-address=**Master IP**

**copy the master join output
in that add
--apiserver-advertise-address=Master IP
in the end**

mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

install weave net/calico 

kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"  #weave net

