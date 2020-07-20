# 监控kubernetes集群节点(二)

通过 Prometheus 来采集节点的监控指标数据，可以通过node_exporter来获取，，node_exporter 就是抓取用于采集服务器节点的各种运行指标，目前 node_exporter 支持几乎所有常见的监控点，比如 conntrack，cpu，diskstats，filesystem，loadavg，meminfo，netstat等。

通过 DaemonSet 控制器来部署该服务，这样每一个节点都会自动运行一个这样的 Pod，如果我们从集群中删除或者添加节点后，也会进行自动扩展。

```yaml
[root@ecm-f543 prometheus]# cat prometheus-node-exporter.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: kube-ops
  labels:
    name: node-exporter
spec:
  selector:
    matchLabels:
      name: node-exporter
  template:
    metadata:
      labels:
        name: node-exporter
    spec:
      hostPID: true
      hostIPC: true
      hostNetwork: true
      containers:
      - name: node-exporter
        image: prom/node-exporter:v0.16.0
        ports:
        - containerPort: 9100
        resources:
          requests:
            cpu: 0.15
        securityContext:
          privileged: true
        args:
        - --path.procfs
        - /host/proc
        - --path.sysfs
        - /host/sys
        - --collector.filesystem.ignored-mount-points
        - '"^/(sys|proc|dev|host|etc)($|/)"'
        volumeMounts:
        - name: dev
          mountPath: /host/dev
        - name: proc
          mountPath: /host/proc
        - name: sys
          mountPath: /host/sys
        - name: rootfs
          mountPath: /rootfs
      tolerations:
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: dev
          hostPath:
            path: /dev
        - name: sys
          hostPath:
            path: /sys
        - name: rootfs
          hostPath:
            path: /
```

由于要获取到的数据是主机的监控指标数据，而我们的 node-exporter 是运行在容器中的，所以我们在 Pod 中需要配置一些 Pod 的安全策略，这里我们就添加了`hostPID: true`、`hostIPC: true`、`hostNetwork: true`3个策略，用来使用主机的 PID namespace、IPC namespace 以及主机网络，这些 namespace 就是用于容器隔离的关键技术，要注意这里的 namespace 和集群中的 namespace 是两个完全不相同的概念。

另外还将主机的`/dev`、`/proc`、`/sys`这些目录挂载到容器中，这些因为我们采集的很多节点数据都是通过这些文件夹下面的文件来获取到的，比如我们在使用`top`命令可以查看当前`cpu`使用情况，数据就来源于文件`/proc/stat`，使用`free`命令可以查看当前内存使用情况，其数据来源是来自`/proc/meminfo`文件。

```
[root@ecm-f543 prometheus]# kubectl create -f prometheus-node-exporter.yaml
[root@ecm-f543 prometheus]# kubectl get pod -n kube-ops -o wide
NAME                          READY   STATUS    RESTARTS   AGE     IP             NODE          NOMINATED NODE   READINESS GATES
node-exporter-8zcvv           1/1     Running   0          108s    192.168.0.6    k8s-node3     <none>           <none>
node-exporter-fxwzq           1/1     Running   0          108s    192.168.0.8    k8s-node2     <none>           <none>
node-exporter-gk4n4           1/1     Running   0          108s    192.168.0.11   k8s-node1     <none>           <none>
node-exporter-plzn8           1/1     Running   0          108s    192.168.0.18   k8s-master1   <none>           <none>
node-exporter-rzj9r           1/1     Running   0          108s    192.168.0.13   k8s-master2   <none>           <none>
```

**服务发现**

在 Kubernetes 下，Promethues 通过与 Kubernetes API 集成，目前主要支持5中服务发现模式，分别是：Node、Service、Pod、Endpoints、Ingress。要让 Prometheus 也能够获取到当前集群中的所有节点信息的话，我们就需要利用 Node 的服务发现模式，同样的，在 prometheus.yml 文件中配置如下的 job 任务即可：

```yml
[root@ecm-f543 prometheus]# cat prometheus-cm.yaml
    - job_name: 'kubernetes-nodes'
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - source_labels: [__address__]
        regex: '(.*):10250'
        replacement: '${1}:9100'
        target_label: __address__
        action: replace
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
```

通过指定`kubernetes_sd_configs`的模式为`node`，Prometheus 就会自动从 Kubernetes 中发现所有的 node 节点并作为当前 job 监控的目标实例，发现的节点`/metrics`接口是默认的 kubelet 的 HTTP 接口。

prometheus 的 ConfigMap 更新完成后，同样的我们执行 reload 操作，让配置生效：

```
[root@ecm-f543 prometheus]# curl -X POST "http://10.0.0.67:9090/-/reload"
```

由于kubelet也带了一些监控指标，为10255端口，所以将kubelet的监控任务也一起配置上：

```yaml
[root@ecm-f543 prometheus]# cat prometheus-cm.yaml
    - job_name: 'kubernetes-kubelet'
      kubernetes_sd_configs:
      - role: node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
        
[root@ecm-f543 prometheus]# curl -X POST "http://10.0.0.67:9090/-/reload"
```

现在打开web ui可以看到kubernetes-kubelet和kubernetes-nodes的两个job已经配置成功了，二者的 Labels 标签都和集群的 node 节点标签保持一致了。











