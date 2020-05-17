BOX_IMAGE = "bento/ubuntu-18.04"
KUBERNETES_VERSION = "stable-1.13"
KUBEADM_VERSION = "1.13.5-00"

Vagrant.configure("2") do |config|
  config.vm.provider :virtualbox do |v|
    v.memory = 2048
    v.cpus = 2
  end

  config.vm.provision :shell, privileged: true, inline: $install_common_tools

  config.vm.define :master do |master|
    master.vm.box = BOX_IMAGE
    master.vm.hostname = "master"
    master.vm.network :private_network, ip: "10.0.0.10"
    master.vm.provision :shell, privileged: false, inline: $provision_master
    master.vm.provider :virtualbox do |vb|
      vb.cpus = 2
    end
  end

  %w{worker1 worker2}.each_with_index do |name, i|
    config.vm.define name do |worker|
      worker.vm.box = BOX_IMAGE
      worker.vm.hostname = name
      worker.vm.provider :virtualbox do |vb|
        vb.customize ["modifyvm", :id, "--memory", "768"]
      end
      worker.vm.network :private_network, ip: "10.0.0.#{i + 11}"
      worker.vm.provision :shell, privileged: false, inline: <<-SHELL
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo chmod 777 /etc/systemd/system/docker.service.d
#cat > /etc/systemd/system/docker.service.d/proxy.conf << "EOF"
#[Service]
#Environment="HTTP_PROXY=http://myproxy.hostname:8080"
#Environment="HTTPS_PROXY=https://myproxy.hostname:8080/"
#Environment="NO_PROXY="localhost,127.0.0.1,::1"
#EOF
#sudo systemctl daemon-reload
#sudo systemctl restart  docker.service	  
sudo /vagrant/join.sh
echo 'Environment="KUBELET_EXTRA_ARGS=--node-ip=10.0.0.#{i + 11}"' | \
  sudo tee -a /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
sudo systemctl daemon-reload
sudo systemctl restart kubelet
SHELL
    end
  end

  config.vm.provision "shell", inline: $install_multicast
end


$install_common_tools = <<-SCRIPT
# disable swap
swapoff -a
sed -i '/swap/d' /etc/fstab

# Install kubeadm, kubectl and kubelet
export DEBIAN_FRONTEND=noninteractive
apt-get -qq install ebtables ethtool
apt-get -qq update
apt-get -qq install -y docker.io apt-transport-https curl
apt-key adv --fetch-keys https://packages.cloud.google.com/apt/doc/apt-key.gpg
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get -qq update
apt-get -qq install -y \
  kubelet=#{KUBEADM_VERSION} \
  kubeadm=#{KUBEADM_VERSION} \
  kubectl=#{KUBEADM_VERSION}
SCRIPT

$provision_master = <<-SHELL
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo chmod 777 /etc/systemd/system/docker.service.d
#cat > /etc/systemd/system/docker.service.d/proxy.conf << "EOF"
#[Service]
#Environment="HTTP_PROXY=http://myproxy.hostname:8080"
#Environment="HTTPS_PROXY=https://myproxy.hostname:8080/"
#Environment="NO_PROXY="localhost,127.0.0.1,::1"
#EOF
#sudo systemctl daemon-reload
#sudo systemctl restart  docker.service
OUTPUT_FILE=/vagrant/join.sh
rm -rf $OUTPUT_FILE

# Start cluster
sudo kubeadm init \
  --ignore-preflight-errors=SystemVerification \
  --apiserver-advertise-address=10.0.0.10 \
  --pod-network-cidr=10.244.0.0/16 \
  --kubernetes-version #{KUBERNETES_VERSION} | \
  grep "kubeadm join" > ${OUTPUT_FILE}
chmod +x $OUTPUT_FILE

# Configure kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Fix kubelet IP
echo 'Environment="KUBELET_EXTRA_ARGS=--node-ip=10.0.0.10"' | \
  sudo tee -a /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

# Configure flannel
curl -Lo kube-flannel.yml \
  https://github.com/agarciafer/kubernetes/blob/master/kube-flannel-v0.11.0.yaml
sed -i.bak -e "s/ip-masq/ip-masq\\n        - --iface=eth1/g" kube-flannel.yml
kubectl create -f kube-flannel.yml

sudo systemctl daemon-reload
sudo systemctl restart kubelet
SHELL

$install_multicast = <<-SHELL
apt-get -qq install -y avahi-daemon libnss-mdns
SHELL