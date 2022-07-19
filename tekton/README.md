# Installing Tekton && Tekton Dashboard on Kubernetes

```bash
kubectl apply -f tekton.yaml
kubectl apply -f tekton-dashboard.yaml
kubectl apply -f tekton-dashboard-ingress.yaml
kubectl apply -f hello-world.yaml
kubectl apply -f hello-world-run.yaml
```

# 参考文档

* https://tekton.dev/docs/getting-started/tasks/
* https://tekton.dev/docs/dashboard/install/#installing-tekton-dashboard-on-kubernetes
* https://tekton.dev/docs/dashboard/install/#using-an-ingress-rule