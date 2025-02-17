# Create your own pure kubernetes cluster and machines without docker
Easy guide (fully extracted from https://kubernetes.io/) to create your own pure kubernetes cluster and machine nodes without docker using linux, kubeadm, containerd, runc and CNI Plugins.

Purpose: create simulations for official in-production kubernetes like AWS, Google Cloud Platform, Azure.

## Recommended/Minimum requiriments for a new linux machine (node)
This settings has been tested on VM (with CentOS Stream 9) hosted on a Hyper-V on Windows Server 2019

Minimum / Recommended specifications
- Storage: 20G ( 12GB is only for S.O and required packages )
- CPU: 2 cores
- RAM: 4GB /  8GB

## Step 1: UPDATE YOUR SYSTEM
```
sudo yum update -y
```

## Step 2: INSTALL REQUIRED PACKAGES
```
sudo yum install -y yum-utils
```

##  Step 3: OPEN REQUIRED PORTS
#### Step 3.1: Open node ports
- If the machine is a Master node
```
sudo firewall-cmd --zone=public --add-port=6443/tcp --permanent
sudo firewall-cmd --zone=public --add-port=2379-2380/tcp --permanent
sudo firewall-cmd --zone=public --add-port=10250/tcp --permanent
sudo firewall-cmd --zone=public --add-port=10251/tcp --permanent
sudo firewall-cmd --zone=public --add-port=10252/tcp --permanent
```

- If the machine is a Worker node
```
sudo firewall-cmd --zone=public --add-port=10250/tcp --permanent
sudo firewall-cmd --zone=public --add-port=30000-32767/tcp --permanent
```

#### Step 3.2: Open Calico ports
```
sudo firewall-cmd --zone=public --add-port=8285/udp --permanent
```

#### Step 3.3: Reload firewall
```
sudo firewall-cmd --reload
```

##  Step 4: CONTAINER RUNTIMES - INSTALL AND CONFIGURE PREREQUISITES
#### Step 4.1: Enable IPv4 packet forwarding
sysctl params required by setup, params persist across reboots
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF
```
Apply sysctl params without reboot
```
sudo sysctl --system
```

## Step 5: INSTALLING CONTAINERD
#### Step 5.1: Installing containerd
```
wget https://github.com/containerd/containerd/releases/download/v1.7.20/containerd-1.7.20-linux-amd64.tar.gz
sudo tar -C /usr/local -xvzf containerd-1.7.20-linux-amd64.tar.gz
sudo mkdir -p /usr/local/lib/systemd/system/
sudo curl -o /usr/local/lib/systemd/system/containerd.service https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
systemctl daemon-reload
systemctl enable --now containerd
```

#### Step 5.2: Installing runc
```
wget https://github.com/opencontainers/runc/releases/download/v1.1.13/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```

#### Step 5.3: Installing CNI plugins
```
wget https://github.com/containernetworking/plugins/releases/download/v1.5.1/cni-plugins-linux-amd64-v1.5.1.tgz
mkdir -p /opt/cni/bin
sudo tar -C /opt/cni/bin -xvzf cni-plugins-linux-amd64-v1.5.1.tgz
```

#### Step 5.4: Generate default containerd configuration
```
sudo mkdir -p /etc/containerd/
sudo /usr/local/bin/containerd config default | sudo tee /etc/containerd/config.toml
```

#### Step 5.5: Configuring the systemd cgroup driver
To use the systemd cgroup driver in **_/etc/containerd/config.toml_** with runc, open the file then search and set:
```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```

#### Step 5.6: Restart containerd
```
sudo systemctl restart containerd
```

## STEP 6: INSTALLING KUBEADM, KUBELET AND KUBECTL
#### Step 6.1: Set SELinux in permissive mode (effectively disabling it)
```
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

#### Step 6.2: Add the Kubernetes yum repository
This overwrites any existing configuration in /etc/yum.repos.d/kubernetes.repo
```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```

#### Step 6.3: Install kubelet, kubeadm and kubectl:
```
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

#### Step 6.4 (Optional): Enable the kubelet service before running kubeadm:
```
sudo systemctl enable --now kubelet
```

## STEP 7: DISABLE SWAP PERMAMENT
edit /etc/fstab and comment swap line
```
sudo vim /etc/fstab
```

## STEP 8 (ONLY ON MASTER NODE): START CLUSTER
#### Step 8.1: Init with podnetwork CIDR notation
```
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

#### Step 8.2: Execute the command that appears after start cluster
The command looks like
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## STEP 9: INSTALL CALICO TO MAKE PODS NETWORK WORKING
#### STEP 9.1: Execute the kubectl create:
```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml
```

#### STEP 9.2: Watch until each pod has the STATUS of Running
```
watch kubectl get pods -n calico-system
```

#### STEP 9.3: Then run:
It should return the following: node/<your-hostname> untainted
```
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

## STEP 10: Check node status
```
kubectl get nodes -o wide
```

## Appendices:
### Joining worker nodes in master node on the cluster
After repeating this guide to create a worker node, you need to create a join command token on the master node:
```
kubeadm token create --print-join-command
```
Then after token creation, copy and paste the generated command on the worker node.
### Utils
Restart all services commands
```
sudo systemctl daemon-reload
sudo systemctl restart containerd
sudo systemctl restart kubelet
```
 
### Util links:
- [Making LoadBalancer work](https://github.com/KevinOliveiraOfficial/own-kubernetes-linux-machine-guide/blob/main/LOAD_BALANCER_GUIDE.md)
- Getting started with the new Gateway API: https://gateway-api.sigs.k8s.io/guides/
