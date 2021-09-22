```
# 查看prometheus的target我们可以看到kube-controller-manager 和 kube-scheduler是0/1
```
![image](https://user-images.githubusercontent.com/39818267/134302331-65be4653-32b4-4cc2-a923-9bdde139babf.png)

```
# 是因为servicemonitor里面没有对应的lable,prometheus-serviceMonitorKubeScheduler.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: kube-scheduler
  name: kube-scheduler
  namespace: monitoring
spec:
  endpoints:
  - interval: 30s  # 每30s获取一次信息
    port: http-metrics  # 对应 service 的端口名
  jobLabel: k8s-app
  namespaceSelector:  # 表示去匹配某一命名空间中的service，如果想从所有的namespace中匹配用any: true
    matchNames:
    - kube-system
  selector:  # 匹配的 Service 的 labels，如果使用 mathLabels，则下面的所有标签都匹配时才会匹配该 service，如果使用 matchExpressions，则至少匹配一个标签的 service 都会被选择
    matchLabels:
      k8s-app: kube-scheduler
上面是一个典型的 ServiceMonitor 资源对象的声明方式，上面我们通过 selector.matchLabels 在 kube-system 这个命名空间下面匹配具有 k8s-app=kube-scheduler 这样的 Service，但是我们系统中根本就没有对应的 Service：

$ kubectl get svc -n kube-system -l k8s-app=kube-scheduler
No resources found.
所以我们需要去创建一个对应的 Service 对象，才能核 ServiceMonitor 进行关联：(prometheus-kubeSchedulerService.yaml)

apiVersion: v1
kind: Service
metadata:
  namespace: kube-system
  name: kube-scheduler
  labels:
    k8s-app: kube-scheduler
spec:
  selector:
    component: kube-scheduler
  ports:
  - name: http-metrics
    port: 10251
    targetPort: 10251
其中最重要的是上面 labels 和 selector 部分，labels 区域的配置必须和我们上面的 ServiceMonitor 对象中的 selector 保持一致，selector 下面配置的是 component=kube-scheduler

$ kubectl apply -f prometheus-kubeSchedulerService.yaml
$ kubectl get svc -n kube-system -l k8s-app=kube-scheduler
NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
kube-scheduler   ClusterIP   172.21.0.139   <none>        10251/TCP   6d4h
创建完成后，隔一小会儿后去 Prometheus 页面上查看 targets 下面 kube-scheduler 已经可以采集到指标数据了。可以用同样的方式来修复下 kube-controller-manager 组件的监控，只需要创建一个如下所示的 Service 对象，只是端口改成 10252 即可：(prometheus-kubeControllerManagerService.yaml)

apiVersion: v1
kind: Service
metadata:
  namespace: kube-system
  name: kube-controller-manager
  labels:
    k8s-app: kube-controller-manager
spec:
  selector:
    component: kube-controller-manager
  ports:
  - name: http-metrics
    port: 10252
    targetPort: 10252
```
![image](https://user-images.githubusercontent.com/39818267/134308663-9c368351-35cd-466d-a4d6-83de1ea8e4eb.png)
```
# 这个时候就可以看到基础组建监控正常了,如果此时你的k8s中有pod的话，你就会发现为啥你的pod没有被监控到呢,那是因为你没有配置自动发现的规则,下面配置一下自动发现
- job_name: 'kubernetes-endpoints'
  kubernetes_sd_configs:
  - role: endpoints
  relabel_configs:
  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
    action: keep
    regex: true
  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
    action: replace
    target_label: __scheme__
    regex: (https?)
  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
    action: replace
    target_label: __metrics_path__
    regex: (.+)
  - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
    action: replace
    target_label: __address__
    regex: ([^:]+)(?::\d+)?;(\d+)
    replacement: $1:$2
  - action: labelmap
    regex: __meta_kubernetes_service_label_(.+)
  - source_labels: [__meta_kubernetes_namespace]
    action: replace
    target_label: kubernetes_namespace
  - source_labels: [__meta_kubernetes_service_name]
    action: replace
    target_label: kubernetes_name
  - source_labels: [__meta_kubernetes_pod_name]
    action: replace
    target_label: kubernetes_pod_name
- job_name: 'kubernetes-pods'
  honor_timestamps: true
  scrape_interval: 30s
  scrape_timeout: 10s
  metrics_path: /metrics
  scheme: http
  kubernetes_sd_configs:
  - role: pod
  relabel_configs:
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    separator: ;
    regex: "true"
    replacement: $1
    action: keep
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scheme]
    separator: ;
    regex: (https?)
    target_label: __scheme__
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
    separator: ;
    regex: (.+)
    target_label: __metrics_path__
    replacement: $1
    action: replace
  - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
    separator: ;
    regex: ([^:]+)(?::\d+)?;(\d+)
    target_label: __address__
    replacement: $1:$2
    action: replace
  - separator: ;
    regex: __meta_kubernetes_service_label_(.+)
    replacement: $1
    action: labelmap
  - source_labels: [__meta_kubernetes_namespace]
    separator: ;
    regex: (.*)
    target_label: kubernetes_namespace
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_service_name]
    separator: ;
    regex: (.*)
    target_label: kubernetes_name
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_pod_name]
    separator: ;
    regex: (.*)
    target_label: kubernetes_pod_name
    replacement: $1
    action: replace
# 要想自动发现集群中的 Service，就需要我们在 Service 的 annotation 区域添加 prometheus.io/scrape=true 的声明
# 要想自动发现集群中的 pod，就需要我们在 pod 的 template.metadata.annotations 区域添加 prometheus.io/scrape=true 的声明
将上面文件直接保存为 prometheus-additional.yaml，然后通过这个文件创建一个对应的 Secret 对象：

$ kubectl create secret generic additional-configs --from-file=prometheus-additional.yaml -n monitoring
secret "additional-configs" created
然后我们需要在声明 prometheus 的资源对象文件中通过 additionalScrapeConfigs 属性添加上这个额外的配置：(prometheus-prometheus.yaml)

# 在最后面添加配置
  ···
  additionalScrapeConfigs:
    name: additional-configs
    key: prometheus-additional.yaml
```
