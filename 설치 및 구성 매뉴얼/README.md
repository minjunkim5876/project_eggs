# 쿠버네티스 설치

### 모든노드에 적용

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
    
### 쿠버네티스 컨트롤 플레인에만 적용

    sudo kubeadm config images pull
    sudo kubeadm init --apiserver-advertise-address 192.168.10.11 --pod-network-cidr 172.30.0.0/16 --upload-certs --control-plane-endpoint kube-controller > token.txt
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
    wget https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml
    sed -i 's/cidr: 192\.168\.0\.0\/16/cidr: 172\.30\.0\.0\/16/g' custom-resources.yaml
    sed -i '7a registry: quay.io/' custom-resources.yaml
    kubectl create -f custom-resources.yaml

# 엘라스틱서치 설치

### 엘라스틱서치 파드 YAML파일

    apiVersion: apps/v1
    kind: Deployment
metadata:
  name: elasticsearch
  namespace: logging
  labels:
    app: elasticsearch
spec:
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: elastic/elasticsearch:6.4.0
        resources:
          limits:
            cpu: 1000m
            memory: 3000Mi
          requests:
            cpu: 200m
            memory: 500Mi
        env:
        - name: discovery.type
          value: single-node
        ports:
        - containerPort: 9200
        - containerPort: 9300
      imagePullSecrets:
      - name: docker-pull-secret  
        
