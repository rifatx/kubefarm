BOX_OS = "centos/7"
BOX_VERSION = "1706.02"
MASTER_COUNT = 2
NODE_COUNT = 4

$token = "123456.1234567812345678"
$hostnamePrefix = "kube"
$masterSuffix = "master"
$nodeSuffix = "node"
$ipGroup = "192.168.100"
$masterStartIp = 100
$nodeStartIp = 10
$masters = []
$nodes = []
$hostsCommand = ""

for i in 1..MASTER_COUNT
$masters << {
    name: "#{$masterSuffix}#{i}",
    hostname: "#{$hostnamePrefix}-#{$masterSuffix}-#{i}",
    ipAddress: "#{$ipGroup}.#{$masterStartIp + i}"
}
end

for i in 1..NODE_COUNT
$nodes << {
    name: "#{$nodeSuffix}#{i}",
    hostname: "#{$hostnamePrefix}-#{$nodeSuffix}-#{i}",
    ipAddress: "#{$ipGroup}.#{$nodeStartIp + i}"
}
end

for master in $masters
    $hostsCommand << master[:ipAddress] + " " + master[:hostname] + "\n"
end

for node in $nodes
    $hostsCommand << node[:ipAddress] + " " + node[:hostname] + "\n"
end

$hostsCommand = "echo \"#{$hostsCommand}\" >> /etc/hosts"

$cmdPreInit = <<SCRIPT
setenforce 0
#{$hostsCommand}
SCRIPT

$cmdRenameMachine = <<SCRIPT
hostname %{hostname}
echo "HOSTNAME=%{hostname}" > /etc/sysconfig/network
systemctl restart network
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
systemctl enable dnsmasq && systemctl restart dnsmasq
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
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address %{ipAddress} --apiserver-cert-extra-sans 10.0.2.15 --token #{$token}
SCRIPT

$cmdInitNode = <<SCRIPT
if ping -c 1 %{masterIpAddress} &> /dev/null; then
  kubeadm join --token #{$token} %{masterIpAddress}:6443
fi
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
    $masters.each do |master|
      config.vm.define master[:name] do |box|
        box.vm.box = BOX_OS
        box.vm.box_version = BOX_VERSION
        box.vm.provider "virtualbox" do |v|
          v.memory = 2048
          v.cpus = 1
        end
        box.vm.hostname = master[:hostname]
        box.vm.network "private_network", ip: master[:ipAddress]
        box.vm.provision "shell", inline: $cmdPreInit
        box.vm.provision "shell", inline: $cmdRenameMachine % {hostname: master[:hostname]}
        box.vm.provision "shell", inline: $cmdInstallDnsmasq
        box.vm.provision "shell", inline: $cmdInstallDocker
        box.vm.provision "shell", inline: $cmdAddKubeRepo
        box.vm.provision "shell", inline: $cmdInstallKube
        # box.vm.provision "shell", inline: $cmdInstallKubectl
        box.vm.provision "shell", inline: $cmdSetKubeletConfig
        box.vm.provision "shell", inline: $cmdIptablesWorkaround
        box.vm.provision "shell", inline: $cmdDisableFirewall
        box.vm.provision "shell", inline: $cmdInitMaster % {ipAddress: master[:ipAddress]}
        box.vm.provision "shell", inline: $cmdSetupEnv
        box.vm.provision "shell", inline: $cmdSetupFlannel
        box.vm.synced_folder ".", "/vagrant", disabled: true
      end
    end

    $nodes.each do |node|
        config.vm.define node[:name] do |box|
        box.vm.box = BOX_OS
        box.vm.box_version = BOX_VERSION
        box.vm.provider "virtualbox" do |v|
          v.memory = 2048
          v.cpus = 1
        end
        box.vm.hostname = node[:hostname]
        box.network "private_network", ip: node[:ipAddress]
        box.vm.provision "shell", inline: $cmdPreInit
        box.vm.provision "shell", inline: $cmdRenameMachine % {hostname: node[:hostname]}
        box.vm.provision "shell", inline: $cmdInstallDnsmasq
        box.vm.provision "shell", inline: $cmdInstallDocker
        box.vm.provision "shell", inline: $cmdAddKubeRepo
        box.vm.provision "shell", inline: $cmdInstallKube
        box.vm.provision "shell", inline: $cmdSetKubeletConfig
        box.vm.provision "shell", inline: $cmdIptablesWorkaround
        box.vm.provision "shell", inline: $cmdDisableFirewall
        $masters.each do |master|
            box.vm.provision "shell", inline: $cmdInitNode % {masterIpAddress: master[:ipAddress]}
        end
        # box.vm.provision "shell", inline: $cmdSetupFlannel
        box.vm.synced_folder ".", "/vagrant", disabled: true
      end
    end
end
