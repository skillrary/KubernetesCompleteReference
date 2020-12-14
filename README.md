# KubernetesCompleteReference

## Install Kubernetes

Use the [link](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) to install Kubernetes

### Pre-Requisite

  One or more machines running one of:
  Ubuntu 16.04+
  Debian 9+
  CentOS 7
  Red Hat Enterprise Linux (RHEL) 7
  Fedora 25+
  HypriotOS v1.0.1+
  Flatcar Container Linux (tested with 2512.3.0)
  2 GB or more of RAM per machine (any less will leave little room for your apps)
  2 CPUs or more
  Full network connectivity between all machines in the cluster (public or private network is fine)
  Unique hostname, MAC address, and product_uuid for every node. See here for more details.
  Certain ports are open on your machines. See here for more details.
  Swap disabled. You MUST disable swap in order for the kubelet to work properly.
  
### Install on CentOS

#### Letting iptables see bridged traffic

```script
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

#### Install Docker CE

```script
# (Install Docker CE)
## Set up the repository
### Install required packages
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```

```script
## Add the Docker repository
sudo yum-config-manager --add-repo \
  https://download.docker.com/linux/centos/docker-ce.repo
```

```script
# Install Docker CE
sudo yum update -y && sudo yum install -y \
  containerd.io-1.2.13 \
  docker-ce-19.03.11 \
  docker-ce-cli-19.03.11
```

```script
## Create /etc/docker
sudo mkdir /etc/docker
```

```script
# Set up the Docker daemon
cat <<EOF | sudo tee /etc/docker/daemon.json
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
EOF
```

```script
# Create /etc/systemd/system/docker.service.d
sudo mkdir -p /etc/systemd/system/docker.service.d
```

```script
# Restart Docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```

```script
sudo systemctl enable docker
```


#### Installing Kubeadm, Kubectl, Kubelet

```script
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet

```

### Creating Cluster using Kubeadm

Install the cluster using Calico Network use [link](https://docs.projectcalico.org/getting-started/kubernetes/quickstart) for installation. 

```script
# Initialize the master using the following command.
kubeadm init --pod-network-cidr=192.168.0.0/16

# Execute the following commands to configure kubectl (also returned by kubeadm init).
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install the Tigera Calico operator and custom resource definitions.
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml

# Install Calico by creating the necessary custom resource. For more information on configuration options available in this manifest
kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml

# Check for network
kubectl get pods --all-namespaces

# Confirm that you now have a node in your cluster with the following command
kubectl get nodes -o wide


# Join the worker node with the command given by kubeadm init

```




