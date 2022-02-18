#请先去该地址下找到自己k8s对应的版本(我是1.18.0k8s)如果是1.20.0版本的话，参考这个链接地址https://github.com/wuchenyanghaoshuai/prometheus-operator

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



