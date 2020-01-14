# Prometheus
在每个企业的数据中心内或多或少都会使用一些开源或者商业的监控系统，从监控对象的角度来看，可以将监控分为网络监控、存储监控、服务器监控和应用监控等。因为需要监控数据中心的各个方面，所以监控系统需要做到面面俱到，在数据中心里充当 "天眼" 角色。


|基础资源监控|中间件监控|应用程序监控|日志监控|
|:-----------:|:---------:|:------------:|:-------:|
|CPU|缓存|调用链|...|
|内存|消息|HTTP延迟|...|
|网络|服务|性能|...|
存储|数据库|监控状态|...|


# Install Prometheus With Kubernetes
```shell script
[root@k8s-master01 data]# ll prometheus-*
-rw-r--r-- 1 root root  395 Jan  7 11:27 prometheus-cm.yaml
-rw-r--r-- 1 root root 1069 Jan  7 11:31 prometheus-rbac.yaml
-rw-r--r-- 1 root root 3166 Jan  8 11:36 prometheus-statefulset.yaml
-rw-r--r-- 1 root root  364 Jan  7 11:32 prometheus-svc.yaml
```

```shell script
[root@k8s-master01 data]# cat prometheus-cm.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: EnsureExists
data:
  prometheus.yml: |
    global:
      scrape_interval: 30s
      scrape_timeout: 30s

    scrape_configs:
    - job_name: 'prometheus'
      static_configs:
        - targets: ['localhost:9090']

```
```shell script
[root@k8s-master01 data]# cat prometheus-rbac.yaml 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
rules:
  - apiGroups:
      - ""
    resources:
      - nodes
      - nodes/metrics
      - services
      - endpoints
      - pods
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - get
  - nonResourceURLs:
      - "/metrics"
    verbs:
      - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: kube-system
```

```shell script
[root@k8s-master01 data]# cat prometheus-statefulset.yaml 
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: prometheus
  namespace: kube-system
  labels:
    k8s-app: prometheus
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    version: v2.2.1
spec:
  serviceName: "prometheus"
  replicas: 1
  podManagementPolicy: "Parallel"
  updateStrategy:
   type: "RollingUpdate"
  selector:
    matchLabels:
      k8s-app: prometheus
  template:
    metadata:
      labels:
        k8s-app: prometheus
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      priorityClassName: system-cluster-critical
      serviceAccountName: prometheus
      initContainers:
      - name: "init-chown-data"
        image: "busybox:latest"
        imagePullPolicy: "IfNotPresent"
        #command: ["chown", "-R", "65534:65534", "/data"]
        volumeMounts:
        - name: prometheus-data
          mountPath: /data
          subPath: ""
        securityContext:
          privileged: true
          capabilities:
            add: ["SYS_ADMIN"]
          allowPrivilegeEscalation: true
      containers:
        - name: prometheus-server-configmap-reload
          image: "jimmidyson/configmap-reload:v0.1"
          imagePullPolicy: "IfNotPresent"
          args:
            - --volume-dir=/etc/config
            - --webhook-url=http://localhost:9090/-/reload
          volumeMounts:
            - name: config-volume
              mountPath: /etc/config
              readOnly: true
          resources:
            limits:
              cpu: 10m
              memory: 10Mi
            requests:
              cpu: 10m
              memory: 10Mi

        - name: prometheus-server
          image: "prom/prometheus:v2.4.3"
          imagePullPolicy: "IfNotPresent"
          args:
            - --config.file=/etc/config/prometheus.yml
            - --storage.tsdb.path=/data
            - --web.console.libraries=/etc/prometheus/console_libraries
            - --web.console.templates=/etc/prometheus/consoles
            - --web.enable-lifecycle
          ports:
            - containerPort: 9090
          readinessProbe:
            httpGet:
              path: /-/ready
              port: 9090
            initialDelaySeconds: 30
            timeoutSeconds: 30
          livenessProbe:
            httpGet:
              path: /-/healthy
              port: 9090
            initialDelaySeconds: 30
            timeoutSeconds: 30
          resources:
            limits:
              cpu: 200m
              memory: 1000Mi
            requests:
              cpu: 200m
              memory: 1000Mi

          volumeMounts:
            - name: config-volume
              mountPath: /etc/config
            - name: prometheus-data
              mountPath: /data
              subPath: ""
      terminationGracePeriodSeconds: 300
      volumes:
        - name: config-volume
          configMap:
            name: prometheus-config
  volumeClaimTemplates:
  - metadata:
      name: prometheus-data
    spec:
      storageClassName: course-nfs-storage
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: "16Gi"
```

```shell script
[root@k8s-master01 data]# cat prometheus-svc.yaml 
kind: Service
apiVersion: v1
metadata:
  name: prometheus
  namespace: kube-system
  labels:
    kubernetes.io/name: "Prometheus"
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  ports:
    - name: http
      port: 9090
      protocol: TCP
      targetPort: 9090
  type: NodePort
  selector:
    k8s-app: prometheus
```


#参考文章：
https://www.jianshu.com/p/8788820f7110
https://www.jianshu.com/p/307f106b8dd8
