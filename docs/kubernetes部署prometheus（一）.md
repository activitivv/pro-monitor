# kubernetes部署prometheus

**为了方便管理配置文件，将`prometheus.ym`l文件用`ConfigMap`的形式管理**

```
[root@ecm-f543 prometheus]# cat prometheus-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: kube-ops
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      scrape_timeout: 15s
    scrape_configs:
    - job_name: 'prometheus'
      static_configs:
      - targets: ['localhost:9090']
```

**这里暂时只配置了对prometheus的监控，创建该资源对象**

```
[root@ecm-f543 prometheus]# kubectl create -f prometheus-cm.yaml
configmap/prometheus-config created
```

配置文件完成了，如果以后有新的资源需要被监控，只需要将上面的`ConfigMap`对象更新即可。接下来创建prometheus的Pod资源（prometheus-deploy.yaml)：

```yaml
[root@ecm-f543 prometheus]# cat prometheus-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: kube-ops
  labels:
    app: prometheus
spec:
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus
      containers:
      - image: prom/prometheus:v2.4.3
        name: prometheus
        command:
        - "/bin/prometheus"
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        - "--storage.tsdb.path=/prometheus"
        - "--storage.tsdb.retention=24h"
        - "--web.enable-admin-api"  # 控制对admin HTTP API的访问，其中包括删除时间序列等功能
        - "--web.enable-lifecycle"  # 支持热更新，直接执行localhost:9090/-/reload立即生效
        ports:
        - containerPort: 9090
          protocol: TCP
          name: http
        volumeMounts:
        - mountPath: "/prometheus"
          subPath: prometheus
          name: data
        - mountPath: "/etc/prometheus"
          name: config-volume
        resources:
          requests:
            cpu: 100m
            memory: 512Mi
          limits:
            cpu: 100m
            memory: 512Mi
      securityContext:
        runAsUser: 0
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: prometheus
      - configMap:
          name: prometheus-config
        name: config-volume
```

启动程序的时候，除了指定prometheus.yml之外，还通过参数`storage.tsdb.path`指定了tsdb的存储路径、通过`web.enable-admin-api`参数来开启对admin api的访问权限，参数`web.enable-lifecycle`非常重要，用来开启支持热更新的，有了这个参数之后，prometheus.yml 配置文件只要更新了，通过执行`localhost:9090/-/reload`就会立即生效，所以一定要加上这个参数

**创建pvc资源**

为了将时间序列数据进行持久化，我们将数据目录和一个 pvc 对象进行了绑定，所以我们需要提前创建好这个 pvc 对象：(prometheus-volume.yaml)

```yaml
[root@ecm-f543 prometheus]# cat prometheus-volume.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: prometheus
spec:
  storageClassName: local  # Local PV
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  local:
    path: /data/prometheus
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - k8s-node3
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prometheus
  namespace: kube-ops
spec:
  storageClassName: local
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
```

通过本地存储来创建一个pv，pvc对象：

```
[root@ecm-f543 prometheus]# kubectl create -f prometheus-volume.yaml
persistentvolume/prometheus created
persistentvolumeclaim/prometheus created
```

配置rbac认证，去在prometheus中访问kubernetes的相关信息，所以创建了一个名为 prometheus 的 serviceAccount 对象：(prometheus-rbac.yaml)

```yaml
[root@ecm-f543 prometheus]# cat prometheus-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: kube-ops
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups:
  - ""
  resources:
  - nodes
  - services
  - endpoints
  - pods
  - nodes/proxy
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - configmaps
  - nodes/metrics
  verbs:
  - get
- nonResourceURLs:
  - /metrics
  verbs:
  - get
---
apiVersion: rbac.authorization.k8s.io/v1beta1
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
  namespace: kube-ops
```

因为要获取的资源信息在每一个namespace都可能存在，用的是clusterRole资源对象。

```
[root@ecm-f543 prometheus]# kubectl create -f prometheus-rbac.yaml
serviceaccount/prometheus created
clusterrole.rbac.authorization.k8s.io/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
```

**创建prometheus的资源对象**

```
[root@ecm-f543 prometheus]# kubectl create -f prometheus-deploy.yaml
deployment.apps/prometheus created
[root@ecm-f543 prometheus]# kubectl get pod -n kube-ops prometheus-75b7f58cb9-fj9gj
NAME                          READY   STATUS    RESTARTS   AGE
prometheus-75b7f58cb9-fj9gj   1/1     Running   0          7m27s
[root@ecm-f543 prometheus]# kubectl logs -f -n kube-ops prometheus-75b7f58cb9-fj9gj
level=info ts=2020-07-19T13:59:40.362707299Z caller=main.go:523 msg="Server is ready to receive web requests."
```

**创建svc**

为了能访问到prometheus的web服务，需要创建一个service对象（prometheus.yaml):

```yaml
[root@ecm-f543 prometheus]# cat prometheus-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: kube-ops
  labels:
    app: prometheus
spec:
  selector:
    app: prometheus
  type: NodePort
  ports:
    - name: web
      port: 9090
      targetPort: http
      nodePort: 30090
```

通过网页访问`任意节点ip:30090`打开web界面。







