# -*- mode: ruby -*-
# vi: set ft=ruby :

servers = [
    {
        :name => "k8s-master",
        :type => "master",
        :box => "ubuntu/bionic64",
        :eth1 => "192.168.50.10",
        :mem => "2048",
        :cpu => "2"
    },
    {
        :name => "k8s-node1",
        :type => "node",
        :box => "ubuntu/bionic64",
        :eth1 => "192.168.50.11",
        :mem => "4096",
        :cpu => "2"
    },
    {
        :name => "k8s-node2",
        :type => "node",
        :box => "ubuntu/bionic64",
        :eth1 => "192.168.50.12",
        :mem => "4096",
        :cpu => "2"
    }
]

$configureBox = <<-SCRIPT

    apt-get update
    apt-get install -y apt-transport-https ca-certificates curl software-properties-common
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
    add-apt-repository "deb https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") $(lsb_release -cs) stable"
    apt-get update && apt-get install -y docker-ce docker-ce-cli containerd.io

    cat <<EOF >/etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
    
    systemctl daemon-reload
    systemctl restart docker
    # run docker commands as vagrant user (sudo not required)
    usermod -aG docker vagrant
    
    # install kubeadm
    apt-get install -y apt-transport-https curl
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
    deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
    apt-get update
    apt-get install -y kubelet kubeadm kubectl
    apt-mark hold kubelet kubeadm kubectl

    # kubelet requires swap off
    swapoff -a

    # keep swap off after reboot
    sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

    # ip of this box
    IP_ADDR=`ifconfig enp0s8 | grep mask | awk '{print $2}'| cut -f2 -d:`
    # set node-ip
    echo "KUBELET_EXTRA_ARGS=--node-ip=$IP_ADDR" > /etc/default/kubelet
    sudo systemctl restart kubelet

    # required for setting up password less ssh between guest VMs
    sudo sed -i "/^[^#]*PasswordAuthentication[[:space:]]no/c\PasswordAuthentication yes" /etc/ssh/sshd_config
    sudo service sshd restart
SCRIPT

$configureMaster = <<-SCRIPT
    echo "This is master"
    
    CALICO_YAML="https://docs.projectcalico.org/v3.14/manifests/calico.yaml"

    # ip of this box
    IP_ADDR=`ifconfig enp0s8 | grep mask | awk '{print $2}'| cut -f2 -d:`
    
    # install k8s master
    HOST_NAME=$(hostname -s)
    kubeadm init --apiserver-advertise-address=$IP_ADDR --apiserver-cert-extra-sans=$IP_ADDR  --node-name $HOST_NAME --pod-network-cidr=172.16.0.0/16

    #copying credentials to regular user - vagrant
    sudo --user=vagrant mkdir -p /home/vagrant/.kube
    cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
    chown $(id -u vagrant):$(id -g vagrant) /home/vagrant/.kube/config

    # install Calico pod network addon
    export KUBECONFIG=/etc/kubernetes/admin.conf
    kubectl apply -f $CALICO_YAML

    kubeadm token create --print-join-command >> /etc/kubeadm_join_cmd.sh
    chmod +x /etc/kubeadm_join_cmd.sh
    
    curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
    
SCRIPT

$configureK8sAddons = <<-SCRIPT
    echo "Install helm3"
    curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
    
    echo "Install k8s dashboard"
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
    
    echo "Install traefik"
    helm repo add traefik https://containous.github.io/traefik-helm-chart
    helm install traefik traefik/traefik
    
SCRIPT


$configureNode = <<-SCRIPT
    echo "This is worker"
    apt-get install -y sshpass
    sshpass -p "vagrant" scp -o StrictHostKeyChecking=no vagrant@192.168.50.10:/etc/kubeadm_join_cmd.sh .
    sh ./kubeadm_join_cmd.sh
SCRIPT

Vagrant.configure("2") do |config|

    servers.each do |opts|
        config.vm.define opts[:name] do |config|

            config.vm.box = opts[:box]
            config.vm.box_version = opts[:box_version]
            config.vm.hostname = opts[:name]
            config.vm.network :private_network, ip: opts[:eth1]

            config.vm.provider "virtualbox" do |v|

                v.name = opts[:name]
                v.customize ["modifyvm", :id, "--groups", "/k8s"]
                v.customize ["modifyvm", :id, "--memory", opts[:mem]]
                v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]

            end

            config.vm.provision "shell", inline: $configureBox

            if opts[:type] == "master"
                config.vm.provision "shell", inline: $configureMaster
            else
                config.vm.provision "shell", inline: $configureNode
            end
            
            if opts[:type] == "master"
                config.vm.provision "shell", inline: $configureK8sAddons
            end

        end

    end

end 