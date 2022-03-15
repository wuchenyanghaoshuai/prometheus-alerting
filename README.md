```
基于prometheus-operator来实现的报警功
step1 安装prometheus-operator
step2 接入grafana
step3 安装完prometheus以后的操作
step4 接入dingding告警,压力测试,实现dingding告警
step5 Thanos存储持久化，多集群查询
step6 Prometheus-operator 文档
step7 code 可以参照本人源代码进行查看比较(本人k8s版本为1.20.0)
k8s prometheus监控
一、node
状态，内存，CPU，内存，负载，磁盘
二、pod
状态，内存，CPU，内存，负载，重启，OOM
三、nginx
Http_4xx_Error、Http_5xx_Error
四、jvm
堆内存，gc次数
```
