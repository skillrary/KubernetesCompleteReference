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

### Pods

- Pod run Containers (that share the pod environment)
- Pods usually have only one container, they can have sidecars
- Sidecars are for example if you have a main container that writes logs + a log scraper that collects the logs and then expose them somewhere that log scraper would be called sidecar container.
- A pod is mortal, if it dies, a new one is created. They are never brought back to life.
- Every time a new pod is spin up it gets a new ip, that is why pod ips are not reliable
- That is why services are useful.

### Replication Controller / Replica Sets

- Are constructs designed to make sure the required number of pods is always running
- Kind of replaced by deployments
- Replica Sets is how they are called inside deployments (with subtle no needed to know diffs)

### Services

- Is a simple object defined by a manifest
- Provides a stable IP and DNS for pods sitting behind it
- Loadbalances requests
- Pods belong to services using labels (for example Prod+BE+1.3, then to update just change label to 1.4, so a rollback and forward is just a matter of changing labels)

### Deployment

- Is defined in yaml as desired state
- Add features to replication controllers/sets and takes care of it
- Simple rolling updates and rollbacks (blue-green / canary)

## Minikube

- Play around with kubernetes locally (single host kubernetes cluster)


### Kubectl commands
> commonly used Kubectl commands

> you can pratice kubectl commands at [katacoda](https://www.katacoda.com/courses/kubernetes/playground) playground

```
kubectl version
kubectl cluster-info
kubectl get storageclass
kubectl get nodes
kubectl get ep kube-dns --namespace=kube-system
kubectl get persistentvolume
kubectl get  PersistentVolumeClaim --namespace default
kubectl get pods --namespace kube-system
kubectl get ep
kubectl get sa
kubectl get serviceaccount
kubectl get clusterroles
kubectl get roles
kubectl get ClusterRoleBinding
# Show Merged kubeconfig settings.
kubectl config view
kubectl config get-contexts
# Display the current-context
kubectl config current-context           
kubectl config use-context docker-desktop
kubectl port-forward service/ok 8080:8080 8081:80 -n the-project
# Delete evicted pods
kubectl get po --all-namespaces | awk '{if ($4 ~ /Evicted/) system ("kubectl -n " $1 " delete pods " $2)}'
```


> see cluster-info
```bash
kubectl cluster-info
```
> nested kubectl commands

```bash
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=servicegraph -o jsonpath='{.items[0].metadata.name}') 8082:8088
```

> kubectl proxy creates proxy server between your machine and Kubernetes API server.
By default it is only accessible locally (from the machine that started it).

```
kubectl proxy --port=8080
curl http://localhost:8080/api/
curl http://localhost:8080/api/v1/namespaces/default/pods
```

### Accessing logs
```bash
# get all the logs for a given pod:
kubectl logs my-pod-name
# keep monitoring the logs
kubectl -f logs my-pod-name
# Or if you have multiple containers in the same pod, you can do:
kubectl -f logs my-pod-name internal-container-name
# This allows users to view the diff between a locally declared object configuration and the current state of a live object.
kubectl alpha diff -f mything.yml
```

### Execute commands in running Pods
```bash
kubectl exec -it my-pod-name -- /bin/sh
```

### CI/CD
> Redeploy newly build image to existing k8s deployment
```
BUILD_NUMBER = 1.5.0-SNAPSHOT // GIT_SHORT_SHA
kubectl diff -f sample-app-deployment.yaml
kubectl -n=staging set image -f sample-app-deployment.yaml sample-app=xmlking/ngxapp:$BUILD_NUMBER
```

### Rolling back deployments
> Once you run `kubectl apply -f manifest.yml`
```bash
# To get all the deploys of a deployment, you can do:
kubectl rollout history deployment/DEPLOYMENT-NAME
# Once you know which deploy you’d like to roll back to, you can run the following command (given you’d like to roll back to the 100th deploy):
kubectl rollout undo deployment/DEPLOYMENT_NAME --to-revision=100
# If you’d like to roll back the last deploy, you can simply do:
kubectl rollout undo deployment/DEPLOYMENT_NAME
```

### Tips and Tricks
```bash
# Show resource utilization per node:
kubectl top node
# Show resource utilization per pod:
kubectl top pod
# if you want to have a terminal show the output of these commands every 2 seconds without having to run the command over and over you can use the watch command such as
watch kubectl top node
# --v=8 for debuging 
kubectl get po --v=8
```

## Services

- Sits in front of pods
- Exposes an IP that can be called to reach the PODs
- It will loadbalance between PODs automatically
- Services matches PODs via labels

#### Service Types

Type           | Description
---------------|------------
`ClusterIP`    | (default) Exposes the service on a cluster-internal IP. Choosing this value makes the service only reachable from within the cluster. This is the default ServiceType. 
`NodePort`     |  Exposes the service on each Node’s IP at a static port (the NodePort). A ClusterIP service, to which the NodePort service will route, is automatically created. You’ll be able to contact the NodePort service, from outside the cluster, by requesting <NodeIP>:<NodePort>. 
`LoadBalancer` | builds on NodePort and creates an external load-balancer (if supported in the current cloud) which routes to the clusterIP.
`ExternalName` | Maps the service to the contents of the externalName field (e.g. `foo.bar.example.com`), by returning a CNAME record with its value. No proxying of any kind is set up.



### Example

#### Iterative

```bash
kubectl expose rc <rc-name> --name=<service-name> --target-port=<port> --type=<type>
# Example: kubectl expose rc hello-rc --name=hello-svc --target-port=8080 --type=NodePort
kubectl describe svc <service-name>
kubectl delete svc <service-name>
```

#### Declatative

```yml
apiVersion: v1
kind: Service
metadata:
  name: hello-svc
  labels:
    app: hello-world
spec:
  type: NodePort
  ports:
    - port: 8080
      nodePort: 30001
      protocol: TCP
  selector:
    app: hello-world
```

- `type`: the service type.
  - `CluserIP`: Stable internal cluster IP. Default. Makes the service only available to other nodes in the same cluster.
  - `NodePort`: Exposes the app outside of the cluster by adding a cluster wide port on top of the IP
  - `LoadBalancer`: Integrates the NodePort with an existing LoadBalancer
- `port`: is the port exposed within the container. It gets mapped through the NodePort (convention something above 30000) on the whole cluster (here 30001).
- `selector`: has to match the label of the PODs/RC

```bash
kubectl create -f <path>
# example: kubectl create -f svc.yml
# kubectl delete svc hello-svc
```

### Updates / Deployments

- Usually you give the pods a version label. E.g. (app=foo;zone=prod;ver=1.0.0)
- The service matches everything but the version label. E.g. (app=foo;zone=prod)
- New version comes: spin up new pods with the new version tag. E.g. (app=foo;zone=prod;ver=2.0.0)
- Now it’s loadbalanced across the new and the old
- If you’re happy, let the service match only the new version by adding the version label to the service. E.g. (app=foo;zone=prod;ver=2.0.0)
- The old pods are still there. To rollback, you just revert the label of the service to the old version  E.g. (app=foo;zone=prod;ver=1.0.0) and it will only target the old app. No downtime.
This is basically called blue/green deployment

## Deployments

- All about rolling updates and simple rollbacks
- Deployments wrap around replication controllers (in the world of deployment called replica set)
- Deployments manage Replica Sets, Replica Sets manage Pods

### Example

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deploy
spec:
  selector:
    matchLabels:
      app: hello-world
  replicas: 10
  minReadySeconds: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
        - name: hello-pod
          imagePullPolicy: IfNotPresent
          image: nigelpoulton/pluralsight-docker-ci:latest
          ports:
            - containerPort: 8080
```

- `minReadySeconds`: let the pod run for 10 second before marking it as ready
- `strategy`: select what kind of update strategy to use (here we use rolling updates)
  - `maxUnavailable` & `maxSurge`: by setting it to 1 we tell the deployment to do them 1 by 1

```bash
kubectl create -f <path>
# example: kubectl create -f deploy.yml
kubectl describe deploy hello-deploy
```

### Updates

- Edit the yml file.
- `kubectl apply -f <file> --record` will apply the changes
- `kubectl rollout status deployment <name-of-deployment>` to watch it
- `kubectl get deploy <name-of-deployment>` check if all are available
- `kubectl rollout history deployment <name-of-deployment>` to see the versions and why it happened
- `kubectl get rs` you’ll see that you have 2 replica sets, 1 with 0 pods and another with the new pods

### Rollbacks

- `kubectl rollout undo deployment <name-of-deployment> --to-revision=<number>` (you can see revisions via `kubectl rollout history deployment <name-of-deployment>`)
- does the same as for the update but in reverse to match the desired state of the specified revision number.

## Secrets

https://kubernetes.io/docs/concepts/configuration/secret/

### Example

```yml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```

- `data`: is a key-value store with the value set as a BASE64 encoded sring.

```
kubectl apply -f ./secret.yaml
```

*Note: secrets can also be created from files:*

```bash
kubectl create secret generic <name> --from-file=<path>
```

- `name`: the name of the secret

And then retrieved with:

```bash
kubectl get secret <name> -o yaml
```

## Useful

### Image from private registry

https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/

```
docker login <registry uri>
```
```
kubectl create secret generic regcred \
    --from-file=.dockerconfigjson=<path/to/.docker/config.json> \
    --type=kubernetes.io/dockerconfigjson
```

- `<path/to/.docker/config.json>`: is usually in the users folder `Users/<username>/.docker/config.json`

OR

```bash
kubectl create secret docker-registry regcred --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>
# Example: kubectl create secret docker-registry regcred --docker-server=https://example.com/ --docker-username=docker --docker-password=password --docker-email=example@mail.com
```

You can get the secret:
```
kubectl get secret regcred --output=yaml
```

Or create it on the spot (don’t forget to add `https://` and `/v2/` to the docker server variable):
```
kubectl create secret docker-registry regcred --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>
```

Now you can add it to your pods:

```yml
…
spec:
  containers:
  …
  imagePullSecrets:
  - name: regcred
```
