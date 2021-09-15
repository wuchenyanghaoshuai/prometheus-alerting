```
git clone https://github.com/prometheus-operator/kube-prometheus.git
git checkout release-0.5
cd manifests
kubectl apply -f setup
kubectl apply -f .
执行完yaml文件以后,kubectl get pods -n monitoring 来查看pod是否running
此处需要修改prometheus和grafana的 svc暴露方式为nodeport ,prometheus-service.yaml grafana-service.yaml
安装完成以后直接访问prometheus跟grafana的nodeport端口即可
grafana的账号密码均为admin,登录进去以后添加一个datasource即可,就是写prometheus的svc,我的是prometheus-k8s.monitoring:9090
点击左侧➕ ,点击Import 在id那一栏输入13105点击load即可
```
![image](https://user-images.githubusercontent.com/39818267/133442411-19ea2d7d-c072-4050-9924-afe1df2317c0.png)

#注意
```
具体在这个位置 https://github.com/wuchenyanghaoshuai/k8s-hpa/tree/main/step3
```
