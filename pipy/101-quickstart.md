# 101 Quickstart

## Web 服务

返回静态内容

- [`listen`](https://flomesh.io/pipy/docs/en/reference/api/Configuration/listen)
- [`serveHTTP`](https://flomesh.io/pipy/docs/en/reference/api/Configuration/serveHTTP)
- [`Message`](https://flomesh.io/pipy/docs/en/reference/api/Message)

```js
pipy()  
  
  .listen(8080)  
  .serveHTTP(new Message('Hi, there!\n'))
```

返回动态内容

- `__inbound`

```js
pipy()  
  
  .listen(8080)  
  .serveHTTP(new Message('Hi, there!\n'))  
  
  .listen(8081)  
  .serveHTTP(msg => new Message(msg.body))  
  
  .listen(8082)  
  .serveHTTP(  
    msg => new Message(  
      `You are requesting ${msg.head.path} from ${__inbound.remoteAddress}\n`  
    )  
  )
```

## 反向代理

### 4 层代理

- [`connect`](https://flomesh.io/pipy/docs/en/reference/api/Configuration/connect)

```js
pipy()  
  
  .listen(8000)  
  .connect('localhost:8080')
```

### 7 层代理

- [`demuxHTTP`](https://flomesh.io/pipy/docs/en/reference/api/Configuration/demuxHTTP)
- [`muxHTTP`](https://flomesh.io/pipy/docs/en/reference/api/Configuration/muxHTTP)
- 子管道
- [`to`](https://flomesh.io/pipy/docs/en/reference/api/Configuration/to)

```js
pipy()  
  
  .listen(8000)  
  .demuxHTTP().to('forward')

  .pipeline('forward')
  .muxHTTP().to('connection')


  .pipeline('connection')
  .connect('localhost:8080')
```

- 匿名管道

```js
pipy()  
  
  .listen(8000)  
  .demuxHTTP().to(
    $=>$.muxHTTP().to(  
      $=>$.connect('localhost:8080')  
    )  
  )
```

## 路由

- [`URLRouter`](https://flomesh.io/pipy/docs/en/reference/api/algo/URLRouter)
- [`handleMessageStart`](https://flomesh.io/pipy/docs/en/reference/api/Configuration/handleMessageStart)
- [`branch`](https://flomesh.io/pipy/docs/en/reference/api/Configuration/branch)
- [`replaceMessage`](https://flomesh.io/pipy/docs/en/reference/api/Configuration/replaceMessage)
- 匿名函数
- 变量定义
  - 全局变量
  - 模块变量
  - 上下文变量

#### Javascript 匿名函数

```js
//匿名函数
(function() {})
//匿名函数调用
(function() {})()
```

PipyJS

```
pipy()

(() => pipy())()

//匿名函数参数默认值

((
  name = 'anonymous'
) => pipy())()
```

#### 实现

```js
((  
  router = new algo.URLRouter({  
    '/hi/*': 'localhost:8080',  
    '/echo': 'localhost:8081',  
    '/ip/*': 'localhost:8082',  
  }),  
  
) => pipy({  
  _target: undefined,  
})  
  
  .listen(8000)  
  .demuxHTTP().to(  
    $=>$  
    .handleMessageStart(  
      msg => (  
        _target = router.find(  
          msg.head.headers.host,  
          msg.head.path,  
        )  
      )  
    )  
    .branch(  
      () => Boolean(_target), (  
        $=>$.muxHTTP(() => _target).to(  
          $=>$.connect(() => _target)  
        )  
      ), (  
        $=>$.replaceMessage(  
          new Message({ status: 404 }, 'No route')  
        )  
      )  
    )  
  )  
  
)()
```

> 通过函数来获取变量的值 `() => _target

## 负载均衡

- [`RoundRobinLoadBalancer`](https://flomesh.io/pipy/docs/en/reference/api/algo/RoundRobinLoadBalancer)

```js
((  
  router = new algo.URLRouter({  
    '/hi/*': new algo.RoundRobinLoadBalancer(['localhost:8080', 'localhost:8082']),  
    '/echo': new algo.RoundRobinLoadBalancer(['localhost:8081']),  
    '/ip/*': new algo.RoundRobinLoadBalancer(['localhost:8082']),  
  }),  
  
) => pipy({  
  _target: undefined,  
})  
  
  .listen(8000)  
  .demuxHTTP().to(  
    $=>$  
    .handleMessageStart(  
      msg => (  
        _target = router.find(  
          msg.head.headers.host,  
          msg.head.path,  
        )?.next?.()  
      )  
    )  
    .branch(  
      () => Boolean(_target), (  
        $=>$.muxHTTP(() => _target).to(  
          $=>$.connect(() => _target.id)  
        )  
      ), (  
        $=>$.replaceMessage(  
          new Message({ status: 404 }, 'No route')  
        )  
      )  
    )  
  )  
  
)()
```

> 可选链操作符 `?.` 访问对象的属性或者函数，对象是 `undefine` 或者 `null` 时直接返回 `undefaine` ，而不是抛出异常。

## 配置

- [`JSON.decode()`](https://flomesh.io/pipy/docs/zh/reference/api/JSON/decode)
- [`pipy.load()`](https://flomesh.io/pipy/docs/zh/reference/api/pipy/load)
- [`Object.entries()`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/entries)

#### JSON 配置

映射关系：

- 路由：请求到服务
- 负载均衡：服务到目标地址

```json
{  
  "listen": 8000,  
  "routes": {  
    "/hi/*": "service-hi",  
    "/echo": "service-echo",  
    "/ip/*": "service-tell-ip"  
  },  
  "services": {  
    "service-hi"      : ["127.0.0.1:8080", "127.0.0.1:8082"],  
    "service-echo"    : ["127.0.0.1:8081"],  
    "service-tell-ip" : ["127.0.0.1:8082"]  
  }  
}
```

#### 加载配置

```js
((  
  config = JSON.decode(pipy.load('config.json')),  
  router = new algo.URLRouter(config.routes),  
  services = Object.fromEntries(  
    Object.entries(config.services).map(  
      ([k, v]) => [  
        k, new algo.RoundRobinLoadBalancer(v)  
      ]  
    )  
  ),  
  
) => pipy({  
  _target: undefined,  
})
//...
)()
```

#### 使用配置

```js
.listen(config.listen)
```

使用匿名函数，进行负载均衡：负载均衡的输入就是路由的结果

```js
.handleMessageStart(  
  msg => (  
    ((  
      s = router.find(  
        msg.head.headers.host,  
        msg.head.path,  
      )  
    ) => (  
      _target = services[s]?.next?.()  
    ))()  
  )  
)
```

## 插件

- [`solve`](https://flomesh.io/pipy/docs/zh/reference/api/pipy/solve)
- [`chain`](https://flomesh.io/pipy/docs/zh/reference/api/Configuration/chain)
- [`export`](https://flomesh.io/pipy/docs/zh/reference/api/Configuration/export)
- [`import`](https://flomesh.io/pipy/docs/zh/reference/api/Configuration/import)

目前已实现的功能：

- 路由
- 负载均衡

#### 插件配置

`config.json`

配置插件列表

```json
{  
  "listen": 8000,  
  "plugins": [  
    "plugins/router.js",  
    "plugins/balancer.js",  
    "plugins/default.js"  
  ],
  //
}
```

#### 使用插件 

- `chain` 指定多个插件
- `chain` 不指定插件

`proxy.js`

```js
((  
  config = pipy.solve('config.js'),  
  
) => pipy()  
  
  .export('main', {  
    __route: undefined,  
  })  
  
  .listen(config.listen)  
  .demuxHTTP().to(  
    $=>$.chain(config.plugins)  
  )  
  
)()
```

`router.js`

```js
((  
  config = pipy.solve('config.js'),  
  router = new algo.URLRouter(config.routes),  
  
) => pipy()  
  
  .import({  
    __route: 'main',  
  })  
  
  .pipeline()  
  .handleMessageStart(  
    msg => (  
      __route = router.find(  
        msg.head.headers.host,  
        msg.head.path,  
      )  
    )  
  )  
  .chain()  
  
)()
```

`balancer.js`

```js
((  
  config = pipy.solve('config.js'),  
  services = Object.fromEntries(  
    Object.entries(config.services).map(  
      ([k, v]) => [  
        k, new algo.RoundRobinLoadBalancer(v)  
      ]  
    )  
  ),  
  
) => pipy({  
  _target: undefined,  
})  
  
  .import({  
    __route: 'main',  
  })  
  
  .pipeline()  
  .branch(  
    () => Boolean(_target = services[__route]?.next?.()), (  
      $=>$  
      .muxHTTP(() => _target).to(  
        $=>$.connect(() => _target.id)  
      )  
    ),  
    $=>$.chain()  
  )  
  
)()
```
