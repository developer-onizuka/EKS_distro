# EKS_distro

# 1. Install CentOS/8
```
# -*- mode: ruby -*-
# vi: set ft=ruby :
```
Vagrant.configure("2") do |config|
  config.vm.box = "centos/8"
#-------------------- master --------------------#
  config.vm.define "master_192.168.33.120" do |server|
    server.vm.network "private_network", ip: "192.168.33.120"
    server.vm.hostname = "master"
    server.vm.provider "libvirt" do |kvm|
      kvm.memory = 8192
      kvm.cpus = 2
      kvm.machine_type = "q35"
      kvm.cpu_mode = "host-passthrough"
      kvm.kvm_hidden = true
      kvm.pci :bus => '0x06', :slot => '0x10', :function => '0x0'
    end
    server.vm.provision "shell", privileged: true, inline: <<-SHELL
      sed -i -e 's/PasswordAuthentication\ no/PasswordAuthentication\ yes/g' /etc/ssh/sshd_config
      systemctl restart sshd
      sudo swapoff -a
      sudo sed -ie "/swap/d" /etc/fstab
      sudo yum install -y yum-utils
      sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
      sudo yum install -y docker-ce docker-ce-cli containerd.io
      sudo mkdir -p /etc/docker
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
      sudo systemctl enable docker && sudo systemctl start docker
      sudo usermod -aG docker vagrant
      setenforce 0
      sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
      cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
      sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
      sudo systemctl enable --now kubelet
      sudo kubeadm init --pod-network-cidr=10.10.0.0/16 --apiserver-advertise-address=192.168.33.120
      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config
      sudo kubectl taint nodes --all node-role.kubernetes.io/master-
      token=$(sudo kubeadm token list |tail -n 1 |awk '{print $1}')
      hashkey=$(openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //')
      ssh vagrant@192.168.33.121 sudo kubeadm join 192.168.33.120:6443 --token $token --discovery-token-ca-cert-hash sha256:$hashkey
      ssh vagrant@192.168.33.122 sudo kubeadm join 192.168.33.121:6443 --token $token --discovery-token-ca-cert-hash sha256:$hashkey
      sudo kubectl label node worker1 node-role.kubernetes.io/node=worker1
      sudo kubectl label node worker2 node-role.kubernetes.io/node=worker2
      #sudo kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
      sudo kubectl apply -f https://github.com/antrea-io/antrea/releases/download/v1.2.3/antrea.yml
      #sudo kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
    SHELL
  end
#-------------------- worker1 --------------------#
  config.vm.define "worker1_192.168.33.121" do |server|
    server.vm.network "private_network", ip: "192.168.33.121"
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
    server.vm.provision "shell", privileged: true, inline: <<-SHELL
      sed -i -e 's/PasswordAuthentication\ no/PasswordAuthentication\ yes/g' /etc/ssh/sshd_config
      systemctl restart sshd
      sudo swapoff -a
      sudo sed -ie "/swap/d" /etc/fstab
      sudo yum install -y yum-utils
      sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
      sudo yum install -y docker-ce docker-ce-cli containerd.io
      sudo apt-get install -y sshpass
      ssh-keygen -t rsa -N '' -f ~/.ssh/id_rsa <<< y
      cat <<EOF > ~/.ssh/config
host 192.168.33.*
   StrictHostKeyChecking no
EOF
      chmod 600 ~/.ssh/config
      sshpass -p "vagrant" ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@192.168.33.121
      sshpass -p "vagrant" ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@192.168.33.122
      sudo mkdir -p /etc/docker
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
      sudo systemctl enable docker && sudo systemctl start docker
      sudo usermod -aG docker vagrant
      setenforce 0
      sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
      cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
      sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
      sudo systemctl enable --now kubelet
    SHELL
  end
#-------------------- worker2 --------------------#
  config.vm.define "worker2_192.168.33.122" do |server|
    server.vm.network "private_network", ip: "192.168.33.122"
    server.vm.hostname = "worker2"
    server.vm.provider "libvirt" do |kvm|
      kvm.memory = 16384
      kvm.cpus = 2
      kvm.machine_type = "q35"
      kvm.cpu_mode = "host-passthrough"
      kvm.kvm_hidden = true
      kvm.pci :bus => '0x06', :slot => '0x11', :function => '0x0'
    end
    server.vm.provision "shell", privileged: true, inline: <<-SHELL
      sed -i -e 's/PasswordAuthentication\ no/PasswordAuthentication\ yes/g' /etc/ssh/sshd_config
      systemctl restart sshd
      sudo swapoff -a
      sudo sed -ie "/swap/d" /etc/fstab
      sudo yum install -y yum-utils
      sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
      sudo yum install -y docker-ce docker-ce-cli containerd.io
      sudo mkdir -p /etc/docker
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
      sudo systemctl enable docker && sudo systemctl start docker
      sudo usermod -aG docker vagrant
      setenforce 0
      sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
      cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
      sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
      sudo systemctl enable --now kubelet
    SHELL
  end
#-------------------------------------------------#
end
```
