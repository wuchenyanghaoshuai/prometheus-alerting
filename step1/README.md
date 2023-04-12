#请先去该地址下找到自己k8s对应的版本(我是1.18.0k8s)如果是1.20.0版本的话，参考这个链接地址https://github.com/wuchenyanghaoshuai/prometheus-operator
#如果你的集群是阿里云的ACK,请先确保集群内没有prometheus，要不然会跟自建的prometheus有冲突，要去控制它卸载，图片附上
```
git clone https://github.com/prometheus-operator/kube-prometheus.git
git checkout release-0.5
cd manifests
kubectl apply -f setup
kubectl apply -f .
执行完yaml文件以后,kubectl get pods -n monitoring 来查看pod是否running
此处需要修改prometheus和grafana的 svc暴露方式为nodeport ,prometheus-service.yaml grafana-service.yaml
安装完成以后直接访问prometheus跟grafana的nodeport端口即可
```
![image](https://user-images.githubusercontent.com/39818267/134333127-541dc380-6813-4aef-8ff0-13c0356a0ee1.png)

<img width="1892" alt="image" src="https://user-images.githubusercontent.com/39818267/231376752-bfe94c18-6187-41f3-969c-2ce44bdefea2.png">


