# kubernetes install

kubernetes installation guideline

## Operative system installation

### Ubuntu 22.04 server

1. Update the installer

2. Select Openssh server and allow password

3. Upgrade packages
   
   ```bash
   sudo apt update
   DEBIAN_FRONTEND=noninteractive sudo apt upgrade -y
   sudo reboot
   ```

## System tune for Kubernetes

### Disable swap

```bash
sudo sed -i 's#^/swap.img#\#/swap.img#' /etc/fstab
sudo swapoff -a
```

### Configure required modules

First, load two modules in the current running environment and configure them to load on boot

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

### Configure required sysctl to persist across system reboots

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

#### Apply sysctl parameters without rebooting to current running environment

```bash
sudo sysctl --system
```
### Configure ufw firewall

#### Master nodes
```bash
sudo ufw allow "OpenSSH"
sudo ufw allow 6443/tcp
sudo ufw allow 2379:2380/tcp
sudo ufw allow 10250/tcp
sudo ufw allow 10259/tcp
sudo ufw allow 10257/tcp
sudo ufw allow 179/tcp
sudo ufw allow 4789/udp
sudo ufw allow 4789/tcp
sudo ufw allow 2379/tcp
sudo ufw enable
sudo ufw status
```
#### Worker nodes
```bash
sudo ufw allow "OpenSSH"
sudo ufw allow 10250/tcp
sudo ufw allow 30000:32767/tcp
sudo ufw allow 179/tcp
sudo ufw allow 4789/udp
sudo ufw allow 4789/tcp
sudo ufw allow 2379/tcp
sudo ufw enable
sudo ufw status
```

### Install containerd

```bash
sudo apt install -y \
    gnupg2 \
    apt-transport-https
```

```bash
curl -sS https://download.docker.com/linux/ubuntu/gpg |\
     gpg --dearmor | \
     sudo tee /usr/share/keyrings/docker.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/docker.gpg arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt update
sudo apt install containerd.io -y
```

### Configure containerd

```bash
sudo systemctl stop containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl start containerd
sudo systemctl enable containerd
```

## Kubeadm install

```bash
curl -sS https://packages.cloud.google.com/apt/doc/apt-key.gpg | \
    gpg --dearmor | \
    sudo tee /usr/share/keyrings/kubernetes.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/kubernetes.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | \
sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
export K8S_VERSION="1.26.3-00"
sudo apt install -y \
    kubeadm="${K8S_VERSION}" \
    kubelet="${K8S_VERSION}" \
    kubectl="${K8S_VERSION}"
sudo apt-mark hold kubelet kubeadm kubectl
kubeadm completion bash | sudo tee /etc/bash_completion.d/kubeadm &>/dev/null
kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl &>/dev/null
source <(kubeadm completion bash)
source <(kubectl completion bash)
sudo systemctl enable --now kubelet
```

### Helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
rm get_helm.sh
helm completion bash | sudo tee /etc/bash_completion.d/helm &>/dev/null
source <(helm completion bash)
```

### install k9s
```bash
export K9S_VERSION="$(curl -sL https://api.github.com/repos/kube-vip/kube-vip/releases | jq -r ".[0].name")"
wget https://github.com/derailed/k9s/releases/download/v0.27.3/k9s_Linux_amd64.tar.gz -O /tmp/k9s_Linux_amd64.tar.gz
cd /tmp
tar -xaf k9s_Linux_amd64.tar.gz
test -d ~/.local/bin && mv k9s ~/.local/bin/k9s || sudo mv k9s /usr/local/bin/k9s
cd -
```

### KubeVIP

- put all the interfaces to `etho` using netplan yaml on each machine

- Put nodes names and ip on `/etc/hosts` on each machine

- rob-node-00 -> 192.168.2.100

- rob-node-01 -> 192.168.2.101

- rob-node-02 -> 192.168.2.102

- DHCP range 192.168.2.50-192.168.2.200

- Virtual IP -> 192.168.2.10

- Do the kube-vip config on each node

```bash
sudo apt install -y jq
export VIP=192.168.2.10
export INTERFACE=eth0
export KVVERSION=$(curl -sL https://api.github.com/repos/kube-vip/kube-vip/releases | jq -r ".[0].name")
alias kube-vip="sudo ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION; sudo ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip /kube-vip"
kube-vip manifest pod \
    --interface $INTERFACE \
    --address $VIP \
    --controlplane \
    --services \
    --arp \
    --leaderElection | sudo tee /etc/kubernetes/manifests/kube-vip.yaml
```

### Kubeadm init node

on node rob-node-00

```bash
sudo kubeadm init \
--pod-network-cidr=10.244.0.0/16 \
--apiserver-advertise-address=192.168.2.100 \
--control-plane-endpoint 192.168.2.10:6443 \
--cri-socket unix:///var/run/containerd/containerd.sock \
--upload-certs \
--apiserver-cert-extra-sans=127.0.0.1,rob-node-00,192.168.2.100,rob-node-01,192.168.2.101,rob-node-02,192.168.2.102
```
### Create cluster configuration for kubectl
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Taint the master node allow workload
```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### Install calico CNI
```bash
export CALICO_VERSION=3.25.1
curl https://raw.githubusercontent.com/projectcalico/calico/v${CALICO_VERSION}/manifests/tigera-operator.yaml -O
kubectl create -f tigera-operator.yaml
curl https://raw.githubusercontent.com/projectcalico/calico/v${CALICO_VERSION}/manifests/custom-resources.yaml -O
sed -i 's#cidr: 192.168.0.0/16#cidr: 10.244.0.0/16#' custom-resources.yaml
kubectl create -f custom-resources.yaml
```

### Install calicoctl
```bash
curl -L https://github.com/projectcalico/calico/releases/latest/download/calicoctl-linux-amd64 -o calicoctl
chmod +x ./calicoctl
test -d ~/.local/bin && mv calicoctl .local/bin/calicoctl || sudo mv calicoctl /usr/local/bin/calicoctl
```

### install metrics server
```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm install --set 'args={--kubelet-insecure-tls}' --namespace kube-system metrics metrics-server/metrics-server
```

### install metallb
```bash
helm repo add metallb https://metallb.github.io/metallb
helm install --create-namespace --namespace metallb-system metallb metallb/metallb
```
#### Configure metallb
```bash
export METALLB_IP_POD="192.168.2.210-192.168.2.219"
cat <<EOF > metallb-config.yaml
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
    - ${METALLB_IP_POD}
  autoAssign: true
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool

EOF
kubectl apply -f metallb-config.yaml
```

### liqo install
```bash
curl --fail -LS "https://github.com/liqotech/liqo/releases/download/v0.7.2/liqoctl-linux-amd64.tar.gz" | tar -xz
sudo install -o root -g root -m 0755 liqoctl /usr/local/bin/liqoctl
liqoctl completion bash | sudo tee /etc/bash_completion.d/liqoctl &>/dev/null
source <(liqoctl completion bash)
rm liqoctl
rm LICENSE
```



