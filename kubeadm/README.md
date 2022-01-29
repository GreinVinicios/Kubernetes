## Step to configure a K8s using kubeadm

### Pre requisites
a. [Virtual Box](https://www.virtualbox.org/wiki/Linux_Downloads)
b. [Vagrant](https://www.vagrantup.com/downloads)
c. [Kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

### Installation
1 - Add the following network to virtualbox
```bash
echo "* 192.168.56.0/24" > /etc/vbox/networks.conf 
```

2 - Provision the VMs
```bash
cd kubeadm/vagrant
vagrant up
```

3 - Making sure that the br_netfilter module is loaded
```bash
sudo modprobe br_netfilter
lsmod | grep br_netfilter # This command check if it is loaded
```

4 - Creating new kernel parameters
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

5 - Installing the container runtime (Docker)
[Installation steps](https://docs.docker.com/engine/install/ubuntu/)
```bash
sudo apt-get remove docker docker-engine docker.io containerd runc

sudo apt-get update

sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io

sudo usermod -aG docker $USER
# restart the node then run a docker command to test:
shutdown -r -t 0
docker run hello-world
```

6 - Installing kube tools
> kubeadm is used to bootstrap the cluster
> kubelet is used to manage pods
> kubectl is a cmd line tool
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet=1.23.2-00 kubeadm=1.23.3-00 kubectl=1.23.3-00
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet
sudo systemctl status kubelet
```

7 - Initializing master node
```bash
kubeadm config images pull
sudo kubeadm init --pod-network-cidr 10.244.0.0/16 --apiserver-advertise-address=192.168.56.2
```
> Take a note of the kubeadm output, you will need the kubeadm join command, ex:
```kubeadm join 192.168.56.2:6443 --token atu2m8.ub3fqbccq9z10f5h --discovery-token-ca-cert-hash sha256:5057c06a9879729e79decbbe40a8935a13d50f76a981d8025b47c03edbb1a084```

>Case kubeadm don't start, run command below to see the logs 
```sudo journalctl -u kubelet -b```
kubelet cgroup driver: "systemd" is different from docker cgroup driver: "cgroupfs"
Case the error is something related to above, follow the next steps
sudo vi /etc/systemd/system/multi-user.target.wants/docker.service
Modify this line
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
To
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --exec-opt native.cgroupdriver=systemd
```systemctl daemon-reload```
```systemctl restart docker```

8 - Making kubectl work for your non-root user
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

9 - Installing Pod network solution (weave)
```bash
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

9.1 - Alternative Pod network solution (flannel)
```bash
kubeadm join 10.44.175.6:6443 --token t0bpzo.zogizrmqd8lrriek --discovery-token-ca-cert-hash sha256:07ae8ea70c36ce5e95f240fda6df884ee06b4548086afb03611355b980ad06af
```

10 - Joining worker nodes
>paste kubeadm join command saved before on each worker node
```bash
sudo kubeadm join 192.168.56.2:6443 --token atu2m8.ub3fqbccq9z10f5h --discovery-token-ca-cert-hash sha256:5057c06a9879729e79decbbe40a8935a13d50f76a981d8025b47c03edbb1a084
```