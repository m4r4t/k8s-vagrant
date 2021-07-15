KUBER_NET = "192.168.50"


Vagrant.configure("2") do |config|
  config.vm.provision "shell" do |s|
    s.privileged = true
    s.inline = $install_common_tools
    s.args = KUBER_NET
  end

  config.vm.define :master do |master|
    master.vm.provider :virtualbox do |vb|
      vb.name = "master"
      vb.memory = 2048
      vb.cpus = 2
    end
    master.vm.box = "ubuntu/bionic64"
    #master.disksize.size = "25GB"
    master.vm.hostname = "master"
    master.vm.network :private_network, ip: KUBER_NET + ".10"
    master.vm.provision "shell" do |s|
      s.privileged = false
      s.inline = $provision_master_node
      s.args = KUBER_NET
    end
  end

  #%w{node1 node2 node3}.each_with_index do |name, i|
  %w{node1 node2}.each_with_index do |name, i|  
    config.vm.define name do |node|
      node.vm.provider "virtualbox" do |vb|
        vb.name = "node#{i + 1}"
        vb.memory = 2048
        vb.cpus = 2
      end
      node.vm.box = "ubuntu/bionic64"
      #node.disksize.size = "25GB"
      node.vm.hostname = name
      node.vm.network :private_network, ip: KUBER_NET + ".#{i + 11}"
      node.vm.provision "shell" do |s|
        s.privileged = false
        s.inline = $node_join
        s.args = [KUBER_NET, i + 11]
      end
    end
  end

  config.vm.provision "shell", inline: $install_multicast
end

$node_join = <<-SHELL
sudo /vagrant/join.sh
echo 'Environment="KUBELET_EXTRA_ARGS=--node-ip='$1.$2'"' | sudo tee -a /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
cat /vagrant/id_rsa.pub >> /home/vagrant/.ssh/authorized_keys

sudo systemctl daemon-reload
sudo systemctl restart kubelet
SHELL

$install_common_tools = <<-SCRIPT
# bridged traffic to iptables is enabled for kube-router.
cat >> /etc/ufw/sysctl.conf <<EOF
net/bridge/bridge-nf-call-ip6tables = 1
net/bridge/bridge-nf-call-iptables = 1
net/bridge/bridge-nf-call-arptables = 1
EOF

# Patch OS
apt-get update && apt-get upgrade -y

# Create local host entries
echo "127.0.0.1 localhost" > /etc/hosts
echo "$1.10 master" >> /etc/hosts
echo "$1.11 node1" >> /etc/hosts
echo "$1.12 node2" >> /etc/hosts
echo "$1.13 node3" >> /etc/hosts

# disable swap
swapoff -a
sed -i '/swap/d' /etc/fstab

# Install kubeadm, kubectl and kubelet
export DEBIAN_FRONTEND=noninteractive
apt-get -qq install ebtables ethtool
apt-get -qq update
apt-get -qq install -y docker.io apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get -qq update
apt-get -qq install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubectl kubeadm

#Change default cgroupDriver for Docker
cp /vagrant/daemon.json /etc/docker/daemon.json
systemctl restart docker.service


# Set external DNS
sed -i -e 's/#DNS=/DNS=8.8.8.8/' /etc/systemd/resolved.conf
service systemd-resolved restart
SCRIPT

$provision_master_node = <<-SHELL
KUB_NET=$1
OUTPUT_FILE=/vagrant/join.sh
KEY_FILE=/vagrant/id_rsa.pub
rm -rf $OUTPUT_FILE
rm -rf $KEY_FILE

# Create key
ssh-keygen -q -t rsa -b 4096 -N '' -f /home/vagrant/.ssh/id_rsa
cat /home/vagrant/.ssh/id_rsa.pub >> /home/vagrant/.ssh/authorized_keys
cat /home/vagrant/.ssh/id_rsa.pub > ${KEY_FILE}

# Start cluster
sudo kubeadm init --apiserver-advertise-address=$1.10 --pod-network-cidr=10.244.0.0/16 | grep -Ei "kubeadm join|discovery-token-ca-cert-hash" > ${OUTPUT_FILE}
chmod +x $OUTPUT_FILE

# Configure kubectl for vagrant and root users
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
sudo mkdir -p /root/.kube
sudo cp -i /etc/kubernetes/admin.conf /root/.kube/config
sudo chown -R root:root /root/.kube

# Fix kubelet IP
echo 'Environment="KUBELET_EXTRA_ARGS=--node-ip='$1'.10"' | sudo tee -a /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

# Use our flannel config file so that routing will work properly
kubectl create -f /vagrant/kube-flannel.yml

# install calico 

#sed "s/KUBNET/$KUB_NET/g" /vagrant/calico.yaml > /tmp/calico.yaml
#kubectl apply -f /tmp/calico.yaml

# Set alias on master for vagrant and root users
echo "alias k=/usr/bin/kubectl" >> $HOME/.bash_profile


# Add kubectl bash completion
kubectl completion bash > /tmp/kubectl
sudo mv /tmp/kubectl /etc/bash_completion.d/kubectl
echo 'complete -F __start_kubectl k' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bash_profile
echo 'complete -o default -F __start_kubectl k' >> ~/.bash_profile


# Install the etcd client
sudo apt install etcd-client

sudo systemctl daemon-reload
sudo systemctl restart kubelet
SHELL

$install_multicast = <<-SHELL
apt-get -qq install -y avahi-daemon libnss-mdns
SHELL