```
# 部署deploy (webhook-dingtalk-deployment.yaml)(webhook-secret-template.yaml)
# 这个注意修改钉钉群的secret 跟token
# 修改完以后直接apply即可,alertmanager-secret 本来就有，可以替换也可以直接apply
```
![image](https://user-images.githubusercontent.com/39818267/134323901-92b22a4a-210d-42e2-adf5-4cba3386813c.png)

```
# 告警 rule
# 直接apply
```
```
# vim prometheus-rules2.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  labels:
    prometheus: k8s
    role: alert-rules
  name: prometheus-k8s-rules2
  namespace: monitoring
spec:
  groups:
  - name: mysql
    rules:
    - alert: Used more than 80% of max connections limited
      annotations:
        message: 使用超过80%连接限制
      expr: |
        mysql_global_status_max_used_connections > mysql_global_variables_max_connections * 0.8
      for: 5m
      labels:
        severity: critical
    - alert: Pod CPU too high
      annotations:
        message: Pod2分钟内cpu太高了
      expr: |
         sum(irate(container_cpu_usage_seconds_total[2m])) by (container, pod) / (sum(container_spec_cpu_quota /100000) by (container, pod))  * 100 > 80
      for: 2m
      labels:
        severity: critical
```
![image](https://user-images.githubusercontent.com/39818267/134330852-d9259b8b-44db-47d0-b6c6-ae14950d4aca.png)
```
# 测试deployment能往钉钉群里发消息
# curl http://localhost:8060/dingtalk/webhook/send   -H 'Content-Type: application/json'    -d '{"msgtype": "text","text": {"content": "监控告警"}}'
# 返回OK
```
![image](https://user-images.githubusercontent.com/39818267/134332049-1a484b3f-e131-4a38-80ba-9e8d4ff495a0.png)
![image](https://user-images.githubusercontent.com/39818267/134332073-ae2fb798-7be8-48a7-85af-465bf18cf916.png)
```
这样的话就是没有问题了,这个链路就打通了，剩下的就是手动让cpu负载变高了，然后过一会就有报警发出了
参考链接https://www.cnblogs.com/bigberg/p/13673033.html
报警和恢复都是有钉钉消息提示的
```
![image](https://user-images.githubusercontent.com/39818267/134332590-0a21cf63-1759-4241-af19-d89b934ddda9.png)
![image](https://user-images.githubusercontent.com/39818267/134332608-d8449f3b-fc92-4221-ae30-e4d8fa671c06.png)


```
