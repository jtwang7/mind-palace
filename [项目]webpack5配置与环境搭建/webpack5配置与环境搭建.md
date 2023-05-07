# webpack5配置与环境搭建

参考文章：

- [前端工程化】webpack5从零搭建完整的react18+ts开发和打包环境](https://juejin.cn/post/7111922283681153038#heading-39)
- [2020年了,再不会webpack敲得代码就不香了(近万字实战)](https://juejin.cn/post/6844904031240863758#heading-22)

## 项目初始化

在开始 webpack 配置之前，先手动初始化一个基本的 react + ts 项目，新建项目文件夹并在项目目录下执行。

1. 完成 `package.json` 文件初始化。

    ```js
    npm init -y
    ```

2. 在项目下新增以下所示目录结构和文件。

    ```yaml
    ├── build
    |   ├── webpack.base.js # 公共配置
    |   ├── webpack.dev.js  # 开发环境配置
    |   └── webpack.prod.js # 打包环境配置
    ├── public
    │   └── index.html # html模板
    ├── src
    |   ├── App.tsx 
    │   └── index.tsx # react应用入口页面
    ├── tsconfig.json  # ts配置
    └── package.json
    ```

3. 安装 webpack 以及 react 依赖：

    ```js
    npm i webpack webpack-cli -D
    npm i react react-dom -S
    npm i @types/react @types/react-dom -D
    ```

至此我们项目初步搭建完成，接下来要对 `webpack.**.js` 文件进行配置。

## webpack 配置文件

`webpack.**.js` 配置文件向 webpack 提供了编译项目文件的具体编译规则。`webpack.**.js` 本质上就是普通的 JS 模块文件，遵循 `CJS` 模块规范，导出包含 `webpack` 一系列配置选项的对象。

其中配置文件可根据环境分割为三种配置文件：

- `webpack.base.js` 公共环境: 配置了一系列公共编译属性，同时应用在生产环境和开发环境。
- `webpack.dev.js` 开发环境: 需要借助 [webpack-dev-server](https://link.juejin.cn/?target=https%3A%2F%2Fwww.npmjs.com%2Fpackage%2Fwebpack-dev-server)在开发环境启动服务器来辅助开发,还需要依赖[webpack-merge](https://link.juejin.cn/?target=https%3A%2F%2Fwww.npmjs.com%2Fpackage%2Fwebpack-merge)来合并基本配置。
- `webpack.prod.js` 生产环境: 需要依赖[webpack-merge](https://link.juejin.cn/?target=https%3A%2F%2Fwww.npmjs.com%2Fpackage%2Fwebpack-merge)来合并基本配置。

### 开发环境

1. 首先安装开发环境的一些 webpack 依赖。

    ```js
    npm i webpack-dev-server webpack-merge -D
    ```

2. 配置开发环境的 webpack 配置文件

    ```js
    // webpack.dev.js
    const path = require('path')
    const { merge } = require('webpack-merge')
    const baseConfig = require('./webpack.base.js')

    // 合并公共配置,并添加开发环境配置
    module.exports = merge(baseConfig, {
      mode: 'development', // 开发模式,打包更加快速,省了代码优化步骤
      devtool: 'eval-cheap-module-source-map', // 源码调试模式
      devServer: {
        port: 3000, // 服务端口号
        compress: false, // gzip压缩,开发环境不开启,提升热更新速度
        hot: true, // 开启热更新，后面会讲react模块热替换具体配置
        historyApiFallback: true, // 解决history路由404问题
        static: {
          directory: path.join(__dirname, "../public"), //托管静态资源public文件夹
        }
      }
    })
    ```

3. `package.json` 添加 `dev` 脚本

    ```js
    // package.json
    "scripts": {
      "dev": "webpack-dev-server -c build/webpack.dev.js"
    },
    ```

    执行**`npm run dev`**，项目就可以借助 `webpack-dev-server` 启动了，通过 [http://localhost:3000/](https://link.juejin.cn/?target=http%3A%2F%2Flocalhost%3A3000%2F) 在本地访问项目。

### 生产环境

1. 配置生产环境的 webpack 配置文件

    ```js
    // webpack.prod.js
    const { merge } = require('webpack-merge')
    const baseConfig = require('./webpack.base.js')
    module.exports = merge(baseConfig, {
      mode: 'production', // 生产模式,会开启tree-shaking和压缩代码,以及其他优化
    })
    ```

2. `package.json` 添加 `build` 脚本

    ```js
    "scripts": {
        "build": "webpack -c build/webpack.prod.js"
    },
    ```

    执行**`npm run build`**,最终打包到指定输出目录下 (参考公共配置 `output` 属性)。

## 深度配置

### entry / output

- [entry](https://webpack.js.org/concepts/entry-points)
- [output](https://webpack.js.org/concepts/output/)

#### 常见配置

```js
const path = require('path')
const HtmlWebpackPlugin = require('html-webpack-plugin')

module.exports = {
  //webpack 编译时访问的入口文件
  entry: path.join(__dirname, '../src/index.tsx'), 
  //打包文件出口
  output: {
    //编译后输出的每个js文件名称
    filename: 'static/js/[name].js', 
    //打包结果输出路径
    path: path.join(__dirname, '../dist'), 
    //打包前删除原有打包文件。webpack4需要配置clean-webpack-plugin；webpack5内置
    clean: true, 
    //打包后文件的公共前缀路径
    publicPath: '/' 
  },
}
```

#### 多入口 / 多出口

webpack 可以配置多个入口及动态出口，实现多模块打包。

```js
// webpack.config.js
module.exports = {
  // entry: {<entryChunkName>:string} | string[]
  entry: {
    app: './src/app.js',
    search: './src/search.js',
  },
  output: {
    filename: '[name].js', // name => entryChunkName
    path: __dirname + '/dist',
  },
};

// writes to disk: ./dist/app.js, ./dist/search.js
```

### loader

#### babel-loader 配置

babel-loader 用于将高版本的 js 代码转化为兼容低版本的 js 代码。除此之外，依赖 babel 插件可以拓展 babel 的语法转换功能。
关于 babel 的具体概念，可以参考 [What is babel?](https://github.com/jtwang7/mind-palace/blob/main/%5B%E9%A1%B9%E7%9B%AE%5DWhat%20is%20babel%3F/What%20is%20babel%3F.md) 这篇文章。

```js
npm i babel-loader @babel/core -D
npm i @babel/preset-react @babel/preset-typescript -D
npm i @babel/preset-env core-js -D
```

- `@babel/preset-react`: 由于 webpack 默认只能识别 js 文件,不能识别 jsx 语法，需要借助预设 [@babel/preset-react](https://link.juejin.cn/?target=https%3A%2F%2Fwww.babeljs.cn%2Fdocs%2Fbabel-preset-react) 来识别 jsx 语法，将其转为 js 文件。
- `@babel/preset-typescript`: ts 语法转换为 js 语法。
- `@babel/preset-env`: babel 编译的预设,可以转换目前最新的 js 标准语法。
- `core-js`: 使用低版本 js 语法模拟高版本的库。




```js
npm i babel-loader @babel/core -D
npm i @babel/preset-react @babel/preset-typescript -D
npm i @babel/preset-env core-js -D
npm i style-loader css-loader -D
npm i less-loader less -D
npm i html-webpack-plugin -D
```

- `html-webpack-plugin`: 向 webpack 提供将构建好的静态资源引入到一个 html 文件中的能力，使打包项目能在浏览器中运行。
- `@babel/preset-react`: 由于 webpack 默认只能识别 js 文件,不能识别 jsx 语法，需要借助预设 [@babel/preset-react](https://link.juejin.cn/?target=https%3A%2F%2Fwww.babeljs.cn%2Fdocs%2Fbabel-preset-react) 来识别 jsx 语法，将其转为 js 文件。
- `@babel/preset-typescript`: ts 语法转换为 js 语法。
- `@babel/preset-env`: babel 编译的预设,可以转换目前最新的 js 标准语法。
- `core-js`: 使用低版本 js 语法模拟高版本的库。
- `style-loader`: 把解析后的 css 代码从 js 中抽离,放到头部的 style 标签中(在运行时做的)
- `css-loader`: 解析 css 文件代码
  - webpack 默认只识别 js,不识别 css 文件,需要使用 loader 来解析 css。
- `less-loader`: 解析 less 文件代码,把 less 编译为 css
  - 项目开发中为了更好的提升开发体验，一般会使用 css 超集 less 或者 scss，对于这些超集也需要对应的 loader 来识别解析。
- `less`: less 核心

### webpack.config.js 配置

> 小技巧：在使用 vscode 配置 webpack.{config}.js 时，可以在文件开头加上：
>
> ```js
> const {Configuration} = require('webpack');
> /**
>  * @type {Configuration}
>  */
> ```
>
> 添加后 vscode 会在你配置 webpack 时自动给予代码提示。
>
> 参考：[webpack配置项智能提示](https://joshuatz.com/posts/2020/vscode-intellisense-autocomplete-for-webpack-config-files/)

```js
const path = require('path')
const HtmlWebpackPlugin = require('html-webpack-plugin')

module.exports = {
  // --- ❇️ 编译访问入口 / 打包输出出口 ---
  entry: path.join(__dirname, '../src/index.tsx'), //webpack 编译时访问的入口文件
  //打包文件出口
  output: {
    filename: 'static/js/[name].js', //编译后输出的每个js文件名称
    path: path.join(__dirname, '../dist'), //打包结果输出路径
    clean: true, //打包前删除原有打包文件。webpack4需要配置clean-webpack-plugin；webpack5内置
    publicPath: '/' //打包后文件的公共前缀路径
  },
  
  // --- ❇️ loader ---
  module: {
    rules: [
      // --- babel-loader ---
      {
        // @include: 只解析该选项指定路径下的模块。
        // @exclude: 不解该选项配置的模块(优先级更高)。
        include: [path.resolve(__dirname, '../src')] // 只对项目src文件的ts,tsx进行loader解析。
        test: /.(ts|tsx)$/, //匹配以.ts或.tsx结尾的文件
        use: 'babel-loader',
        // options: { ... }
      },
      // --- 样式 ---
      {
        test: /.(css|less)$/, //匹配 css 文件
        use: ['style-loader','css-loader', 'less-loader'] // 匹配到less文件后, 使用less-loader解析为css, 再用css-loader解析, 再借助style-loader把css解析结果插入到头部style标签中
      }
    ]
  }
  
  // --- ❇️ plugin ---
  plugins: [
    new HtmlWebpackPlugin({
      template: path.resolve(__dirname, '../public/index.html'), // 模板取定义root节点的模板
      inject: true, // 自动注入静态资源
    })
  ]
  
  // --- ❇️ 持久化缓存 ---
  cache: {
    type: 'filesystem', // 使用文件缓存
  },
  
  // --- ❇️ 额外配置 ---
  resolve: {
    // @extensions: 在引入模块不带文件后缀时，遍历该配置数组元素，依次添加后缀查找文件。
    // (注意把高频出现的文件后缀放在前面，减少查找次数。)
    extensions: ['.js', '.tsx', '.ts'], 
    // @alias: 配置路径别名
    alias: {
      '@': path.join(__dirname, '../src')
    }
  }
}
```

```js
// babel.config.js
// 为了避免 webpack 配置文件过于庞大, 把 babel-loader 的配置抽离出来, 新建 babel.config.js 文件
module.exports = {
  "presets": [
    "@babel/preset-env",
    "@babel/preset-react",
    "@babel/preset-typescript"
  ]
}
```

## 配置环境变量

环境变量按作用可分两种

- 开发模式 or 打包构建模式

  > 区分开发模式还是打包构建模式可以用 **process.env.NODE_ENV** 

- 开发/测试/预测/正式环境

  > 区分项目接口环境可以自定义一个环境变量**process.env.BASE_ENV** 

自定义环境变量需要借助[cross-env](https://link.juejin.cn?target=https%3A%2F%2Fwww.npmjs.com%2Fpackage%2Fcross-env)和[webpack.DefinePlugin](https://link.juejin.cn?target=https%3A%2F%2Fwww.webpackjs.com%2Fplugins%2Fdefine-plugin%2F)来设置。

- **cross-env**：兼容各系统的设置环境变量的包
- **webpack.DefinePlugin**：**webpack**内置的插件,可以为业务代码注入环境变量

1. 安装**cross-env**

```sh
npm i cross-env -D
```

2. 修改**package.json**的**scripts**脚本字段

   **dev**开头是开发模式,**build**开头是打包模式,冒号后面对应的**dev**/**test**/**pre**/**prod**是对应的业务环境的**开发**/**测试**/**预测**/**正式**环境。

   ```js
   "scripts": {
     "dev:dev": "cross-env NODE_ENV=development BASE_ENV=development webpack-dev-server -c build/webpack.dev.js",
     "dev:test": "cross-env NODE_ENV=development BASE_ENV=test webpack-dev-server -c build/webpack.dev.js",
     "dev:pre": "cross-env NODE_ENV=development BASE_ENV=pre webpack-dev-server -c build/webpack.dev.js",
     "dev:prod": "cross-env NODE_ENV=development BASE_ENV=production webpack-dev-server -c build/webpack.dev.js",
   
     "build:dev": "cross-env NODE_ENV=production BASE_ENV=development webpack -c build/webpack.prod.js",
     "build:test": "cross-env NODE_ENV=production BASE_ENV=test webpack -c build/webpack.prod.js",
     "build:pre": "cross-env NODE_ENV=production BASE_ENV=pre webpack -c build/webpack.prod.js",
     "build:prod": "cross-env NODE_ENV=production BASE_ENV=production webpack -c build/webpack.prod.js",
   }
   ```

   > **process.env.NODE_ENV** 环境变量**webpack**会自动根据设置的**mode**字段来给业务代码注入对应的**development**和**prodction**。这里在命令中再次设置环境变量**NODE_ENV**是为了在**webpack**和**babel**的配置文件中访问到。
   >
   
   执行**`npm run build:dev`**命令，表明当前是打包模式，业务环境是开发环境。

3. 修改**webpack.base.js**

   这里需要把**process.env.BASE_ENV**注入到业务代码里面, 业务代码中就可以通过访问该环境变量的接口地址获取其他数据,要借助**webpack.DefinePlugin**插件。

   ```js
   // webpack.base.js
   // ...
   const webpack = require('webpack')
   module.export = {
     // ...
     plugins: [
       // ...
       new webpack.DefinePlugin({
         'process.env.BASE_ENV': JSON.stringify(process.env.BASE_ENV)
       })
     ]
   }
   ```

   配置后会把值注入到业务代码里面去,**webpack**解析代码匹配到**process.env.BASE_ENV**,就会设置到对应的值。我们可以在业务代码中通过 `process.env.BASE_ENV` 访问到这个自定义的环境变量。

   可以通过以下代码测试一下，在**src/index.tsx**打印一下两个环境变量

   ```js
   // src/index.tsx
   // ...
   console.log('NODE_ENV', process.env.NODE_ENV)
   console.log('BASE_ENV', process.env.BASE_ENV)
   ```

   

## 🔥 进阶

**[webpack documentation](https://webpack.js.org/concepts/)**

**[webpack plugins](https://webpack.js.org/plugins/)**


---



**打包 css 样式至一个 .css 文件内**

- [MiniCssExtractPlugin](https://webpack.js.org/plugins/mini-css-extract-plugin)

  > 收集项目中 .css 文件并统一打包到一个 .css 文件中

- [CssMinimizerWebpackPlugin](https://webpack.js.org/plugins/css-minimizer-webpack-plugin/#root)

  > 优化和压缩 .css 文件

webpack5 利用 **[MiniCssExtractPlugin](https://webpack.js.org/plugins/mini-css-extract-plugin)** 插件打包 css 文件，利用  [CssMinimizerWebpackPlugin](https://webpack.js.org/plugins/css-minimizer-webpack-plugin/#root) 插件压缩 .css 文件体积。

> 由于 MiniCssExtractPlugin 建立在 webpack v5 的一个新功能之上，需要webpack 5才能工作。
>
> webpack v4 及以下版本用户请用 extract-text-webpack-plugin 插件代替。

```js
npm install --save-dev mini-css-extract-plugin css-minimizer-webpack-plugin
// or
yarn add -D mini-css-extract-plugin css-minimizer-webpack-plugin
// or
pnpm add -D mini-css-extract-plugin css-minimizer-webpack-plugin
```

**mini-css-extract-plugin 通常和 css-loader 联用，首先对 css 进行解析，然后再通过插件打包。** 

最终 webpack.config.js 配置如下：

```js
const MiniCssExtractPlugin = require("mini-css-extract-plugin");
const CssMinimizerPlugin = require("css-minimizer-webpack-plugin");
const path = require('path')

module.exports = {
  plugins: [new MiniCssExtractPlugin({
    filename: path.join(__dirname, './dist/index.css') // 指定打包输出的 .css 文件路径
  })],
  optimization: {
    minimizer: [
      // For webpack@5 you can use the `...` syntax to extend existing minimizers (i.e. `terser-webpack-plugin`), uncomment the next line
      // `...`,
      new CssMinimizerPlugin(),
    ],
  },
  module: {
    rules: [
      {
        test: /\.css$/,
        use: [MiniCssExtractPlugin.loader, "css-loader"],
      },
      {
        test: /\.less$/,
        use: [MiniCssExtractPlugin.loader, "css-loader", "less-loader"],
      },
    ],
  },
};
```
