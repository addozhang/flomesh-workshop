# 103 新语法

## 1. 静态站

- [`listen`](https://flomesh.io/pipy/docs/en/reference/api/Configuration/listen)
- [`demuxHTTP`](https://flomesh.io/pipy/docs/en/reference/api/Configuration/demuxHTTP)
- [`replaceMessage`](https://flomesh.io/pipy/docs/en/reference/api/Configuration/replaceMessage)
- [`Message`](https://flomesh.io/pipy/docs/en/reference/api/Message)

`index.html`

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Hello, People!</title>
  </head>
  <body>
    <h1>The Quick Brown Fox Jumps Over the Lazy Dog</h1>
    <h2>The Quick Brown Fox Jumps Over the Lazy Dog</h2>
    <h3>The Quick Brown Fox Jumps Over the Lazy Dog</h3>
    <h4>The Quick Brown Fox Jumps Over the Lazy Dog</h4>
    <h5>The Quick Brown Fox Jumps Over the Lazy Dog</h5>
    <h6>The Quick Brown Fox Jumps Over the Lazy Dog</h6>
  </body>
</html>

```

```javascript
var files = Object.fromEntries(
  pipy.list('.').map(f => ([`/${f}`, http.File.from(`./${f}`)])).concat([['/', http.File.from('./index.html')]])
)

pipy.listen(8080, ($ => $
  .demuxHTTP().to($ => $.replaceMessage(
    msg => (
      files[msg.head.path]?.toMessage?.(msg.head.headers['accept-encoding']) || new Message({ status: 404 }, 'not found!\n')
    )
  ))
))
```

> 可选链操作符 `?.` 访问对象的属性或者函数，对象是 `undefine` 或者 `null` 时直接返回 `undefaine` ，而不是抛出异常。

## 2. Web 服务

返回静态内容

- [`listen`](https://flomesh.io/pipy/docs/en/reference/api/Configuration/listen)
- [`serveHTTP`](https://flomesh.io/pipy/docs/en/reference/api/Configuration/serveHTTP)
- [`Message`](https://flomesh.io/pipy/docs/en/reference/api/Message)

```js
pipy.listen(8080, ($ => $
  .serveHTTP(new Message('Hi, there!\n'))
))
```

返回动态内容

- `__inbound`

```js
var text = pipeline($ => $
  .serveHTTP(new Message('Hi, there!\n'))
)

var echo = pipeline($ => $
  .serveHTTP(msg => new Message(msg.body))
)

var $inbound
var ip = pipeline($ => $
  .serveHTTP(
    msg => new Message(
      `You are requesting ${msg.head.path} from ${$inbound.remoteAddress}\n`
    )
  )
)

pipy.listen(8080, $ => $.pipe(text))
pipy.listen(8081, $ => $.pipe(echo))
pipy.listen(8082, $ => $
  .onStart(ib => void ($inbound = ib))
  .pipe(ip)
)
```

## 3. 反向代理

### 4 层代理

- [`connect`](https://flomesh.io/pipy/docs/en/reference/api/Configuration/connect)

```js
pipy.listen(8000, $ => $
  .connect('localhost:8080')
)
```

### 7 层代理

- [`demuxHTTP`](https://flomesh.io/pipy/docs/en/reference/api/Configuration/demuxHTTP)
- [`muxHTTP`](https://flomesh.io/pipy/docs/en/reference/api/Configuration/muxHTTP)
- [子管道](https://flomesh.io/pipy/docs/en/intro/concepts#sub-pipeline)
- [`to`](https://flomesh.io/pipy/docs/en/reference/api/Configuration/to)

```js
var connection = pipeline($ => $
  .connect('localhost:8080')
)

var forward = pipeline($ => $
  .muxHTTP().to($ => $.pipe(connection))
)

pipy.listen(8000, $ => $
  .demuxHTTP().to($ => $
    .pipe(forward)
  )
)
```

- 匿名管道

```js
pipy.listen(8000, $ => $
  .demuxHTTP().to($ => $
    .muxHTTP().to($ => $
      .connect('localhost:8080'))
  )
)
```

## 4. 路由

- [`URLRouter`](https://flomesh.io/pipy/docs/en/reference/api/algo/URLRouter)
- [`handleMessageStart`](https://flomesh.io/pipy/docs/en/reference/api/Configuration/handleMessageStart)
- [`branch`](https://flomesh.io/pipy/docs/en/reference/api/Configuration/branch)
- [`replaceMessage`](https://flomesh.io/pipy/docs/en/reference/api/Configuration/replaceMessage)
- 匿名函数
- 变量定义
  - 全局变量
  - 模块变量
  - 上下文变量

```js
var routers = new algo.URLRouter({
  '/hi/*': 'localhost:8080',
  '/echo': 'localhost:8081',
  '/ip/*': 'localhost:8082',
})
var $ctx
var router = pipeline($ => $
  .handleMessageStart(
    msg => (
      $ctx.route = routers.find(
        msg.head.headers.host,
        msg.head.path,
      )
    )
  )
  .pipeNext()
)
var connection = pipeline($ => $
  .pipe(function () {
    if ($ctx?.route) {
      return 'proxy'
    }
    return 'pass'
  }, {
    'proxy': ($ => $
      .muxHTTP(() => $ctx.route).to($ => $
        .connect(() => $ctx.route)
      )),
    'pass': $ => $.pipeNext()
  })
)

var pass = pipeline($ => $
  .replaceMessage(new Message({ status: 404 }, 'No route'))
)

pipy.listen(8000, $ => $
  .demuxHTTP().to($ => $
    .onStart(function () {
      $ctx = { route: null }
    })
    .pipe([router, connection, pass])
  )
)
```

> 通过函数来获取变量的值 `() => _target

## 5. 负载均衡

- [`RoundRobinLoadBalancer`](https://flomesh.io/pipy/docs/en/reference/api/algo/RoundRobinLoadBalancer)

```js
var routers = new algo.URLRouter({
  '/hi/*': 'hi',
  '/echo': 'echo',
  '/ip/*': 'ip',
})
var balancers = {
  'hi': new algo.LoadBalancer(['localhost:8080', 'localhost:8082'], { algorithm: 'round-robin' }),
  'echo': new algo.LoadBalancer(['localhost:8081'], { algorithm: 'round-robin' }),
  'ip': new algo.LoadBalancer(['localhost:8082'], { algorithm: 'round-robin' }),
}
var $ctx
var router = pipeline($ => $
  .handleMessageStart(
    msg => (
      $ctx.route = routers.find(
        msg.head.headers.host,
        msg.head.path,
      )
    )
  )
  .pipeNext()
)
var $conn
var balancer = pipeline($ => $
  .pipe(function () {
    var lb = balancers[$ctx?.route]
    if (lb) {
      $conn = lb.allocate()
      console.log(`$conn: ${JSON.stringify($conn.target)}`)
      if ($conn) {
        return 'proxy'
      }
    }
    return 'pass'
  }, {
    'proxy': ($ => $
      .muxHTTP(() => $conn).to($ => $
        .connect(() => $conn.target)
      )),
    'pass': $ => $.pipeNext()
  })
)

var pass = pipeline($ => $
  .replaceMessage(new Message({ status: 404 }, 'No route'))
)

pipy.listen(8000, $ => $
  .demuxHTTP().to($ => $
    .onStart(function () {
      $ctx = { route: null }
    })
    .pipe([router, balancer, pass])
  )
)
```

## 6. 配置

- [`YAML.decode()`](https://flomesh.io/pipy/docs/zh/reference/api/JSON/decode)
- [`pipy.load()`](https://flomesh.io/pipy/docs/zh/reference/api/pipy/load)
- [`Object.entries()`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/entries)

### YAML 配置

映射关系：

- 路由：请求到服务
- 负载均衡：服务到目标地址

```yaml
listen: 8000
routes:
  /hi/*: hi
  /echo: echo
  /ip/*: ip
services:
  hi:
  - localhost:8080
  - localhost:8082
  echo:
  - localhost:8081
  ip:
  - localhost:8082
```

### 加载配置

```js
var config = YAML.decode(
  pipy.load('config.yaml')
)
```

### 使用配置

```js
.listen(config.listen)
```

使用配置初始化路由和负载均衡器。

```js
var routers = new algo.URLRouter(config.routes)
var balancers = Object.fromEntries(
  Object.entries(config.services).map(
    function([svc, endpoints]) {
      return [svc, new algo.LoadBalancer(endpoints, {algorithm: 'round-robin'})]
    }
  )
)
//...
```

## 7. 插件

- - [`import from`]()

目前已实现的功能：

- 路由
- 负载均衡

### 插件配置

`config.yaml`

增加插件列表配置

```yaml
plugins:
- plugins/router.js
- plugins/balancer.js
- plugins/default.js
```

### 使用插件

- `pipe` 指定多个插件

`config.js`

```javascript
export default (
  YAML.decode(
    pipy.load('config.yaml')
  )
)
```

`main.js`

```js
import config from './config.js'
var $ctx
var plugins = config.plugins.map(
  plugin => pipy.import(`./${plugin}.js`).default
)
pipy.listen(config.listen, $ => $
  .demuxHTTP().to($ => $
    .onStart(function () {
      $ctx = { route: null }
    })
    .pipe(plugins, () => $ctx)
  )
)
```

`router.js`

```js
import config from '../config.js'

var routers = new algo.URLRouter(config.routes)
var $ctx
export default pipeline($ => $
  .onStart(context => void ($ctx = context))
  .handleMessageStart(
    msg => (
      $ctx.route = routers.find(
        msg.head.headers.host,
        msg.head.path,
      )
    )
  )
  .pipeNext()
)
```

`balancer.js`

```js
import config from '../config.js'

var balancers = Object.fromEntries(
  Object.entries(config.services).map(
    function([svc, endpoints]) {
      return [svc, new algo.LoadBalancer(endpoints, {algorithm: 'round-robin'})]
    }
  )
)
var $ctx
var $conn
export default pipeline($ => $
  .onStart(context => void ($ctx = context))
  .pipe(function () {
    var lb = balancers[$ctx?.route]
    if (lb) {
      $conn = lb.allocate()
      console.log(`$conn: ${JSON.stringify($conn.target)}`)
      if ($conn) {
        return 'proxy'
      }
    }
    return 'pass'
  }, {
    'proxy': ($ => $
      .muxHTTP(() => $conn).to($ => $
        .connect(() => $conn.target)
      )),
    'pass': $ => $.pipeNext()
  })
)
```

`default.js`

```javascript
export default pipeline($ => $
  .replaceMessage(new Message({ status: 404 }, 'No route'))
)
```
