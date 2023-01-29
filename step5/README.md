# 
```
多集群监控组件Thanos
边车组件（Sidecar）：连接 Prometheus，并把 Prometheus 暴露给查询网关（Querier/Query），以供实时查询，并且可以上传 Prometheus 数据给云存储，以供长期保存
查询网关（Querier/Query）：实现了 Prometheus API，与汇集底层组件（如边车组件 Sidecar，或是存储网关 Store Gateway）的数据
存储网关（Store Gateway）：将云存储中的数据内容暴露出来
压缩器（Compactor）：将云存储中的数据进行压缩和下采样
接收器（Receiver）：从 Prometheus 的 remote-write WAL（Prometheus 远程预写式日志）获取数据，暴露出去或者上传到云存储
规则组件（Ruler）：针对监控数据进行评估和报警
Bucket：主要用于展示对象存储中历史数据的存储情况，查看每个指标源中数据块的压缩级别，解析度，存储时段和时间长度等信息
```

```
#1.创建storageClass
直接apply目录里面的yaml即可,注意修改nfs的地址跟挂载的目录
```
```
2.由于prometheus-operator 支持thanos扩展，我们直接在prometheus-prometheus.yaml最后添加 thanos 配置,
  注意此图内还有一个storage的字段要添加,这个是查询时用的如果不添加的话,使用thanos的querier的时候会报错  
  "No StoreAPIs matched for this queryNo StoreAPIs matched for this query"
  thanos:  #  添加 thanos 配置
    objectStorageConfig:
      key: thanos.yaml
      name: thanos-objectstorage
```
![image](https://user-images.githubusercontent.com/39818267/134810002-996f9efc-7c8b-4005-bf02-f821fe31143f.png)
![image](https://user-images.githubusercontent.com/39818267/158297871-e9818f30-18f3-41ff-887c-48ec15e06242.png)
<img width="268" alt="image" src="https://user-images.githubusercontent.com/39818267/158347687-bfa069a4-0ddc-4dc4-8a5a-84022d6ae9a6.png">


```
Thanos 的 Querier 组件来提供一个全局的统一查询入口。对于 Quierier 最重要的就是要配置上 Thanos 的 Sidecar 地址，
我们这里完全可以直接使用 Headless Service 去自动发现：(querier.yaml)
apiVersion: v1
kind: Service
metadata:
  name: thanos-querier
  namespace: monitoring
  labels:
    app: thanos-querier
spec:
  ports:
  - port: 9090
    protocol: TCP
    targetPort: http
    nodePort: 60000
    name: http
  selector:
    app: thanos-querier
  type: NodePort

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: thanos-querier
  namespace: monitoring
  labels:
    app: thanos-querier
spec:
  selector:
    matchLabels:
      app: thanos-querier
  template:
    metadata:
      labels:
        app: thanos-querier
    spec:
      containers:
      - name: thanos
        image: thanosio/thanos:v0.11.0
        args:
        - query
        - --log.level=debug
        - --query.replica-label=prometheus_replica
        # Discover local store APIs using DNS SRV.
        - --store=dnssrv+thanos-store-gateway:10901
        ports:
        - name: http
          containerPort: 10902
        - name: grpc
          containerPort: 10901
        resources:
          requests:
            memory: "2Gi"
            cpu: 500m
          limits:
            memory: "4Gi"
            cpu: "2"
        livenessProbe:
          httpGet:
            path: /-/healthy
            port: http
          initialDelaySeconds: 10
        readinessProbe:
          httpGet:
            path: /-/healthy
            port: http
          initialDelaySeconds: 15
```

```
Compactor 组件
Thanos 的 Compactor 组件，用来将对象存储中的数据进行压缩和下采样。
Compactor 组件的部署和 Store 非常类似，指定对象存储的配置文件即可，如下所示的资源清单文件：（compactor.yaml）
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: thanos-compactor
  namespace: monitoring
  labels:
    app: thanos-compactor
spec:
  replicas: 1
  selector:
    matchLabels:
      app: thanos-compactor
  serviceName: thanos-compactor
  template:
    metadata:
      labels:
        app: thanos-compactor
    spec:
      containers:
      - name: thanos
        image: thanosio/thanos:v0.11.0
        args:
        - "compact"
        - "--log.level=debug"
        - "--data-dir=/data"
        - "--objstore.config-file=/etc/secret/thanos.yaml"
        - "--wait"
        ports:
        - name: http
          containerPort: 10902
        livenessProbe:
          httpGet:
            port: 10902
            path: /-/healthy
          initialDelaySeconds: 10
        readinessProbe:
          httpGet:
            port: 10902
            path: /-/ready
          initialDelaySeconds: 15
        volumeMounts:
        - name: object-storage-config
          mountPath: /etc/secret
          readOnly: false
      volumes:
      - name: object-storage-config   # 挂载对象存储配置
        secret:
          secretName: thanos-objectstorage
```

```
Store 组件
Thanos Store 组件，将历史监控指标存储在对象存储中
目前 Thanos 支持的对象存储有
```
![image](https://user-images.githubusercontent.com/39818267/134810026-79e0d050-10ef-4c83-acdb-a6445d7195d1.png)

```
我们这里使用流行的第三方软件模拟S3存储

安装 Minio

MinIO 是一个基于 Apache License v2.0 开源协议的对象存储服务。它兼容亚马逊 S3 云存储服务接口，非常适合于存储大容量非结构化的数据，

例如图片、视频、日志文件、备份数据和容器/虚拟机镜像等，而一个对象文件可以是任意大小，从几 kb 到最大 5T 不等。


要安装 Minio 非常容易的，同样我们这里将 Minio 安装到 Kubernetes 集群中，可以直接参考官方文档 使用Kubernetes部署MinIO，

在 Kubernetes 集群下面可以部署独立、分布式或共享几种模式，可以根据实际情况部署，我们这里为了简单直接部署独立模式。


为了方便管理，将所有的资源对象都部署在一个名为 minio 的命名空间中，如果没有的话需要手动创建。

直接使用 Deployment 来管理 Minio 的服务：（minio-deploy.yaml）
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-pvc
  namespace: minio
spec:
  storageClassName: managed-nfs-storage
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi

---
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: minio
spec:
  ports:
  - port: 9000
    targetPort: 9000
    protocol: TCP
    nodePort: 30009
  type: NodePort
  selector:
    app: minio

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: minio
spec:
  selector:
    matchLabels:
      app: minio
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: minio
    spec:
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: minio-pvc
      containers:
      - name: minio
        volumeMounts:
        - name: data
          mountPath: "/data"
        image: minio/minio:RELEASE.2020-11-06T23-17-07Z
        args:
        - server
        - /data
        env:
        - name: MINIO_ACCESS_KEY
          value: "minio"
        - name: MINIO_SECRET_KEY
          value: "minio123"
        ports:
        - containerPort: 9000
        resources:
          requests:
            cpu: 100m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 2048Mi
        readinessProbe:
          httpGet:
            path: /minio/health/ready
            port: 9000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /minio/health/live
            port: 9000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
```
```
#点击右下角创建bucket即可 prometheus-thanos(看配置文件里面定义的)
#账号密码 minio/minio123
```
![image](https://user-images.githubusercontent.com/39818267/134810075-1b482545-13c5-466a-ae52-48b61ab63560.png)

```
#登录上minio创建一个bucket
```

```
创建一个对象存储配置文件：（09thanos-storage-minio.yaml） 
apiVersion: v1
kind: Secret
metadata:
  name: thanos-objectstorage
  namespace: monitoring
type: Opaque
data: {}
stringData:
  thanos.yaml: |-
    type: s3
    config:
      bucket: prometheus-thanos
      endpoint: minio.minio.svc.cluster.local:9000
      access_key: minio
      secret_key: minio123
      insecure: true
      signature_version2: false
```

```
安装 Thanos Store
Prometheus和thanos-store-gateway的有状态集被标记为thanos-store-api：“ true”，以便无头服务发现每个pod。

Thanos Query将使用此无头服务来查询所有Prometheus实例中的数据（store.yaml）

apiVersion: v1
kind: Service
metadata:
  name: thanos-store-gateway
  namespace: monitoring
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: grpc
    port: 10901
    targetPort: grpc
  selector:
    thanos-store-api: "true"   # 注意这里标签

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: thanos-store-gateway
  namespace: monitoring
  labels:
    app: thanos-store-gateway
spec:
  replicas: 1
  selector:
    matchLabels:
      app: thanos-store-gateway
  serviceName: thanos-store-gateway
  template:
    metadata:
      labels:
        app: thanos-store-gateway
        thanos-store-api: "true"   # 注意这里标签
    spec:
      containers:
        - name: thanos
          image: thanosio/thanos:v0.11.0
          args:
          - "store"
          - "--log.level=debug"
          - "--data-dir=/data"
          - "--objstore.config-file=/etc/secret/thanos.yaml"
          - "--index-cache-size=500MB"
          - "--chunk-pool-size=500MB"
          ports:
          - name: http
            containerPort: 10902
          - name: grpc
            containerPort: 10901
          livenessProbe:
            httpGet:
              port: 10902
              path: /-/healthy
          readinessProbe:
            httpGet:
              port: 10902
              path: /-/ready
          requests:
            memory: "2Gi"
            cpu: 500m
          limits:
            memory: "4Gi"
            cpu: "2"
          volumeMounts:
            - name: object-storage-config
              mountPath: /etc/secret
              readOnly: false
      volumes:
        - name: object-storage-config   # 挂载对象存储配置
          secret:
            secretName: thanos-objectstorage 
```

```
查询组件，能抓取sidecar的热数据，和store的旧数据

kubectl -n monitoring port-forward svc/thanos-querier 9090:9090
```
![image](https://user-images.githubusercontent.com/39818267/134810312-8e729c2f-e59f-4b34-b745-a766ecda913e.png)
```
#最后将grafana的抓取地址改为thanos-querier的地址,然后用新的地址来创建图即可
```
![image](https://user-images.githubusercontent.com/39818267/134810345-f321d83e-65f6-4ff3-9215-11707a78d138.png)
![image](https://user-images.githubusercontent.com/39818267/134810430-18901912-303e-46f9-a157-3d0a4fd06847.png)

```
#参考文档
https://github.com/prometheus-operator/kube-prometheus
https://github.com/timonwong/prometheus-webhook-dingtalk
https://www.cnblogs.com/bigberg/p/13673033.html
https://github.com/prometheus-operator/prometheus-operator/issues/926  # kubelet targets down - 401 Unauthorized
https://docs.min.io/cn/deploy-minio-on-kubernetes.html
https://www.mdeditor.tw/pl/pi4m
https://www.metricfire.com/blog/ha-kubernetes-monitoring-using-prometheus-and-thanos/?GAID=1579914937.1612180807
https://k8s.imroc.io/monitoring/build-cloud-native-large-scale-distributed-monitoring-system/thanos-deploy/
```

