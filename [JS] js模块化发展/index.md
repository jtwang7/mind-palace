# js模块化发展

参考文章：

- [What the heck are CJS, AMD, UMD, and ESM in Javascript?](https://dev.to/iggredible/what-the-heck-are-cjs-amd-umd-and-esm-ikm)
- [聊聊 js 模块化(CommonJS, AMD, UMD, CMD, ES6)](https://juejin.cn/post/7203968787325960229)

## JS 为什么需要模块化？

最早的 JS 并没有导入/导出模块的概念，所有功能均写在一个文件内。这就导致在开发大型项目时，**无法并行开展团队协作和代码管理**。虽然开发者可以将不同功能的模块抽离出来放入不同的 JS 文件，通过引用的方式构建项目，但这可能会引起**命名冲突、污染作用域**等一系列问题。为了解决这些问题，模块化的概念及实现方法应运而生 —— **”隔离并维护 JS 文件自身的作用域“**。

> JS 的模块化发展经历了：`CJS / AMD => UMD => ESM` 这些阶段，目前 `ESM` 是最成熟且最佳的 JS 模块化规范。

## CJS (CommonJS 规范)

❇️ **代码形式**

```js
// 定义模块
var basicNum = 0;
function add(a, b) {
  return a + b;
}
// 在这里写上需要向外暴露的函数、变量
module.exports = { 
  add: add,
  basicNum: basicNum
}
```

```js
// 引入模块
var math = require('./math');
math.add(2, 3); // 5
```

🌈 **特点**

- CJS imports module synchronously.

  >  CJS 采用同步方式导入模块。

- When CJS imports, it will give you a **copy** of the imported object.

  > CJS 模块输出的是一个值的“拷贝”。

- CJS will not work in the browser. It will have to be transpiled and bundled.

  > (因为“同步“模块加载) CJS 模块无法直接用于浏览器环境，它主要使用于 node 环境开发。

❌ **不足**

- 无法实现模块非阻塞并行加载。
- 不适用于浏览器环境。

> CJS 规范的不足归根到底是因为其同步加载模块的特性造成的。

## AMD (Asynchronous Module Definition 规范)

**`AMD:异步模块定义`** => 采用**异步**方式加载模块，模块的加载不影响它后面语句的运行。所有依赖这个模块的语句，都定义在一个**回调函数**中，模块加载完成之后会自动执行所挂载的回调函数。

❇️ **代码形式**

```js
/* 模块定义
 * id(可选): 定义的模块名(可以理解为“本模块被调用时所使用的名称”)。如果没有定义该参数，模块名默认为当前脚本名，如果有该参数，模块名必须是顶级的绝对的。 
 * dependencies(可选): 定义的模块中所依赖的模块数组(可以理解为“本模块中所依赖的其他模块文件”)，依赖模块优先级执行，并且执行结果按照数组中的排序依次以参数的形式传入 factory。
 * factory(必需): 模块初始化时执行的函数或对象(即本模块内部逻辑)。
 */
define(id?, dependencies?, factory);
```

```js
/* 加载模块
 * module: 数组类型，成员为要加载的模块。
 * callback: 模块加载成功之后执行的回调函数，接收模块加载结果作为回调参数。
 */
require([module], callback);
```

🌰 **例子**

```js
// 定义 math.js 模块
define(function() {
  var basicNum = 0;
  function add(a, b) {
    return a + b;
  }
  return {
    add: add,
  	basicNum: basicNum
  }
})
```

```js
// 使用模块
require(['./math'], function (math) {
  math.add(2, 3);
});
```

🌈 **特点**

- AMD imports modules asynchronously.

  >  AMD 采用异步方式导入模块。这与 CJS 完全相反，采取了不同的加载方式。

- AMD is [made for frontend](http://tagneto.blogspot.com/2011/04/on-inventing-js-module-formats-and.html) (when it was proposed) (while CJS backend).

  > 基于 AMD 异步的加载方式，它更适用于前端 (其本身就是为前端模块化加载制定的)，而 CJS 则偏向后端。

- AMD 可以并行非阻塞加载模块 (得益于它的异步加载方式)。

❌ **不足**

- 异步语法不符合通用的模块化思维方式，CJS 同步加载模块的语法更加直观。



## UMD (Universal Module Definition 规范)

**`UMD:通用模块定义`** => 鉴于`AMD` (浏览器优先，异步加载)和`CommonJS` (服务器优先，同步加载)的不同特点，为实现通用，`UMD`对二者做了整合。其本质就是：先判断是否支持 `node` 的模块，支持就使用 `node`；再判断是否支持 `AMD`，支持则使用 `AMD` 的方式加载。

❇️ **代码形式**

```js
(function (window, factory) {
  if (typeof exports === "object") {
    // CommonJS
    module.exports = factory();
  } else if (typeof define === "function" && define.amd) {
    // AMD
    define(factory);
  } else {
    // 浏览器全局定义
    window.eventUtil = factory();
  }
})(this, function () {
  // do something
});
```

🌈 **特点**

- Works on front and back end.

  > 前后端通用的模块化方案

- Unlike CJS or AMD, UMD is more like a pattern to configure several module systems. Check [here](https://github.com/umdjs/umd/) for more patterns.

  > UMD更像是一个模式，其可以基于不同环境配置不同的模块操作方案。

- UMD is usually used as a fallback module when using bundler like Rollup/ Webpack

  > 当使用像Rollup/Webpack这样的打包编译时，UMD通常是作为一个后备模块。
  >
  > (可以理解为，在如今的开发场景中，Webpack 等打包编译已经能应对更丰富的模块化场景，UMD 通常作为一个兜底选项，避免了打包环节失效导致程序无法运行的问题。)

`UMD` 目前通常作为 `ESM` 不工作情况下的备用兜底方案。

## ESM (ECMAScript Module 规范)

**`ESM:ES 模块`** => JS 提出的**标准模块系统实现**。其设计思想是尽量的**静态化**，使得**编译时就能确定模块的依赖关系**，以及输入和输出的变量。

> `CommonJS` 和 `AMD` 模块，都只能在运行时确定这些东西。

❇️ **代码形式**

```js
// 模块定义 & 导出
let basicNum = 0;
function add(a, b) {
  return a + b;
}

export { basicNum, add };
```

```js
// ES6模块引入
import { stat, exists, readFile } from 'fs';
```

🌈 **特点**

- **编译时加载 (静态加载)**： `ES6` 可以在编译时就完成模块加载。不同于 `CJS` 和 `AMD` 的运行时加载方式，`ESM`本质上是在编译阶段加载、解析并运行模块，并将实际代码所引用的实体与预设接口名相互关联映射，代码运行中基于接口关联的引用路径，从各模块调用到对应的实体。其整体的运行效率要比 `CommonJS` 模块的加载方式高。但编译时加载方法也导致了没法直接引用 `ES6` 模块本身，因为它不是对象。

- 便于静态分析以及 **`Tree-Shakeable`**: 得益于`ESM`在编译阶段就构建了模块间的引用依赖，这使得开发人员可以多模块依赖进行提前分析，同时可以发现不必要的模块引入，对其进行剔除。这些都是 `CJS` 和 `AMD` 等运行时加载的模块化方案所做不到的。

- Works in [many modern browsers](https://caniuse.com/#feat=es6-module)

  > `ESM` 规范已被众多现代浏览器所支持

- It has the best of both worlds: CJS-like simple syntax and AMD's async
  
  > `ESM` 模块既有 `CJS` 的简单语法结构，也有 `AMD` 的异步加载能力。其异步加载主要指的是在编译阶段的异步加载和解析，而实际代码执行阶段依然是根据模块被导入的顺序进行序列化并同步执行的。
  >
  > 关于 `ESM` 是否异步请参照 **Is ESM asynchronous** 讨论内容。

### 🔥 Is ESM asynchronous and high operating efficiency?

🧑‍💻**A:** 

ES6 modules aren't asynchronous, at least not as they are in use right now most places. Put simply, unless you're using dynamic imports (see below), each import statement runs to completion before the next statement starts executing, and the process of fetching (and parsing) the module is part of that. They also don't do selective execution of module code like you seem to imply. `import {bar, baz} from './foo.js'` loads, parses, and runs all of './foo.js', then binds the exported entities named 'bar', and 'baz' to those names in the local scope, and only then does the next import statement get evaluated. They do, however, cache the results of this execution and do direct binding, so the above line called from separate files will produce multiple references to the single 'bar' and 'baz' entities.

Now, there is a way to make them asynchronous called 'dynamic import'. In essence, you use `import` as a function in the global scope, which then returns a Promise that resolves to the module you're importing once it's fetched and parsed. However, dynamic import support is somewhat limited right now (IE will never support it, Edge is only going to get it when they finish the switch to Chromium under the hood, and UC Browser, Opera Mini, and a handful of others still don't have it either), so you can't really use them if you want to be truly portable (especially since static imports (the original ES6 import syntax) are only valid at the top level, so you can't conditionally use them if you don't happen to have dynamic import support).

As a result of this, code built around ES6 modules is often slower than equivalent code built on AMD (or a good UMD syntax).

```
--- 核心内容 ---

1. ESM 并非选择性的加载和解析引入的模块方法，相反，在编译阶段，需要加载、解析并运行模块所有的内容。以 `import {bar as bar1, baz} from './foo.js'` 为例，编译阶段需要加载、解析、运行整个 `./foo.js` 模块文件，然后将模块中名为 'bar' 和 'baz' 的导出实体绑定到预设的接口名 'bar1' 和 'baz' 上(所以本质上是引用地址间映射关系的建立过程)，然后才会继续编译下一个导入语句。与运行时加载不同的是，静态加载确实可以缓存模块的执行结果，并做了直接绑定，所以从不同文件中对同一模块方法进行调用时，将产生对模块内'bar'和'baz'实体的多个引用。

2. ESM 的异步需要依靠“动态导入”这一API import() 实现。其需要在全局范围内使用 import() 作为一个函数，然后返回一个 Promise，一旦 import() 所预设的模块有被引入的需要，它就会执行并解析你要导入的模块，并在模块成功解析后调用传递给 Promise 执行预设的回调函数。
围绕ES6模块构建的代码往往比建立在AMD（或一个好的UMD语法）上的同等代码要慢。
```

🧑‍💻**B:** 

Hi A, thanks for the reply! I appreciate you taking a lot of time to write this.

1. According to [this link](https://exploringjs.com/es6/ch_modules.html#_moduleshttps://exploringjs.com/es6/ch_modules.html#_modules), it says "ECMAScript 6 gives you the best of both worlds: The synchronous syntax of Node.js plus the asynchronous loading of AMD. ", and [this article](https://blog.logrocket.com/how-to-use-ecmascript-modules-with-node-js/) also says that " ESM is asynchronously loaded, while CommonJS is synchronous."
2. Regarding ESM speed, what I meant to say is that ESM creates static module structure ([source](https://exploringjs.com/es6/ch_modules.html#_benefit-faster-lookup-of-imports), [source](https://dev.to/bennypowers/you-should-be-using-esm-kn3)), allowing bundlers to remove unnecessary code. If we remove unnecessary codes using bundlers like webpack/ rollup, wouldn't this allow the shipping of less codes, and if we ship less code, we get faster load time? (btw, just reread the article, I definitely didn't mention rollup usage. Will revise that).

There is a good chance I am wrong (still learning about JS modules) or interpreted what I read incorrectly (also likely, happened before), but based on what I've read, ESM is async and ESM in overall is faster because it removes unnecessary code. I really appreciate your comment - it forced me to look up more stuff and do more research!

```
--- 核心内容 ---

ESM创建了静态的模块结构，允许 Webpack/rollup 等编译器基于其 Tree-shakeable 的特点删除不必要的代码。如果我们使用webpack/rollup 等提前移除了不必要的代码，这不就可以运送更少的代码，如果我们运送更少的代码，我们就可以获得更快的加载时间。总之，ESM 更高效的原因主要体现在其可以预先根据静态结构，剔除冗余的模块依赖引入。
```

🧑‍💻**A:** 

Digging a bit further myself, I think I know why I misunderstood the sync/async point. Put concretely based on looking further at the ES6 spec, the Node.js implementation of CJS, and the code for Require.js and Alameda):

1. CJS executes imports as it finds them, blocking until they finish.
2. ESM waits to execute any code in a module until all of it's imports have been loaded and parsed, then does the binding/side-effects stuff in the relative order that they happen.
3. AMD also waits to run module code until it's dependencies are loaded and parsed, but it runs each dependency as it's loaded in the order in which they finish loading, instead of the order they're listed in the file.

So, in a way, we're kind of both right. The loading and parsing for ESM modules is indeed asynchronous, but the execution of the code in them is synchronous and serialized based on the order they are imported, while for AMD, even the execution of the code in the modules is asynchronous and based solely on the order they are loaded.

That actually explains why the web app I recently converted from uisng Alameda to ESM took an almost 80% hit to load times, the dependency tree happened to be such that that async execution provided by AMD modules actually significantly cut down on wait times.

```
--- 核心内容 ---

1. CJS 在发现模块导入时执行它们，阻塞直到它们完成。
2. ESM 等待执行模块中的任何代码，直到它的所有导入都被加载和解析，然后按照它们发生的相对顺序进行绑定/副作用的处理。
3. AMD 也等待运行模块代码，直到它的依赖关系被加载和解析，但它在每个依赖关系被加载时，按照它们完成加载的顺序运行，而不是按照它们在文件中列出的顺序。这也意味着，模块加载并不会阻塞实际的代码运行。

所以，在某种程度上，我们都是对的。ESM模块的加载和解析确实是异步的，但其中代码的执行是同步的，并根据它们被导入的顺序进行序列化，而对于AMD来说，即使是模块中代码的执行也是异步的，完全基于它们被加载的顺序。
```

❇️ **核心讨论内容：ESM 是否是异步的 / 高效的?**

综合上述两者的讨论，可以提取出以下几点关键信息：

1. `ESM` 在编译和解析阶段是异步执行的，但是在代码运行阶段依然是同步的。但 `ESM` 提供了在运行时异步加载模块的 **`API: import()`**，它允许 `ESM` 同 `AMD` 类似，在运行到模块加载命令处再异步加载模块，并通过回调方式执行后续依赖于该模块的代码。
2. `ESM` 高效主要体现在其 `Tree-shakeable` 的特性上，依据静态结构剔除冗余的模块依赖引入，从而实现更快的加载。
3. ESM 并非选择性的加载和解析引入的模块方法，它仍需要完整执行整个模块，但其通过“接口-实体”映射方式，缓存了模块的执行结果，不同文件对同一模块实体进行调用时，将产生对模块内指定实体的多个引用，而不会多次运行模块。从这一角度分析，它相比于 `CJS` 和 `AMD` 而言是高效的。

## 对比

### AMD vs CJS

- 对于依赖的模块，`AMD` 是 **提前执行**，`CMD` 是 **延迟执行**。
- `AMD` 推崇 **依赖前置**，`CMD` 推崇 **依赖就近**。
- `AMD` 的 API 默认是一个当多个用，`CMD` 的 API 严格区分，推崇职责单一。

### ESM vs CJS

- `CommonJS` 模块输出的是一个 **值的拷贝**，`ES6` 模块输出的是 **值的引用**。
- `CommonJS` 模块是运行时加载，`ES6` 模块是编译时输出接口。
- `CommonJS` 模块的 `require()` 是 **同步加载** 模块，`ES6` 模块的 `import` 命令是 **异步加载**，有一个独立的模块依赖的解析阶段。
