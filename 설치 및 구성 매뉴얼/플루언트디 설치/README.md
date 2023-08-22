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

### 플루언트디 로그수집 건피그 YAML파일

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