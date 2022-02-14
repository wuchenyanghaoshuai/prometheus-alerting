新版本的k8s要加这个参数,在kube-apiserver.yaml里
原来是1.20版本（我的是1.20.2）默认禁止使用selfLink
- --feature-gates=RemoveSelfLink=false
