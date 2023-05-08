# What is babel-plugin-import?

参考文章:

- 🔥 [babel-plugin-import 使用](https://juejin.cn/post/7051206427402043423)
- [用 babel-plugin 实现按需加载](https://juejin.cn/post/6873740183078961166)
- [组件库按需加载 借助babel-plugin-import实现](https://juejin.cn/post/6844903855604367367)
- 🔥 [【前端工程化基础 - Babel 篇】简单实现 babel-plugin-import 插件](https://juejin.cn/post/6905708824703795214)

## 前言

近阶段开始接触了一些组件库开发相关的知识，发现 `babel-plugin-import` 这个 `babel` 插件应用在了众多组件库开发中 (例如 `antd` 从 `2.x ~ 4.x` 都使用了 `babel-plugin-import` )。然而 `antd 5.x` 版本中却没有了 `babel-plugin-import` 的身影，转而使用了 `css-in-js` 技术进行开发，这不禁让我好奇，`babel-plugin-import` 在组件库开发中究竟起到了什么作用？现在的组件库开发还需要它吗？

带着这些疑问，我们接下来将从 **`babel-plugin-import` 的概念/作用/原理/...** 等多个角度了解这个插件，同时我们还会涉及 **`tree-shaking` / `css-in-js` / 组件库 CSS 样式方案 / ...** 等一系列相关的知识，帮助你快速掌握基于 `babel-plugin-import` 展开的相关领域知识。读完整篇文章，或许可以解答你一下几点疑惑：

- `babel-plugin-import` 是什么？
- `babel-plugin-import` 在组件库开发中有什么作用？
- `babel-plugin-import` 它是怎么工作的 (工作流程)？
- 现在的组件库开发还需要 `babel-plugin-import` 吗？
- 组件库的 CSS 样式方案有哪些？`babel-plugin-import` 属于哪种样式解决方案？
- 在非 `babel` 环境中调用包含 `babel-plugin-import` 的组件库，会报错吗？

## 🔥 What is babel-plugin-import ?

### 作用

我们首先来看一个🌰例子：

```js
// 一个二次封装 arco-design button 的 React 组件

import { Button } from '@arco-design/web-react'

function AntdButton() {
  return <Button>arco button</Button>
}

export default AntdButton
```

我们用 `rollup` 对该组件进行打包编译，打包配置如下所示：

```js
// @ts-check
import glob from 'glob'
import { defineConfig } from 'rollup'
import typescript from 'rollup-plugin-typescript2'
import { getBabelOutputPlugin } from '@rollup/plugin-babel'

export default defineConfig([
  {
    input: await glob('./src/**/*.{ts,tsx}'),
    output: [
      {
        dir: 'dist/rollup',
        format: 'esm',
      },
    ],
    plugins: [
      getBabelOutputPlugin({
        plugins: [
          [
            'babel-plugin-import',
            {
              libraryName: '@arco-design/web-react',
              libraryDirectory: 'es',
              camel2DashComponentName: false,
              style: 'css',
            },
          ],
        ],
      }),
      typescript({ check: false }),
    ],
  },
])
```

在上述 `rollup` 配置中，我们需要关注两个重点信息：

- `babel-plugin-import` 中 `options - style` 配置为了 `css` 
- `output - format` 配置了 `esm`

我们基于上述配置，利用 `rollup` 对组件进行一次打包，查看 `dist/rollup` 文件夹下的输出文件，发现组件被编译成了如下代码：

```js
import "@arco-design/web-react/es/Button/style/css";
import _Button from "@arco-design/web-react/es/Button";
import { jsx } from 'react/jsx-runtime';
function AntdButton() {
  return jsx(_Button, {
    children: "arco button"
  });
}
export { AntdButton as default };
```

对比打包前后的组件文件，我们很容易就发现了几点变化：

- `import { Button } from '@arco-design/web-react'` 被编译为了 `import _Button from "@arco-design/web-react/es/Button"` 。这意味着，我们引用 `Button` 组件的地址从整个 `arco-design` 组件库具体到了 `arco-design/**/Button` 对应组件文件目录下。
- 打包组件文件开头被引入了 `import "@arco-design/web-react/es/Button/style/css";` 这行样式代码。

👉 **`babel-plugin-import` 的作用:** 其实上述打包文件的两点变化就是 `babel-plugin-import` 所做的工作了，我将其进行归纳为以下两点：

1. 实现组件的**按需引入**

   > 在 `babel` 转码的时候，把整个组件库 `arco-design` 的引用，变为 `arco-desgin/**/Button` 具体模块的引用。这样 `webpack` 收集依赖 `module` 就不是整个 `arco-design`，而是里面的 `button`.

2. 实现组件**样式的自动导入**

### 原理

> 具体参考：
>
> - [【前端工程化基础 - Babel 篇】简单实现 babel-plugin-import 插件](https://juejin.cn/post/6905708824703795214)
> - [babel-plugin-import 使用](https://juejin.cn/post/7051206427402043423)

`babel-plugin-import` 是 `babel` 的一种插件，所以其和普遍的 `babel` 插件一样，会遍历代码的 `ast`，然后在 `ast` 上做了一些事情：

1. **收集依赖**：找到 `importDeclaration`，分析出包 `a` 和依赖 `b,c,d....`，假如 `a` 和 `libraryName` 一致，就将 `b,c,d...` 在内部收集起来
2. **判断是否使用**：在多种情况下（比如 `CallExpression`）判断收集到的 `b,c,d...` 是否在代码中被使用，如果有使用的，就调用 `importMethod` 生成新的 `import` 语句
3. **生成引入代码**：根据配置项生成代码和样式的 `import` 语句

> 除此之外，还有很多细节这里就没提到，比如如何删除旧的 `import` 等... 感兴趣的可以自行阅读源码。

知道 `babel-plugin-import` 大致的工作流程后，我们具体深入了解下 `babel-plugin-import` 的具体代码实现，**`babel-plugin-import` 核心源码实现代码👇**

❇️ **前提：配置 `babel-plugin-import` opions**

```js
{
  "libraryName": "@arco-design/web-react",     // 包名
  "libraryDirectory": "lib", // 目录，默认 lib
  "style": true,             // 是否引入 style
}
```

❇️ **第一步：依赖收集**

`babel-plubin-import` 会在 `ImportDeclaration` 里将所有的 `specifier` 收集起来。

我们已知 `babel-plugin-import` 在配置的时候，要求开发者填写了 `libraryName` 指定具体应用转换的包名，所以在收集依赖阶段，`babel-plugin-import` 会基于  `ImportDeclaration` 内对应的 `AST` 内部节点信息判断当前遍历的包名是否是 `@arco-design/web-react`，若命中了配置项，则收集这个 `import` 语句导入的组件名，即 `Button` 和 `Input`。

```js
// ImportDeclaration 语句收集的对象👇
import { Button, Input } from '@arco-design/web-react';
ReactDOM.render(<Button />);
                
// ---
// 其在 AST 结构树中的表示形式：
{
  "type": "Program",
  "body": [
    {
      "type": "ImportDeclaration",
      "specifiers": [
        {
          "type": "ImportSpecifier",
          "imported": {
            "type": "Identifier",
            "name": "Button"
          },
          "local": {
            "type": "Identifier",
            "name": "Button"
          }
        },
        {
          "type": "ImportSpecifier",
          "imported": {
            "type": "Identifier",
            "name": "Input"
          },
          "local": {
            "type": "Identifier",
            "name": "Input"
          }
        }
      ],
      "source": {
        "type": "Literal",
        "value": "antd",
      }
    }
  ],
  "sourceType": "module"
}
```

```js
// ImportDeclaration 逻辑：
// traverser 遍历到的所有 ImportDeclaration 类型的节点都会被传入该类型逻辑函数中处理。
ImportDeclaration(path, state) {
  const { node } = path;
  if (!node) return;
  // 代码里 import 的包名
  const { value } = node.source;
  // 配在插件 options 的包名
  const { libraryName } = this;
  // babel-type 工具函数
  const { types } = this;
  // 内部状态
  const pluginState = this.getPluginState(state);
  // 判断是不是需要使用该插件的包
  if (value === libraryName) {
    // node.specifiers 表示 import 了什么
    node.specifiers.forEach(spec => {
      // 判断是不是 ImportSpecifier 类型的节点，也就是是否是大括号的
      if (types.isImportSpecifier(spec)) {
        // 收集依赖
        // 也就是 pluginState.specified.Button = Button
        // local.name 是导入进来的别名，比如 import { Button as MyButton } from 'antd' 的 MyButton
        // imported.name 是真实导出的变量名
        pluginState.specified[spec.local.name] = spec.imported.name;
      } else { 
        // ImportDefaultSpecifier 和 ImportNamespaceSpecifier
        pluginState.libraryObjs[spec.local.name] = true;
      }
    });
    pluginState.pathsToRemove.push(path);
  }
}
```

待 `babel` 遍历了所有的 `ImportDeclaration` 类型的节点之后，就收集好了依赖关系，下一步就是如何加载它们了。

❇️ **第二步：判断组件是否被使用 (按需引入)**

收集了依赖关系之后，得要判断一下这些 `import` 的变量是否被使用到了，`babel-plugin-import` 会在编译阶段舍弃未被应用的组件引入，从而减少模块的实际依赖。那么 `babel-plugin-import` 是如何判断我们的组件是否被使用呢？

我们知道，`JSX` 最终是变成 `React.createElement()` 执行的：

```js
ReactDOM.render(<Button>Hello</Button>);

      ↓ ↓ ↓ ↓ ↓ ↓

React.createElement(Button, null, "Hello");
```

因此，通过在 `AST` 树中定位 `createElement` 并获取它的第一个参数，就可以收集到实际会被渲染的组件的名称，从而过滤到未被实际使用的组件。除了 `React.createElement(Button)` 之外，还有 `const btn = Button` / `[Button]` ... 等多种情况会使用 `Button`，源码中都有对应的处理方法，感兴趣的可以自己看一下： [github.com/ant-design/…](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fant-design%2Fbabel-plugin-import%2Fblob%2Fmaster%2Fsrc%2FPlugin.js%23L163-L272) 。

`babel-plugin-import` 具体定位了有效组件引入后，会调用 `importMethod` 方法生成新的 `import` 导入语句，新生成的导入语句会具体指向组件目录的路径地址。具体代码实现：

```js
CallExpression(path, state) {
  const { node } = path;
  const file = (path && path.hub && path.hub.file) || (state && state.file);
  // 方法调用者的 name
  const { name } = node.callee;
  // babel-type 工具函数
  const { types } = this;
  // 内部状态
  const pluginState = this.getPluginState(state);

  // 如果方法调用者是 Identifier 类型
  if (types.isIdentifier(node.callee)) {
    if (pluginState.specified[name]) {
      node.callee = this.importMethod(pluginState.specified[name], file, pluginState);
    }
  }

  // 遍历 arguments 找我们要的 specifier
  node.arguments = node.arguments.map(arg => {
    const { name: argName } = arg;
    if (
      pluginState.specified[argName] &&
      path.scope.hasBinding(argName) &&
      path.scope.getBinding(argName).path.type === 'ImportSpecifier'
    ) {
      // 找到 specifier，调用 importMethod 方法
      return this.importMethod(pluginState.specified[argName], file, pluginState);
    }
    return arg;
  });
}
```

❇️ **第三步 生成引入代码 (核心)**

这一步是插件的核心步骤，即实现新导入路径的生成，让 `babel` 基于设置的 `options` 去修改代码并且生成一个新的 `import` 以及一个样式的 `import` 。

```js
import { addSideEffect, addDefault, addNamed } from '@babel/helper-module-imports';

importMethod(methodName, file, pluginState) {
  if (!pluginState.selectedMethods[methodName]) {
    // libraryDirectory：目录，默认 lib
    // style：是否引入样式
    const { style, libraryDirectory } = this;
    
    // 组件名转换规则
    // 优先级最高的是配了 camel2UnderlineComponentName：是否使用下划线作为连接符
    // camel2DashComponentName 为 true，会转换成小写字母，并且使用 - 作为连接符
    const transformedMethodName = this.camel2UnderlineComponentName
      ? transCamel(methodName, '_')
      : this.camel2DashComponentName
      ? transCamel(methodName, '-')
      : methodName;
    // 兼容 windows 路径
    // path.join('antd/lib/button') == 'antd/lib/button'
    const path = winPath(
      this.customName
        ? this.customName(transformedMethodName, file)
        : join(this.libraryName, libraryDirectory, transformedMethodName, this.fileName),
    );
    // 根据是否有导出 default 来判断使用哪种方法来生成 import 语句，默认为 true
    // addDefault(path, 'antd/lib/button', { nameHint: 'button' })
    // addNamed(path, 'button', 'antd/lib/button')
    pluginState.selectedMethods[methodName] = this.transformToDefaultImport
      ? addDefault(file.path, path, { nameHint: methodName })
      : addNamed(file.path, methodName, path);
    // 根据不同配置 import 样式
    if (this.customStyleName) {
      const stylePath = winPath(this.customStyleName(transformedMethodName));
      addSideEffect(file.path, `${stylePath}`);
    } else if (this.styleLibraryDirectory) {
      const stylePath = winPath(
        join(this.libraryName, this.styleLibraryDirectory, transformedMethodName, this.fileName),
      );
      addSideEffect(file.path, `${stylePath}`);
    } else if (style === true) {
      addSideEffect(file.path, `${path}/style`);
    } else if (style === 'css') {
      addSideEffect(file.path, `${path}/style/css`);
    } else if (typeof style === 'function') {
      const stylePath = style(path, file);
      if (stylePath) {
        addSideEffect(file.path, stylePath);
      }
    }
  }
  return { ...pluginState.selectedMethods[methodName] };
}
```

其中 `addSideEffect`, `addDefault` 和 `addNamed` 是 `@babel/helper-module-imports` 的三个方法，作用都是创建一个 `import` 语句，伪代码如下：

```js
addSideEffect(path, 'source');

      ↓ ↓ ↓ ↓ ↓ ↓

import "source"
```

```js
addDefault(path, 'source', { nameHint: "hintedName" })

      ↓ ↓ ↓ ↓ ↓ ↓

import hintedName from "source"
```

```js
addNamed(path, 'named', 'source', { nameHint: "hintedName" });

      ↓ ↓ ↓ ↓ ↓ ↓

import { named as _hintedName } from "source"
```

❇️ **总结：** `babel-plugin-import` 本质工作就是收集各种 `import` 语句内容，并基于 `libraryName` 调用逻辑处理匹配的包名，为其生成新的 `import` 语句。

因此，`babel-plugin-import` 并非只能用于 `antd` 和 `element` 等大型组件库。它实际是对 `import` 语句的转换，所以符合使用环境的第三方库都可以使用 `babel-plugin-import` 来实现按需加载和自动加载样式。比如我们常用的 `lodash`，也可以使用 `babel-plugin-import` 来加载它的各种方法。

### 使用方法

1. 安装

   ```js
   npm install babel-plugin-import --save-dev
   ```

2. 在 babel 配置文件 `.babelrc` or `babel-loader` 中配置

   ```js
   {
     plugins: [
       ["import", { 
         "libraryName": "antd", // 指定导入包的名称
         "libraryDirectory": "lib", // 指定模块的存放目录
         style: "css", // 导入 css 样式
       }]
     ]
   }
   ```

3. 配置多个包按需引入

   > 有时候项目中需要对多个依赖包做按需引入处理，这里需要区分 babel@ 版本，不同的版本配置方式存在差异。

   ```js
   // 如果是 babel@6 版本，可以将 import.options 配置为一个数组：
   [
     {
       "libraryName": "antd",
       "libraryDirectory": "lib",
       "style": true
     },
     {
       "libraryName": "antd-mobile"
     },
   ]
   
   // 如果是 babel@7+ 版本，可以配置多个 `import` 插件实例：
   {
     "plugins": [
       ["import", { "libraryName": "antd", "libraryDirectory": "lib"}, "antd"],
       ["import", { "libraryName": "antd-mobile", "libraryDirectory": "lib"}, "antd-mobile"]
     ]
   }
   ```

## 2023年了，我们还需要 `babel-plugin-import` 吗？

参考文章：

- [如何让npm包支持tree-shaking](https://juejin.cn/post/6847902223653928967)
- [Webpack Tree shaking 深入探究](https://juejin.cn/post/6844903687412776974)
- [Webpack 原理系列九：Tree-Shaking 实现原理](https://juejin.cn/post/7002410645316436004)
- [这样做，你的组件也可以支持tree-shaking](https://juejin.cn/post/7045172831272828936)
- 🔥 [Tree-Shaking性能优化实践 - 原理篇](https://juejin.cn/post/6844903544756109319)

我们已经了解了 `babel-plugin-import` 的相关知识以及其背后的原理，并且回答了我们最初提出的三个问题：

✅ `babel-plugin-import` 是什么？

✅ `babel-plugin-import` 在组件库开发中有什么作用？

✅ `babel-plugin-import` 它是怎么工作的 (工作流程)？

我们接下来将从本节开始讨论后三个问题：

- 现在的组件库开发还需要 `babel-plugin-import` 吗？
- 组件库的 CSS 样式方案有哪些？`babel-plugin-import` 属于哪种样式解决方案？
- 在非 `babel` 环境中调用包含 `babel-plugin-import` 的组件库，会报错吗？

我们在前文曾提及，在 `antd 5.x` 中，已经没有了 `babel-plugin-import` 的身影，我们已知 `babel-plugin-import` 实现了按需导入和样式关联的功能，那么 `antd 5.x` 为什么要舍弃它呢？其中一个很重要的原因就是：`Tree-shaking` 技术的成熟以及 `css-in-js` 技术的出现。

**👉 引用自 [babel-plugin-import 使用](https://juejin.cn/post/7051206427402043423):** 早期（没有 `tree shaking` 的时代）为了实现按需引入功能，我们会通过 `babel-plugin-import` 来优化我们的项目打包体积，做到只打包我们项目中所用到的模块。但在现在新版的 antd 和 material-ui 中，默认已支持基于 ES modules 的 `tree shaking` 功能；而打包工具如：Webpack、Rollup 等在打包层面也支持了 `tree shaking`，使得我们不需要额外配置 `babel-plugin-import` 也能实现按需引入，这得益于 `tree shaking`。

### 🔥 What is Tree-Shaking ?

`Tree-Shaking` 是一种**基于 `ES Module` 规范**的 `Dead Code Elimination (DCE)` 技术，它会在项目运行的编译阶段静态分析模块之间的导入导出，确定 `ESM` 模块中哪些导出值未曾其它模块使用，并将其删除，以此实现打包产物的优化。

以 webpack 项目为例，具体来说，在 webpack 项目中，有一个入口文件，相当于一棵树的主干，入口文件有很多依赖的模块，相当于树枝。实际情况中，虽然依赖了某个模块，但其实只使用其中的某些功能。通过 tree-shaking，将没有使用的模块摇掉，这样来达到删除无用代码的目的。

Tree Shaking 较早前由 Rich Harris 在 Rollup 中率先实现，Webpack 自 2.0 版本开始接入，至今已经成为一种应用广泛的性能优化手段。

---

**❇️ `Tree-Shaking` 本质:** 消除无用的js代码。

无用代码消除广泛存在于传统的编程语言编译器中，编译器可以判断出某些代码根本不影响输出，然后消除这些代码，这个称之为`DCE(dead code elimination)`。`Tree-shaking` 是 DCE 的一种新的实现，JS 同传统的编程语言不同的是，JS 绝大多数情况需要通过网络进行加载，然后执行，**加载的文件大小越小，整体执行时间更短，所以去除无用代码以减少文件体积，对 JS 项目来说更有意义**。

`Tree-shaking` 和传统的 `DCE` 的方法处理对象和目的并不一样：传统的 `DCE` 消灭**不可能执行**的代码，而 `Tree-shaking` 更注重**消除没有用到的 JS 模块**。

---

❇️ **`Tree-Shaking` 的应用基础:**  基于 `ES6` 的模块特性。

`Tree-shaking` 的消除原理是依赖于ES6的模块特性。通过了解 JS 模块的发展史，我们可以知道 `ESM` 模块的特点包括：

- 只能作为模块顶层的语句出现
- import 的模块名只能是字符串常量
- **编译时加载模块**

等。因此ES6模块依赖关系是确定的，和运行时的状态无关，可以进行可靠的静态分析，这就是tree-shaking的基础。所谓静态分析就是**不执行代码，从字面量上对代码进行分析**。更详细的解释就是👇：

ES6之前的模块化，例如 AMD / CJS / UMD 等，都属于**"动态时加载模块"**，只有执行代码中的加载语句后，才能知道引用的什么模块。正因为这些模块化规范的导入导出行为是高度动态，难以预测的，因此不能通过静态分析去做优化。而 ESM 方案则从规范层面规避这一行为，它要求**所有的导入导出语句只能出现在模块顶层，且导入导出的模块名必须为字符串常量**。所以，ESM 下模块之间的依赖关系是高度确定的，**与运行状态无关**，编译工具只需要对 ESM 模块做静态分析，就可以**从代码字面量中推断出哪些模块值未曾被其它模块使用**，这是实现 Tree Shaking 技术的必要条件。

👉 正是基于 **编译时加载** 这个基础，使得 Webpack 等工具能够在项目运行前就确定 `ESM` 模块的静态依赖关系，在此基础上才能实现对无用模块的判断与舍弃操作 (`Tree-Shaking`)。这也是为什么 rollup 和 webpack 2 都要用 `ES6 module syntax` 才能 tree-shaking 的原因。

---

**Webpack Tree-Shaking 的最佳实践**

虽然 Webpack 自 2.x 开始就原生支持 Tree Shaking 功能，但**受限于 JS 的动态特性与模块的复杂性**，直至最新的 5.0 版本依然**没有解决许多代码副作用带来的问题**，使得优化效果并不如 Tree Shaking 原本设想的那么完美，所以需要使用者有意识地**优化代码结构**，或使用一些补丁技术帮助 Webpack 更精确地检测无效代码，完成 Tree Shaking 操作。

因为本节主要是对 `Tree-Shaking` 做初步的了解和介绍，因此不会讲到具体的 `Webpack` 或其他编译工具在 `Tree-Shaking` 上最佳实践的具体方法，想要了解的可阅读以下几篇文章：

- [Webpack 原理系列九：Tree-Shaking 实现原理](https://juejin.cn/post/7002410645316436004)
- [Webpack Tree shaking 深入探究](https://juejin.cn/post/6844903687412776974)

总的来说，`Tree-Shaking` 是一个基于 `ESM` 的性能优化方案，不同的编译工具对于这个方案由自己不同的实现方式，但现阶段而言看似并不能完全适应所有的模块导入导出场景，因此仍需要开发者去调整代码结构，以触发各编译工具各自的 `Tree-Shaking` 检测条件。

### 🌈 如何启用 `Webpack Tree-Shaking` 功能 ？

在 Webpack 中，启动 Tree Shaking 功能必须同时满足三个条件：

- 使用 ESM 规范编写模块代码
- 配置 `optimization.usedExports` 为 `true`，启动标记功能
- 启动代码优化功能，可以通过如下方式实现：
  - 配置 `mode = production`
  - 配置 `optimization.minimize = true`
  - 提供 `optimization.minimizer` 数组

代码示例：

```js
// webpack.config.js
module.exports = {
  entry: "./src/index",
  mode: "production",
  devtool: false,
  optimization: {
    usedExports: true,
  },
};
```



### 🌈 如何让你的 npm 包支持 `Tree-Shaking` ?

关于 `Webpack Tree-Shaking` 的启用是针对实际业务项目而言的，一般现有脚手架下构建的项目通常会默认开启 `Tree-Shaking` 这一功能。而对于组件库 (npm 包) 开发而言，我们需要在第三方库打包编译时，通过配置打包工具的选项，使我们的第三方工具能够支持 `Tree-Shaking` 这一功能。

> 可用用 http 跨域请求简单理解，在 http cors 中，前端需要配置某些配置项以开启跨域请求，而后端也需要配置某些配置项，支持前端的跨域请求操作。同理，在“项目 - 组件库”交互中，项目就相当于“前端”，而组件库则相当于”后端“，它需要通过配置支持项目的一些操作。

在组件库开发中，我们通常需要配置 `package.json` 和打包工具配置项，例如 `webpack.config.js`，来支持 `Tree-Shaking`。

1. 🔥 **`ESM`** : 需要支持 `tree-shaking`，那么你的组件库就必须时基于 `ESM` 编写的，以满足静态分析模块依赖关系这一需求。假如你的代码库如果基于 `ESM` 编写，也输出 `ESM` 文件供外部使用，其实在 js 的部分已经支持了 `tree-shaking`，这是各种构建平台自带的功能。

   但值得注意的是，许多组件库还支持 cjs 模式，在 `package.json` 配置文件中表现为如下形式：

   ```js
   "main": "lib/index.js",
   "module": "es/index.js",
   ```

   打包工具(`Webpack, Rollup`)会优先通过`package.json`来判断一个npm包是否支持 `tree-shaking`：

   > 这里的打包工具指的是业务项目的打包环境，而非组件库。组件库的 `package.json` 配置中的 `main` 或 `module` 字段实际向外提供了访问这个组件库入口文件的路径，业务项目的打包工具会在运行时，访问这些 `package.json` 配置以判断该组件库是否支持 `Tree-Shaking` 这项功能。

   `main` 是 `CJS` 模块的入口，`module` 是 `ESM` 的入口。一般的组件库都会配置这两个入口，同时支持 `CJS` 和 `ESM` 两种模块，具体决定用哪种，其实是和你的源代码使用哪种模块绑定的，你用 `ESM` 语法写代码，那么就会去加载 `module` 入口文件，反之亦然。

   如果只有一个 `main` 入口，那么没有选择，只能使用这个了，当然 `webpack` 这类工具会帮忙进行`cjs->esm`的转换，因此我们写的代码是无感知的(如果你使用过ts，那么ts默认是不会帮你转换的，需要你配置编译选项)，如果你没配置 `module` 且 `mian` 入口是 `CJS` 模块，那么就无法支持 `Tree-Shaking` 这一项功能了。

2. **`sideEffects` 配置**：在组件库 `package.json` 配置中，我们还能经常看到这个选项，其接收一个包含文件路径的数组或一个布尔值。该属性的作用是告诉 `webpack` 等各类打包工具，该属性所包含的目录文件都不可被 `Tree-Shaking` 剔除。若 `sideEffects:false` 则表示所有文件均可被 `Tree-Shaking` 剔除。

   ```js
   // antd package.json
   {
     "sideEffects": [
       "dist/*",
       "es/components/**/style/*",
       "lib/components/**/style/*",
       "*.less"
     ]
   }
   ```

   在组件库`package.json`配置中，通常需要将样式文件等添加到 `sideEffects` 中，防止样式文件被 `Tree-Shaking` 识别为无用模块**(没有任何引用，只有单纯的导入)**而删除，导致组件样式失效的问题。假设有以下场景，组件库中使用了 `babel-plugin-import`，业务项目在访问该组件库组件时，该组件引用被转换为了具体的路径并自动导入了 `*.css` 模块。

   ```js
   // 业务项目中对组件进行引用
   import {Button} from '@arco-design/web-react';
   
   // 实际编译后运行的代码
   import "@arco-design/web-react/es/Button/style/css";
   import _Button from "@arco-design/web-react/es/Button";
   ```

   假设我们所使用的组件库`package.json`没有将`*.css`写入 `sideEffects`中，那么业务项目 `webpack` 在编译阶段，会识别到只有导入但并未被“引用”的样式文件，导致触发 `Tree-Shaking` 将其删除，组件样式失效。所以在组件库开发中，我们要将其写入 `package.json sideEffects`，告诉业务项目的 `webpack` 这些样式文件是不能被删除的。

3. **`Babel`** 配置：由于 `Babel` 通常会注入语法转换的功能 (`babel-loader @babel/preset-env`)，确保代码能够向后兼容，所以实际打包后的代码可能是 `CommonJS` 模块类型，导致无法支持 `Webpack` 等打包工具的 `Tree-Shaking` 功能。因此，我们在模块打包时要确保打包文件不会被编译到低于 `ES6` 的版本环境，从而保证满足 `Tree-Shaking` 最基本的条件。

   > 值得注意的是，最新的 `babel-loader` 中已经默认添加了判断当前环境是否支持 `ESM` 的逻辑代码，因此在实际使用中，`@babel/preset-env` 并不会发生 `ESM` 向 `CJS` 的转换。具体讲解参考[勘误！Babel 会导致 Webpack Tree-shaking 失效？](https://www.bilibili.com/video/BV1oy4y1p7CC/?vd_source=d15754fa27c9a4102615004bd019e951) 这个视频。

### `babel-plugin-import` or `Tree-Shaking` ?

好了，我们现在已经知道 `Webpack` / `rollup` 等编译打包工具实现了 `Tree-Shaking` ，并且组件库对其还做了一定的支持。这也就意味着，现在的组件库自身已经能够支持“按需引入”这一功能了，那么 `babel-plugin-import` 对于组件库而言还是必须的吗？

答案显而易见：**`babel-plugin-import`已不是应用组件库时所必须的了。**

`Tree-Shaking` 相较于 `babel-plugin-import` 有以下优势：

1. 基于 `ECMAScript` 规范基础的概念性性能优化方案，相对于 `babel-plugin-import` 插件而言，基础更稳定。
2. **不需要在项目中额外配置插件 (对于使用方而言):** 在 `babel-plugin-import` 中，按需导入功能需要在业务项目方进行配置，如果项目需要开启按需导入则必须投入额外的配置工作。而 `Tree-Shaking` 则更加方便，按需导入功能统一由组件库自身通过 `package.json` 支持，项目方在使用组件库的时候，不需要额外的配置，`webpack`等工具可以基于组件库 `package.json` 自动判断并完成 `Tree-Shaking` 这一过程。

👉 那么可能就有人会问了，`Tree-Shaking` 不过是完成了 `babel-plugin-import` 中按需导入的功能，`babel-plugin-import` 不是还有自动关联样式的功能吗？

这就涉及到 [组件库 CSS 样式方案](https://juejin.cn/post/7097100515535765534) 等相关知识了，后续将另起一篇讲关于这方面的知识。这里大致说一下：组件库开发其实有很多样式方案，大体上包括了 **样式和逻辑分离** / **样式和逻辑结合** / **样式和逻辑关联** 三大类，国内组件库 (`antd 4.x` 及之前版本；`arco-design` 等) 开发大多采用了 **样式和逻辑分离** 这一方案，也就是所谓的 `js` 与 `css` 分开单独打包，使用时单独引入。

这里以 `arco-design` [官方文档](https://arco.design/react/docs/start)为例：

```js
// 基础使用
// 以 Button 组件为例。

import React from "react";
import ReactDOM from "react-dom";
import { Button } from "@arco-design/web-react";
import "@arco-design/web-react/dist/css/arco.css";

ReactDOM.render(
  <Button type="primary">Hello Arco</Button>,
  document.querySelector("#root")
);
```

可以看到，要引用 `arco-design` 的组件，还需要在项目顶层文件中引入全局样式文件 `import "@arco-design/web-react/dist/css/arco.css";` 才能保证样式正确加载。

当然，若不想引入样式文件，也可以配置 `babel-plugin-import` 并开启 `style:'true'` 实现样式的自动关联。此时导入全局样式文件将不再是必须的。

```js
// 按需加载
@arco-design/web-react 的组件默认支持 tree shaking, 使用 import { Button } from '@arco-design/web-react'; 方式引入即可按需加载。

如果按需加载失效，或者需要样式按需加载以及图标按需加载的可使用以下两种方式处理：

使用 Arco 官方插件
...

使用 babel-plugin-import
- 安装
npm i babel-plugin-import -D

- 添加配置
  组件和样式的按需加载
  - 在 babel 配置中加入：
  plugins: [
    [
      'babel-plugin-import',
      {
        libraryName: '@arco-design/web-react',
        libraryDirectory: 'es',
        camel2DashComponentName: false,
        style: true, // 样式按需加载
      },
    ],
  ];

  Icon 按需加载
  - 在 babel 配置中加入：
  plugins: [
    [
      'babel-plugin-import',
      {
        libraryName: '@arco-design/web-react/icon',
        libraryDirectory: 'react-icon',
        camel2DashComponentName: false,
      },
    ],
  ];
```



### 总结

目前来看，`babel-plugin-import` 的应用场景已经被压缩了，从一开始的按需引入变成了如今主要负责样式的自动关联。实际上，关于样式的自动关联，现在也有了很多解决方案可供替代，例如`antd 5.x` 开始尝试使用 `css-in-js`，通过 js 来编写组件样式，直接抛弃了 `.css` 文件导入这种样式加载的形式。

此外，`babel-plugin-import` 其实也存在一些问题，例如它依赖于 `babel`，本质是对 `import` 语句的转换。假设在 SSR 渲染场景或某些不支持以 `import "xxx.css"` 方式加载样式文件的场景中使用依赖于 `babel-plugin-import` 的组件库，将同样会导致样式无法正确加载。

> 因为依赖于 `babel-plugin-import`的组件库会在打包发布过程中自动添加 `import xxx.css` 语句，其实就相当于采用了 css 与 js 分开打包，但内部逻辑紧密耦合的 CSS 方案了，这就会导致项目在使用组件时，访问到的组件文件代码中包含了 `import xxx.css`， 环境不支持这种 css 识别方式，导致报错。

---

之前的问题可以被概括为以下几点：

❇️ `babel-plugin-import` 是什么？在组件库开发中有什么作用？

babel-plugin-import 是一种组件库(但并非局限于组件库)的按需导入 babel 插件。属于 babel Transform 转换的环节之一。

❇️ `babel-plugin-import` 它是怎么工作的 (工作流程)？

本质上是对 babel AST 树 ImportDeclaration 类型节点语句的重构。以 `import {Button} from "arco-design"` 为例，babel-plugin-import 会在 `@babel/traverser` 访问器遍历到 AST ImportDeclaration 时被调用，判断当前导入包名( `arco-design` )是否命中预设的 `babel-plugin-import options => libraryName` 来决定是否处理该导入语句。若命中则会对其进行转换，将包名指向具体引用组件的路径。若`style:true`则还需要添加 `import xxx.css` 实现样式自动关联。

❇️ 现在的组件库开发还需要 `babel-plugin-import` 吗？

可以用，但不是必须的。因为 `babel-plugin-import` 的作用：1.按需导入 2.样式自动关联。而按需导入这一功能现在可由 `webpack` 等工具启用 `tree-shaking` 实现，组件库只需要支持 `tree-shaking` 就可以了 (这里就涉及到了 `ESM` 相关知识，`tree-shaking` 实现的基础是 `ESM` 模块) 。样式的自动关联其实并非是必需的，像 `antd 4.x` `arco-design` 等一般都会在头文件中直接加载全局的组件库样式文件，`antd 5.x` 则引入了 `css-in-js` 技术进行实践。

❇️ 组件库的 CSS 样式方案有哪些？`babel-plugin-import` 属于哪种样式解决方案？

这一块还没有具体做总结，但是可大致分为三类：

1. css 与 js 分开打包和使用 (例如  `antd 4.x` `arco-design` ) 
2. css 与 js 耦合，只输出 js 文件 (例如 `antd 5.x` 对于 `css-in-js` 的应用) 
3. css 与 js 分开打包，但逻辑关联 (即在组件中直接引入对应的 css)

`babel-plugin-import` 实际是方案1的一种优化手段，其可以自动实现样式关联，而不需要导入全局样式，即所谓的样式也实现按需导入。

❇️ 在非 `babel` 环境中调用包含 `babel-plugin-import` 的组件库，会报错吗？

这个问题其实在提出的时候有一点理解的偏差。`babel-plugin-import` 是业务项目为了按需加载组件库使用的一种手段，其是在业务项目上配置和实现的。现在的 `tree-shaking` 手段则更加注重组件库上的 `package.json` 配置是否支持 `tree-shaking` ，业务项目中的 `webpack` 等工具会判断第三方库是否支持并自动启动 `tree-shaking` 功能，所以不需要做额外配置。

假设组件库内部使用了 `babel-plugin-import` 对开发中所使用的第三方库做了按需加载等转换操作，则可能会出现问题。原因在于：使用了 `babel-plugin-import` 的组件库在打包时会在各组件引入 `import xxx.css` 样式语句，假设业务项目的环境不支持 `import xxx.css` 这种语句的识别，那么整个构建过程就会报错。
