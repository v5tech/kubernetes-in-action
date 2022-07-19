# Install Argo CD

```bash
kubectl apply -f install.yaml
kubectl apply -f argocd-server-ingress.yaml
kubectl apply -f argocd-application.yaml
```

# 参考文档

* https://argo-cd.readthedocs.io/en/stable/getting_started/
* https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#kubernetesingress-nginx