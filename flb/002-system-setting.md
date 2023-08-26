## 管理员配置

### 创建组织

创建一个名为 `org1` 的组织。

### 创建工程

创建名为 `project1` 和 `project2` 的工程，隶属于组织 `org1`。

## 组件管理

### 日志组件

创建 `Clickhouse` 类型的组件 `ch`（注 `192.168.1.12` 为集群 node IP 地址）。

```yaml
---
  host: "192.168.1.12"
  port: 30123
  user: "flomesh"
  database: "default"
  password: "flomesh"   
```

创建日志组件 `log`：

- Log Type：`Clickhouse`
- Log Table：`log`
- Log Address：选择上面创建的 `ch`

### Prometheus 组件

创建名为 `prometheus` 的组件，使用下面的配置：

```yaml
---
  host: "prometheus-server.flomesh.svc:80"
```

## 证书管理

```shell
openssl genrsa 2048 > ca-key.pem
# CA cert
openssl req -new -x509 -nodes -days 365000 \
   -key ca-key.pem \
   -out ca-cert.pem \
   -subj '/CN=flomesh.io'
# Server cert
openssl genrsa -out server-key.pem 2048
openssl req -new -key server-key.pem -out server.csr -subj '/CN=example.com'
openssl x509 -req -in server.csr -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial -out server-cert.pem -days 365
```

### TLS 证书

创建名为 `4lb-tls` 类型为 `API/LB/Website TLS Secret` 的证书。

- 在 `key` 中填 `server-key.pem` 的内容
- 在 `cert` 中填 `server-cert.pem` 的内容

## 模拟后端服务

```shell
pipy -e "pipy().listen(8081).serveHTTP(()=>new Message('Hi, Pipy\n'))" &
pipy -e "pipy().listen(8082).serveHTTP(()=>new Message('Hi, World\n'))" &
```