# 由于pm2设置代理导致的服务超时问题

## 问题描述

新服务A，在测试环境和预发环境验证ok后，准备上线，线上环境发布后进行验证，发现其中一个功能异常，页面显示接口超时。于是开始排查问题。

## 排查过程

异常的功能是请求一个服务B，然后返回数据。所以首先确定服务B有无异常，检查B的日志，发现没有A的访问记录，所以B没有收到来自A的请求，所以问题的范围缩小为：服务A内部问题。

增加A的错误日志，发现如下错误：![](https://raw.githubusercontent.com/love477/picture/main/images/20210223174656.png?token=AGTQAHCTZGSGC6KVF5L7XQDAGTHU4)

查阅资料后发现这是一个squid代理返回的错误，表明A的请求被代理了，而代理服务器无法访问B的服务。

然后就排查哪里设置了代理？检查完代码后，没有发现在代码、配置中设置代理，所以怀疑是机器的问题。于是重新申请了一台相同环境的机器，部署服务，发现服务是正常的可以访问。所以基本确定是机器配置中设置了代理，于是开始测试基本服务，发现机器能正常的访问这个B的服务。我们的服务是使用pm2进行管理的，于是手动启动服务，发现手动启动的服务是正常的，不合理，手动启动的服务正常，那就表明机器的配置是没有问题，只能是启动服务的环境配置有问题。

检查过服务的启动配置，没有发现问题，而且相同的代码，换一台机器是可以运行的，所以应该是隐性的设置什么，但是对我们是不可见的，所以只能排查底层的源码。由于我们已经确认是由于代理导致的服务超时，于是开始排查我们使用的http相关的库，目前的项目中使用的还是request（已经废弃的库，后面我们已经使用axios代替），由于只在特定的机器有问题，无法debug（也无法远程debug，生生产环境机器和内网不通），只能使用加log。由于确定是代理导致的问题，开始排查request库中请求的执行顺序，增加了request错误返回的完整打印，截取片段如下(xx表示加密)：

```javascript
proxy: 
    Url: {
        protocol: 'http:',
        slashes: true,
        auth: null,
        host: '172.xx.xx.xx:xxxx',
        port: 'xxxx',
        hostname: '172.xx.xx.xx',
        hash: null,
        search: null,
        query: null,
        pathname: '/',
        path: '/',
        href: 'http://172.xx.xx.xx:xxx/'
    },
```

发现request返回的错误体中会包含request的配置，其中proxy部分的配置如上，验证了我们之前的问题：是代理导致的服务超时。

于是开始排查request设置代理的逻辑：

```javascript
// 文件地址：request lib/getProxyFromURI.js
'use strict'

function formatHostname (hostname) {
  // canonicalize the hostname, so that 'oogle.com' won't match 'google.com'
  return hostname.replace(/^\.*/, '.').toLowerCase()
}

function parseNoProxyZone (zone) {
  zone = zone.trim().toLowerCase()

  var zoneParts = zone.split(':', 2)
  var zoneHost = formatHostname(zoneParts[0])
  var zonePort = zoneParts[1]
  var hasPort = zone.indexOf(':') > -1

  return {hostname: zoneHost, port: zonePort, hasPort: hasPort}
}

function uriInNoProxy (uri, noProxy) {
  var port = uri.port || (uri.protocol === 'https:' ? '443' : '80')
  var hostname = formatHostname(uri.hostname)
  var noProxyList = noProxy.split(',')

  // iterate through the noProxyList until it finds a match.
  return noProxyList.map(parseNoProxyZone).some(function (noProxyZone) {
    var isMatchedAt = hostname.indexOf(noProxyZone.hostname)
    var hostnameMatched = (
      isMatchedAt > -1 &&
        (isMatchedAt === hostname.length - noProxyZone.hostname.length)
    )
    // noProxyZone.hasPort, port, noProxyZone.port, hostnameMatched:  false 8088 undefined false
    if (noProxyZone.hasPort) {
      return (port === noProxyZone.port) && hostnameMatched
    }

    return hostnameMatched
  })
}

function getProxyFromURI (uri) {
  // Decide the proper request proxy to use based on the request URI object and the
  // environmental variables (NO_PROXY, HTTP_PROXY, etc.)
  // respect NO_PROXY environment variables (see: https://lynx.invisible-island.net/lynx2.8.7/breakout/lynx_help/keystrokes/environments.html)

  var noProxy = process.env.NO_PROXY || process.env.no_proxy || ''

  // if the noProxy is a wildcard then return null

  if (noProxy === '*') {
    return null
  }

  // if the noProxy is not empty and the uri is found return null

  if (noProxy !== '' && uriInNoProxy(uri, noProxy)) {
    return null
  }

  // Check for HTTP or HTTPS Proxy in environment Else default to null

  if (uri.protocol === 'http:') {
    return process.env.HTTP_PROXY ||
      process.env.http_proxy || null
  }

  if (uri.protocol === 'https:') {
    return process.env.HTTPS_PROXY ||
      process.env.https_proxy ||
      process.env.HTTP_PROXY ||
      process.env.http_proxy || null
  }

  // if none of that works, return null
  // (What uri protocol are you using then?)

  return null
}

module.exports = getProxyFromURI
```

request会根据proxy.env.HTTP_PROXY、process.env.https_proxy、process.env.http_proxy、process.env.HTTPS_PROXY中的值蛇追代理，然后我们打印process.env的值：

```javascript
process.env.http_proxy = http://172.xx.xx.xx:xxxx
```

果然，由于设置这个环境变量，导致A的请求被代理。但是我们检查过配置，没有设置代理的地方，于是开始联系运维，最终从运维了解到，在安装机器的pm2时，会设置这个代理。

好了，问题明白了，由于设置http_proxy的代理，导致服务请求被代理，而代理服务器无法访问目标机器，从而导致服务超时。

## 解决方案

了解了request设置代理的逻辑，于是提出了如下的解决方案：

1. 重启pm2进程，刷新process.env.http_proxy的值
2. 手动设置node应用的process.env.http_proxy为空
3. 设置process.env.no_proxy=*，忽略所有的代理设置

最终还是需要优化pm2的安装方式，避免引入其他的数据，从而影响到应用。

## 深入剖析

那么问题来了：为什么pm2设置了一个环境变量，Node的应用会受到该环境变量的影响呢？

关于这个问题，我们需要了解pm2的机制 [pm2 cluster mode](https://pm2.keymetrics.io/docs/usage/cluster-mode/) ,其中有一段话：Under the hood, this uses the Node.js [cluster module](https://nodejs.org/api/cluster.html) such that the scaled application’s child processes can automatically share server ports. 也就是说pm2的cluster mode底层是使用Node的cluster实现的，Node的cluster创建子进程是使用child_process.fork()方法，我们看下fork的方法定义：![](https://raw.githubusercontent.com/love477/picture/main/images/11a540c0-6cf3-4ccd-add6-fe5776ff7965.png?token=AGTQAHHQA6OVI42LDTW4MSTAGTUNI)

options中的env默认使用的是父进程的process.env，所以在pm2上启动cluster模式的服务的时候，pm2的process.env后默认设置到对应的应用中。

到这里，我们就明白了为什么pm2设置的process.env.http_proxy的值，会在应用中造成影响。