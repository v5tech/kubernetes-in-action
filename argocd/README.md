# Install Argo CD

```bash
kubectl create namespace argocd
# daocloud镜像加速服务
wget -O image-filter.sh https://github.com/DaoCloud/public-image-mirror/raw/main/hack/image-filter.sh && chmod +x image-filter.sh
wget https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
cat ./install.yaml | ./image-filter.sh | kubectl apply -n argocd -f -
```

## 访问Argo CD UI

修改svc argocd-server为`LoadBalancer`

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

创建argocd-server-ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/ingress.class: nginx
    kubernetes.io/tls-acme: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  rules:
  - host: argocd.anchnet.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: 
            name: argocd-server
            port:
              name: https
  tls:
  - hosts:
    - argocd.anchnet.com
    secretName: argocd-secret
```

查看`admin`用户密码

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

浏览器访问 http://argocd.anchnet.com 输入用户名 `admin`及上面的密码登陆即可。


## 创建Argo CD Application

```bash
kubectl apply -f argocd-application.yaml
```

# 参考文档

* https://argo-cd.readthedocs.io/en/stable/getting_started/
* https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#kubernetesingress-nginx