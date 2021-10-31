# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2004"
  #config.vm.box = "roboxes/ubuntu2004"
#---------- master ----------
  config.vm.define "master_192.168.33.100" do |server|
    server.vm.network "private_network", ip: "192.168.33.100"
    server.vm.hostname = "master"
    server.vm.provider "libvirt" do |kvm|
      kvm.memory = 8192 
      kvm.cpus = 2
      kvm.machine_type = "q35"
      kvm.cpu_mode = "host-passthrough"
      kvm.kvm_hidden = true
      kvm.pci :bus => '0x06', :slot => '0x10', :function => '0x0'
    end
    server.vm.provision "shell", inline: <<-SHELL
      cat <<EOF > /etc/netplan/99_config.yaml
network:
  ethernets:
    eth2:
      dhcp4: false
      addresses:
         - 192.168.200.100/24
  version: 2
EOF
      sudo ip a add 192.168.200.100/24 dev eth2
      sudo ip link set eth2 up
      sudo swapoff -a
      #sudo systemctl mask "swap.img.swap"
      #sudo systemctl mask "dev-vda2.swap"
      sudo sed -ie "/swap/d" /etc/fstab
      sudo cat <<EOF > /etc/modprobe.d/blacklist-nouveau.conf
blacklist nouveau
options nouveau modeset=0
EOF
      sudo update-initramfs -u 
      sudo apt-get update
      sudo apt-get install -y curl
      sudo apt-get install -y docker.io
      sudo apt-get install -y sshpass
      ssh-keygen -t rsa -N '' -f ~/.ssh/id_rsa <<< y
      cat <<EOF > ~/.ssh/config
host 192.168.33.*
   StrictHostKeyChecking no
EOF
      chmod 600 ~/.ssh/config
      sshpass -p "vagrant" ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@192.168.33.101
      sshpass -p "vagrant" ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@192.168.33.102
      sudo cat <<EOF > /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
      sudo systemctl enable docker
      sudo systemctl daemon-reload
      sudo systemctl restart docker
      curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
      sudo cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
      sudo apt-get update
      sudo apt-get install -y -q kubelet kubectl kubeadm
      sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=192.168.33.100
      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config
      sudo kubectl taint nodes --all node-role.kubernetes.io/master-
      sudo kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
      token=$(sudo kubeadm token list |tail -n 1 |awk '{print $1}')
      hashkey=$(openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //')
      ssh vagrant@192.168.33.101 sudo kubeadm join 192.168.33.100:6443 --token $token --discovery-token-ca-cert-hash sha256:$hashkey
      ssh vagrant@192.168.33.102 sudo kubeadm join 192.168.33.100:6443 --token $token --discovery-token-ca-cert-hash sha256:$hashkey
      sudo kubectl label node worker1 node-role.kubernetes.io/node=worker1
      sudo kubectl label node worker2 node-role.kubernetes.io/node=worker2
      sudo sed -ie "/swap/d" /etc/fstab
      sudo cat /etc/fstab
      sudo cp -p /etc/docker/daemon.json /tmp/daemon.json
      sudo tac /tmp/daemon.json |sed -e '2s/$/,/' |sed -e '2i \ \ "insecure-registries":["192.168.33.1:5000"]'|tac > /etc/docker/daemon.json
    SHELL
  end
#---------- worker1 ----------
  config.vm.define "worker1_192.168.33.101" do |server|
    server.vm.network "private_network", ip: "192.168.33.101"
    server.vm.hostname = "worker1"
    server.vm.provider "libvirt" do |kvm|
      kvm.memory = 16384
      kvm.cpus = 2
      kvm.machine_type = "q35"
      kvm.cpu_mode = "host-passthrough"
      kvm.kvm_hidden = true
      kvm.pci :bus => '0x06', :slot => '0x10', :function => '0x4'
      kvm.pci :bus => '0x01', :slot => '0x00', :function => '0x0'
    end
    server.vm.provision "shell", inline: <<-SHELL
      cat <<EOF > /etc/netplan/99_config.yaml
network:
  ethernets:
    eth2:
      dhcp4: false
      addresses:
         - 192.168.200.101/24
  version: 2
EOF
      sudo ip a add 192.168.200.101/24 dev eth2
      sudo ip link set eth2 up
      sudo swapoff -a
      #sudo systemctl mask "swap.img.swap"
      #sudo systemctl mask "dev-vda2.swap"
      sudo sed -ie "/swap/d" /etc/fstab
      sudo cat <<EOF > /etc/modprobe.d/blacklist-nouveau.conf
blacklist nouveau
options nouveau modeset=0
EOF
      sudo update-initramfs -u 
      sudo apt-get update
      sudo apt-get install -y curl
      sudo apt-get install -y docker.io
      sudo cat <<EOF > /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
      sudo systemctl enable docker
      sudo systemctl daemon-reload
      sudo systemctl restart docker
      curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
      sudo cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
      sudo apt-get update
      sudo apt-get install -y -q kubelet kubectl kubeadm
#      cat <<EOF > /etc/rc.local
##!/bin/sh
#sudo iptables -t nat -A POSTROUTING -s 192.168.0.0/16 -j SNAT --to 192.168.33.101
#EOF
      chmod 775 /etc/rc.local
      sudo sed -ie "/swap/d" /etc/fstab
      sudo cp -p /etc/docker/daemon.json /tmp/daemon.json
      sudo tac /tmp/daemon.json |sed -e '2s/$/,/' |sed -e '2i \ \ "insecure-registries":["192.168.33.1:5000"]'|tac > /etc/docker/daemon.json
    SHELL
  end
#---------- worker2 ----------
  config.vm.define "worker2_192.168.33.102" do |server|
    server.vm.network "private_network", ip: "192.168.33.102"
    server.vm.hostname = "worker2"
    server.vm.provider "libvirt" do |kvm|
      kvm.memory = 8192 
      kvm.cpus = 2
      kvm.machine_type = "q35"
      kvm.cpu_mode = "host-passthrough"
      kvm.kvm_hidden = true
      kvm.pci :bus => '0x06', :slot => '0x11', :function => '0x0'
    end
    server.vm.provision "shell", inline: <<-SHELL
      cat <<EOF > /etc/netplan/99_config.yaml
network:
  ethernets:
    eth2:
      dhcp4: false
      addresses:
         - 192.168.200.102/24
  version: 2
EOF
      sudo ip a add 192.168.200.102/24 dev eth2
      sudo ip link set eth2 up
      sudo swapoff -a
      #sudo systemctl mask "swap.img.swap"
      #sudo systemctl mask "dev-vda2.swap"
      sudo sed -ie "/swap/d" /etc/fstab
      sudo cat <<EOF > /etc/modprobe.d/blacklist-nouveau.conf
blacklist nouveau
options nouveau modeset=0
EOF
      sudo update-initramfs -u 
      sudo apt-get update
      sudo apt-get install -y curl
      sudo apt-get install -y docker.io
      sudo cat <<EOF > /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
      sudo systemctl enable docker
      sudo systemctl daemon-reload
      sudo systemctl restart docker
      curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
      sudo cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
      sudo apt-get update
      sudo apt-get install -y -q kubelet kubectl kubeadm
#      cat <<EOF > /etc/rc.local
##!/bin/sh
#sudo iptables -t nat -A POSTROUTING -s 192.168.0.0/16 -j SNAT --to 192.168.33.102
#EOF
      chmod 775 /etc/rc.local
      sudo sed -ie "/swap/d" /etc/fstab
      sudo cp -p /etc/docker/daemon.json /tmp/daemon.json
      sudo tac /tmp/daemon.json |sed -e '2s/$/,/' |sed -e '2i \ \ "insecure-registries":["192.168.33.1:5000"]'|tac > /etc/docker/daemon.json
    SHELL
  end
#--------------------
end
