
## 新建 "monitor" 命名空间

-------------------------

```
apiVersion: v1
kind: Namespace
metadata:
  name: monitor
```

## Node node_exporter

--------------------------

### 仅部署 Node Exporter 到 Node 可以不需要 Service

    apiVersion: v1
    kind: Service
    metadata:
      namespace: prom
      annotations:
        prometheus.io/scrape: 'true'
      labels:
        name: node-exporter
        app: node-exporter
    spec:
      clusterIP: None
      ports:
      - name: scrape
        port: 9100
        protocol: TCP
      selector:
        app: node-exporter
      type: ClusterIP

### DaemonSet 在每个 Node 部署 node_exporter


> 优点: 新增 Node 自动部署上 node_exporter

> 存在的问题, Master 与 etcd 需二进制文件部署, 可用 ansible 自动化部署

    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      namespace: monitor
      name: node-exporter
      labels:
        app: node-exporter
    spec:
      selector:
        matchLabels:
          app: node-exporter
      template:
        metadata:
          namespace: monitor
          labels:
            app: node-exporter
          name: node-exporter
        spec:
          containers:
          - image: prom/node-exporter:latest
            name: node-exporter
            ports:
            - containerPort: 9100
              hostPort: 9100
              name: scrape
          hostNetwork: true
          hostPID: true
          restartPolicy: Always

## kube-state-metrics 监控集群内资源对象

-------------------------------------

[kube-state-metrics GitHub](https://github.com/kubernetes/kube-state-metrics "kube-state-metrics GitHub")

可监控对象

* CronJob Metrics
* DaemonSet Metrics
* Deployment Metrics
* Job Metrics
* LimitRange Metrics
* Node Metrics
* PersistentVolume Metrics
* PersistentVolumeClaim Metrics
* Pod Metrics
* ReplicaSet Metrics
* ReplicationController Metrics
* ResourceQuota Metrics
* Service Metrics
* StatefulSet Metrics
* Namespace Metrics
* Horizontal Pod Autoscaler Metrics
* Endpoint Metrics

### kube-state-metrics Ingress

    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: kube-state-metrics
      namespace: monitor
    spec:
      rules:
      - host: prom.yud.io
        http:
          paths:
          - path:
            backend:
              serviceName: kube-state-metrics
              servicePort: 8080
          - path:
            backend: /telemetry
              serviceName: kube-state-metrics
              servicePort: 8081

## Kubernetes 部署 Prometheus

---------------------------

### RBAC Setup

    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: prometheus
    rules:
    - apiGroups: [""] # "" 表示 core API group
      resources:  # 资源列表
      - nodes
      - nodes/proxy
      - services
      - endpoints
      - pods
      verbs: ["get", "list", "watch"] # 授权的动作
    - apiGroups:
      - extensions
      resources:
      - ingresses
      verbs: ["get", "list", "watch"]
    - nonResourceURLs: ["/metrics"] # 有权访问的部分网址，如果需要访问其他 url 获取指标，应在此给予权限, 多个 url 用逗号分隔, 如下示例
      verbs: ["get"]
    #- nonResourceURLs: ["/metrics", "/healthz"]
    #  verbs: ["get"]
    ---
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: prometheus
      namespace: monitor
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: prometheus
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: prometheus
    subjects:
    - kind: ServiceAccount
      name: prometheus
      namespace: monitor

### ConfigMap

[官方示例 prometheus-kubernetes.yml](https://github.com/prometheus/prometheus/blob/master/documentation/examples/prometheus-kubernetes.yml)

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitor
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: EnsureExists
data:
  prometheus.yml: |
    scrape_configs: # 指定采集数据的描述和参数
    - job_name: 'kubernetes-apiservers' # 作业名称
      kubernetes_sd_configs:  # 配置从 Kubernetes 中通过 REST API 采集的参数和目标, 为自动发现设置
      - role: endpoints   # role 类型为 endpoint, 表示应被自动发现的
      scheme: https # 默认使用 https, 可禁用或改为 http
      tls_config:   # 配置 TLS 连接相关参数
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt # ca.crt 在 pod 内的路径, 可能是 prom image 默认创建, 并获取 kubernetes 集群的 ca.crt 和 token
        insecure_skip_verify: true  # 禁用服务器证书的验证, 可根据情况选择开启, 如自签证书等可控情况
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token  # token 文件路径, 可使用 "bearer_token" 直接填写 token, 但两个选项不能同时使用

      relabel_configs:  # 重新标记，可以在抓取目标之前动态重写目标的标签集。每个采集配置可以配置多个重新标记步骤。它们按照它们在配置文件中的出现顺序应用于每个目标的标签集
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]  # 源标签, 指定标签以进一步操作
        action: keep  # 对基于下面的 regex 匹配的内容执行的操作, keep 表示 regex 与 source_labels 不匹配的项都删除
        regex: default;kubernetes;https   # 正则表达式, 是 action 所必需的

    - job_name: 'kubernetes-nodes'
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

      kubernetes_sd_configs:
      - role: node  # node 表示自动发现每个集群节点一个采集目标, 目标地址默认为 kubelet 的 HTTP 端口
      relabel_configs:
      - action: labelmap  # labelmap 表示将 regex 与所有标签名称匹配; 然后将符合匹配标签的值复制到 replacement 给出的标签名称，匹配组引用(${1},${2},...)替换为其值
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__   # target_label表示标签在替换操作时将结果写入; __address__ 表示目标的 host:port 标签
        replacement: kubernetes.default.svc:443   # 如果正则表达式匹配, 则执行 regex 匹配的内容替换的 replacement 值中的匹配组引用, 默认 $1
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__  # __metrics_path__ 标签表示获取指标的HTTP资源路径, 默认 /metrics
        replacement: /api/v1/nodes/${1}/proxy/metrics

    - job_name: 'kubernetes-cadvisor'
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor

    - job_name: 'kubernetes-service-endpoints'
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_example_io_should_be_scraped]
        action: keep
        regex: true

      - source_labels: [__meta_kubernetes_service_annotation_example_io_metric_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)

      - source_labels: [__address__, __meta_kubernetes_service_annotation_example_io_scrape_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__

      - source_labels: [__meta_kubernetes_service_annotation_example_io_scrape_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        action: replace
        target_label: kubernetes_name

    - job_name: 'kubernetes-services'
      metrics_path: /probe
      params:
        module: [http_2xx]
      kubernetes_sd_configs:
      - role: service
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_example_io_should_be_probed]
        action: keep
        regex: true
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: blackbox-exporter.example.com:9115
      - source_labels: [__param_target]
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        target_label: kubernetes_name

    - job_name: 'kubernetes-ingresses'
      metrics_path: /probe
      params:
        module: [http_2xx]
      kubernetes_sd_configs:
      - role: ingres
      relabel_configs:
      - source_labels: [__meta_kubernetes_ingress_annotation_example_io_should_be_probed]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_ingress_scheme,__address__,__meta_kubernetes_ingress_path]
        regex: (.+);(.+);(.+)
        replacement: ${1}://${2}${3}
        target_label: __param_target
      - target_label: __address__
        replacement: blackbox-exporter.example.com:9115
      - source_labels: [__param_target]
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_ingress_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_ingress_name]
        target_label: kubernetes_name

    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_example_io_should_be_scraped]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_example_io_metric_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_example_io_scrape_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name

```

### Deployment

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    name: prometheus-deployment
  name: prometheus
  namespace: monitor
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      #- image: prom/prometheus:v2.3.2
      - image: registry.cn-hangzhou.aliyuncs.com/choerodon-tools/prometheus:init
        name: prometheus
        command:
        - "/bin/prometheus"
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        - "--storage.tsdb.path=/prometheus"
        - "--storage.tsdb.retention=18h"
        ports:
        - containerPort: 9090
          protocol: TCP
        volumeMounts:
        - mountPath: "/prometheus"
          name: data
        - mountPath: "/etc/prometheus"
          name: config-volume
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 500m
            memory: 2500Mi
      serviceAccountName: prometheus
      imagePullSecrets:
        - name: regsecret
      volumes:
      - name: data
        emptyDir: {}
      - name: config-volume
        configMap:
          name: prometheus-config
```

### Service

```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: prometheus
  name: prometheus
  namespace: monitor
spec:
  type: NodePort
  ports:
  - port: 9090
    targetPort: 9090
    nodePort: 9090
spec:
  ports:
  - name: prom_port
    port: 9090
    targetPort: 9090
    protocol: TCP
  selector:
    app: prometheus
```

### Ingress

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: prometheus
  namespace: monitor
spec:
  rules:
  - host: {{ prometheus_domain }}
    http:
      paths:
      - path:
        backend:
          serviceName: prometheus
          servicePort: 9090
```

## ETCD

----------------


```
curl -v --cert /etc/kubernetes/pki/etcd/server.pem --key /etc/kubernetes/pki/etcd/server-key.pem  https://192.168.3.134:2379/metrics

# prometheus.yml 配置
scrape_configs:
  ......
  - job_name: test-etcd
    static_configs:
    - targets:
      - '192.168.3.134:2379'
      - '192.168.3.133:2379'
      - '192.168.3.132:2379'
```


# prometheus.yml

```
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - "192.168.3.145:9093"
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  - "alert-rules.yml"
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']
  - job_name: 'all-hpc-node-exporter'
    static_configs:
    - targets:
      - 192.168.3.11:9100
      - 192.168.3.12:9100
      - 192.168.3.13:9100
      - 192.168.3.14:9100
      - 192.168.3.15:9100
      - 192.168.3.17:9100
      - 192.168.3.18:9100
      - 192.168.3.19:9100
  - job_name: 'all-hpc-nvidia-gpu-exporter'
    static_configs:
    - targets:
      - 192.168.3.11:9445
      - 192.168.3.12:9445
      - 192.168.3.13:9445
      - 192.168.3.14:9445
      - 192.168.3.15:9445
      - 192.168.3.17:9445
      - 192.168.3.18:9445
      - 192.168.3.19:9445
  - job_name: 'k8s-test-node-exporter'
    static_configs:
    - targets:
      - 192.168.3.129:9100
      - 192.168.3.130:9100
      - 192.168.3.131:9100
  - job_name: 'kube-state-metrics'
    static_configs:
    - targets:
      - prom.yud.io
  - job_name: 'federate'
    scrape_interval: 15s
    honor_labels: true
    metrics_path: '/federate'
    params:
      'match[]':
        - '{job=~"kubernetes-.*"}'
    static_configs:
      - targets:
        - '192.168.3.129:30003'
  - job_name: etcd-server-metrics
    static_configs:
    - targets:
      - '192.168.3.134:2379'
      - '192.168.3.133:2379'
      - '192.168.3.132:2379'
```

## 还原

```
kubectl delete -f 
```

