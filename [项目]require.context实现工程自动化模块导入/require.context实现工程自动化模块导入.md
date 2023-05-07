# require.context实现工程自动化模块导入

参考文章：

- [工程自动化导入:require.context](https://juejin.cn/post/6982813323217600543#heading-5)
- [webpack 与 vite 实现模块自动引入-导入多个模块](https://blog.csdn.net/qq_43869822/article/details/121570878)
- [自动化导入模块：require.context](https://blog.csdn.net/Superman_peng/article/details/121181831)
- [require.context 使用](https://juejin.cn/post/6844904163298508807)

## 使用场景
写这篇文章的时候，我正在构建一个 react 通用组件库。在项目打包时，我需要将所有组件文件夹路径下的 css 文件统一通过 rollup 工具抽离到 index.css 文件中 (就像 antd4.x 那样)。

```js
// 组件库的项目结构
// 期望将所有 style 下的 index.less 统一抽离到一个 index.css 文件中
├─ components
│  ├─ Button
│  │  ├─ button.tsx
│  │  ├─ index.tsx
│  │  ├─ interface.ts
│  │  └─ style
│  │     ├─ index.less
│  │     └─ index.tsx
│  ├─ InputTag
│  │  ├─ index.tsx
│  │  ├─ input-tag.tsx
│  │  ├─ interface.ts
│  │  └─ style
│  │     ├─ index.less
│  │     └─ index.tsx
│  └─ index.tsx
```

但是在实际打包过程中发现，rollup 的 postcss 插件只能识别和抽取模块依赖树下关联的 js 文件中的 css 引入，无法探测和打包未被引用的 .css 静态资源。有人可能会疑惑，在每个组件中引入对应的样式文件不就可以了吗？确实，该思路可以保证每个 .css 文件都被引用，从而被 rollup 识别和抽离打包。但是，组件库开发我们通常不推荐这么做，原因在于组件库需要兼容不同的场景，举个例子：`import index.css` 语句在 web 端可以被正常识别，但无法被 server 端解析，假如前端项目需要 server render，而我们组件库中各个组件都写死了 `import index.css` 样式导入语句，那么就会导致组件库整体无法适配 server render 的场景。
总结而言，为了保证组件库的通用，通常需要单独打包 js 和 css 文件，并避免在 js 中直接引入 css 样式。

❌ css 与组件耦合，无法在服务端使用

```js
// ./components/Button/index.tsx

import "./index.less";

export default function Button(props) {
  return <button className="some-style"></button>
}
```

```js
// ./app.tsx

import Button from 'myNpmPackage'

function App() {
  return (
    <Button />
  )
}
```

✅ 正确的组件开发方式

```js
// ./components/Button/index.tsx

export default function Button(props) {
  return <button className="some-style"></button>
}
```

```js
// ./app.tsx

import Button from 'myNpmPackage';
import "myNpmPackage/style/index.css";

function App() {
  return (
    <Button />
  )
}
```

## 解决方法

已知 rollup 等编译打包工具抽离样式并生成 .css 文件，需要在 js 文件中主动引入 css 样式。为了打包所有的静态 css 资源，我们需要通过 `import xxx.css` 语句遍历引入所有的 css 样式文件。为了避免直接在组件中添加样式引入语句，我们可以将所有 `import xxx.css` 语句放在组件导出的入口文件 index.ts 中主动调用，确保 rollup 可以通过访问入口文件，收集到 css 样式的引用，单独抽离并打包到 index.css。

```js
import "./component/Button/style/index.less";
import "./component/InputTag/style/index.less";
// more css import ...

export {default as Button} from "./component/Button";
export {default as InputTag} from "./component/InputTag";
// more component export ...
```

上述解决方法的思路来自于 antd4.x-stable 的[相关代码](https://github.com/ant-design/ant-design/blob/4.x-stable/index-style-only.js)：

```js
// index-style-only.js

function pascalCase(name) {
  return name.charAt(0).toUpperCase() + name.slice(1).replace(/-(\w)/g, (m, n) => n.toUpperCase());
}

// Just import style for https://github.com/ant-design/ant-design/issues/3745
const req = require.context('./components', true, /^\.\/[^_][\w-]+\/style\/index\.tsx?$/);

req.keys().forEach(mod => {
  let v = req(mod);
  if (v && v.default) {
    v = v.default;
  }
  const match = mod.match(/^\.\/([^_][\w-]+)\/index\.tsx?$/);
  if (match && match[1]) {
    if (match[1] === 'message' || match[1] === 'notification') {
      // message & notification should not be capitalized
      exports[match[1]] = v;
    } else {
      exports[pascalCase(match[1])] = v;
    }
  }
});

module.exports = exports;
```

```js
// index.js
require('./index-style-only');

module.exports = require('./components');
```

简单分析可以发现，antd4.x 在 index-style-only.js 文件中，使用到了 `require.context()` 这一方法，它递归遍历了 ./components 文件夹下的所有正确匹配的 style/index.tsx 模块，并将这些模块加载并导出。

## require.context(directory，useSubdirectories，regExp)

[require.context](https://webpack.js.org/guides/dependency-management/#requirecontext) 是 webpack 中的一个 api。当在其他环境使用时，需要单独引入一个 npm 模块 [require-context.macro](https://www.npmjs.com/package/require-context.macro)

你可以用require.context()函数创建模块导入的上下文，通过执行 require.context() 函数，可以获取指定的文件夹内的特定文件，在需要多次从同一个文件夹内倒入的模块，使用这个函数可以自动倒入，不用每个都显示的写import来引入。

> Webpack在构建过程中会解析代码中的require.context()。

### 参数

- directory: 要搜索的目录
- useSubdirectories: 是否递归查询子目录
- regExp: 用于匹配文件的正则表达式

### 返回结果

require.context() 返回一个上下文模块。该上下文模块是一个 require() 函数，该函数接收一个参数：request。

此外，该函数还附带3个属性：

- resolve: 函数，执行该函数返回解析后的请求的模块id。
- keys: 函数，执行后返回一个数组，其包含上下文模块可以处理的所有请求(request)。
- id: 上下文模块的 id。

常见的是 `.keys()` 和 `require()` 的搭配使用，就像 antd4.x index-style-only 中那样：

```js
// 获取 ./components 文件夹下的所有模块请求(导入)
const req = require.context('./components', true, /^\.\/[^_][\w-]+\/style\/index\.tsx?$/);

req.keys().forEach(mod => {
  // mod 为可以被 request 的模块
  let v = req(mod);
  
  if (v && v.default) {
    v = v.default;
  }
  const match = mod.match(/^\.\/([^_][\w-]+)\/index\.tsx?$/);
  if (match && match[1]) {
    if (match[1] === 'message' || match[1] === 'notification') {
      // message & notification should not be capitalized
      exports[match[1]] = v;
    } else {
      exports[pascalCase(match[1])] = v;
    }
  }
});
```