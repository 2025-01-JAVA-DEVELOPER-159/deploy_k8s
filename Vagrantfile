
NodeCnt = 2

Vagrant.configure("2") do |config|
  
  config.vm.box = "rockylinux/8"
  config.vm.box_version="8.8-20230518.0"  
  config.disksize.size = "30GB"  
  config.vbguest.auto_update = false
  config.vm.synced_folder "./", "/vagrant", disabled: true  
  config.vm.provision :shell, privileged: true, inline: $install_common_tools

  config.vm.define "k8s-master" do |master|
    master.vm.hostname = "k8s-master"
    master.vm.network "private_network", ip: "192.168.56.30"
    master.vm.provider :virtualbox do |vb|
      vb.memory = 4096
      vb.cpus = 4
      vb.customize ["modifyvm", :id, "--firmware", "efi"]
      vb.customize ["modifyvm", :id, "--nested-hw-virt", "on"]
	end
    master.vm.provision :shell, privileged: true, inline: $provision_master_node
  end

  (1..NodeCnt).each do |i|
    config.vm.define "k8s-node#{i}" do |node| 
      node.vm.hostname = "k8s-node#{i}"
      node.vm.network "private_network", ip: "192.168.56.#{i + 30}"
      node.vm.provider :virtualbox do |vb|
        vb.memory = 2048
        vb.cpus = 2
        vb.customize ["modifyvm", :id, "--firmware", "efi"]
	  end
    end
  end
end

$install_common_tools = <<-SHELL

echo '======== [4] Rocky Linux 기본 설정 ========'
echo '======== [4-1] 패키지 업데이트 ========'
# 실습 자료실과 동일한 실습 환경을 유지하기 위해 Linux Update 주석 처리
# yum -y update

echo '======== [4-2] 타임존 설정 ========'
timedatectl set-timezone Asia/Seoul

echo '======== [4-3] Disk 확장 / Bug: soft lockup 설정 추가========'
# https://cafe.naver.com/kubeops/25
yum install -y cloud-utils-growpart
growpart /dev/sda 4
xfs_growfs /dev/sda4
echo 0 > /proc/sys/kernel/hung_task_timeout_secs
echo "kernel.watchdog_thresh = 20" >> /etc/sysctl.conf

echo '======== [4-4] [WARNING FileExisting-tc]: tc not found in system path 로그 관련 업데이트 ========'
yum install -y yum-utils iproute-tc

echo '======== [4-5] Hosts 등록 ========'
cat << EOF >> /etc/hosts
192.168.56.30 k8s-master
192.168.56.31 k8s-node1
192.168.56.32 k8s-node2
EOF

echo '======== [5] kubeadm 설치 전 사전작업 ========'
echo '======== [5] 방화벽 해제 ========'
systemctl stop firewalld && systemctl disable firewalld

echo '======== [5] Swap 비활성화 ========'
swapoff -a && sed -i '/ swap / s/^/#/' /etc/fstab


echo '======== [6] 컨테이너 런타임 설치 ========'
echo '======== [6-1] 컨테이너 런타임 설치 전 사전작업 ========'
echo '======== [6-1] iptable 세팅 ========'
cat <<EOF |tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

cat <<EOF |tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system

echo '======== [6-2] 컨테이너 런타임 (containerd 설치) ========'
echo '======== [6-2-1] containerd 패키지 설치 (option2) ========'
echo '======== [6-2-1-1] docker engine 설치 ========'
echo '======== [6-2-1-1] repo 설정 ========'
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

echo '======== [6-2-1-1] containerd 설치 ========'
yum install -y containerd.io-1.6.21-3.1.el8
systemctl daemon-reload
systemctl enable --now containerd

echo '======== [6-3] 컨테이너 런타임 : cri 활성화 ========'
# defualt cgroupfs에서 systemd로 변경 (kubernetes default는 systemd)
containerd config default > /etc/containerd/config.toml
sed -i 's/ SystemdCgroup = false/ SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd


echo '======== [7] kubeadm 설치 ========'
echo '======== [7] repo 설정 ========'
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.27/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.27/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

echo '======== [7] SELinux 설정 ========'
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

echo '======== [7] kubelet, kubeadm, kubectl 패키지 설치 ========'
yum install -y kubelet-1.27.2-150500.1.1.x86_64 kubeadm-1.27.2-150500.1.1.x86_64 kubectl-1.27.2-150500.1.1.x86_64 --disableexcludes=kubernetes
systemctl enable --now kubelet

SHELL



$provision_master_node = <<-SHELL

echo '======== [8] kubeadm으로 클러스터 생성  ========'
echo '======== [8-1] 클러스터 초기화 (Pod Network 세팅) ========'
kubeadm init --pod-network-cidr=20.96.0.0/16 --apiserver-advertise-address 192.168.56.30
kubeadm token create --print-join-command > ~/join.sh

echo '======== [8-2] kubectl 사용 설정 ========'
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

echo '======== [8-3] Pod Network 설치 (calico) ========'
kubectl create -f https://raw.githubusercontent.com/k8s-1pro/install/main/ground/k8s-1.27/calico-3.26.4/calico.yaml
kubectl create -f https://raw.githubusercontent.com/k8s-1pro/install/main/ground/k8s-1.27/calico-3.26.4/calico-custom.yaml


echo '======== [9] 쿠버네티스 편의기능 설치 ========'
echo '======== [9-1] kubectl 자동완성 기능 ========'
echo "source <(kubectl completion bash)" >> ~/.bashrc
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc

echo '======== [9-2] Dashboard 설치 ========'
kubectl create -f https://raw.githubusercontent.com/k8s-1pro/install/main/ground/k8s-1.27/dashboard-2.7.0/dashboard.yaml

echo '======== [9-3] Metrics Server 설치 ========'
kubectl create -f https://raw.githubusercontent.com/k8s-1pro/install/main/ground/k8s-1.27/metrics-server-0.6.3/metrics-server.yaml

SHELL
