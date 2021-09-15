```
git clone https://github.com/prometheus-operator/kube-prometheus.git
git checkout release-0.5
cd manifests
kubectl apply -f setup
kubectl apply -f .
执行完yaml文件以后,kubectl get pods -n monitoring 来查看pod是否running
此处需要修改prometheus和grafana的 svc暴露方式为nodeport ,prometheus-service.yaml grafana-service.yaml
安装完成以后直接访问prometheus跟grafana的nodeport端口即可
然后请看step2
```
![image](https://user-images.githubusercontent.com/39818267/133442411-19ea2d7d-c072-4050-9924-afe1df2317c0.png)

#注意
```
具体在这个位置 https://github.com/wuchenyanghaoshuai/k8s-hpa/tree/main/step3
```
