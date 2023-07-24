# 判断生产or测试环境

在日常开发中，我们常常需要判断当前项目环境是生产环境还是测试环境，进而做一些参数配置。通常有判断 `process.env.NODE_ENV` 和 `location.host` 两种方法。

## process.env.NODE_ENV

传统的方法是通过 `process.env.NODE_ENV` 这个参数来区分当前的生产环境：

```js
// 生产环境
process.env.NODE_ENV === "production"

// 测试环境
process.env.NODE_ENV === "development"

// ...
```

但是实际上，`process.env` 上并不存在 NODE_ENV 属性，参考[使用process.env.NODE_ENV的正确姿势](https://juejin.cn/post/7070347341282148365)可知:

`process.env` 属性返回一个包含用户环境信息的对象。在 node 环境中，当我们打印 `process.env` 时，发现它并没有NODE_ENV这一个属性。实际上，`process.env.NODE_ENV` 是在 `package.json` 的scripts 命令中注入的，也就是 NODE_ENV 并不是 node 自带的，而是由用户定义的，约定成俗的命名为 NODE_ENV。

```js
{
  "scripts": {
    "dev": "NODE_ENV=development webpack --config webpack.dev.config.js"
  }
}
```

可以看到 `NODE_ENV=development`，当执行 `npm run dev` 时，我们就可以在 `webpack.dev.config.js` 脚本以及它所关联的脚本中访问到 `process.env.NODE_ENV`，而无法在其它脚本中访问。在 scripts 命令中注入的 NODE_ENV 只能被 webpack 的构建脚本访问，而被 webpack 打包的源码中是无法访问到的，此时可以借助 webpack 的 DefinePlugin 插件，创建全局变量。

```js
const webpack = require('webpack');
module.exports = {
  plugins: [
    new webpack.DefinePlugin({
      'process.env.NODE_ENV': '"development"'
    })
  ]
}
```

> DefinePlugin 不仅仅可以定义 `process.env.NODE_ENV`，你也可以根据自己的需要定义其他的全局变量。定义完成之后，就可以在项目代码中直接使用了。

### cross-env: 跨平台设置 NODE_ENV

在 window 平台下直接设置 NODE_ENV 是会报错的，cross-env 能够提供一个设置环境变量的 scripts，这样我们就能够以 unix 方式设置环境变量，然后在 windows 上也能够兼容。

```js
npm install cross-env --save

// 在 package.json 设置 NODE_ENV 前添加 cross-env
"scripts": {
  "dev": "cross-env NODE_ENV=development webpack-dev-server"
}
```

## location.host

基于 NODE_ENV 判断环境可能失效的问题，例如忘记在 package.json scripts 中配置 NODE_ENV 环境变量。我们可以通过生产环境和测试环境域名不同判断。其主要通过 `location.host` 判断：

```js
// 测试环境
location.host === "[development 环境域名]"

// 生产环境
location.host === "[production 环境域名]"
```

## 总结

1. 配置 package.json scripts NODE_ENV 环境变量

```js
"scripts": {
  "dev": "cross-env NODE_ENV=development webpack-dev-server",
  "build": "rm -rf dist && cross-env NODE_ENV=production webpack --config config/webpack.prod.js --progress"
}
```

2. 判断语句

```js
process.env.NODE_ENV === "production" || location.host === "[production 环境域名]"

process.env.NODE_ENV === "development" || location.host === "[development 环境域名]"
```
