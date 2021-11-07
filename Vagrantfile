# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2004"
  #config.vm.box = "roboxes/ubuntu2004"
#---------- master ----------
  config.vm.define "master_192.168.33.120" do |server|
    server.vm.network "private_network", ip: "192.168.33.120"
    server.vm.hostname = "master"
    server.vm.provider "libvirt" do |kvm|
      kvm.memory = 8192 
      kvm.cpus = 2
      kvm.machine_type = "q35"
    end
    server.vm.provision "shell", inline: <<-SHELL
      sudo swapoff -a
      sudo sed -i "/swap/d" /etc/fstab
      sudo cat <<EOF > /etc/modprobe.d/blacklist-nouveau.conf
blacklist nouveau
options nouveau modeset=0
EOF
      sudo update-initramfs -u 
      sudo apt-get update
      sudo apt-get install -y snapd
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
      sleep 120
      sudo apt-get install -y sshpass
      ssh-keygen -t rsa -N '' -f ~/.ssh/id_rsa <<< y
      cat <<EOF > ~/.ssh/config
host 192.168.33.*
   StrictHostKeyChecking no
EOF
      chmod 600 ~/.ssh/config
      sshpass -p "vagrant" ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@192.168.33.121
      sshpass -p "vagrant" ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@192.168.33.122
      sudo sed -i '2i \ \ "insecure-registries":["192.168.33.1:5000"],' /etc/docker/daemon.json
      sudo snap install eks --classic --edge
      eks kubectl config view --raw > $HOME/.kube/config
    SHELL
  end
#---------- worker1 ----------
  config.vm.define "worker1_192.168.33.121" do |server|
    server.vm.network "private_network", ip: "192.168.33.121"
    server.vm.hostname = "worker1"
    server.vm.provider "libvirt" do |kvm|
      kvm.memory = 16384
      kvm.cpus = 2
      kvm.machine_type = "q35"
      kvm.cpu_mode = "host-passthrough"
      kvm.kvm_hidden = true
      kvm.pci :bus => '0x01', :slot => '0x00', :function => '0x0'
    end
    server.vm.provision "shell", inline: <<-SHELL
      sudo swapoff -a
      sudo sed -i "/swap/d" /etc/fstab
      sudo cat <<EOF > /etc/modprobe.d/blacklist-nouveau.conf
blacklist nouveau
options nouveau modeset=0
EOF
      sudo update-initramfs -u 
      sudo apt-get update
      sudo apt-get install -y snapd
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
      sudo sed -i '2i \ \ "insecure-registries":["192.168.33.1:5000"],' /etc/docker/daemon.json
      sudo snap install eks --classic --edge
    SHELL
  end
#---------- worker2 ----------
  config.vm.define "worker2_192.168.33.122" do |server|
    server.vm.network "private_network", ip: "192.168.33.122"
    server.vm.hostname = "worker2"
    server.vm.provider "libvirt" do |kvm|
      kvm.memory = 8192
      kvm.cpus = 2
      kvm.machine_type = "q35"
    end
    server.vm.provision "shell", inline: <<-SHELL
      sudo swapoff -a
      sudo sed -i "/swap/d" /etc/fstab
      sudo cat <<EOF > /etc/modprobe.d/blacklist-nouveau.conf
blacklist nouveau
options nouveau modeset=0
EOF
      sudo update-initramfs -u 
      sudo apt-get update
      sudo apt-get install -y snapd
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
      sudo sed -i '2i \ \ "insecure-registries":["192.168.33.1:5000"],' /etc/docker/daemon.json
      sudo snap install eks --classic --edge
    SHELL
  end
end

