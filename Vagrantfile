BOX_OS = "centos/7"
BOX_VERSION = "1706.02"

$token = "123456.1234567812345678"

$master = "master"
$masterIpAddress = "192.168.100.10"
$masterHostname = "kube-#{$master}"

$node1 = "node1"
$node1IpAddress = "192.168.100.11"
$node1Hostname = "kube-node1"

$node2 = "node2"
$node2IpAddress = "192.168.100.12"
$node2Hostname = "kube-node2"

$node3 = "node3"
$node3IpAddress = "192.168.100.13"
$node3Hostname = "kube-node3"

$cmdPreInit = <<SCRIPT
setenforce 0
echo "#{$masterIpAddress} #{$masterHostname}
#{$node1IpAddress} #{$node1Hostname}
#{$node2IpAddress} #{$node2Hostname}
#{$node3IpAddress} #{$node3Hostname}" >> /etc/hosts
SCRIPT

$cmdRenameMaster = <<SCRIPT
hostname #{$masterHostname}
echo "HOSTNAME=#{$masterHostname}" > /etc/sysconfig/network
SCRIPT

$cmdRenameNode1 = <<SCRIPT
hostname #{$node1Hostname}
echo "HOSTNAME=#{$node1Hostname}" > /etc/sysconfig/network
SCRIPT

$cmdRenameNode2 = <<SCRIPT
hostname #{$node2Hostname}
echo "HOSTNAME=#{$node2Hostname}" > /etc/sysconfig/network
SCRIPT

$cmdRenameNode3 = <<SCRIPT
hostname #{$node3Hostname}
echo "HOSTNAME=#{$node3Hostname}" > /etc/sysconfig/network
SCRIPT

$cmdInstallDnsmasq = <<SCRIPT
yum install -y dnsmasq
chattr -i /etc/resolv.conf
rm -rf /etc/resolv.conf
echo -e "nameserver 127.0.0.1\nnameserver $(hostname -i)" >> /etc/resolv.conf
chmod 444 /etc/resolv.conf
chattr +i /etc/resolv.conf
echo "server=8.8.8.8
server=8.8.4.4" > /etc/dnsmasq.conf
service dnsmasq restart
SCRIPT

$cmdInstallDocker = <<SCRIPT
yum install -y docker-1.12.6
systemctl enable docker && systemctl start docker
SCRIPT

$cmdAddKubeRepo = <<SCRIPT
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
SCRIPT

$cmdInstallKubectl = <<SCRIPT
curl -LOs https://storage.googleapis.com/kubernetes-release/release/v1.6.1/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
echo "source <(kubectl completion bash)" >> ~/.bashrc
SCRIPT

$cmdInstallKube = <<SCRIPT
yum install -y kubernetes-cni-0.5.1 kubelet-1.6.1 kubeadm-1.6.1 kubectl-1.6.1
systemctl enable kubelet && systemctl start kubelet
SCRIPT

$cmdSetKubeletConfig = <<SCRIPT
mkdir /etc/systemd/system/kubelet.service.d
cat <<EOF > /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
[Service]
Environment=\"KUBELET_KUBECONFIG_ARGS=--kubeconfig=/etc/kubernetes/kubelet.conf --require-kubeconfig=true\"
Environment=\"KUBELET_SYSTEM_PODS_ARGS=--pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true\"
Environment=\"KUBELET_NETWORK_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin\"
Environment=\"KUBELET_DNS_ARGS=--cluster-dns=10.96.0.10 --cluster-domain=cluster.local\"
Environment=\"KUBELET_AUTHZ_ARGS=--authorization-mode=Webhook --client-ca-file=/etc/kubernetes/pki/ca.crt\"
Environment=\"KUBELET_EXTRA_ARGS=\"
Environment=\"KUBELET_CGROUP_ARGS=--cgroup-driver=systemd\"
ExecStart=
ExecStart=/usr/bin/kubelet \\$KUBELET_KUBECONFIG_ARGS \\$KUBELET_SYSTEM_PODS_ARGS \\$KUBELET_NETWORK_ARGS \\$KUBELET_DNS_ARGS \\$KUBELET_AUTHZ_ARGS \\$KUBELET_EXTRA_ARGS \\$KUBELET_CGROUP_ARGS
EOF
systemctl daemon-reload
systemctl enable kubelet && systemctl restart kubelet
SCRIPT

$cmdIptablesWorkaround = <<SCRIPT
echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
SCRIPT

$cmdDisableFirewall = <<SCRIPT
systemctl disable firewalld
systemctl stop firewalld
SCRIPT

$cmdInitMaster = <<SCRIPT
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address #{$masterIpAddress} --apiserver-cert-extra-sans 10.0.2.15 --token #{$token}
SCRIPT

$cmdInitNode = <<SCRIPT
kubeadm join --token #{$token} #{$masterIpAddress}:6443
SCRIPT

$cmdSetupEnv = <<SCRIPT
sudo cp /etc/kubernetes/admin.conf $HOME/
sudo chown $(id -u):$(id -g) $HOME/admin.conf
echo 'export KUBECONFIG="$HOME/admin.conf"' >> $HOME/.bashrc
SCRIPT

$cmdSetupFlannel = <<SCRIPT
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel-rbac.yml
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
SCRIPT

Vagrant.configure("2") do |config|
  config.vm.define $master do |box|
    box.vm.box = BOX_OS
    box.vm.box_version = BOX_VERSION
    box.vm.provider "virtualbox" do |v|
      v.memory = 2048
      v.cpus = 1
    end
    box.vm.network "private_network", ip: "192.168.100.10"
    config.vm.provision "shell", inline: $cmdPreInit
    config.vm.provision "shell", inline: $cmdRenameMaster
    config.vm.provision "shell", inline: $cmdInstallDnsmasq
    config.vm.provision "shell", inline: $cmdInstallDocker
    config.vm.provision "shell", inline: $cmdAddKubeRepo
    config.vm.provision "shell", inline: $cmdInstallKube
    config.vm.provision "shell", inline: $cmdInstallKubectl
    config.vm.provision "shell", inline: $cmdSetKubeletConfig
    config.vm.provision "shell", inline: $cmdIptablesWorkaround
    config.vm.provision "shell", inline: $cmdDisableFirewall
    config.vm.provision "shell", inline: $cmdInitMaster
    config.vm.provision "shell", inline: $cmdSetupEnv
    config.vm.provision "shell", inline: $cmdSetupFlannel
    config.vm.synced_folder ".", "/vagrant", disabled: true
  end

  config.vm.define $node1 do |box|
    box.vm.box = BOX_OS
    box.vm.box_version = BOX_VERSION
    box.vm.provider "virtualbox" do |v|
      v.memory = 2048
      v.cpus = 1
    end
    box.vm.network "private_network", ip: "192.168.100.11"
    config.vm.provision "shell", inline: $cmdPreInit
    config.vm.provision "shell", inline: $cmdRenameNode1
    config.vm.provision "shell", inline: $cmdInstallDnsmasq
    config.vm.provision "shell", inline: $cmdInstallDocker
    config.vm.provision "shell", inline: $cmdAddKubeRepo
    config.vm.provision "shell", inline: $cmdInstallKube
    config.vm.provision "shell", inline: $cmdSetKubeletConfig
    config.vm.provision "shell", inline: $cmdIptablesWorkaround
    config.vm.provision "shell", inline: $cmdDisableFirewall
    config.vm.provision "shell", inline: $cmdInitNode
    # config.vm.provision "shell", inline: $cmdSetupFlannel
    config.vm.synced_folder ".", "/vagrant", disabled: true
  end

  config.vm.define $node2 do |box|
    box.vm.box = BOX_OS
    box.vm.box_version = BOX_VERSION
    box.vm.provider "virtualbox" do |v|
      v.memory = 2048
      v.cpus = 1
    end
    box.vm.network "private_network", ip: "192.168.100.12"
    config.vm.provision "shell", inline: $cmdPreInit
    config.vm.provision "shell", inline: $cmdRenameNode2
    config.vm.provision "shell", inline: $cmdInstallDnsmasq
    config.vm.provision "shell", inline: $cmdInstallDocker
    config.vm.provision "shell", inline: $cmdAddKubeRepo
    config.vm.provision "shell", inline: $cmdInstallKube
    config.vm.provision "shell", inline: $cmdSetKubeletConfig
    config.vm.provision "shell", inline: $cmdIptablesWorkaround
    config.vm.provision "shell", inline: $cmdDisableFirewall
    config.vm.provision "shell", inline: $cmdInitNode
    # config.vm.provision "shell", inline: $cmdSetupFlannel
    config.vm.synced_folder ".", "/vagrant", disabled: true
  end

  config.vm.define $node3 do |box|
    box.vm.box = BOX_OS
    box.vm.box_version = BOX_VERSION
    box.vm.provider "virtualbox" do |v|
      v.memory = 2048
      v.cpus = 1
    end
    box.vm.network "private_network", ip: "192.168.100.13"
    config.vm.provision "shell", inline: $cmdPreInit
    config.vm.provision "shell", inline: $cmdRenameNode3
    config.vm.provision "shell", inline: $cmdInstallDnsmasq
    config.vm.provision "shell", inline: $cmdInstallDocker
    config.vm.provision "shell", inline: $cmdAddKubeRepo
    config.vm.provision "shell", inline: $cmdInstallKube
    config.vm.provision "shell", inline: $cmdSetKubeletConfig
    config.vm.provision "shell", inline: $cmdIptablesWorkaround
    config.vm.provision "shell", inline: $cmdDisableFirewall
    config.vm.provision "shell", inline: $cmdInitNode
    # config.vm.provision "shell", inline: $cmdSetupFlannel
    config.vm.synced_folder ".", "/vagrant", disabled: true
  end

end
