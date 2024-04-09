# 使用 Kubernetes 部署 MySQL 主从复制、读写分离

1、创建ConfigMap

```bash
kubectl apply -f mysql-configmap.yaml
```

2、创建Secret

```bash
kubectl apply -f mysql-secret.yaml
```

3、创建Services

```bash
kubectl apply -f mysql-services.yaml
```

4、创建StatefulSet

```bash
kubectl apply -f mysql-statefulset.yaml
```

5、查看Pod

```bash
kubectl get pods -l app=mysql --watch
```

6、验证主从状态

```bash
kubectl exec mysql-0 -c mysql -- bash -c "mysql -uroot -p123456 -e 'show master status \G'"
kubectl exec mysql-1 -c mysql -- bash -c "mysql -uroot -p123456 -e 'show slave status \G'"
kubectl exec mysql-2 -c mysql -- bash -c "mysql -uroot -p123456 -e 'show slave status \G'"
```

7、操作主库创建数据库及表结构

```bash
kubectl run mysql-client --image=mysql:5.7 -i --rm --restart=Never --\
  mysql -h mysql-0.mysql -uroot -p123456 <<EOF
CREATE DATABASE test;
CREATE TABLE test.messages (message VARCHAR(250));
INSERT INTO test.messages VALUES ('hello');
EOF
```

8、查询读库验证数据

```bash
kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
  mysql -h mysql-read -uroot -p123456 -e "SELECT * FROM test.messages"
# 或者
kubectl exec mysql-1 -c mysql -- bash -c "mysql -uroot -p123456 -e 'SELECT * FROM test.messages'"
kubectl exec mysql-2 -c mysql -- bash -c "mysql -uroot -p123456 -e 'SELECT * FROM test.messages'"
```

9、循环打印@@server_id

```bash
kubectl run mysql-client-loop --image=mysql:5.7 -i -t --rm --restart=Never --\
  bash -ic "while sleep 1; do mysql -h mysql-read -uroot -p123456 -e 'SELECT @@server_id,NOW()'; done"
```

10、副本扩容

```bash
kubectl scale statefulset mysql  --replicas=5
kubectl get pods -l app=mysql --watch
```

11、副本缩容

```bash
kubectl scale statefulset mysql --replicas=3
kubectl get pods -l app=mysql --watch
```

12、查看pvc

```bash
kubectl get pvc -l app=mysql
```

13、清理环境

```bash
kubectl delete pod mysql-client-loop --now
kubectl delete statefulset mysql
kubectl get pods -l app=mysql
kubectl delete configmap,secret,service,pvc -l app=mysql
```


# 参考文档

- https://github.com/rancher/local-path-provisioner
- https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/
