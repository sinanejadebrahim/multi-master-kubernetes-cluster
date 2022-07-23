# multi-master-kubernetes-cluster
simple way to deploy a multi master kubernetes cluster with HA proxy.

to do this we need at least 4 servers(2 master nodes - 1 worker node - 1 for ha proxy), this document was tested on July 23 2022 with ubuntu 20 - kuber v1.24.3.
### lets define our environment:
node01 192.168.1.10 <br>
node02 192.168.1.11 <br>
ha 192.168.1.20 <br>
worker1 192.168.1.30

it's also better to connect to a VPN server if you're in IRAN (:  you can use these rules to allow you to connect to your server after a vpn connection.
```
ip rule add from $(ip route get 1 | grep -Po '(?<=src )(\S+)') table 128
ip route add table 128 to $(ip route get 1 | grep -Po '(?<=src )(\S+)')/32 dev $(ip -4 route ls | grep default | grep -Po '(?<=dev )(\S+)')
ip route add table 128 default via $(ip -4 route ls | grep default | grep -Po '(?<=via )(\S+)')
```
now you can easily connect to any vpn server and you'll have ssh and other connections to your server afterwards.
<br><br>
we need a SSL tool to generate some certifictes, i'm using cloudflare SSL tool,install it on node01.

### installing cfssl on node01
```
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
chmod +x cfssl*
mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
cfssl version   #to verify installation
```

### installing HA 
```
# ssh to your ha server
add these lines to /etc/hosts , you can also do this on node01 and node02 cause we'll need it later
192.168.1.10 node01
192.168.1.11 node02

# install ha
apt-get install haproxy

# Configure HAProxy to load balance the traffic between the two Kubernetes master nodes.
# don't change anything just add these lines to the end of this file
vim /etc/haproxy/haproxy.cfg

frontend kubernetes
bind *:6443
option tcplog
mode tcp
default_backend kubernetes-master-nodes


backend kubernetes-master-nodes
mode tcp
balance roundrobin
option tcp-check
server node01 192.168.1.10:6443 check fall 3 rise 2
server node02 192.168.1.11:6443 check fall 3 rise 2

# restart ha.
systemctl restart haproxy

# you can use this to make sure ha is ok
touch node01    #on node01
touch node02    #on node02
python3 -m http.server 6443    #node01
python3 -m http.server 6443    #node02
# now if you go to HA-server-IP:6443 with your browser and refresh - it should change between node01 and node01
```
### generating our certs
```
#on node01
$ vim ca-config.json
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}

$ vim ca-csr.json
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
  {
    "C": "IE",
    "L": "Cork",
    "O": "Kubernetes",
    "OU": "CA",
    "ST": "Cork Co."
  }
 ]
}

cfssl gencert -initca ca-csr.json | cfssljson -bare ca
ls -la
# you should have ca-key.pem and ca.pem 
```

### generating certs for ETCD cluster
```
#on node01
$ vim kubernetes-csr.json
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
  {
    "C": "IE",
    "L": "Cork",
    "O": "Kubernetes",
    "OU": "Kubernetes",
    "ST": "Cork Co."
  }
 ]
}

$ cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=192.168.1.10,192.168.1.11,127.0.0.1,kubernetes.default \
  -profile=kubernetes kubernetes-csr.json | \
  cfssljson -bare kubernetes

ls -la
# you should have kubernetes-key.pem and kubernetes.pem
```
now copy all of these to your nodes
```
scp ca.pem kubernetes.pem kubernetes-key.pem root@192.168.1.10:~
scp ca.pem kubernetes.pem kubernetes-key.pem root@192.168.1.11:~
scp ca.pem kubernetes.pem kubernetes-key.pem root@192.168.1.30:~
```
### install docker on all kuber nodes.
```
apt-get install ca-certificates curl gnupg lsb-release
mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
apt update
apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```
### install kubeadm, kublet, and kubectl on all kuber nodes.
```
apt-get install -y apt-transport-https ca-certificates curl
curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```
### you also need cri-dockerd installed for kuber V1.24+ 
```
git clone https://github.com/Mirantis/cri-dockerd.git
wget https://storage.googleapis.com/golang/getgo/installer_linux
chmod +x ./installer_linux
./installer_linux
source ~/.bash_profile
cd cri-dockerd
mkdir bin
go get && go build -o bin/cri-dockerd
mkdir -p /usr/local/bin
install -o root -g root -m 0755 bin/cri-dockerd /usr/local/bin/cri-dockerd
cp -a packaging/systemd/* /etc/systemd/system
sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
systemctl daemon-reload
systemctl enable cri-docker.service
systemctl enable cri-docker.socket
```
### don't forget to disable swap on all your nodes cause for some reason kuber doesn't like swap :D
```
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab
```

### installing and configuring ETCD
```
# on node01
mkdir /etc/etcd /var/lib/etcd
mv ~/ca.pem ~/kubernetes.pem ~/kubernetes-key.pem /etc/etcd
wget https://github.com/etcd-io/etcd/releases/download/v3.3.13/etcd-v3.3.13-linux-amd64.tar.gz
tar xvzf etcd-v3.3.13-linux-amd64.tar.gz
mv etcd-v3.3.13-linux-amd64/etcd* /usr/local/bin/
# also Create an etcd systemd unit file
vim /etc/systemd/system/etcd.service

[Unit]
Description=etcd

[Service]
ExecStart=/usr/local/bin/etcd \
  --name 192.168.1.10 \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls https://192.168.1.10:2380 \
  --listen-peer-urls https://192.168.1.10:2380 \
  --listen-client-urls https://192.168.1.10:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://192.168.1.10:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster node01=https://node01:2380,node02=https://node02:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target


$ systemctl daemon-reload
systemctl enable --now etcd
# use this command to check if etcd is working
ETCDCTL_API=3 etcdctl member list

repeate this steps on node02 only replace 192.168.1.10 with 192.168.1.11.
use the member list command again and you should have a output like this.
bd8766fds66b3d3f, started, 192.168.1.10, https://192.168.1.10:2380, https://192.168.1.10:2379
f95a7656dg85c823, started, 192.168.1.11, https://192.168.1.11:2380, https://192.168.1.11:2379
```

### now we're going to initialize our master nodes.
```
on node01
vim config.yaml

apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
nodeRegistration:
  criSocket: "unix:///var/run/cri-dockerd.sock"
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: stable
apiServer:
  certSANs:
    - "192.168.1.20"
  extraArgs:
    apiserver-count: "2" 
controlPlaneEndpoint: "192.168.1.20:6443"
etcd:
  external:
    endpoints:
    - https://192.168.1.10:2379
    - https://192.168.1.11:2379
    caFile: /etc/etcd/ca.pem
    certFile: /etc/etcd/kubernetes.pem
    keyFile: /etc/etcd/kubernetes-key.pem
networking:
  podSubnet: 10.30.0.0/24
  
# use migrate to change this file to a newer and more complcated version
$ kubeadm config migrate --old-config config.yaml  --new-config new.yaml
now we initialize our cluster using the new config file
kubeadm init --config=new.yaml
```
the output will give you a kubeadm join command with --control-plane at the end,use this to join node02 as master to your cluster.<br>
also add --cri-socket unix:///var/run/cri-dockerd.sock in your join command<br>
if you faced any problems do these, then use the join command again.
```
rm ~/pki/apiserver.*
mv ~/pki /etc/kubernetes/
```
you can use the join command without the --control-plane command to join a server as worker.<br>
get the join command with $ kubeadm token create --print-join-command
## a quick recap
we need etcd - kubeadm - kubectl - kubelet - docker - cri-dockerd on our master nodes.<br>
we need  kubeadm - kubelet - docker - cri-dockerd on our worker nodes.<br>
we need a seperate server for HAproxy.<br>

feel free to recommend any fix or changes :D

