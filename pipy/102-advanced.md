# 102 Advanced

## TLS/SSL

- [`acceptTLS`](https://flomesh.io/pipy/docs/zh/reference/api/Configuration/acceptTLS)

```shell
mkdir secret
pushd secret

openssl genrsa 2048 > ca-key.pem

openssl req -new -x509 -nodes -days 365000 \
   -key ca-key.pem \
   -out ca-cert.pem \
   -subj '/CN=flomesh.io'

openssl genrsa -out server-key.pem 2048
openssl req -new -key server-key.pem -out server.csr -subj '/CN=example.com'
openssl x509 -req -in server.csr -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial -out server-cert.pem -days 365
popd
```

`config.json`

```json
{
//
  "listenTLS": 8443,
  "certificates": {
    "cert": "secret/server-cert.pem",
    "key": "secret/server-key.pem"
  },
//
}
```

`proxy.js`

```javascript
((
  config = pipy.solve('config.js'),
) => pipy()

  .export('main', {
    __route: undefined,
  })

  .listen(config.listen)
  .link('http-tcp')

  .listen(config.listenTLS)
  .acceptTLS({
    certificate: {
      cert: new crypto.Certificate(pipy.load(config.certificates.cert)),
      key: new crypto.PrivateKey(pipy.load(config.certificates.key)),
    }
  }).to('http-tcp')

  .pipeline('http-tcp')
  .demuxHTTP().to(
    $ => $.chain(config.plugins)
  )

)()
```

`curl --cacert secret/ca-cert.pem https://example.com/ip --connect-to example.com:443:127.0.0.1:8443`

## mTLS

TODO

## 隧道

`proxy.js`

```javascript
pipy({
  _isTunnel: false,
  _target: false,
})

  .listen(8888)
  .demuxHTTP().to($ => $
    .handleMessageStart(
      msg => msg.head.method === 'CONNECT' && (_isTunnel = true)
    )
    .branch(
      () => _isTunnel, (
      $ => $.acceptHTTPTunnel(
        msg => (
          _target = msg.head.path,
          new Message({ status: 200 })
        )
      ).to($ => $
        .connect(() => _target)
      )
    ),
      $ => $.replaceMessage(new Message({ status: 404 }, 'Not Found!\n'))
    )
  )
```

`peer.js`

```javascript
pipy({
  _tunnel: '127.0.0.1:8888'
})

  .listen(8000)
  .connectHTTPTunnel(() => new Message({ method: 'CONNECT', path: '127.0.0.1:8081' }))
  .to($=>$
    .muxHTTP(() => _tunnel, { version: 2 }).to($=>$
      .connect(() => _tunnel)
    )
  )
```

`curl localhost:8000`

## 加密隧道

TODO