# GPU Operator with preinstalled driver in Host (the Case #2 below)

*** But now I am in the failure of nvidia-driver-daemonset. ***
-------------------------------------------------------------------
gpu-operator-resources   nvidia-driver-daemonset-wvwmp                                     0/1     CrashLoopBackOff   4 (75s ago)   9m48s
-------------------------------------------------------------------


-One master node (No GPU machine)

-Two worker nodes (GPU machine and CPU machine)

https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/getting-started.html#chart-customization-options

| # | Scenario | Nvidia Driver | Nvidia Toolkit |
| --- | --- | --- | --- |
| #1 | No GPU operator | In Host | In Host |
| #2 | GPU Operator (default) | DaemonSet | DaemonSet |
| #3 | GPU Operator (w/ driver.enabled=false) | In Host | DaemonSet |
| #4 | GPU Operator (w/ toolkit.enabled=false) | DaemonSet | In Host |

# 1. Master node (no GPU machine)
# 1-1. Disable Swapping and Blacklisting Nouveau driver
```
$ uname -a
Linux worker 5.11.0-27-generic #29~20.04.1-Ubuntu SMP Wed Aug 11 15:58:17 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
$ sudo vi /etc/fstab
# comment out like below:
#/swapfile                                 none            swap    sw              0       0
-----
$ sudo swapoff -a
$ sudo systemctl mask "swapfile.swap"
Created symlink /etc/systemd/system/swapfile.swap → /dev/null.
```
The nouveau driver for NVIDIA GPUs must be blacklisted before starting the GPU Operator. 
Create a file at /etc/modprobe.d/blacklist-nouveau.conf with the following contents:
```
$ sudo vi /etc/modprobe.d/blacklist-nouveau.conf
blacklist nouveau
options nouveau modeset=0

$ sudo update-initramfs -u
```
# 1-2. Install Curl
```
$ sudo apt-get install curl
```

It's convenient if you clone this image thru KVM. 
After booting , do "hostnamectl set-hostname <hostname>" then you can rename the hostname.


# 1-3. Install Docker-CE
```
$ curl https://get.docker.com | sh \
&& sudo systemctl --now enable docker
```
# 1-4. Configuring about using systemd instead of cgroups.
```
$ cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

$ sudo systemctl enable docker
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
```
# 1-5. Install kubernetes (Necessary Two CPU cores for k8s)
```
$ sudo apt-get update \
&& sudo apt-get install -y apt-transport-https curl

$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
$ cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

$ sudo apt-get update \
&& sudo apt-get install -y -q kubelet kubectl kubeadm \
&& sudo kubeadm init --pod-network-cidr=192.168.0.0/16

Copy the output below:
********************************************************************************************************
kubeadm join 192.168.122.147:6443 --token xxxxxxxxxxxxxxxxxxxxxxx \
	--discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
********************************************************************************************************

$ mkdir -p $HOME/.kube \
&& sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config \
&& sudo chown $(id -u):$(id -g) $HOME/.kube/config

$ kubectl taint nodes --all node-role.kubernetes.io/master-
node/master untainted

$ kubectl get nodes
NAME     STATUS     ROLES                  AGE     VERSION
master   NotReady   control-plane,master   8m41s   v1.22.1
```

# 2. Worker node1 (GPU machine) and Worker node2 (CPU macine)
# 2-1. Disable Swapping and Blacklisting Nouveau driver
Same as #1-1.

# 2-2. Install Nvidia-driver (Only GPU machine)
Skip. I will use DaemonSet as a driver.

# 2-3. Install Curl
```
$ sudo apt-get install curl
```
# 2-4. Install Docker-CE
```
$ curl https://get.docker.com | sh \
&& sudo systemctl --now enable docker
```
# 2-5. Configuring about using systemd instead of cgroups.
```
$ cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

$ sudo systemctl enable docker
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
```
# 2-6. Install kubernetes
```
$ sudo apt-get update \
&& sudo apt-get install -y apt-transport-https curl

$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
$ cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

$ sudo apt-get update \
&& sudo apt-get install -y -q kubelet kubectl kubeadm
```
# 2-7. Join cluster
```
$ sudo kubeadm join 192.168.122.147:6443 --token xxxxxxxxxxxxxxxxxxxxxxx \
	--discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

# 3. Master node (no GPU machine)
# 3-1. Comfirm the nodes in the cluster at Master node
```
$ kubectl get nodes
NAME      STATUS     ROLES                  AGE     VERSION
master    NotReady   control-plane,master   26m     v1.22.1
worker1   NotReady   <none>                 8m11s   v1.22.1
worker2   NotReady   <none>                 2m52s   v1.22.1
```
# 3-2. Label node at Master node
```
$ kubectl label node worker1 node-role.kubernetes.io/node=worker1
node/worker1 labeled
$ kubectl label node worker2 node-role.kubernetes.io/node=worker2
node/worker2 labeled

$ kubectl get nodes
NAME      STATUS     ROLES                  AGE     VERSION
master    NotReady   control-plane,master   26m     v1.22.1
worker1   NotReady   node                   9m6s    v1.22.1
worker2   NotReady   node                   3m47s   v1.22.1
```

# 3-3. Install contoller Pods at Master node 
```
$ kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

$ kubectl get nodes
NAME      STATUS   ROLES                  AGE     VERSION
master    Ready    control-plane,master   29m     v1.22.1
worker1   Ready    node                   11m     v1.22.1
worker2   Ready    node                   6m20s   v1.22.1
```

Images in Master node
```
master:~/Desktop$ sudo docker images
REPOSITORY                           TAG       IMAGE ID       CREATED        SIZE
k8s.gcr.io/kube-apiserver            v1.22.1   f30469a2491a   2 weeks ago    128MB
k8s.gcr.io/kube-controller-manager   v1.22.1   6e002eb89a88   2 weeks ago    122MB
k8s.gcr.io/kube-scheduler            v1.22.1   aca5ededae9c   2 weeks ago    52.7MB
k8s.gcr.io/kube-proxy                v1.22.1   36c4ebbc9d97   2 weeks ago    104MB
calico/node                          v3.20.0   5ef66b403f4f   5 weeks ago    170MB
calico/pod2daemon-flexvol            v3.20.0   5991877ebc11   5 weeks ago    21.7MB
calico/cni                           v3.20.0   4945b742b8e6   5 weeks ago    146MB
k8s.gcr.io/etcd                      3.5.0-0   004811815584   2 months ago   295MB
k8s.gcr.io/coredns/coredns           v1.8.4    8d147537fb7d   3 months ago   47.6MB
k8s.gcr.io/pause                     3.5       ed210e3e4a5b   5 months ago   683kB
```

Images in Worker1 and Worker2 node
```
worker1:~/Desktop$ sudo docker images
REPOSITORY                  TAG       IMAGE ID       CREATED        SIZE
k8s.gcr.io/kube-proxy       v1.22.1   36c4ebbc9d97   2 weeks ago    104MB
calico/node                 v3.20.0   5ef66b403f4f   5 weeks ago    170MB
calico/pod2daemon-flexvol   v3.20.0   5991877ebc11   5 weeks ago    21.7MB
calico/cni                  v3.20.0   4945b742b8e6   5 weeks ago    146MB
k8s.gcr.io/pause            3.5       ed210e3e4a5b   5 months ago   683kB
```

# 4. Master node (no GPU machine)
# 4-1. Install Helm chart at Master node
```
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 \
&& chmod 700 get_helm.sh \
&& ./get_helm.sh

$ helm repo add nvidia https://nvidia.github.io/gpu-operator \
&& helm repo update

$ helm install --wait --generate-name \
nvidia/gpu-operator
```
