### 쿠버네티스 설치
sudo apt update -y && sudo apt -y full-upgrade
sudo apt install -y gnupg2 software-properties-common apt-transport-https ca-certificates
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
curl -fsSL https://dl.k8s.io/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update -y
sudo apt install -y containerd.io
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo apt -y install kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo sed -i '/swap/s/^/#/' /etc/fstab
sudo swapoff -a
sudo mount -a
sudo su - -c "echo 'net.bridge.bridge-nf-call-ip6tables = 1' >> /etc/sysctl.d/kubernetes.conf"
sudo su - -c "echo 'net.bridge.bridge-nf-call-iptables = 1' >> /etc/sysctl.d/kubernetes.conf"
sudo su - -c "echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.d/kubernetes.conf"
sudo su - -c "echo 'overlay' >> /etc/modules-load.d/containerd.conf"
sudo su - -c "echo 'br_netfilter' >> /etc/modules-load.d/containerd.conf"
sudo modprobe overlay
sudo modprobe br_netfilter
sudo sysctl --system
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl restart kubelet
sudo systemctl enable kubelet
sudo su - -c 'echo "192.168.10.11 kube-controller " >> /etc/hosts'
sudo su - -c 'echo "192.168.10.12 kube-worker-node " >> /etc/hosts'
sudo kubeadm config images pull
sudo kubeadm init --apiserver-advertise-address 192.168.10.11 --pod-network-cidr 172.30.0.0/16 --upload-certs --control-plane-endpoint kube-controller > token.txt
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
wget https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml
sed -i 's/cidr: 192\.168\.0\.0\/16/cidr: 172\.30\.0\.0\/16/g' custom-resources.yaml
sed -i '7a registry: quay.io/' custom-resources.yaml

### 도커명령(자주씀)

    1. docker cp (로컬 파일을 컨테이너 파일로 복사)
    2. docker inspect (자세한 내용확인)
    3. watch -n 1 docker ps -a (터미널에서 변경사항 수시확인)
    4. docker run --restart (재시작 정책설정)
    5. docker stats (서버자원 사용량 확인)
    6. docker run -it -d -v --name 이름 --restart=설정 컨테이너이름  

### 베이그란트로 손쉽게 초기 세팅 해보기

    # -*- mode: ruby -*-
    # vi: set ft=ruby :
    Vagrant.configure("2") do |config|
    config.vm.box = "ubuntu/focal64"

    config.vm.define "ubuntu" do |ubuntu|
        ubuntu.vm.hostname = "mz-server"
        ubuntu.vm.provider "virtualbox" do |vb|
        vb.name = "docker-server"
        vb.cpus = 4
        vb.memory = 8192
        end
        ubuntu.vm.network "forwarded_port", guest: 80, host: 80
        ubuntu.vm.network "private_network", ip: "192.168.33.10"
        ubuntu.vm.provision "shell", inline: <<-SCRIPT
        sudo apt-get update -y
        sudo apt-get install -y ca-certificates curl gnupg
        sudo install -m 0755 -d /etc/apt/keyrings
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
        sudo chmod a+r /etc/apt/keyrings/docker.gpg
        echo \
            "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
            "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
            sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
        sudo apt-get update -y
        sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
        sudo usermod -a -G docker vagrant
        docker run -it -d -p 80:80 --name nginx-server nginx
        docker cp /vagrant/env/sample2/ nginx-server:/usr/share/nginx/html

### 도커허브 로그인 토큰 등록
    1.env/docker_token 파일을 생성하여 토큰번호를 등록 합시다       

### 도커 스토리지 관리 해보자
    1.바인드마운트 호스트 파일시스템에서 직접 관리(내가 지정)
    2.볼륨마운트 도커 시스템에서 간접 관리(/var/lib/docker/volumes)
    3.tmpfs 메모리에만 저장, 호스트에 저장안함,성능이 좋고 빠름(시스템 메모리)
    4.볼륨마운트 커맨드
    5.볼륨 크리에이트 하고 컨테이너에 마운트하기
    6.docker run -d -p (8080:80) -v web_src_vol:/usr/local/apache2/htdocs --name 컨테이너이름 이미지(httpd)
