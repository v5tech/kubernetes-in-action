# helm安装harbor

参考文章

https://blog.csdn.net/networken/article/details/126295863

https://blog.csdn.net/networken/article/details/126321152

https://blog.csdn.net/networken/article/details/124071068

## bitnami repo

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm upgrade --install -f harbor.yml harbor bitnami/harbor \
--namespace harbor \
--create-namespace
```

## goharbor repo

### Ingress方式暴露服务

#### 部署openebs 持久存储

harbor默认启用了数据持久化，依赖默认存储类提供pv卷，这里使用openebs

```bash
helm repo add openebs https://openebs.github.io/charts
helm repo update
helm install openebs openebs/openebs --namespace openebs --create-namespace
```

#### 部署ingress nginx 控制器

harbor 默认使用 ingress 方式暴露服务，依赖ingress控制器，这里使用ingress-nginx：

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.type=NodePort
```

获取nodeport

```bash
root@node01:~# kubectl -n ingress-nginx get svc
NAME                                 TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             NodePort    10.96.2.133   <none>        80:31523/TCP,443:31718/TCP   150m
ingress-nginx-controller-admission   ClusterIP   10.96.1.2     <none>        443/TCP                      150m
```

#### 部署 harbor 镜像仓库

添加harbor helm仓库

```bash
helm repo add harbor https://helm.goharbor.io
```

部署harbor仓库，ingress-nginx使用nodeport方式暴露自身，需要在externalURL中配置其 NodePort 端口号。

```bash
helm upgrade --install harbor harbor/harbor --namespace harbor --create-namespace \
  --set "expose.ingress.annotations.kubernetes\.io\/ingress\.class=nginx" \
  --set externalURL=https://harbor.lab.com:31718
```

查看harbor pods运行状态

```bash
root@node01:~# kubectl -n harbor get pods
NAME                                    READY   STATUS    RESTARTS        AGE
harbor-chartmuseum-787ff97489-7p45b     1/1     Running   0               4h53m
harbor-core-777f5cfc9c-46vvq            1/1     Running   0               4h53m
harbor-database-0                       1/1     Running   0               4h53m
harbor-jobservice-6d8c485bf8-s8f2k      1/1     Running   0               4h53m
harbor-notary-server-5fbf9fcb58-42nhq   1/1     Running   1 (4h52m ago)   4h53m
harbor-notary-signer-5894f4c77c-tn55p   1/1     Running   1 (4h52m ago)   4h53m
harbor-portal-685498cc69-tk4b7          1/1     Running   0               4h53m
harbor-redis-0                          1/1     Running   0               4h53m
harbor-registry-d9bb75d7b-8pvql         2/2     Running   0               4h53m
harbor-trivy-0                          1/1     Running   0               4h53m
```

查看pvc

```bash
root@node01:~# kubectl -n harbor get pvc
NAME                              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     AGE
data-harbor-redis-0               Bound    pvc-f9839aa4-89ed-4971-92ae-047b271e6205   1Gi        RWO            local-hostpath   17m
data-harbor-trivy-0               Bound    pvc-4b25125b-0fb8-40ed-8d3b-80c70f90cc5a   5Gi        RWO            local-hostpath   17m
database-data-harbor-database-0   Bound    pvc-7d826b47-066e-4da6-8fb8-734be3823667   1Gi        RWO            local-hostpath   17m
harbor-chartmuseum                Bound    pvc-432582f3-62f8-49da-96d0-37a01e015c57   5Gi        RWO            local-hostpath   17m
harbor-jobservice                 Bound    pvc-080863a2-277d-4293-93ec-dea149b051ec   1Gi        RWO            local-hostpath   17m
harbor-registry                   Bound    pvc-906d49c6-0fd8-435d-84f1-46b6e7132802   5Gi        RWO            local-hostpath   17m
```

查看ingress

```bash
root@node01:~# kubectl -n harbor get ingress
NAME                    CLASS    HOSTS                  ADDRESS       PORTS     AGE
harbor-ingress          <none>   harbor.lab.com     10.96.2.133   80, 443   4h53m
harbor-ingress-notary   <none>   notary.harbor.domain   10.96.2.133   80, 443   4h53m
```

浏览器访问harbor管理界面，如果没有DNS解析，注意将192.168.72.50 harbor.lab.com 加入本地hosts文件中，其中192.168.72.50 为kubernetes集群任意节点IP地址。

```
https://harbor.lab.com:31718/
```

默认用户名密码为`admin/Harbor12345`

集群外docker客户端验证上传镜像。

首先导出ca.crt证书

```bash
kubectl -n harbor get secrets harbor-ingress -o jsonpath="{.data.ca\.crt}" | base64 -d >ca.crt
```

复制ca.crt到docker客户端所在机器

```bash
root@ubuntu:~# mkdir -p /etc/docker/certs.d/harbor.lab.com:31718/
root@ubuntu:~# ls /etc/docker/certs.d/harbor.lab.com:31718/
ca.crt
```

配置hosts解析

```bash
echo "192.168.72.50 harbor.lab.com" >>/etc/hosts
```

登录harbor仓库

```bash
root@ubuntu:~# docker login -u admin -p Harbor12345 https://harbor.lab.com:31718
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See

Login Succeeded
```

推送镜像到harbor仓库

```bash
root@ubuntu:~# docker tag centos:latest harbor.lab.com:31718/library/centos:latest

root@ubuntu:~# docker push harbor.lab.com:31718/library/centos:latest
The push refers to repository [harbor.lab.com:31718/library/centos]
74ddd0ec08fa: Pushed 
latest: digest: sha256:a1801b843b1bfaf77c501e7a6d3f709401a1e0c83863037fa3aab063a7fdb9dc size: 529
```

### NodePort方式暴露服务

helm 部署harbor仓库，使用nodePort方式暴露服务，无需部署ingress-nginx控制器：

```bash
export node_ip=192.168.72.50
helm upgrade --install harbor harbor/harbor --namespace harbor --create-namespace \
  --set expose.type=nodePort \
  --set expose.tls.auto.commonName=$node_ip \
  --set externalURL='https://$node_ip:30003'
```

说明：其中 192.168.72.50 为kubernetes集群任一节点IP地址。

查看service

```bash
root@ubuntu:~# kubectl -n harbor get svc
NAME                   TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                                     AGE
harbor                 NodePort    10.96.1.217   <none>        80:30002/TCP,443:30003/TCP,4443:30004/TCP   100s
harbor-chartmuseum     ClusterIP   10.96.0.116   <none>        80/TCP                                      100s
harbor-core            ClusterIP   10.96.0.125   <none>        80/TCP                                      100s
harbor-database        ClusterIP   10.96.1.189   <none>        5432/TCP                                    100s
harbor-jobservice      ClusterIP   10.96.1.99    <none>        80/TCP                                      100s
harbor-notary-server   ClusterIP   10.96.0.5     <none>        4443/TCP                                    100s
harbor-notary-signer   ClusterIP   10.96.0.164   <none>        7899/TCP                                    100s
harbor-portal          ClusterIP   10.96.1.25    <none>        80/TCP                                      100s
harbor-redis           ClusterIP   10.96.0.224   <none>        6379/TCP                                    100s
harbor-registry        ClusterIP   10.96.3.233   <none>        5000/TCP,8080/TCP                           100s
harbor-trivy           ClusterIP   10.96.2.193   <none>        8080/TCP                                    100s
```

浏览器访问harbor，使用节点IP+nodePort方式访问，使用默认用户名密码`admin/Harbor12345`进行登录：

```
https://192.168.72.50:30003/
```

docker客户端配置。

首先导出ca.crt证书

```bash
kubectl -n harbor get secrets harbor-nginx -o jsonpath="{.data.ca\.crt}" | base64 -d >ca.crt
```

复制ca.crt到docker客户端所在机器

```bash
root@ubuntu:~# mkdir -p /etc/docker/certs.d/192.168.72.50:30003/
root@ubuntu:~# ls /etc/docker/certs.d/192.168.72.50:30003/
ca.crt
```

登录harbor仓库

```bash
root@ubuntu:~# docker login -u admin -p Harbor12345 https://192.168.72.50:30003
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See

Login Succeeded
```

推送镜像到harbor仓库

```bash
root@ubuntu:~# docker tag centos:latest 192.168.72.50:30003/library/centos:latest

root@ubuntu:~# docker push 192.168.72.50:30003/library/centos:latest
The push refers to repository [192.168.72.50:30003/library/centos]
74ddd0ec08fa: Pushed 
latest: digest: sha256:a1801b843b1bfaf77c501e7a6d3f709401a1e0c83863037fa3aab063a7fdb9dc size: 529
```