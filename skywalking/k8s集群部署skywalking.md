# k8s集群部署skywalking

> svc_name.ns_name.svc.cluster.local

## 1. 添加helm repo

```bash
helm repo add skywalking https://apache.jfrog.io/artifactory/skywalking-helm
helm repo add elastic https://helm.elastic.co 
helm repo update
```
## 2. 部署skywalking

* 一键部署skywalking

```bash
kubectl create ns skywalking
helm install myskywalking skywalking/skywalking -n skywalking \
  --set oap.image.repository=apache/skywalking-oap-server \
  --set oap.image.tag=8.9.1 \
  --set oap.storageType=elasticsearch \
  --set oap.javaOpts='-Xms4g -Xmx4g' \
  --set ui.image.repository=apache/skywalking-ui \
  --set ui.image.tag=8.9.1 \
  --set elasticsearch.image=docker.elastic.co/elasticsearch/elasticsearch-oss \
  --set elasticsearch.imageTag=7.10.2 \
  --set elasticsearch.esJavaOpts='-Xmx4g -Xms4g'
```

* 使用已经存在的es集群

```bash
kubectl create ns skywalking
helm install myskywalking skywalking/skywalking -n skywalking \
  --set oap.image.repository=apache/skywalking-oap-server \
  --set oap.image.tag=8.9.1 \
  --set oap.storageType=elasticsearch \
  --set oap.javaOpts='-Xms2g -Xmx2g' \
  --set ui.image.repository=apache/skywalking-ui \
  --set ui.image.tag=8.9.1 \
  --set elasticsearch.enabled=false \
  --set elasticsearch.config.host=elasticsearch.graylog.svc.cluster.local \
  --set elasticsearch.config.port.http=9200 \
  --set elasticsearch.config.user= \
  --set elasticsearch.config.password=
```

* 修改svc为`NodePort`

```bash
kubectl --namespace skywalking patch svc myskywalking-ui -p '{"spec": {"type": "NodePort"}}'
```

## 3. 部署skywalking java agent

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-eureka-client
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-eureka-client
  template:
    metadata:
      labels:
        app: my-eureka-client
    spec:
      volumes:
        - name: skywalking-agent
          emptyDir: { }

      initContainers:
        - name: agent-container
          image: apache/skywalking-java-agent:8.8.0-alpine
          volumeMounts:
            - name: skywalking-agent
              mountPath: /agent
          command: [ "/bin/sh" ]
          args: [ "-c", "cp -R /skywalking/agent /agent/" ]

      containers:
        - name: my-eureka-client
          image: myeurekaclient:1.0
          volumeMounts:
            - name: skywalking-agent
              mountPath: /skywalking
          env:
            - name: JAVA_TOOL_OPTIONS
              value: "-javaagent:/skywalking/agent/skywalking-agent.jar"
            - name: SW_AGENT_NAME
              value: "my-eureka-client"
            - name: SW_AGENT_COLLECTOR_BACKEND_SERVICES
              value: "myskywalking-oap.skywalking.svc.cluster.local:11800"
```
