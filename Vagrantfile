Vagrant.configure("2") do |config|
  config.vm.provider :virtualbox do |v|
    v.memory = 2048
    v.cpus = 1
  end

  config.vm.define :master do |master|
    master.vm.box = "learnk8s/networking"
    master.vm.box_version = "2.0.0"
    master.vm.hostname = "master"
    master.vm.network :private_network, ip: "10.0.0.10"
    master.vm.provision :shell, privileged: false, inline: $provision_master
    master.vm.provider :virtualbox do |vb|
      vb.cpus = 2
	  vb.name = "master"
    end
  end

  %w{worker1 worker2}.each_with_index do |name, i|
    config.vm.define name do |worker|
      worker.vm.box = "learnk8s/networking"
      worker.vm.box_version = "2.0.0"
      worker.vm.hostname = name
      worker.vm.provider :virtualbox do |vb|
        vb.customize ["modifyvm", :id, "--memory", "2048"]
		vb.name = name
      end
      worker.vm.network :private_network, ip: "10.0.0.#{i + 11}"
      worker.vm.provision :shell, privileged: false, inline: <<-SHELL
sudo /vagrant/join.sh
echo 'Environment="KUBELET_EXTRA_ARGS=--node-ip=10.0.0.#{i + 11}"' | \
  sudo tee -a /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
sudo systemctl daemon-reload
sudo systemctl restart kubelet
sudo systemctl enable docker
SHELL
    end
  end

end

$provision_master = <<-SHELL
OUTPUT_FILE=/vagrant/join.sh
rm -rf $OUTPUT_FILE

# Start cluster
sudo kubeadm init \
  --ignore-preflight-errors=SystemVerification \
  --apiserver-advertise-address=10.0.0.10 \
  --pod-network-cidr=10.244.0.0/16 \
  --kubernetes-version v1.18.3 | \
  grep -A1 "kubeadm join" > ${OUTPUT_FILE}
chmod +x $OUTPUT_FILE

# Configure kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
cp $HOME/.kube/config /vagrant
cp /kube/nginx-ingress-v0.24.1.yaml /vagrant
sudo apt-get install bash-completion
echo 'source <(kubectl completion bash)' >>~/.bashrc
# Fix kubelet IP
echo 'Environment="KUBELET_EXTRA_ARGS=--node-ip=10.0.0.10"' | \
  sudo tee -a /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

# Configure flannel
kubectl apply -f /kube/kube-flannel-v0.12.0.yaml

sudo systemctl daemon-reload
sudo systemctl restart kubelet
sudo systemctl enable docker

SHELL