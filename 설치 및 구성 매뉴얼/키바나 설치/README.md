# 키바나 설치

### 키바나 파드 YAML파일

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: kibana
      namespace: logging
      labels:
        app: kibana
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: kibana
      template:
        metadata:
          labels:
            app: kibana
        spec:
          containers:
          - name: kibana
            image: elastic/kibana:6.4.0
            resources:
              limits:
                cpu: 1000m
                memory: 1000Mi
              requests:
                cpu: 200m
                memory: 500Mi
            env:
            - name: SERVER_NAME
              value: kube-worker-node
            - name: ELASTICSEARCH_URL
              value: http://192.168.10.12:30920
            ports:
            - containerPort: 5601
          imagePullSecrets:
          - name: docker-pull-secret    
### 키바나 서비스 YAML파일
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        app: kibana
      name: kibana-svc
      namespace: logging
    spec:
      ports:
      - nodePort: 30561
        port: 5601
        protocol: TCP
        targetPort: 5601
      selector:
        app: kibana
      type: NodePort