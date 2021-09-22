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

