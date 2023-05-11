# What is babel?

参考文章：

**原理相关**

- 🔥 [深入浅出 Babel 上篇：架构和原理 + 实战](https://juejin.cn/post/6844903956905197576)
- [面试官(7): 聊一聊 Babel?](https://juejin.cn/post/6844903849442934798#heading-12)
- [babel 官网](https://babeljs.io/)

**应用相关**

- [一口(很长的)气了解 babel](https://juejin.cn/post/6844903743121522701)
- [「前端基建」带你在Babel的世界中畅游](https://juejin.cn/post/7025237833543581732)
- [不容错过的 Babel7 知识](https://juejin.cn/collection/7217097071310929980)

## 简介

进入 [babel 官网](https://babeljs.io/)，映入眼帘的就是一行大字：**“Babel is a JavaScript compiler.”** 没错，这就是 babel —— 一个 JS 编译器。进入 [官方文档](https://babeljs.io/docs/) 我们可以读到官方对其作出的定义：

Babel is a toolchain that is mainly used to convert ECMAScript 2015+ code into a backwards compatible version of JavaScript in current and older browsers or environments. Here are the main things Babel can do for you:

- Transform syntax

- Polyfill features that are missing in your target environment (through a third-party polyfill such as [core-js](https://github.com/zloirock/core-js))
- Source code transformations (codemods)
- And more! (check out these [videos](https://babeljs.io/videos) for inspiration)

我又尝试咨询了 `chatGPT` ，其给出了以下答案：

- Babel是一种用于将 ECMAScript 2015+（ES6及更高版本）的代码转换为**向后兼容**的 JavaScript 版本的 **JavaScript 编译器**。由于不同的浏览器和环境对 ES6+ 的支持程度不同，使用 Babel 可以使我们编写的最新版本的 JavaScript 代码得以在更多的浏览器和环境上运行。

  > Babel可以用作独立的命令行工具，也可以与构建工具（例如Webpack、Rollup）和开发框架（例如React、Angular）等工具配合使用，以便更容易地将 ES6+ 的代码转换为低版本的 JavaScript 代码。
  > 兼容不同版本的 ECMAScript 规范。采用统一转为低版本规范的代码实现。

- Babel不仅可以转换ES6+的语法，还**支持一些新的语言特性**，比如 jsx（一种 JavaScript 的扩展语法，常用于React前端开发）和 TypeScript（一种静态类型的 JavaScript 超集）。Babel还支持编写自定义插件（Plugin），可以对代码进行更多的自定义转换，以便满足项目特定的需求。

  > 除了转译 ECMAScript 规范外，babel 还可以利用插件转换一些新的语言特性。

通过上面的解释，目前对 `babel` 有了一个大概的认知：它是一种 JS 编译器，服务的语法对象 (例如 jsx, typescript 等) 很广，但是最终都会被转换为 JS 语法。它最常用的应用是实现 ECMAScript 规范的向后兼容，通过将高版本规范的语法类型转换为低版本规范的语法类型，使代码能够兼容更多的低版本环境。


## 🔥 babel 的工作流程

了解完 `babel` 的定义后，我们可能会有一个疑问：`babel`是如何实现语法转换的？那么我们就接着来讲下它的工作原理。

> `babel` 的工作流程图在 [深入浅出 Babel 上篇：架构和原理 + 实战](https://juejin.cn/post/6844903956905197576) 中已有展示，这边就不再挂出了，本文主要是对各类学习资料的学习和整理总结，如有兴趣，可以点击🔗链接深入了解下各篇好文。

`babel` 的工作流程大致可以归纳为以下步骤：

```text
词法解析(Lexical Analysis) => 语法解析(Syntactic Analysis) => 遍历/访问器模式(Traverser) => 转换(Transform) => 生成(Generator)
```

1. ❇️ **解析(Parsing)**

   `babel` 对 js 进行语法转换或其他编译操作，首先就要“读懂”代码具体做什么，`Parse` 这一过程其实就是 `babel` 读取代码的过程。

   - **词法解析 (Lexical Analysis):** `词法解析器(Tokenizer)`在这个阶段将字符串形式的代码转换为`Tokens(令牌)`. Tokens 可以视作是一些语法片段组成的数组，其中每个 `Token` 中包含了语法片段、位置信息、以及一些类型信息，这些信息有助于后续的语法分析。例如`for (const item of items) {}` 词法解析后的结果如下:

     ```text
     - tokens: [
     	- Token (for) {
     		type: {label, keyword, beforeExpr, ...}
     		value: "for"
     		start: 0
     		end: 3
     		loc: {start, end}
     	}
     	+ Token (() {type, value, start, end, loc}
     	+ Token (const) {type, value, start, end, loc}
     	+ Token (name) {type, value, start, end, loc}
     	+ Token (name) {type, value, start, end, loc}
     	+ Token (name) {type, value, start, end, loc}
     	+ Token ()) {type, value, start, end, loc}
     	+ Token ({) {type, value, start, end, loc}
     	+ Token (}) {type, value, start, end, loc}
     	+ Token (eof) {type, value, start, end, loc}
     ]
     ```

   - **语法解析(Syntactic Analysis)**：在这个阶段语法 `解析器(Parser)` 会把 `Tokens` 转换为 `抽象语法树(Abstract Syntax Tree，AST)`，与词法解析得到的语法片段 Token 数组不同的是，`AST` 是一颗**“对象树”**，其能够更清晰地表示代码的语法结构，例如`console.log('hello world')`会解析成为:

     ```text
     - File {
     	- program: Program {
           sourceType: "module"
         - body: [
           - ExpressionStatement {
             - expression: CallExpression {
               - callee: MemberExpression {
                 - object: Identifier {
                   name: "console"
                 }
                 - property: Identifier {
                   name: "log"
                 }
                 computed: false
               }
               - arguments: [
                 - StringLiteral {
                   + extra: {rawValue, raw}
                     value: "hello world"
                 }
               ]
             }
           }
         ]
         directives: []
     	}
     	comments: []
     }
     ```

     `Program`、`CallExpression`、`Identifier` 都表示的是**节点的类型**，每个节点都是一个**有意义的语法单元**。 这些节点类型定义了一些属性来描述节点的信息。在 `babel` 中其实除了支持最新的JavaScript规范语法, 还支持 `JSX`、`Flow`、`Typescript`等。因此 AST 的节点类型实际是非常丰富的，我们不需要去记住这么多类型、也记不住，**插件开发者会利用 [`ASTExplorer`](https://link.juejin.cn?target=https%3A%2F%2Fastexplorer.net) 来审查解析后的AST树**。

     > 我们只需要知道一点：**AST 能够清晰表示代码间的层级关系依赖于它的对象结构以及丰富的对象节点语义表达，AST 是 Babel 转译的核心数据结构，后续的操作都依赖于 AST。**

2. **重构**

   转换阶段就是 `babel` 应用一系列方法对 `AST` 进行重构的过程。但值得一提的是：**`babel` 自身没有转换这一功能，这些功能都是依赖于 babel 的 `plugin` 实现的**，这些 `plugin` 既包括了官方的一些产出，也有外部开发人员的贡献。**`babel` 自身只提供了解析以及还原的过程**，其暴露的 `AST` 结构树才是转换的关键，这种**微内核架构风格**向贡献者提供了最大的操作自由度。

   >  没有应用`plugin`的`babel`输入和输出是相同的。

   - ❇️ **访问者模式 `Traverser` (遍历):** 

     `plugin` 操作的对象单元是 `AST` 树的各个节点。但是想象一下，Babel 有那么多插件，如果每个插件自己去遍历AST，对不同的节点进行不同的操作，维护自己的状态。这样子不仅低效，它们的逻辑分散在各处，会让整个系统变得难以理解和调试， 最后插件之间关系就纠缠不清，乱成一锅粥。**所以转换器操作 AST 一般都是使用`访问器模式`，由这个`访问者(Visitor)`来 ① 进行统一的遍历操作，② 提供节点的操作方法，③ 响应式维护节点之间的关系；而插件(设计模式中称为‘具体访问者’)只需要定义自己感兴趣的节点类型，当访问者访问到对应节点时，就调用插件的访问(visit)方法**。

     **`babel Visitor`遍历器:** 在 `babel` 中，遍历 `AST` 节点的事统一交由 `babel` 自身实现 (`@babel/traverse`)，访问者会以`深度优先`的顺序, 或者说递归地对 `AST` 进行遍历，其代码表示如下：

     ```js
     const babel = require('@babel/core')
     const traverse = require('@babel/traverse').default
     
     const ast = babel.parseSync(code)
     
     let depth = 0
     traverse(ast, {
       enter(path) {
         console.log(`enter ${path.type}(${path.key})`)
         depth++
       },
       exit(path) {
         depth--
         console.log(`  exit ${path.type}(${path.key})`)
       }
     })
     ```

   - **`Transform` (转换):** 

     我们了解了 `babel` 是如何遍历 `AST` 树之后，我们就可以利用 `plugin` 对兴趣节点进行转换了。
     
     首先，我们要知道，当访问者进入一个节点时都会调用 `enter(进入)` 方法，反之离开该节点时会调用 `exit(离开)` 方法。 一般情况下，插件不会直接使用`enter`方法，只会关注少数几个节点类型，所以访问者也可以这样声明访问方法:
     
     ```js
     traverse(ast, {
       // 访问标识符
       Identifier(path) {
         console.log(`enter Identifier`)
       },
       // 访问调用表达式
       CallExpression(path) {
         console.log(`enter CallExpression`)
       },
       // 上面是enter的简写，如果要处理exit，也可以这样
       // 二元操作符
       BinaryExpression: {
         enter(path) {},
         exit(path) {},
       },
       // 更高级的, 使用同一个方法访问多种类型的节点
       "ExportNamedDeclaration|Flow"(path) {}
     })
     ```

     **上下文信息：** 访问者在访问一个节点时, 会无差别地调用 `enter` 方法，为了获取当前访问节点的上下文信息 (与其他节点的关联关系)，`babel` 在各个访问节点中都注入了其的位置信息以及与其他节点间的关联关系。

     从上述代码中可知，每个`visit`方法都接收一个 **`Path` 对象**, 你可以将它当做一个‘上下文’对象，这里面包含了很多信息：
     
     - 当前节点信息
     
     - 节点的关联信息: 父节点、子节点、兄弟节点等等
     - 作用域信息
     - 上下文信息
     - 节点操作方法: 增删查改
     - 断言方法
     
     下面是它的主要结构:
     
     ```js
     export class NodePath<T = Node> {
         constructor(hub: Hub, parent: Node);
         parent: Node;
         hub: Hub;
         contexts: TraversalContext[];
         data: object;
         shouldSkip: boolean;
         shouldStop: boolean;
         removed: boolean;
         state: any;
         opts: object;
         skipKeys: object;
         parentPath: NodePath;
         context: TraversalContext;
         container: object | object[];
         listKey: string; // 如果节点在一个数组中，这个就是节点数组的键
         inList: boolean;
         parentKey: string;
         key: string | number; // 节点所在的键或索引
         node: T;  // 🔴 当前节点
         scope: Scope; // 🔴当前节点所在的作用域
         type: T extends undefined | null ? string | null : string; // 🔴节点类型
         typeAnnotation: object;
         // ... 还有很多方法，实现增删查改
     }
     ```

     ❇️ **`plugin` 插件本质上就是通过访问这些 `path` 上下文对象重构 `AST` 结构的。** `plugin` 会在自身实现中写入一层逻辑判断，将当前 `path` 对象中的属性标识与自身期望处理的节点类型进行比对，若符合则调用后续插件逻辑对其 `AST` 节点进行转换，若不符合则跳过。
     
     > 你可以通过这个[手册](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fjamiebuilds%2Fbabel-handbook%2Fblob%2Fmaster%2Ftranslations%2Fzh-Hans%2Fplugin-handbook.md%23toc-visitors)来学习怎么通过 Path 来转换 AST.
     
     > 这节并没有详细介绍 `babel` 插件具体的使用方法以及以何种顺序被 `babel` 调用的，这将在后文单独起一小节讲述。
     
     ---
     
     👉 **`Transform` 过程中需要注意的一些问题**
     
     - **`Transform` 副作用:**
     
       实际上访问者的工作比我们想象的要复杂的多，上面示范的是静态 `AST` 的遍历过程。而 `AST` 转换本身是有副作用的，比如插件将旧的节点替换了，那么访问者就没有必要再向下访问旧节点了，而是继续访问新的节点。
     
       由于插件可以对 `AST` 进行任意的操作，比如删除父节点的兄弟节点、删除第一个子节点、新增兄弟节点...，因此这些操作极易污染 `AST `树的原本结构。 **当这些操作'污染'了 AST 树后，访问者需要记录这些状态，响应式(Reactive)更新 Path 对象的关联关系, 保证正确的遍历顺序，从而获得正确的转译结果**。
     
     - **`Transform` 作用于问题:**
     
       对于转换器来说，另一个比较棘手的是对作用域的处理，这个责任落在了插件开发者的头上。插件开发者必须非常谨慎地处理作用域，不能破坏现有代码的执行逻辑。**AST 转换的前提是保证程序的正确性**。 我们在添加和修改`引用`时，需要确保与现有的所有引用不冲突。Babel本身不能检测这类异常，只能依靠插件开发者谨慎处理。
     
     

3. **Javascript In Javascript Out:**  `babel`最后阶段还是要把 `AST` 转换回字符串形式的 Javascript，同时这个阶段还会生成 Source Map。



## 🔥 babel 架构

`Babel` 为了适应复杂的定制需求和频繁的功能变化，使用了[微内核](https://juejin.cn/post/6844903943068205064#heading-10) 的架构风格。

> **微内核：核心非常小，大部分功能都是通过插件扩展实现的**。

`Babel` 是一个 [`MonoRepo`](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Flerna%2Flerna) 项目，其包括了：**核心** \ **核心周边支撑** \ **插件** \ 插件开发辅助 \ 工具。

> Monorepo 是一种软件开发术语，指的是将所有相关项目都放置在一个版本库（repository）中，而不是拆分成多个独立的版本库。在 Monorepo 中，每个项目以文件夹的形式存在，并且使用共同的构建脚本、工具链和依赖项。这种方法旨在帮助更好地管理大型项目和团队的代码库，提高开发效率和代码复用性。
>
> Monorepo 意味着你的所有代码都存储在一个单一的版本库中，每个包或项目使用相同的版本管理系统。它为团队共享代码和更好的代码共享提供了更好的途径。这也可以帮助通过减少跨库API变更的需要来减少额外维护开销。
>
> 另外，Monorepo 还可以促进项目的横向拓展，让开发人员更容易地在现有的应用中添加新的功能，而无需创建一个全新的应用。这种方法可以更加灵活地管理和组织代码，并提高开发实践的一致性和可重用性。
>
> Monorepo 的一个例子是 Google 的代码库，他们将数百万行的代码都存储在一个单一的版本库中，这样可以更轻松地共享代码和跨团队开发，提高整个团队和项目的代码质量。然而，Monorepo 也需要一些额外的配置和管理成本，因此需要考虑组织大小、项目复杂性等多个因素，以确定使用 Monorepo 是否是合适的选择。

❇️ **核心**

`@babel/core` 这也是上面说的‘微内核’架构中的‘内核’。对于Babel来说，这个内核主要干这些事情：

- 加载和处理配置(config)
- 加载插件
- 调用 `Parser` 进行语法解析，生成 `AST`
- 调用 `Traverser` 遍历AST，并使用`访问者模式`应用'插件'对 AST 进行转换
- 生成代码，包括SourceMap转换和源代码生成

❇️ **核心周边支撑**

- **Parser(`@babel/parser`)**： 将源代码解析为 AST 就靠它了。 它已经内置支持很多语法. 例如 JSX、Typescript、Flow、以及最新的ECMAScript规范。目前为了执行效率，parser是[不支持扩展的](https://link.juejin.cn?target=https%3A%2F%2Fbabeljs.io%2Fdocs%2Fen%2Fbabel-parser%23faq)，由官方进行维护。如果你要支持自定义语法，可以 fork 它，不过这种场景非常少。

- **Traverser(`@babel/traverse`)**：  实现了`访问者模式`，对 AST 进行遍历，`转换插件`会通过它获取感兴趣的AST节点，对节点继续操作。

- **Generator(`@babel/generator`)**： 将 AST 转换为源代码，支持 SourceMap

❇️ **插件**

打开 Babel 的源代码，会发现有好几种类型的‘插件’。

- **语法插件(`@babel/plugin-syntax-\*`)**：上面说了 `@babel/parser` 已经支持了很多 JavaScript 语法特性，Parser也不支持扩展. **因此`plugin-syntax-\*`实际上只是用于开启或者配置Parser的某个功能特性**。

  一般用户不需要关心这个，Transform 插件里面已经包含了相关的`plugin-syntax-*`插件了。用户也可以通过[`parserOpts`](https://link.juejin.cn?target=https%3A%2F%2Fbabeljs.io%2Fdocs%2Fen%2Foptions%23parseropts)配置项来直接配置 Parser

- **转换插件**： 用于对 AST 进行转换, 实现转换为ES5代码、压缩、功能增强等目的. Babel仓库将转换插件划分为两种(只是命名上的区别)：

  - `@babel/plugin-transform-*`： 普通的转换插件
  - `@babel/plugin-proposal-*`： 还在'提议阶段'(非正式)的语言特性, 目前有[这些](https://link.juejin.cn?target=https%3A%2F%2Fbabeljs.io%2Fdocs%2Fen%2Fnext%2Fplugins%23experimental)

- **预定义集合(`@babel/presets-\*`)**： 插件集合或者分组，主要方便用户对插件进行管理和使用。比如`preset-env`含括所有的标准的最新特性; 再比如`preset-react`含括所有react相关的插件.

**插件开发辅助**

- `@babel/template`： 某些场景直接操作AST太麻烦，就比如我们直接操作DOM一样，所以Babel实现了这么一个简单的模板引擎，可以将字符串代码转换为AST。比如在生成一些辅助代码(helper)时会用到这个库
- `@babel/types`： AST 节点构造器和断言. 插件开发时使用很频繁
- `@babel/helper-*`： 一些辅助器，用于辅助插件开发，例如简化AST操作
- `@babel/helper`： 辅助代码，单纯的语法转换可能无法让代码运行起来，比如低版本浏览器无法识别class关键字，这时候需要添加辅助代码，对class进行模拟。

**工具**

- `@babel/node`： Node.js CLI, 通过它直接运行需要 Babel 处理的JavaScript文件
- `@babel/register`： Patch NodeJs 的require方法，支持导入需要Babel处理的JavaScript模块
- `@babel/cli`： CLI工具



## babel 插件的应用方式及加载顺序

Babel 会按照插件定义的顺序来应用访问方法，在实际配置时，需要按照你预期调用插件的顺序，将其写入如下数组中。比如你注册了多个插件，`babel-core` 最后传递给访问器的数据结构大概长这样：

```js
{
  Identifier: {
    enter: [plugin-xx, plugin-yy,] // 数组形式
  }
}
```

当进入一个节点时，这些插件会按照注册的顺序被执行。大部分插件是不需要开发者关心定义的顺序的，有少数的情况需要稍微注意以下，例如`plugin-proposal-decorators`:

```js
{
  "plugins": [
    "@babel/plugin-proposal-decorators",     // 必须在plugin-proposal-class-properties之前
    "@babel/plugin-proposal-class-properties"
  ]
}
```

所有插件定义的顺序，按照惯例，应该是**新的或者说实验性的插件在前面，老的插件定义在后面**。因为可能需要新的插件将 AST 转换后，老的插件才能识别语法**（向后兼容）**。但是对于预设的插件而言，其执行的顺序相反，原因详见官方[文档](https://link.juejin.cn?target=https%3A%2F%2Fbabeljs.io%2Fdocs%2Fen%2Fnext%2Fplugins%23plugin-ordering)。

> 预设插件`preset plugin` 是什么？为什么执行顺序相反？这些问题不是本章的介绍重点，具体可以参考[一口(很长的)气了解 babel](https://juejin.cn/post/6844903743121522701) \ [「前端基建」带你在Babel的世界中畅游](https://juejin.cn/post/7025237833543581732) \ [不容错过的 Babel7 知识](https://juejin.cn/collection/7217097071310929980)等文章。



## 小结

`babel` 本质上是一个 js 编译器，它采用了 **微内核** 的架构思想，向外部开发人员提供了最大的自由度。通过`@babel/parser`解析代码文件的词法结构生成 `AST` 语法结构树，`@babel/traverse` 遍历这颗 `AST` 树，同时在每个遍历节点暴露了访问方法，开发人员通过向这些访问方法内注入自定义的`plugin`插件，重构 `AST` 节点以达到转换语法等的目的，最终通过 `@babel/generator` 将重构后的 `AST` 逆解析还原为 js 代码。

