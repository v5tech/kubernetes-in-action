# Kubernetes 集群中部署 MySQL

在 Kubernetes 中如何使用 PersistentVolume 和 Deployment 运行一个单实例有状态应用。 该示例应用是 MySQL。


部署PV 和 PVC
```bash
kubectl apply -f mysql-pv.yaml
```

部署 Deployment

```bash
kubectl apply -f mysql-deployment.yaml
```

查看 Deployment 信息

```bash
kubectl describe deployment mysql
```

查看 Deployment 创建的 Pod 信息

```bash
kubectl get pods -l app=mysql
```

查看 PersistentVolumeClaim

```bash
kubectl describe pvc mysql-pv-claim
```

访问 MySQL 数据库

```bash
kubectl run -it --rm --image=mysql:5.7 --restart=Never mysql-client -- mysql -h mysql -ppassword
```

清理资源

```bash
kubectl delete deployment,svc mysql
kubectl delete pvc mysql-pv-claim
kubectl delete pv mysql-pv-volume
```

# 参考文档

https://kubernetes.io/zh-cn/docs/tasks/run-application/run-single-instance-stateful-application/