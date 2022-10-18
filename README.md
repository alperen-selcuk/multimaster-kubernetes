# multimaster-kubernetes

## haproxy installation

```
apt-get update && apt-get install haproxy -y
```

haproxy kurduktan sonra config yapacağız. /etc/haproxy/haproxy.cfg de bulunan default confiğin altına aşağıdaki şekilde frontend ve backend ekleyeceğiz.

```
frontend masters
  bind *:6443
  mode tcp
  option tcplog
  default_backend masters
   
backend masters
    mode tcp
    option tcplog
    option tcp-check
    balance roundrobin
    default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
    server master1 <master1-ip>:6443 
    server master2 <masyer2-ip>:6443 
```

ardından servisi restart edelim. systemctl restart haproxy

bu config haproxy sunucusu 6443 portunu dinleyecek ve gelen istekleri round robin bi şekilde kuracağımız masterlera iletecek.

>>> BÜTÜN NODELARA CONTAINERD kuracağız ve pre-req leri gireceğiz.

## pre-req

swap disable

```
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

bridge ayarları

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```


## containerd kurulum

```
sudo su -

wget https://github.com/containerd/containerd/releases/download/v1.6.8/containerd-1.6.8-linux-amd64.tar.gz
tar Cxzvf /usr/local containerd-1.6.8-linux-amd64.tar.gz

wget https://github.com/opencontainers/runc/releases/download/v1.1.3/runc.amd64
install -m 755 runc.amd64 /usr/local/sbin/runc

wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz

mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz

mkdir -p /etc/containerd/
containerd config default | sudo tee /etc/containerd/config.toml

sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

curl -L https://raw.githubusercontent.com/containerd/containerd/main/containerd.service -o /etc/systemd/system/containerd.service

systemctl daemon-reload

systemctl start containerd
systemctl enable containerd
```

## kubeadm, kubelet kurulumları

```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm 
sudo apt-mark hold kubelet kubeadm 
```


## kubeadm init domain name ile birlikte

```
kubeadm init --pod-network-cidr=10.222.0.0/16 --apiserver-advertise-address=0.0.0.0 --apiserver-cert-extra-sans=k8s.dev-ops.expert
```

config dosyasını alıp cluster IP sini k8s.dev-ops.expert ile değiştireceğiz.

DNS üzerinde de k8s.dev-ops.expert için A kaydı girip HAPROXY dış IP sini göstereceğiz. 

## kubectl kurulum

daha sonra erişeceğimiz yere kubectl kuralım.

```
curl https://raw.githubusercontent.com/alperen-selcuk/kubectl-install/main/install-kubectl-helm.sh | bash -
```
