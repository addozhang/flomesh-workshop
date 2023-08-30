### 前提条件

- 4c16g VM * 1 (FLB 控制面)
- 2c8g VM * 1 (FLB 数据面)

环境说明：

在下面的操作中用到了 4 台虚拟机，在演示环境中，数据面和控制面可以运行与同一 VM 上：

- `192.168.1.11` BGP 路由器的 IP 地址，在 4LB 的 BGP 场景下会用到。
- `192.168.1.12` FLB 的控制面，运行 k3s 集群，部署了所有 FLB 控制面所需要的组件
- `192.168.1.13` FLB 的两个数据面之一，用于运行模拟的后端服务。
- `192.168.1.14` FLB 的两个数据面之一，用于运行模拟的后端服务。

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

### FRR BGP Router

我们使用 FRR 来运行一个 BGP 路由器，这里将需要 Docker 来运行 FRR 程序，因此请确保已经安装了 Docker。

执行下面的命令创建 FRR daemon 配置。

```shell
mkdir frr

tee > ./frr/daemons <<EOF
# This file tells the frr package which daemons to start.
#
# Sample configurations for these daemons can be found in
# /usr/share/doc/frr/examples/.
#
# ATTENTION:
#
# When activating a daemon for the first time, a config file, even if it is
# empty, has to be present *and* be owned by the user and group "frr", else
# the daemon will not be started by /etc/init.d/frr. The permissions should
# be u=rw,g=r,o=.
# When using "vtysh" such a config file is also needed. It should be owned by
# group "frrvty" and set to ug=rw,o= though. Check /etc/pam.d/frr, too.
#
# The watchfrr, zebra and staticd daemons are always started.
#
bgpd=yes
ospfd=no
ospf6d=no
ripd=no
ripngd=no
isisd=no
pimd=no
pim6d=no
ldpd=no
nhrpd=no
eigrpd=no
babeld=no
sharpd=no
pbrd=no
bfdd=no
fabricd=no
vrrpd=no
pathd=no

#
# If this option is set the /etc/init.d/frr script automatically loads
# the config via "vtysh -b" when the servers are started.
# Check /etc/pam.d/frr if you intend to use "vtysh"!
#
vtysh_enable=yes
zebra_options="  -A 127.0.0.1 -s 90000000"
bgpd_options="   -A 127.0.0.1"
ospfd_options="  -A 127.0.0.1"
ospf6d_options=" -A ::1"
ripd_options="   -A 127.0.0.1"
ripngd_options=" -A ::1"
isisd_options="  -A 127.0.0.1"
pimd_options="   -A 127.0.0.1"
pim6d_options="  -A ::1"
ldpd_options="   -A 127.0.0.1"
nhrpd_options="  -A 127.0.0.1"
eigrpd_options=" -A 127.0.0.1"
babeld_options=" -A 127.0.0.1"
sharpd_options=" -A 127.0.0.1"
pbrd_options="   -A 127.0.0.1"
staticd_options="-A 127.0.0.1"
bfdd_options="   -A 127.0.0.1"
fabricd_options="-A 127.0.0.1"
vrrpd_options="  -A 127.0.0.1"
pathd_options="  -A 127.0.0.1"


# If you want to pass a common option to all daemons, you can use the
# "frr_global_options" variable.
#
#frr_global_options=""


# The list of daemons to watch is automatically generated by the init script.
# This variable can be used to pass options to watchfrr that will be passed
# prior to the daemon list.
#
# To make watchfrr create/join the specified netns, add the the "--netns"
# option here. It will only have an effect in /etc/frr/<somename>/daemons, and
# you need to start FRR with "/usr/lib/frr/frrinit.sh start <somename>".
#
#watchfrr_options=""


# configuration profile
#
#frr_profile="traditional"
#frr_profile="datacenter"


# This is the maximum number of FD's that will be available.  Upon startup this
# is read by the control files and ulimit is called.  Uncomment and use a
# reasonable value for your setup if you are expecting a large number of peers
# in say BGP.
#
#MAX_FDS=1024


# For any daemon, you can specify a "wrap" command to start instead of starting
# the daemon directly. This will simply be prepended to the daemon invocation.
# These variables have the form daemon_wrap, where 'daemon' is the name of the
# daemon (the same pattern as the daemon_options variables).
#
# Note that when daemons are started, they are told to daemonize with the `-d`
# option. This has several implications. For one, the init script expects that
# when it invokes a daemon, the invocation returns immediately. If you add a
# wrap command here, it must comply with this expectation and daemonize as
# well, or the init script will never return. Furthermore, because daemons are
# themselves daemonized with -d, you must ensure that your wrapper command is
# capable of following child processes after a fork() if you need it to do so.
#
# If your desired wrapper does not support daemonization, you can wrap it with
# a utility program that daemonizes programs, such as 'daemonize'. An example
# of this might look like:
#
# bgpd_wrap="/usr/bin/daemonize /usr/bin/mywrapper"
#
# This is particularly useful for programs which record processes but lack
# daemonization options, such as perf and rr.
#
# If you wish to wrap all daemons in the same way, you may set the "all_wrap"
# variable.
#
#all_wrap=""
EOF
```

运行 FRR 程序。

```shell
docker run -d --rm --privileged --name frr --net=host -e watchfrr_debug=true -v ./frr:/etc/frr frrouting/frr:latest
```

FRR 成功启动后会在前面创建的 `frr` 目录中初始化 `bgpd.conf`、`staticd.conf` 和 `zebra.conf` 三个配置文件。

执行下面的命令，将 BGP 邻居的信息写入到 `frr.conf` 配置文件中。比如我们两个邻居 `192.168.1.13` 和 `192.168.1.14` 属于 AS `65001`；而当前 BGP 路由器属于 AS `65000`，id 为 `192.168.1.11`，也是运行 BGP 路由器这台虚拟机的 IP 地址。

```shell
tee >> ./frr/frr.conf <<EOF
ip forwarding
!
router bgp 65000
 bgp router-id 192.168.1.11
 no bgp ebgp-requires-policy
 bgp bestpath peer-type multipath-relax
 neighbor 192.168.1.13 remote-as 65001
 neighbor 192.168.1.14 remote-as 65001
exit
!
EOF
```