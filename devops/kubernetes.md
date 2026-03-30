# Kubernetes

`Kubernetes`（K8s）是容器编排系统，提供容器化应用的自动化部署、扩缩容与生命周期管理。

## 资源（Resources）

```bash
# 查看资源名称与对应的 apiVersion
kubectl api-resources
```

## 工作负载（Workloads）

### Pod

```bash
# 运行一个 busybox 的 Pod（交互式）
kubectl run -i -t busybox --image=busybox
```

#### Probe 健康检查

容器可配置三类探针：

- **存活（Liveness）**：决定何时重启容器，例如应用死锁后通过重启恢复可用性。
- **就绪（Readiness）**：决定容器何时可以接收流量并对外提供服务。
- **启动（Startup）**：保护慢启动容器，在启动完成前避免被存活探针误杀。

> 存活探针是从应用故障中恢复的强力手段，但必须谨慎配置，确保它只在不可恢复的故障（例如死锁）时才判定失败。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-test
  labels:
    name: probe-test
spec:
  containers:
    - name: nginx
      image: nginx
      livenessProbe: # 存活探针：按业务语义定义“不可恢复故障”
        httpGet:
          path: /
          port: 80
        failureThreshold: 30 # 连续失败 n 次判定失败
        initialDelaySeconds: 15 # 容器启动后延迟多少秒再开始探测
        periodSeconds: 3 # 每隔多少秒探测一次
      readinessProbe: # 就绪探针：决定是否接收流量
        tcpSocket:
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 10
      startupProbe: # 启动探针：保护慢启动
        httpGet:
          path: /
          port: 80
        failureThreshold: 30
        periodSeconds: 10
# 有启动探测时：最多 5 分钟（30 * 10 = 300s）完成启动；成功一次后由存活探针接管；
# 若启动探测一直失败，容器将在 300 秒后被杀死，并按 restartPolicy 进一步处置。
```

### Deployment

```bash
# 根据现有 YAML 创建/更新 Deployment
kubectl apply -f user-service-deployment.yaml

# 生成 Deployment YAML（不创建，仅输出；用于模板生成）
kubectl create deployment user-uservice-deployment --image=nginx --dry-run=client -o yaml > user-uservice-deployment.yaml

# Deployment 更新镜像（常与 CI/CD 联动）
kubectl set image deployment/deployment-name nginx=nginx:1.24

# 查看 rollout 历史
kubectl rollout history deployment user-service-deployment
kubectl rollout history deployment user-service-deployment --revision=4

# 回滚：多次回滚通常是在最近两个版本间切换
kubectl rollout undo deployment user-service-deployment
kubectl rollout undo deployment user-service-deployment --to-revision=4
```

### StatefulSet

有状态应用管理，适用于有状态场景，例如：redis-cluster、MongoDB 集群、Kafka 集群、Eureka 等。

创建 `StatefulSet` 通常需要配套的无头 `Service`（Headless Service）。

```yaml
# redis headless service
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: default
  labels:
    name: redis-service
spec:
  type: ClusterIP
  clusterIP: None # 无头服务：不分配 ClusterIP
  selector:
    name: redis-pod
  ports:
    - name: redis-port
      port: 6379
      protocol: TCP
      targetPort: 6379
```

```yaml
# redis statefulset
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-sts
  namespace: default
  labels:
    name: redis-sts
spec:
  replicas: 2
  serviceName: redis-service # 上一步创建的 Service 名称
  selector:
    matchLabels:
      name: redis-pod # 必须与 template.labels 匹配
  template:
    metadata:
      labels:
        app: redis-app
        name: redis-pod
    spec:
      containers:
        - image: redis:latest
          imagePullPolicy: IfNotPresent
          name: redis-container
          ports:
            - containerPort: 6379
              name: redis-port
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0 # 指定灰度发布分区
```

### DaemonSet

在匹配的节点（或所有节点）上创建一个副本，常用于：节点日志采集、CNI（如 Calico/Flannel）、Ingress Controller、Node Exporter 等。

```bash
# 给节点添加 label：ingress-nginx=true
kubectl label node k8s-node000012 ingress-nginx=true
```

```yaml
# DaemonSet nginx
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ingress-nginx
  labels:
    name: ingress-nginx-ds
spec:
  selector:
    matchLabels:
      name: ingress-node
  template:
    metadata:
      labels:
        name: ingress-node
    spec:
      nodeSelector:
        ingress-nginx: "true" # 不设置则表示选择所有节点
      containers:
        - image: nginx:latest
          imagePullPolicy: IfNotPresent
          name: ingress-nginx-container
```

## Service

Service 类型：`ExternalName`、`LoadBalancer`、`NodePort`、`ClusterIP`、`ClusterIP(None)`。

## Label & Selector

### Label

`label` 可以给任意资源（Service/Pod/Node 等）添加标签分组。

```bash
kubectl label pod busybox version=v1 # 添加 label
kubectl label pod busybox version=v2 --overwrite # 修改 label
kubectl get pod -l version=v2 # 筛选 version=v2 的 Pod
kubectl get pod -l 'version in (v2, v1)' # 通过 in 筛选
kubectl label pod busybox version- # 删除 label
```

### Selector

`selector` 用于筛选符合标签的资源，例如：

- 通过 `service.spec.selector` 选择对应的 `Pod`
- 通过 `Deployment.spec.template.spec.nodeSelector` 选择在合适的 `Node` 创建 Pod

```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
  labels:
    name: user-service
spec:
  selector:
    name: user-service # 选择 name=user-service 的 Pod
```

## ConfigMap & Secret

### ConfigMap

```bash
kubectl create configmap myconf --from-file conf.yml # 通过配置文件创建 ConfigMap
```

在 `Deployment` 中使用 ConfigMap（以 volume 方式挂载）：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: default
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - image: nginx
          imagePullPolicy: Always
          name: nginx
          volumeMounts:
            - mountPath: /var/conf # 容器内路径
              name: config # 与 volumes.name 一致
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: myconf # ConfigMap 名称
            defaultMode: 420
            items:
              - key: conf.yml
                path: conf.yml
```

### Secret

`Secret` 用于保存少量敏感信息（例如密码、令牌、密钥）。它与 ConfigMap 类似，但专门用于机密数据。

```bash
# 创建 docker-registry 认证 Secret
kubectl create secret docker-registry myregister \
  --docker-server=DOCKER_REGISTRY_SERVER \
  --docker-username=xxxx \
  --docker-password=xxxx \
  --docker-email=xxxx

# 创建 TLS 证书 Secret
kubectl create secret tls NAME --cert=path/to/cert/file --key=path/to/key/file
```

## Scale & Autoscale（HPA）

```bash
# 为 Deployment 创建 HPA
kubectl autoscale deployment user-service-deployment --min=3 --max=6

# 查看 HPA
kubectl get hpa -o wide
```

## Ingress

[ingress-nginx](https://kubernetes.github.io/ingress-nginx) 与 [nginx-ingress](https://github.com/nginxinc/kubernetes-ingress) 是两个不同项目：

- `ingress-nginx`：K8s 社区/官方维护的 Ingress Controller
- `nginx-ingress`：Nginx 官方维护的 Ingress Controller

两者配置风格相近，常见选择是 `ingress-nginx`。

```bash
helm pull ingress-nginx/ingress-nginx # 下载 ingress-nginx chart
tar xf ingress-nginx-4.7.1.tgz # 解压
vim ingress-nginx/values.yaml # 修改 values.yaml
```

下载 `ingress-nginx` 后通常需要按业务调整 `values.yaml` 的关键项：

1. controller、admissionWebhooks 的 image 地址（可切换到阿里云等镜像源）
2. `hostNetwork: true`（常见配置）
3. `dnsPolicy: ClusterFirstWithHostNet`
4. `nodeSelector` 添加标识（例如 `ingress-nginx: "true"`）以固定部署节点
5. `kind: DaemonSet`

```bash
# 安装 ingress-nginx（在 chart 目录下执行）
helm install ingress-nginx -n ingress-nginx .

# 创建一个简单 Ingress
kubectl create ingress ingress-nginx-svc --class=nginx --rule="ingress.test.com/*=nginx-svc:80"
```

```yaml
# ingress 配置示例
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-nginx-svc
  namespace: default # 需与被代理的 Service 处于同一 namespace
spec:
  ingressClassName: nginx # kubectl get ingressclasses.networking.k8s.io
  rules:
    - host: ingress.test.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-svc
                port:
                  number: 80
```

```bash
curl ingress.test.com # 域名解析后或添加 hosts 后测试
```

> 访问 Ingress 代理域名时，域名需解析到运行 `ingress-nginx-controller` Pod 的 Node 节点才能访问。

### Annotations

`ingress-nginx` 通过 `annotations` 可以实现常用能力：

- [速率限制](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#rate-limiting)
- [重定向](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#permanent-redirect)
- [白名单](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#whitelist-source-range)
- [代理重定向](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#proxy-redirect)

```yaml
# 常用 metadata.annotations 配置示例
nginx.ingress.kubernetes.io/rewrite-target: /$2
nginx.ingress.kubernetes.io/use-regex: "true"
nginx.ingress.kubernetes.io/limit-rps: "1"
nginx.ingress.kubernetes.io/permanent-redirect: https://www.google.com
nginx.ingress.kubernetes.io/temporal-redirect: https://www.google.com
nginx.ingress.kubernetes.io/whitelist-source-range: 10.0.0.0/24,172.10.0.1
```

```yaml
# rewrite 示例
# foo.com/api -> foo.com
# foo.com/api/new -> foo.com/new
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: foo.com
      http:
        paths:
          - path: /api(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: nginx-svc
                port:
                  number: 80
```

### ingress-nginx 实现原理

`ingress-nginx` 会把用户的 Ingress 配置渲染/注入到 Controller Pod 内的 Nginx 配置文件中，从而实现反向代理。

```bash
# 查看 ingress-nginx Pod 中的 nginx.conf（注意替换 Pod 名称）
kubectl exec -it ingress-nginx-controller-9qwnw -n ingress-nginx -- \
  sh -c "cat /etc/nginx/nginx.conf | grep foo.com -A 10"
```

## Metrics Server

## Prometheus

[prometheus](https://github.com/prometheus/prometheus)

### Install（kube-prometheus）

```bash
git clone https://github.com/prometheus-operator/kube-prometheus.git
cd kube-prometheus/manifests
kubectl apply --server-side -f setup
kubectl wait --for condition=Established --all CustomResourceDefinition -n monitoring
kubectl apply -f .
```

[kube-prometheus](https://github.com/prometheus-operator/kube-prometheus)

### 数据可视化

安装完成后通常会创建一系列服务，例如：

- `alertmanager-main`：告警配置入口（Alertmanager）
- `grafana`：指标可视化 Web UI
- `prometheus-k8s`：Prometheus 实例服务

通过 `kubectl get svc -n monitoring` 查看服务列表，例如：

```text
alertmanager-main  ClusterIP  10.108.145.122  <none>  9093/TCP,8080/TCP  19h
grafana            ClusterIP  10.98.163.188   <none>  3000/TCP           19h
prometheus-k8s     ClusterIP  10.105.23.112   <none>  9090/TCP,8080/TCP  19h
```

若希望通过域名访问，可通过 [Ingress](https://iscod.github.io/#/devops/kubernetes?id=ingress) 进行反向代理：

```bash
# namespace 必须与服务在同一命名空间；留意端口号
kubectl create ingress prometheus --class=nginx \
  --rule="grafana.test.com/*=grafana:3000" \
  --rule="alert.test.com/*=alertmanager-main:9093" \
  --rule="prometheus.test.com/*=prometheus-k8s:9090" \
  -n monitoring
```

### 告警配置

Prometheus 常用通知渠道配置：

- [email_config](https://prometheus.io/docs/alerting/latest/configuration/#email_config)
- [wechat_config](https://prometheus.io/docs/alerting/latest/configuration/#wechat_config)

#### 邮件配置

在 `alertmanager-secret.yaml` 中添加邮件配置示例：

```yaml
global:
  resolve_timeout: 6m
  smtp_from: xxxx@163.com
  smtp_smarthost: smtp.163.com:465
  smtp_hello: prometheus
  smtp_auth_username: xxxx@163.com
  smtp_auth_password: xxxx
  smtp_require_tls: false # 注意：部分邮箱服务商可能不支持 TLS
receivers:
  - name: Default
    email_configs:
      - to: xxxx@163.com
        send_resolved: true
```

#### 微信配置

### 架构图

![prometheus](https://iscod.github.io/images/prometheus.png)

## Istio

### 服务网格