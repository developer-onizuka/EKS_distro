# EKS_distro

# 1. Install CentOS/8
```
# -*- mode: ruby -*-
# vi: set ft=ruby :

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


```
    1  cat /etc/docker/daemon.json 
    2  kubeadm init --pod-network-cidr=10.10.0.0/16
    3  kubectl get nodes
    4  kubectl get nodes
    5    mkdir -p $HOME/.kube
    6    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    7    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    8  kubectl get nodes
    9  kubectl apply -f https://docs.projectcalico.org/v3.11/manifests/calico.yaml
   10  kubectl get nodes
   11  kubectl get nodes
   12  watch -x kubectl get pods -A
   13  kubectl delete -f https://docs.projectcalico.org/v3.11/manifests/calico.yaml
   14  watch -x kubectl get pods -A
   15  kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
   16  watch -x kubectl get pods -A
   17  kubectl get nodes
   18  kubectl delete node worker1
   19  kubectl delete node worker2
   20  watch -x kubectl get pods -A
   21  watch -x kubectl get pods -A -o wide
   22  kubeadm reset
   23  yum remove -y kubelet kubeadm kubectl --disableexcludes=kubernetes
   24  yum -y update
   25  yum install -y yum-utils device-mapper-persistent-data lvm2 wget
   26  sed -i '/^ExecStart/ s/$/ --exec-opt native.cgroupdriver=systemd/' /usr/lib/systemd/system/docker.service
   27  systemctl daemon-reload
   28  systemctl restart docker
   29  systemctl status docker.service
   30  systemctl restart docker
   31  systemctl daemon-reload
   32  systemctl status docker.service
   33  vi /usr/lib/systemd/system/docker.service 
   34  vi /usr/lib/systemd/system/docker.service 
   35  systemctl status docker.service
   36  systemctl daemon-reload
   37  systemctl restart docker
   38  mkdir -p /opt/cni/bin
   39  wget -q https://distro.eks.amazonaws.com/kubernetes-1-18/releases/1/artifacts/plugins/v0.8.7/cni-plugins-linux-amd64-v0.8.7.tar.gz
   40  tar zxf cni-plugins-linux-amd64-v0.8.7.tar.gz -C /opt/cni/bin/
   41  wget -q https://distro.eks.amazonaws.com/kubernetes-1-18/releases/1/artifacts/kubernetes/v1.18.9/bin/linux/amd64/kubeadm
   42  wget -q https://distro.eks.amazonaws.com/kubernetes-1-18/releases/1/artifacts/kubernetes/v1.18.9/bin/linux/amd64/kubelet
   43  wget -q https://distro.eks.amazonaws.com/kubernetes-1-18/releases/1/artifacts/kubernetes/v1.18.9/bin/linux/amd64/kubectl
   44  mv kubeadm kubelet kubectl /usr/bin/
   45  chmod +x /usr/bin/kubeadm /usr/bin/kubelet /usr/bin/kubectl
   46  yum -y install conntrack ebtables socat
   47  cat /etc/docker/daemon.json 
   48  cat <<EOF > /etc/sysconfig/kubelet
   49  KUBELET_EXTRA_ARGS='--cgroup-driver=systemd'
   50  EOF
   51  mkdir -p /etc/kubernetes/manifests
   52  mkdir -p /usr/lib/systemd/system/kubelet.service.d
   53  cat <<EOF > /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
   54  # Note: This dropin only works with kubeadm and kubelet v1.11+
   55  [Service]
   56  Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
   57  Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
   58  # This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
   59  EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
   60  # This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
   61  # the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
   62  EnvironmentFile=-/etc/sysconfig/kubelet
   63  ExecStart=
   64  ExecStart=/usr/bin/kubelet \$KUBELET_KUBECONFIG_ARGS \$KUBELET_CONFIG_ARGS \$KUBELET_KUBEADM_ARGS \$KUBELET_EXTRA_ARGS
   65  EOF
   66  cat <<EOF > /usr/lib/systemd/system/kubelet.service
   67  [Unit]
   68  Description=kubelet: The Kubernetes Node Agent
   69  Documentation=https://kubernetes.io/docs/
   70  Wants=network-online.target
   71  After=network-online.target
   72  [Service]
   73  ExecStart=/usr/bin/kubelet
   74  Restart=always
   75  StartLimitInterval=0
   76  RestartSec=10
   77  [Install]
   78  WantedBy=multi-user.target
   79  EOF
   80  systemctl enable kubelet
   81  docker pull public.ecr.aws/eks-distro/etcd-io/etcd:v3.4.14-eks-1-18-1
   82  docker pull public.ecr.aws/eks-distro/kubernetes/pause:v1.18.9-eks-1-18-1
   83  docker pull public.ecr.aws/eks-distro/kubernetes/kube-scheduler:v1.18.9-eks-1-18-1
   84  docker pull public.ecr.aws/eks-distro/kubernetes/kube-proxy:v1.18.9-eks-1-18-1
   85  docker pull public.ecr.aws/eks-distro/kubernetes/kube-apiserver:v1.18.9-eks-1-18-1
   86  docker pull public.ecr.aws/eks-distro/kubernetes/kube-controller-manager:v1.18.9-eks-1-18-1
   87  docker pull public.ecr.aws/eks-distro/coredns/coredns:v1.7.0-eks-1-18-1
   88  docker tag public.ecr.aws/eks-distro/kubernetes/pause:v1.18.9-eks-1-18-1 public.ecr.aws/eks-distro/kubernetes/pause:3.2
   89  docker tag public.ecr.aws/eks-distro/coredns/coredns:v1.7.0-eks-1-18-1 public.ecr.aws/eks-distro/kubernetes/coredns:1.6.7
   90  cat <<EOF > kubeadm-config.yaml
   91  apiVersion: kubeadm.k8s.io/v1beta2
   92  kind: ClusterConfiguration
   93  networking:
   94    podSubnet: "192.168.0.0/16"
   95  etcd:
   96    local:
   97      imageRepository: public.ecr.aws/eks-distro/etcd-io
   98      imageTag: v3.4.14-eks-1-18-1
   99      extraArgs:
  100        listen-peer-urls: "https://0.0.0.0:2380"
  101        listen-client-urls: "https://0.0.0.0:2379"
  102  imageRepository: public.ecr.aws/eks-distro/kubernetes
  103  kubernetesVersion: v1.18.9-eks-1-18-1
  104  EOF
  105  vi kubeadm-config.yaml 
  106  kubeadm init --config kubeadm-config.yaml
  ```
  ```
    2  kubeadm join 192.168.121.197:6443 --token jhiv0b.l8i0cx1bnj2g0mqz --discovery-token-ca-cert-hash sha256:5cafadbe1203b5682e038eb8faf14ddc9201df5783a1890cc4786153ab6360cf
    3  kubeadm reset
    4  yum remove -y kubelet kubeadm kubectl --disableexcludes=kubernetes
    5  yum -y update
    6  yum install -y yum-utils device-mapper-persistent-data lvm2 wget
    7  mkdir -p /opt/cni/bin
    8  wget -q https://distro.eks.amazonaws.com/kubernetes-1-18/releases/1/artifacts/plugins/v0.8.7/cni-plugins-linux-amd64-v0.8.7.tar.gz
    9  tar zxf cni-plugins-linux-amd64-v0.8.7.tar.gz -C /opt/cni/bin/
   10  wget -q https://distro.eks.amazonaws.com/kubernetes-1-18/releases/1/artifacts/kubernetes/v1.18.9/bin/linux/amd64/kubeadm
   11  wget -q https://distro.eks.amazonaws.com/kubernetes-1-18/releases/1/artifacts/kubernetes/v1.18.9/bin/linux/amd64/kubelet
   12  wget -q https://distro.eks.amazonaws.com/kubernetes-1-18/releases/1/artifacts/kubernetes/v1.18.9/bin/linux/amd64/kubectl
   13  mv kubeadm kubelet kubectl /usr/bin/
   14  chmod +x /usr/bin/kubeadm /usr/bin/kubelet /usr/bin/kubectl
   15  yum -y install conntrack ebtables socat
   16  cat <<EOF > /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS='--cgroup-driver=systemd'
EOF

   17  mkdir -p /etc/kubernetes/manifests
   18  mkdir -p /usr/lib/systemd/system/kubelet.service.d
   19  cat <<EOF > /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/sysconfig/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet \$KUBELET_KUBECONFIG_ARGS \$KUBELET_CONFIG_ARGS \$KUBELET_KUBEADM_ARGS \$KUBELET_EXTRA_ARGS
EOF

   20  cat <<EOF > /usr/lib/systemd/system/kubelet.service
[Unit]
Description=kubelet: The Kubernetes Node Agent
Documentation=https://kubernetes.io/docs/
Wants=network-online.target
After=network-online.target

[Service]
ExecStart=/usr/bin/kubelet
Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

   21  systemctl enable kubelet
   22  docker pull public.ecr.aws/eks-distro/etcd-io/etcd:v3.4.14-eks-1-18-1
   23  docker pull public.ecr.aws/eks-distro/kubernetes/pause:v1.18.9-eks-1-18-1
   24  docker pull public.ecr.aws/eks-distro/kubernetes/kube-scheduler:v1.18.9-eks-1-18-1
   25  docker pull public.ecr.aws/eks-distro/kubernetes/kube-proxy:v1.18.9-eks-1-18-1
   26  docker pull public.ecr.aws/eks-distro/kubernetes/kube-apiserver:v1.18.9-eks-1-18-1
   27  docker pull public.ecr.aws/eks-distro/kubernetes/kube-controller-manager:v1.18.9-eks-1-18-1
   28  docker pull public.ecr.aws/eks-distro/coredns/coredns:v1.7.0-eks-1-18-1
   29  docker tag public.ecr.aws/eks-distro/kubernetes/pause:v1.18.9-eks-1-18-1 public.ecr.aws/eks-distro/kubernetes/pause:3.2
   30  docker tag public.ecr.aws/eks-distro/coredns/coredns:v1.7.0-eks-1-18-1 public.ecr.aws/eks-distro/kubernetes/coredns:1.6.7
   31  kubeadm join 192.168.121.197:6443 --token zs4qt3.k64abd630brugvil     --discovery-token-ca-cert-hash sha256:5a5ca9e3443ad61ee7c80eb60347677fc11eaadd3d831538a6217f54ac7aedb2 
   32  history
```
