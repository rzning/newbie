微信小程序 > 开发 > 指南 > 基础能力

# 基础能力

1. [网络](#network)
  - [局域网通信](#mDNS)
2. [存储](#storage)
3. [文件系统](#fs)
4. [画布](#canvas)
5. [分包加载](#subpackages)
6. [多线程 Worker](#workers)
7. [服务器端能力](#server)
8. [自定义 tabBar](#tabbar)
9. [周期性更新](#fetch)


<hr id="network"/>

## 1. 网络

> [dev/framework/ability/network](https://developers.weixin.qq.com/miniprogram/dev/framework/ability/network.html)

### 1.1 服务器域名配置

每个微信小程序需要事先设置通信域名。

小程序只能跟指定的域名进行网络通信。

包括：

- 普通 HTTPS 请求 [`wx.request()`][1.1.1]
- 上传文件 [`wx.uploadFile()`][1.1.2]
- 下载文件 [`wx.downloadFile()`][1.1.3]
- WebSocket 通信 [`wx.connectSocket()`][1.1.4]
- UDP 通信 [`wx.createUDPSocket()`][1.1.5]

从基础库 2.4.0 开始，网络可以与局域网 IP 通信，但不允许与本机 IP 通信。

服务器域名在 `小程序后台 > 开发 > 开发设置 > 服务器域名` 中配置：

- 域名只支持 HTTPS 和 WSS 协议
  - HTTPS : [`wx.request()`][1.1.1] , [`wx.uploadFile()`][1.1.2] , [`wx.downloadFile()`][1.1.3]
  - WSS : [`wx.connectSocket()`][1.1.4]
- 域名不能使用 IP 地址或 `localhost` ，局域网 IP 除外
- 可以配置端口号
- 若未配置端口号，则请求的 URL 也不能包含端口号
- 域名必须经过 ICP 备案
- `api.weixin.qq.com` 不能被配置为服务器域名，相关 API 也不能在小程序内调用
- 每个接口最多配置 20 个域名

### 1.2 网络请求

- 超时时间

  - 默认为 60s
  - 可在 [`app.json`](./config.md#app) 中通过 `networkTimeout` 配置

- 使用限制

  - 网络请求的 `referer` header 不可设置，其固定格式为：
    - `https://servicewechat.com/{appid}/{version}/page-frame.html`
      - `{appid}` 小程序 appid
      - `{version}` 小程序版本号
  - HTTPS 最大并发限制 10 个
  - WSS 最大并发限制 5 个
  - 小程序进入后台，5s 内网络请求未结束，抛出 `fail interrupted` 错误

- 返回值编码

  - 使用 UTF-8 编码
  - 自动对 BOM 头过滤

- 回调函数

  - 只要成功接收到服务器返回，无论 `statusCode` 多少，都会进入 `success` 回调

### 1.3 常见问题

- HTTPS 证书
- 跳过域名校验


<hr id="mDNS"/>

### 1.4 局域网通信

> [dev/framework/ability/mDNS](https://developers.weixin.qq.com/miniprogram/dev/framework/ability/mDNS.html)

- [`wx.startLocalServiceDiscovery()`][1.4.1] 搜索局域网内提供 mDNS 服务的设备 IP
- [`wx.request()`][1.1.1] 等请求的 URL 参数允许为 `${IP}:${PORT}/${PATH}` 格式
  - 当且仅当 IP 与 手机 IP 处于同一网段，且非本机 IP 时，请求或链接才会成功
  - 此时不会进行安全域校验，可以使用 HTTP 或 WS

目前小程序只支持通过 mDNS 协议获取局域网内其他设备的 IP

- IOS 基于 [Bonjour](https://developer.apple.com/bonjour/)
- Android 基于 [NSD 网络服务发现](https://developer.android.com/training/connect-devices-wirelessly/nsd)
  - 参考 [使用网络服务发现][android.nsd]
  - 要在本地网络上注册服务，需首先创建一个 [`NsdServiceInfo`](https://developer.android.google.cn/reference/android/net/nsd/NsdServiceInfo.html) 对象

服务类型 serviceType

发起 mDNS 服务搜索需传入 `serviceType` 参数：

```js
wx.startLocalServiceDiscovery({
  serviceType: '_http._tcp.',
  success () {},
  fail () {}
})
```

- [域命名约定 | Bonjour Overview - IOS ](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/NetServices/Articles/domainnames.html)
  - 格式 `_ServiceType._TransportProtocolName.`
    - `_ServiceType` : 服务类型 `ftp` , `http` , `printer`
    - `_TransportProtocolName` : 传输协议名称 `tcp` , `udp`
  - 例如在 TCP 上运行的 FTP 服务的注册类型为 `_ftp._tcp.`
- [使用网络服务发现 | Android Developers][android.nsd]
  - 参数 `serviceType` 指定应用程序使用的协议和传输层
  - 语法 `_<protocol>._<transportlayer>`
- [服务名称和传输协议端口号注册表](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml)


[1.1.1]: <https://developers.weixin.qq.com/miniprogram/dev/api/network/request/wx.request.html>
[1.1.2]: <https://developers.weixin.qq.com/miniprogram/dev/api/network/upload/wx.uploadFile.html>
[1.1.3]: <https://developers.weixin.qq.com/miniprogram/dev/api/network/download/wx.downloadFile.html>
[1.1.4]: <https://developers.weixin.qq.com/miniprogram/dev/api/network/websocket/wx.connectSocket.html>
[1.1.5]: <https://developers.weixin.qq.com/miniprogram/dev/api/network/udp/wx.createUDPSocket.html>
[1.4.1]: <https://developers.weixin.qq.com/miniprogram/dev/api/network/mdns/wx.startLocalServiceDiscovery.html>
[android.nsd]: <https://developer.android.google.cn/training/connect-devices-wirelessly/nsd>


<hr id="storage"/>

## 2. 存储

> [dev/framework/ability/storage](https://developers.weixin.qq.com/miniprogram/dev/framework/ability/storage.html)



<hr id="fs"/>

## 3. 文件系统

> [dev/framework/ability/file-system](https://developers.weixin.qq.com/miniprogram/dev/framework/ability/file-system.html)



<hr id="canvas"/>

## 4. 画布

> [dev/framework/ability/canvas](https://developers.weixin.qq.com/miniprogram/dev/framework/ability/canvas.html)



<hr id="subpackages"/>

## 5. 分包加载

> [dev/framework/subpackages](https://developers.weixin.qq.com/miniprogram/dev/framework/subpackages.html)



<hr id="workers"/>

## 6. 多线程 Worker

> [dev/framework/workers](https://developers.weixin.qq.com/miniprogram/dev/framework/workers.html)



<hr id="server"/>

## 7. 服务器端能力

### 7.1 后端 API

> [dev/framework/server-ability/backend-api](https://developers.weixin.qq.com/miniprogram/dev/framework/server-ability/backend-api.html)


### 7.2 消息推送

> [dev/framework/server-ability/message-push](https://developers.weixin.qq.com/miniprogram/dev/framework/server-ability/message-push.html)



<hr id="tabbar"/>

## 8. 自定义 tabBar

> [dev/framework/ability/custom-tabbar](https://developers.weixin.qq.com/miniprogram/dev/framework/ability/custom-tabbar.html)



<hr id="fetch"/>

## 9. 周期性更新

> [dev/framework/ability/background-fetch](https://developers.weixin.qq.com/miniprogram/dev/framework/ability/background-fetch.html)
