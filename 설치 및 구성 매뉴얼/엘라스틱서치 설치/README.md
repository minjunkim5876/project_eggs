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
        