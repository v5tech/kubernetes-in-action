# helm安装drone、drone-runner-kube

## 安装drone repo

```bash
helm repo add drone https://charts.drone.io
helm repo update
```

## 获取drone、drone-runner-kube

```bash
helm search repo drone -l
helm pull drone/drone
helm pull drone/drone-runner-kube
# helm pull drone/drone-runner-kube --version 0.1.10
tar zxvf drone-0.6.4.tgz
tar zxvf drone-runner-kube-0.1.10.tgz
```

## 安装drone

```bash
kubectl create ns drone-ci
kubectl create secret generic drone-secret \
  --from-literal=DRONE_RPC_SECRET=1c0601db6b8d55072c34df5ca2456a0a \
  --from-literal=DRONE_GITEA_CLIENT_ID=63dfbe1f-7fc8-4351-a6ca-12a7f5ca6439 \
  --from-literal=DRONE_GITEA_CLIENT_SECRET=J4a5L58ZovqpnVBc7v1EteUriLpBjpsiW8PdgfkSO71W \
  -n drone-ci
helm install drone-ci drone/drone --namespace drone-ci -f values.yaml
```

## 安装drone-runner-kube

.drone.yml

```bash
kubectl apply -f pv.yaml -f pvc.yaml -n drone-ci
helm install drone-runner-kube drone/drone-runner-kube --namespace drone-ci -f values.yaml
```

## 构建测试

```yaml
kind: pipeline
type: kubernetes
name: 构建部署

# 禁用默认克隆
clone:
  disable: true

steps:
  - name: 环境变量
    image: busybox
    pull: if-not-exists
    privileged: true
    commands:
      - env

  - name: 克隆源码
    image: drone/git
    pull: if-not-exists
    privileged: true
    commands:
      - pwd
      - rm -rf /drone/src/.gradle
      - echo "${DRONE_GIT_HTTP_URL} -> ${DRONE_BRANCH}"
      - git version
      - git clone ${DRONE_GIT_HTTP_URL} .
      - git checkout ${DRONE_BRANCH}
      - git status
      - pwd
      - ls -lah
      - git diff --name-only HEAD~ HEAD | grep '/' |grep -v '"' | awk -F "/" '{print $1}'| uniq >> /drone/src/modules
      - cat /drone/src/modules
    depends_on:
      - 环境变量

  - name: 编译源码
    image: maven:3-jdk-10
    pull: if-not-exists
    privileged: true
    volumes:
      - name: gradle-cache
        path: /drone/src/.gradle
    commands:
      # - export GRADLE_USER_HOME=./.gradle
      - pwd && mvn --version
    depends_on:
      - 克隆源码

  - name: 构建镜像
    image: plugins/docker
    pull: if-not-exists
    privileged: true
    volumes:
      - name: docker-sock
        path: /var/run/
    commands:
      - docker info
      - docker login reg.anchnet.com --username smartops --password Smartops@123 && echo "登录harbor成功!"
    depends_on:
      - 编译源码

volumes:
  - name: gradle-cache
    host:
      path: /tmp/.gradle
  - name: docker-sock
    host:
      path: /var/run/
```