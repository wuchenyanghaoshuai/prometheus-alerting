```
grafana的账号密码均为admin,登录进去以后添加一个datasource即可,就是写prometheus的svc,我的是prometheus-k8s.monitoring:9090
点击左侧➕ ,点击Import 在id那一栏输入13105点击load即可
然后你就得到一个漂亮的图案了

```
![image](https://user-images.githubusercontent.com/39818267/133442411-19ea2d7d-c072-4050-9924-afe1df2317c0.png)
