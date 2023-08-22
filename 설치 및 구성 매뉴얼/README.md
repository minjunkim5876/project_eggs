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

### 네임스페이스 YAML파일

    apiVersion: v1
    kind: Namespace
    metadata:
      name: logging

### 아파치 웹 파드 YAML파일

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: web-example
      namespace: logging
    spec:
      selector:
        matchLabels:
          app: web-example
      replicas: 3
      template:
        metadata:
          labels:
            app: web-example
        spec:
          containers:
          - name: web-example
            image: httpd:latest
            resources:
              limits:
                cpu: 1000m
                memory: 1000Mi
              requests:
                cpu: 200m
                memory: 500Mi
            ports:
            - containerPort: 8080
          imagePullSecrets:
          - name: docker-pull-secret  

### 아파치 웹 파드 서비스 YAML파일

    apiVersion: v1
    kind: Service
    metadata:
      name: service-example
      namespace: logging
    spec:
      selector:
        app: web-example
      ports:
      - name: http
        protocol: TCP
        nodePort: 30000
        port: 8080
        targetPort: 80
      type: LoadBalancer

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

### 엘라스틱서치 서비스 YAML파일

    apiVersion: v1
    kind: Service
    metadata:
      labels:
        app: elasticsearch
      name: elasticsearch-svc
      namespace: logging
    spec:
      ports:
      - name: elasticsearch-rest
        nodePort: 30920
        port: 9200
        protocol: TCP
        targetPort: 9200
      - name: elasticsearch-nodecom
        nodePort: 30930
        port: 9300
        protocol: TCP
        targetPort: 9300
      selector:
        app: elasticsearch
      type: NodePort
        
# 플루언트디 설치

### 서비스 어카운트 및 롤&롤바인 YAML파일

    apiVersion: v1
    kind: Namespace
    metadata:
      name: logging

    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: fluentd
    rules:
    - apiGroups:
      - ""
      resources:
      - pods
      - namespaces
      verbs:
      - get
      - list
      - watch

    kind: ClusterRoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: fluentd
    roleRef:
      kind: ClusterRole
      name: fluentd
      apiGroup: rbac.authorization.k8s.io
    subjects:
    - kind: ServiceAccount
      name: fluentd
      namespace: logging


### 플루언트디 데몬셋 YAML파일

    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: fluentd
      namespace: logging
      labels:
        k8s-app: fluentd-logging
        version: v1
    spec:
      selector:
        matchLabels:
          k8s-app: fluentd-logging
          version: v1
      template:
        metadata:
          labels:
            k8s-app: fluentd-logging
            version: v1
        spec:
          serviceAccount: fluentd
          serviceAccountName: fluentd
          tolerations:
          - key: node-role.kubernetes.io/master
            effect: NoSchedule
          containers:
          - name: fluentd
            image: fluent/fluentd-kubernetes-daemonset:v1.3.0-debian-elasticsearch
            env:
              - name:  FLUENT_ELASTICSEARCH_HOST
                value: "192.168.10.12"
              - name:  FLUENT_ELASTICSEARCH_PORT
                value: "9200"
              - name: FLUENT_ELASTICSEARCH_SCHEME
                value: "http"
            resources:
              limits:
                cpu: 500m
                memory: 500Mi
              requests:
                cpu: 200m
                memory: 500Mi
            volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
          terminationGracePeriodSeconds: 30
          volumes:
          - name: varlog
            hostPath:
              path: /var/log
          - name: varlibdockercontainers
            hostPath:
              path: /var/lib/docker/containers

### 플루언트디 로그수집 컨피 YAML파일

    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: fluent-d-config
      namespace: logging
      labels:
        k8s-app: fluent-bit
    data:
      # Configuration files: server, input, filters and output
      # ======================================================
      fluent-bit.conf: |
        [SERVICE]
            Flush         1
            Log_Level     info
            Daemon        off
            Parsers_File  parsers.conf
            HTTP_Server   On
            HTTP_Listen   0.0.0.0
            HTTP_Port     2020
    
        @INCLUDE input-kubernetes.conf
        @INCLUDE filter-kubernetes.conf
        @INCLUDE output-elasticsearch.conf
    
      input-kubernetes.conf: |
        [INPUT]
            Name              tail
            Tag               kube.*
            Path              /var/log/containers/*.log
            Parser            docker
            DB                /var/log/flb_kube.db
            Mem_Buf_Limit     5MB
            Skip_Long_Lines   On
            Refresh_Interval  10
    
      filter-kubernetes.conf: |
        [FILTER]
            Name                kubernetes
            Match               kube.*
            Kube_URL            https://kubernetes.default.svc:443
            Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
            Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
            Kube_Tag_Prefix     kube.var.log.containers.
            Merge_Log           On
            Merge_Log_Key       log_processed
            K8S-Logging.Parser  On
            K8S-Logging.Exclude Off
    
      output-elasticsearch.conf: |
        [OUTPUT]
            Name            es
            Match           *
            Host            ${FLUENT_ELASTICSEARCH_HOST}
            Port            ${FLUENT_ELASTICSEARCH_PORT}
            Logstash_Format On
            Replace_Dots    On
            Retry_Limit     False
    
      parsers.conf: |
        [PARSER]
            Name   apache
            Format regex
            Regex  ^(?<host>[^ ]*) [^ ]* (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*?)(?: +\S*)?)?" (?<code>[^ ]*) (?<size>[^ ]*)(?: "(?<referer>[^\"]*)" "(?<agent>[^\"]*)")?$
            Time_Key time
            Time_Format %d/%b/%Y:%H:%M:%S %z
    
        [PARSER]
            Name   apache2
            Format regex
            Regex  ^(?<host>[^ ]*) [^ ]* (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^ ]*) +\S*)?" (?<code>[^ ]*) (?<size>[^ ]*)(?: "(?<referer>[^\"]*)" "(?<agent>[^\"]*)")?$
            Time_Key time
            Time_Format %d/%b/%Y:%H:%M:%S %z
    
        [PARSER]
            Name   apache_error
            Format regex
            Regex  ^\[[^ ]* (?<time>[^\]]*)\] \[(?<level>[^\]]*)\](?: \[pid (?<pid>[^\]]*)\])?( \[client (?<client>[^\]]*)\])? (?<message>.*)$
    
        [PARSER]
            Name   nginx
            Format regex
            Regex ^(?<remote>[^ ]*) (?<host>[^ ]*) (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*?)(?: +\S*)?)?" (?<code>[^ ]*) (?<size>[^ ]*)(?: "(?<referer>[^\"]*)" "(?<agent>[^\"]*)")?$
            Time_Key time
            Time_Format %d/%b/%Y:%H:%M:%S %z
    
        [PARSER]
            Name   json
            Format json
            Time_Key time
            Time_Format %d/%b/%Y:%H:%M:%S %z
    
        [PARSER]
            Name        docker
            Format      json
            Time_Key    time
            Time_Format %Y-%m-%dT%H:%M:%S.%L
            Time_Keep   On
    
        [PARSER]
            # http://rubular.com/r/tjUt3Awgg4
            Name cri
            Format regex
            Regex ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>[^ ]*) (?<message>.*)$
            Time_Key    time
            Time_Format %Y-%m-%dT%H:%M:%S.%L%z
    
        [PARSER]
            Name        syslog
            Format      regex
            Regex       ^\<(?<pri>[0-9]+)\>(?<time>[^ ]* {1,2}[^ ]* [^ ]*) (?<host>[^ ]*) (?<ident>[a-zA-Z0-9_\/\.\-]*)(?:\[(?<pid>[0-9]+)\])?(?:[^\:]*\:)? *(?<message>.*)$
            Time_Key    time
            Time_Format %b %d %H:%M:%S
