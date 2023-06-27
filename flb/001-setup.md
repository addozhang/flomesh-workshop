### 前提条件

- 4c16g VM * 1 (FLB 控制面)
- 2c8g VM * 1 (FLB 数据面)

### 准备

```shell
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### 集群

```shell
export INSTALL_K3S_VERSION=v1.27.1+k3s1
curl -sfL https://get.k3s.io | sh -s - --disable traefik --disable metrics-server --write-kubeconfig-mode 644 --write-kubeconfig ~/.kube/config
```

```shell
kubectl create ns flomesh
```

### Prometheus

```shell
# prometheus
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/prometheus --set alertmanager.enabled=false,prometheus-pushgateway.enabled=false,server.global.scrape_interval=10s --namespace flomesh --create-namespace

kubectl apply -n flomesh -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: prometheus
  name: prometheus
spec:
  ports:
  - name: 9090-9090
    nodePort: 30090
    port: 9090
    protocol: TCP
    targetPort: 9090
  selector:
    app.kubernetes.io/component: server
    app.kubernetes.io/instance: prometheus
  type: NodePort
EOF
```

### Mysql

```shell
kubectl apply -n flomesh -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mysql
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
    targetPort: 3306
    name: client
    appProtocol: tcp
  selector:
    app: mysql
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "local-path"
      resources:
        requests:
          storage: 5Gi
  template:
    metadata:
      labels:
        app: mysql
    spec:
      serviceAccountName: mysql
      containers:
      - image: mysql:5.7.41
        name: mysql
        args:
          - --character-set-server=utf8mb4
          - --collation-server=utf8mb4_unicode_ci
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: Flomesh@2022
        - name: MYSQL_DATABASE
          value: flomesh
        - name: MYSQL_USER
          value: flomesh
        - name: MYSQL_PASSWORD
          value: Flomesh@2022
        - name: MYSQL_CHARACTER_SET_SERVER
          value: utf8mb4
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - mountPath: /var/lib/mysql
          name: data
        readinessProbe:
          tcpSocket:
            port: 3306
          initialDelaySeconds: 15
          periodSeconds: 10
EOF
```

### ClickHouse

```shell
kubectl apply -n flomesh -f - <<EOF
apiVersion: apps/v1
kind: StatefulSet
metadata:
  creationTimestamp: null
  labels:
    app: clickhouse
  name: clickhouse
spec:
  serviceName: clickhouse
  selector:
    matchLabels:
      app: clickhouse
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: clickhouse
    spec:
      volumes:
      - name: clickhouse-pvc
        persistentVolumeClaim:
          claimName: clickhouse-pvc
      containers:
      - image: clickhouse/clickhouse-server:22.3
        name: clickhouse
        env: 
        - name: CLICKHOUSE_USER
          value: flomesh
        - name: CLICKHOUSE_DEFAULT_ACCESS_MANAGEMENT
          value: "1"
        - name: CLICKHOUSE_PASSWORD
          value: flomesh
        volumeMounts:
        - mountPath: /var/lib/clickhouse/
          name: clickhouse-pvc
        resources: {}
---
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: clickhouse
  name: clickhouse
spec:
  ports:
  - name: 8123-8123
    port: 8123
    protocol: TCP
    targetPort: 8123
    nodePort: 30123
  selector:
    app: clickhouse
  type: NodePort
status:
  loadBalancer: {}
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: clickhouse-pvc
spec:
  storageClassName: local-path
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1000Mi
EOF
```

### Pipy Repo

```shell
kubectl apply -n flomesh -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: pipy-repo
  name: pipy-repo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pipy-repo
  template:
    metadata:
      labels:
        app: pipy-repo
    spec:
      containers:
      - image: flomesh/pipy-repo:0.90.0-185
        name: pipy-repo
        command:
        - pipy
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: pipy-repo
  name: pipy-repo
spec:
  ports:
  - port: 6060
    protocol: TCP
    targetPort: 6060
    nodePort: 30060
  selector:
    app: pipy-repo
  type: NodePort
EOF
```

### Flomesh-GUI

```shell
kubectl apply -n flomesh -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: flomesh-gui
  name: flomesh-gui
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flomesh-gui
  template:
    metadata:
      labels:
        app: flomesh-gui
    spec:
      containers:
      - image: flomesh/traffic-guru-pro:latest
        name: flomesh-gui
        imagePullPolicy: Always
        ports:
        - name: flomesh-gui-web
          containerPort: 8080
        env:
        - name: HOST
          value: '0.0.0.0'
        - name: PORT
          value: "8080"
        - name: PIPY_REPO_HOST
          value: 'pipy-repo.flomesh.svc'
        - name: PIPY_REPO_PORT
          value: '6060'
        - name: DATABASE_HOST
          value: 'mysql.flomesh.svc'
        - name: DATABASE_PORT
          value: '3306'
        - name: DATABASE_NAME
          value: 'flomesh'
        - name: DATABASE_USERNAME
          value: 'flomesh'
        - name: DATABASE_PASSWORD
          value: 'Flomesh@2022'
        volumeMounts:
      serviceAccount: default
      serviceAccountName: default
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: flomesh-gui
  name: flomesh-gui
spec:
  ports:
  - name: web
    port: 8080
    protocol: TCP
    targetPort: 8080
    nodePort: 30000
  selector:
    app: flomesh-gui
  type: NodePort
EOF
```

打开 http://HOST_IP:30000/flomesh/