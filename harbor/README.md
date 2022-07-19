# Harbor安装

```bash
helm repo add harbor https://helm.goharbor.io
helm repo update
helm install harbor harbor/harbor -f values-prod.yaml --create-namespace harbor
```

# 参考文档

* https://computingforgeeks.com/install-harbor-image-registry-on-kubernetes-openshift-with-helm-chart/
* https://mp.weixin.qq.com/s/ZqLu97S6VluRc1Mu0jL1Uw