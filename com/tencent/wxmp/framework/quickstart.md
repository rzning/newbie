微信小程序 > 开发 > 指南 > 起步

# 起步

1. [小程序简介](#intro)
2. [开始](#getstart)
3. [小程序代码构成](#code)
4. [小程序宿主环境](#framework)
5. [小程序协同工作和发布](#release)

<hr id="intro"/>

## 1. 小程序简介

> [dev/framework/quickstart/](https://developers.weixin.qq.com/miniprogram/dev/framework/quickstart/)

- 小程序技术发展史
- 小程序与普通网页开发的区别
- 体验小程序

<hr id="getstart"/>

## 2. 开始

> [dev/framework/quickstart/getstart](https://developers.weixin.qq.com/miniprogram/dev/framework/quickstart/getstart.html)

- 申请账号

  - 进入 [小程序注册页](https://mp.weixin.qq.com/wxopen/waregister?action=step1) 填写信息，注册账号。
  - 登录 [小程序公众平台](https://mp.weixin.qq.com/) 进入 `开发` > `开发设置` 菜单查看小程序 `AppID` 信息。

- 安装开发者工具

  - 在 [下载页面](https://developers.weixin.qq.com/miniprogram/dev/devtools/download.html) 下载开发者工具。
  - 在 [文档页面](https://developers.weixin.qq.com/miniprogram/dev/devtools/devtools.html) 查看开发者工具介绍。

- 你的第一个小程序

  1. 打开微信开发者工具，使用绑定微信登录
  2. 选择 `小程序` 标签页，点击 `新建项目` 图标
  3. 填写 `项目名称` ，选择项目存放 `目录`
  4. 填写你的小程序 `AppID`
  5. 点击 `新建` 按钮

- 编译预览

  - 点击开发者工具上方 `工具栏` 中的 `预览` 按钮进行预览。

<hr id="code"/>

## 3. 小程序代码构成

> [dev/framework/quickstart/code](https://developers.weixin.qq.com/miniprogram/dev/framework/quickstart/code.html)

| 后缀    | 说明              |
| ------- | ----------------- |
| `.json` | `JSON` 配置文件   |
| `.wxml` | `WXML` 模板文件   |
| `.wxss` | `WXSS` 样式文件   |
| `.js`   | `JS` 脚本逻辑文件 |

### 3.1 JSON 配置

```
|- pages/
|   |- page/
|       |- page.json
|- app.json
|- project.config.json
```

- `app.json` [小程序配置](./config.md#app)
- `project.config.json` [开发者工具配置](../devtools/projectconfig.md)
- `page.json` [页面配置](./config.md#page)

### 3.2 WXML 模板

- [WXML](./view/view.md#wxml)

  1. `{{ data }}`
  2. `wx:for="{{ list }}"`
  3. `wx:if="{{ conditional }}"`
  4. `<template name="">` -> `<template is="" data="">`
  5. `<import src="template.wxml">`
  6. `<include src="snippet.wxml">`

### 3.3 WXSS 样式

- [WXSS](./view/view.md#wxss)

  1. 新增尺寸单位 `rpx`
  2. 提供全局样式 `app.wxss` 和局部样式 `page.wxss`
  3. `WXSS` 仅支持部分 `CSS` 选择器

### 3.3 JS 逻辑交互

- 处理用户交互事件

  - [WXML - 事件](./view/event.md)

- 调用微信提供的 API 能力
  - [小程序 API](./app-service.md#api)

<hr id="framework"/>

## 4. 小程序宿主环境

> [dev/framework/quickstart/framework](https://developers.weixin.qq.com/miniprogram/dev/framework/quickstart/framework.html)

微信客户端给小程序提供的环境称为宿主环境。

### 4.1 渲染层和逻辑层

小程序的运行环境分为 `渲染层` 和 `逻辑层` ：

- WXML 模板和 WXSS 样式工作在渲染层。
- JS 脚本运行在逻辑层。

小程序的渲染层和逻辑层分别由两个线程管理：

- 渲染层界面使用 WebView 进行渲染。
- 逻辑层采用 JsCore 线程运行 JS 脚本。

小程序多个界面中，每个界面都有独立 WebView 线程管理，
两个线程间的通信将通过微信客户端 ( 常称为 Native ) 进行中转。

逻辑层发送的网络请求也经由 Native 转发。

![4-1](https://res.wx.qq.com/wxdoc/dist/assets/img/4-1.ad156d1c.png)

有关渲染层和逻辑层的详细信息，参考 @ [小程序框架](./MINA.md)

### 4.2 程序与页面

在打开小程序之前，微信客户端会将小程序代码包下载到本地。

通过 `app.json` 的 `pages` 字段获取当前小程序所有页面路径：

```json
{
  "pages": ["pages/index/index", "pages/logs/logs"]
}
```

`pages` 字段的第一个页面为小程序的首页。

小程序启动后，将触发 `app.js` 中定义的 App 实例的 `onLaunch()` 方法：

```js
App({
  onLaunch() {
    // 小程序启动后触发
  }
})
```

每个小程序只有一个 App 实例，且所有页面共享。

- 详细 App 事件回调参考 @ [注册小程序](./app-service.md#app)

每个小程序页面都包括四种文件。

微信客户端首先根据 `page.json` 文件生成一个界面，页面顶部的颜色和文本都在 `page.json` 文件中指定。

接着客户端将装载页面的 WXML 结构和 WXSS 样式。

最后加载 `page.js` 脚本：

```js
Page({
  data: {},
  onLoad() {}
})
```

JS 脚本中 `Page()` 方法为一个页面构造器，传入页面配置，返回页面实例。

页面加载后，将触发 `onLoad()` 事件回调。

- 关于 `Page()` 构造器参考 @ [注册页面](./app-service.md#page)

### 4.3 组件

小程序提供丰富的基础组件供开发者使用。

- 参考 @ [小程序组件](../component/component.md)

### 4.4 API

为了让开发者方便调用微信提供的各种能力，如获取用户信息、微信支付等，
小程序提供丰富 API 供开发者使用。

获取用户地理位置：

```js
wx.getLocation({
  type: 'wgs84',
  success: (res) => {
    var latitude = res.latitude // 纬度
    var longitude = res.longitude // 经度
  }
})
```

调用微信扫一扫：

```js
wx.scanCode({
  success: (res) => {
    console.log(res)
  }
})
```

- 参考 @ [小程序 API](./app-service.md#api)

<hr id="release"/>

## 5. 小程序协同工作和发布

> [dev/framework/quickstart/release](https://developers.weixin.qq.com/miniprogram/dev/framework/quickstart/release.html)

### 5.1 协同工作

### 5.2 小程序的版本

### 5.3 发布上线

### 5.4 运营数据
