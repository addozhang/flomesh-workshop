# 快速入门

课程目标：
* osm-edge 的安装
* 通过实例应用了解 osm-edge 的基本功能

> 环境准备，参考 [环境搭建](./001-setup-env.md)。

* [安装 osm-edge](#安装-osm-edge)
  * [下载 CLI](#下载-cli)
  * [安装](#安装)
* [部署示例应用](#部署示例应用)
  * [创建命名空间](#创建命名空间)
  * [将命名空间加入到网格中](#将命名空间加入到网格中)
  * [部署应用](#部署应用)
  * [访问应用](#访问应用)
* [访问控制](#访问控制)
  * [关闭宽松流量策略模式](#关闭宽松流量策略模式)
  * [部署流量访问策略](#部署流量访问策略)
* [灰度发布](#灰度发布)
  * [部署 bookstore-v2](#部署-bookstore-v2)
  * [初始化流量拆分策略](#初始化流量拆分策略)
  * [调整 bookstore-v2 的权重](#调整-bookstore-v2-的权重)
  * [将所有流量导入 bookstore-v2](#将所有流量导入-bookstore-v2)
* [创建 Ingress](#创建-ingress)
* [Grafana 监控](#grafana-监控)
  * [Dashboard](#dashboard)
* [Jaeger 链路跟踪](#jaeger-链路跟踪)
  * [功能](#功能)

## 安装 osm-edge

osm-edge 的安装可通过 Helm、osm CLI 安装，推荐使用 CLI 的方式安装。

### 下载 CLI

```shell
system=$(uname -s | tr [:upper:] [:lower:])
arch=$(dpkg --print-architecture)
release=v1.1.2
curl -L https://github.com/flomesh-io/osm-edge/releases/download/${release}/osm-edge-${release}-${system}-${arch}.tar.gz | tar -vxzf -
./${system}-${arch}/osm version
cp ./${system}-${arch}/osm /usr/local/bin/
```

### 安装

通过参数设置会安装 Prometheus、Grafana、Jaeger、fsm，并开启宽松流量模式和链路跟踪。

```shell
export osm_namespace=osm-system 
export osm_mesh_name=osm 

osm install \
    --mesh-name "$osm_mesh_name" \
    --osm-namespace "$osm_namespace" \
    --set=osm.enablePermissiveTrafficPolicy=true \
    --set=osm.deployPrometheus=true \
    --set=osm.deployGrafana=true \
    --set=osm.deployJaeger=true \
    --set=osm.tracing.enable=true \
    --set=fsm.enabled=true
```

检查组件是否启动并运行。

```shell
kubectl get po -n osm-system
```

## 部署示例应用

我们将部署 5 个不同的 Pod，并且我们将应用一些策略来控制它们之间的流量。

  * `bookbuyer` 是一个 HTTP 客户端，它发送请求给 `bookstore`。这个流量是**允许**的。
  * `bookthief` 是一个 HTTP 客户端，很像 `bookbuyer`，也会发送 HTTP 请求给 `bookstore`。这个流量应该被**阻止**。
  * `bookstore` 是一个服务器，负责对 HTTP 请求给与响应。同时，该服务器也是一个客户端，发送请求给 `bookwarehouse` 服务。这个流量是被**允许**的。
  * `bookwarehouse` 是一个服务器，应该只对 `bookstore` 做出响应。`bookbuyer` 和 `bookthief` 都应该被其阻止。
  * `mysql` 是一个 MySQL 数据库，只有 `bookwarehouse` 可以访问。

### 创建命名空间

```shell
kubectl create namespace bookstore
kubectl create namespace bookbuyer
kubectl create namespace bookthief
kubectl create namespace bookwarehouse
```

### 将命名空间加入到网格中

```shell
osm namespace add bookstore bookbuyer bookthief bookwarehouse
```

收集命名空间中应用的指标：

```
osm metrics enable --namespace "bookstore,bookbuyer,bookthief,bookwarehouse"
```

### 部署应用

```shell
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/main/manifests/apps/bookbuyer.yaml
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/main/manifests/apps/bookthief.yaml
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/main/manifests/apps/bookstore.yaml
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/main/manifests/apps/bookwarehouse.yaml
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/main/manifests/apps/mysql.yaml
```

如果遇到域名 `raw.githubusercontent.com` 解析问题，请使用下面的命令进行部署。

<details>
  <summary>展开</summary>

**bookstore**

```shell
kubectl apply -f - <<EOF
# Create bookbuyer Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookbuyer
  namespace: bookbuyer

---

# Create bookbuyer Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bookbuyer
  namespace: bookbuyer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bookbuyer
      version: v1
  template:
    metadata:
      labels:
        app: bookbuyer
        version: v1
    spec:
      serviceAccountName: bookbuyer
      nodeSelector:
        kubernetes.io/os: linux
      containers:
        - name: bookbuyer
          image: flomesh/bookbuyer:latest
          imagePullPolicy: Always
          command: ["/bookbuyer"]
          env:
            - name: "BOOKSTORE_NAMESPACE"
              value: bookstore
            - name: "BOOKSTORE_SVC"
              value: bookstore
EOF
```

**bookthief**

```shell
kubectl apply -f - <<EOF
# Create bookthief ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookthief
  namespace: bookthief

---

# Create bookthief Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bookthief
  namespace: bookthief
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bookthief
  template:
    metadata:
      labels:
        app: bookthief
        version: v1
    spec:
      serviceAccountName: bookthief
      nodeSelector:
        kubernetes.io/os: linux
      containers:
        - name: bookthief
          image: flomesh/bookthief:latest
          imagePullPolicy: Always
          command: ["/bookthief"]
          env:
            - name: "BOOKSTORE_NAMESPACE"
              value: bookstore
            - name: "BOOKSTORE_SVC"
              value: bookstore
            - name: "BOOKTHIEF_EXPECTED_RESPONSE_CODE"
              value: "503"
EOF
```

**bookstore**

```shell
kubectl apply -f - <<EOF
# Create bookstore Service
apiVersion: v1
kind: Service
metadata:
  name: bookstore
  namespace: bookstore
  labels:
    app: bookstore
spec:
  ports:
  - port: 14001
    name: bookstore-port
  selector:
    app: bookstore

---

# Create bookstore Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookstore
  namespace: bookstore

---

# Create bookstore Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bookstore
  namespace: bookstore
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bookstore
  template:
    metadata:
      labels:
        app: bookstore
    spec:
      serviceAccountName: bookstore
      nodeSelector:
        kubernetes.io/os: linux
      containers:
        - name: bookstore
          image: flomesh/bookstore:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 14001
              name: web
          command: ["/bookstore"]
          args: ["--port", "14001"]
          env:
            - name: BOOKWAREHOUSE_NAMESPACE
              value: bookwarehouse
            - name: IDENTITY
              value: bookstore-v1
EOF
```

**bookwarehouse**

```shell
kubectl apply -f - <<EOF
# Create bookwarehouse Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookwarehouse
  namespace: bookwarehouse

---

# Create bookwarehouse Service
apiVersion: v1
kind: Service
metadata:
  name: bookwarehouse
  namespace: bookwarehouse
  labels:
    app: bookwarehouse
spec:
  ports:
  - port: 14001
    name: bookwarehouse-port
  selector:
    app: bookwarehouse

---

# Create bookwarehouse Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bookwarehouse
  namespace: bookwarehouse
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bookwarehouse
  template:
    metadata:
      labels:
        app: bookwarehouse
        version: v1
    spec:
      serviceAccountName: bookwarehouse
      nodeSelector:
        kubernetes.io/os: linux
      containers:
        - name: bookwarehouse
          image: flomesh/bookwarehouse:latest
          imagePullPolicy: Always
          command: ["/bookwarehouse"]
EOF
```

**mysql**

```shell
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mysql
  namespace: bookwarehouse
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: bookwarehouse
spec:
  ports:
  - port: 3306
    targetPort: 3306
    name: client
    appProtocol: tcp
  selector:
    app: mysql
  clusterIP: None
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: bookwarehouse
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      serviceAccountName: mysql
      nodeSelector:
        kubernetes.io/os: linux
      containers:
      - image: mariadb:10.7.4
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: mypassword
        - name: MYSQL_DATABASE
          value: booksdemo
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - mountPath: /mysql-data
          name: data
        readinessProbe:
          tcpSocket:
            port: 3306
          initialDelaySeconds: 15
          periodSeconds: 10
      volumes:
        - name: data
          emptyDir: {}
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 250M
EOF
```
</details>

检查所有应用都正常启动并运行。

```shell
kubectl get pod,svc -n bookstore
kubectl get pod,svc -n bookbuyer
kubectl get pod,svc -n bookthief
kubectl get pod,svc -n bookwarehouse
```

### 访问应用

使用 port-forward 的方式来访问应用。

```shell
git clone https://github.com/flomesh-io/osm-edge.git -b release-v1.1
cd osm-edge
cp .env.example .env
./scripts/port-forward-all.sh #可以忽略错误信息
```

浏览器访问应用（将 `localhost` 替换成主机的 IP 地址，下同）。

* [http://localhost:8080](http://localhost:8080/) - **bookbuyer**
* [http://localhost:8083](http://localhost:8083/) - **bookthief**
* [http://localhost:8084](http://localhost:8084/) - **bookstore**


应用页面计数器清零：

```shell
export ingress_host=localhost
curl -sIXGET http://$ingress_host:8083/reset http://$ingress_host:8084/reset http://$ingress_host:8080/reset http://$ingress_host:8082/reset
```

## 访问控制

**目标**：禁止 bookthief 从 bookstore 偷书，不影响书籍的正常购买。

### 关闭宽松流量策略模式

```shell
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}'  --type=merge
```

检查 bookthief 和 bookbuyer 的计数器。

### 部署流量访问策略

部署访问策略，允许正常的访问，然后检查页面上计数器的变化。

```shell
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/main/manifests/access/traffic-access-v1.yaml
```

或者：

<details>
  <summary>展开</summary>

```shell
kubectl apply -f - <<EOF
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
  name: bookstore
  namespace: bookstore
spec:
  destination:
    kind: ServiceAccount
    name: bookstore
    namespace: bookstore
  rules:
  - kind: HTTPRouteGroup
    name: bookstore-service-routes
    matches:
    - buy-a-book
    - books-bought
  sources:
  - kind: ServiceAccount
    name: bookbuyer
    namespace: bookbuyer
---
apiVersion: specs.smi-spec.io/v1alpha4
kind: HTTPRouteGroup
metadata:
  name: bookstore-service-routes
  namespace: bookstore
spec:
  matches:
  - name: books-bought
    pathRegex: /books-bought
    methods:
    - GET
    headers:
    - "user-agent": ".*-http-client/*.*"
    - "client-app": "bookbuyer"
  - name: buy-a-book
    pathRegex: ".*a-book.*new"
    methods:
    - GET
---
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
  name: bookstore-access-bookwarehouse
  namespace: bookwarehouse
spec:
  destination:
    kind: ServiceAccount
    name: bookwarehouse
    namespace: bookwarehouse
  rules:
  - kind: HTTPRouteGroup
    name: bookwarehouse-service-routes
    matches:
    - restock-books
  sources:
  - kind: ServiceAccount
    name: bookstore
    namespace: bookstore
---
apiVersion: specs.smi-spec.io/v1alpha4
kind: HTTPRouteGroup
metadata:
  name: bookwarehouse-service-routes
  namespace: bookwarehouse
spec:
  matches:
    - name: restock-books
      methods:
      - POST
---
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
  name: mysql
  namespace: bookwarehouse
spec:
  destination:
    kind: ServiceAccount
    name: mysql
    namespace: bookwarehouse
  rules:
  - kind: TCPRoute
    name: mysql
  sources:
  - kind: ServiceAccount
    name: bookwarehouse
    namespace: bookwarehouse
---
apiVersion: specs.smi-spec.io/v1alpha4
kind: TCPRoute
metadata:
  name: mysql
  namespace: bookwarehouse
spec:
  matches:
    ports:
    - 3306
EOF
```

</details>

## 灰度发布

### 部署 bookstore-v2

```shell
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/main/manifests/apps/bookstore-v2.yaml
```

或者:

<details>
  <summary>展开</summary>

```shell
kubectl apply -f - <<EOF
# Create bookstore-v2 Service
apiVersion: v1
kind: Service
metadata:
  name: bookstore-v2
  namespace: bookstore
  labels:
    app: bookstore-v2
spec:
  ports:
  - port: 14001
    name: bookstore-port
  selector:
    app: bookstore-v2

---

# Create bookstore-v2 Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookstore-v2
  namespace: bookstore

---

# Create bookstore-v2 Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bookstore-v2
  namespace: bookstore
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bookstore-v2
  template:
    metadata:
      labels:
        app: bookstore-v2
    spec:
      serviceAccountName: bookstore-v2
      nodeSelector:
        kubernetes.io/os: linux
      containers:
        - name: bookstore
          image: flomesh/bookstore:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 14001
              name: web
          command: ["/bookstore"]
          args: ["--port", "14001"]
          env:
            - name: BOOKWAREHOUSE_NAMESPACE
              value: bookwarehouse
            - name: IDENTITY
              value: bookstore-v2

---

kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
  name: bookstore-v2
  namespace: bookstore
spec:
  destination:
    kind: ServiceAccount
    name: bookstore-v2
    namespace: bookstore
  rules:
  - kind: HTTPRouteGroup
    name: bookstore-service-routes
    matches:
    - buy-a-book
    - books-bought
  sources:
  - kind: ServiceAccount
    name: bookbuyer
    namespace: bookbuyer

---

kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
  name: bookstore-v2-access-bookwarehouse
  namespace: bookwarehouse
spec:
  destination:
    kind: ServiceAccount
    name: bookwarehouse
    namespace: bookwarehouse
  rules:
  - kind: HTTPRouteGroup
    name: bookwarehouse-service-routes
    matches:
    - restock-books
  sources:
  - kind: ServiceAccount
    name: bookstore-v2
    namespace: bookstore
EOF
```
</details>

停止并重启 port-forward脚本。

```
./scripts/port-forward-all.sh
```

在浏览器中访问 bookstore-v2。

* [http://localhost:8082](http://localhost:8082/) - **bookstore-v2**

### 初始化流量拆分策略

```shell
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/main/manifests/split/traffic-split-v1.yaml
```

或者

```shell
kubectl apply -f - <<EOF
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: bookstore-split
  namespace: bookstore
spec:
  service: bookstore.bookstore
  backends:
  - service: bookstore
    weight: 100
EOF
```

### 调整 bookstore-v2 的权重

将 50% 的流量放行到 bookstore-v2 中。

```shell
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/main/manifests/split/traffic-split-50-50.yaml
```

或者

```shell
kubectl apply -f - <<EOF
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: bookstore-split
  namespace: bookstore
spec:
  service: bookstore.bookstore
  backends:
  - service: bookstore
    weight: 50
  - service: bookstore-v2
    weight: 50
EOF
```

查看 bookstore-v1 和 bookstore-v2 页面的计数器。

### 将所有流量导入 bookstore-v2

```shell
kubectl apply -f https://raw.githubusercontent.com/flomesh-io/osm-edge-docs/main/manifests/split/traffic-split-v2.yaml
```

或者

```shell
kubectl apply -f - <<EOF
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: bookstore-split
  namespace: bookstore
spec:
  service: bookstore.bookstore
  backends:
  - service: bookstore
    weight: 0
  - service: bookstore-v2
    weight: 100
EOF
```

查看 bookstore-v1 和 bookstore-v2 页面的计数器。

## 创建 Ingress

```shell
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: bookstore
  namespace: bookstore
spec:
  ingressClassName: pipy
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: bookstore-v2
            port:
              number: 14001
---
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: bookstore
  namespace: bookstore
spec:
  backends:
  - name: bookstore-v2
    port:
      number: 14001
      protocol: http
  sources:
  - kind: Service
    namespace: osm-system
    name: ingress-pipy-controller
EOF
```

浏览器中使用 ingress 的 IP 地址和端口访问 bookstore。

## Grafana 监控

通过 osm CLI 安装的 Grafana，如果是本机访问，可以通过下面的命令打开 Grafana。

```shell
osm dashboard
```

如果是访问远程远端的 cluster，继续使用上面的 port-forward 脚本，并使用下面的地址访问。

* [http://localhost:3000](http://localhost:3000/) - **Grafana**

在浏览器中打开 Grafana 后，使用用户 `admin` 密码 `admin` 访问。

### Dashboard

* 控制平面与 sidecar
* 数据面资源占用
* Pod 到 Service 的指标
* Service 到 Service 的指标
* 工作负载到 Service 的指标
* 工作负载到工作负载的指标

## Jaeger 链路跟踪

在浏览器中打开下面的地址访问 Jaeger。

* [http://localhost:16686](http://localhost:16686/) - **Grafana**

### 功能

* 搜索服务链路
* 查看链路详情
* 链路跟踪比较
* 系统拓扑
