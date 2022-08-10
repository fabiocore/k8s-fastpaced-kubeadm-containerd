## Fast paced guide to install Kubernetes 1.24.3 with Kubeadm and Containerd on Ubuntu 20.04


This is my recipe to install a fully operational Kubernetes cluster, with 1 master and 1 worker node, using Kubeadm and Containerd on Ubuntu 20.04.


## Prerequisites

- 2 x Amazon t2.medium or similar machines with 2 vCPUs and 4 Gi RAM
- Ubuntu 20.04
- Full access between the master and worker node
- SSH access to master and worker node


## Let's do it!

**Do the steps below for both machines until the last steps!**


#### Configuring the operating system

Run the following commands in the master and worker node:

```bash
sudo apt update && sudo apt upgrade -y
sudo hostnamectl set-hosname <HOSTNAME>
sudo reboot
```

**[Notes]**: For hostname, I've used:

Master = k8s1-master
Worker = k8s1-worker1


#### Disable swapp

```bash
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
sudo swapoff -a
```


#### Install Kubeadm

```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update -y
sudo apt -y install vim git curl wget kubelet=1.24.3-00 kubeadm=1.24.3-00 kubectl=1.24.3-00
sudo apt-mark hold kubelet kubeadm kubectl
```


#### Load br_netfilter and configure iptables

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```


#### Install and set up Containerd

**[Containerd] Install**

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update -y
sudo apt install -y containerd.io
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

#### [Containerd] Set up required sysctl params

```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sudo sysctl --system
```

**[Containerd] Module config**

```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```

**[Containerd] Start and enable on boot**

```bash
sudo systemctl daemon-reload
sudo systemctl restart containerd
sudo systemctl enable containerd
```


#### Pull Kubernetes container images (MASTER ONLY!)

```bash
sudo kubeadm config images pull --cri-socket /run/containerd/containerd.sock --kubernetes-version v1.24.3
```


#### Run Kubeadm init (MASTER ONLY!)

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --upload-certs --kubernetes-version=v1.24.3 --control-plane-endpoint=<MASTER-IP_ADDR> --cri-socket /run/containerd/containerd.sock
```

**ATTENTION**: Change => <MASTER_IPADDR> <= 

- Get the master ip address using  `ip a` or `ifconfig` command.


The output should looks like below:

```txt
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf
  sudo chmod 744 /etc/kubernetes/admin.conf # don't forget about this, not default

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 172.31.20.245:6443 --token j5xrj9.8l6cpqsd1qgusuvw \
	--discovery-token-ca-cert-hash sha256:fa4bc6764374b2224569c11639fafa3f73d7d2f8d74f4c8184e902bd3cc9dc2d \
	--control-plane --certificate-key 9a083fe153e54e5623af7075fac340823629bf77755d5771c6c9c8accfdddb65

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

sudo kubeadm join 172.31.20.245:6443 --token j5xrj9.8l6cpqsd1qgusuvw \
	--discovery-token-ca-cert-hash sha256:fa4bc6764374b2224569c11639fafa3f73d7d2f8d74f4c8184e902bd3cc9dc2d
```


#### Export the kubeconfig file and install Flannel CNI (MASTER ONLY!)

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl apply -f https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml
```


**Optional if you are the root**

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
sudo chmod 744 /etc/kubernetes/admin.conf
```


#### Run the join command on the WORKER NODE

```bash
sudo kubeadm join 172.31.20.245:6443 --token j5xrj9.8l6cpqsd1qgusuvw \
	--discovery-token-ca-cert-hash sha256:fa4bc6764374b2224569c11639fafa3f73d7d2f8d74f4c8184e902bd3cc9dc2d
```


**READY TO PLAY! 😉**


## References

[Kubernetes 1.23 + containerd](https://kubesimplify.com/kubernetes-containerd-setup) 👊
