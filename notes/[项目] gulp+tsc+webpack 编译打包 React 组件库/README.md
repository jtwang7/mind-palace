# gulp+tsc+webpack 编译打包 React 组件库

**🚀 [github仓库地址](https://github.com/jtwang7/gulp-ts-webpack-pack.git)**

**💡 仓库中包含:**

- 基于 `Gulp+TypeScript+Webpack` 的构建打包工具
- 基于 `Rollup` 的构建打包工具
- 基于 React App 的测试环境及测试用例

## 编译(构建) & 打包

参考: [前端为什么要工程化？](https://www.cnblogs.com/songyao666/p/14286880.html)

代码编译和打包是两个不同的概念：

- 打包: 将分散的模块依照引用依赖合并到一个文件的过程。
- 编译: 将源代码转换为可在指定开发(生产)环境下运行的代码。

✨ **为什么打包?**

✅ 减少 HTTP 请求数。模块都需要通过`<script>`标签引入到 HTML 文件中，这就意味着每个模块都需要被浏览器单独加载。过多的资源请求容易导致页面载入时间过长。模块打包可以把所有的模块合并到一个或几个文件中，以此来减少 HTTP 请求数。在现在的项目开发过程中，我们通常采用模块化开发，好处在于:

- 避免命名冲突；
- 便于依赖管理；
- 利于性能优化；
- 提高可维护性；
- 提高代码可复用性；

模块化开发能够帮助我们更好地推进合作项目，其优势在于开发环境。但在生产环境中，我们只负责使用它，而不关心其项目的结构构成是否合理，因此需要对散落在各处的模块统一收集并合并为一个文件，对其进行文件压缩等一系列打包优化，减少其在生产环境中的体积占比，提高它的加载效率。

✨ **为什么要编译?**

✅ 前端开发的所有文件最终归属是要交给宿主环境(例如浏览器)去解析、渲染，并将页面呈现给用户，编译(构建)就是将前端开发中的所有源代码转化为宿主环境可以执行的代码，若不进行编译，我们仅凭源代码是无法在宿主环境执行的。通常需要对以下内容进行编译:

- 无法被浏览器直接识别的JS代码，包括ES6/7等符合ECMAScript规范的JS代码以及TS代码等；
- 无法被浏览器直接识别的CSS代码，包括SASS/LESS等预编译的CSS语法；
- 无法被浏览器识别的HTML模板代码，包括jade、ejs、artTemplate、mustache等Node.js模板引擎；

构建其实就是为了弥补浏览器自身的缺陷和不足，是一种面向语言的编译过程。

## 为什么选择 gulp+tsc+webpack ?

### Gulp & Webpack & Rollup

参考：[前端为什么要工程化？](https://www.cnblogs.com/songyao666/p/14286880.html)

现有技术方法中，有许多种【编译/打包】工具(Rollup, Webpack, Gulp 等)供我们选择使用，我们可以选择某种工具单独处理特定的业务场景，也可以通过不同组合方式应对不同的编译打包场景需求。在选择合适的工具前，我们需要简要了解各个工具的特点，当下发展阶段有3款主流的打包工具：gulp，webpack， rollup，以发布时间为顺序简要介绍如下:

- [Gulp](https://www.gulpjs.com.cn/)：基于 Node.js 文件流的构建工具，其遵循代码优于配置的策略，为开发者提供了基于 JavaScript 脚本方式自由管理编译打包流程的能力。
- Webpack: 模块化管理工具和打包工具。通过 loader 的转换，任何形式的资源都可以视作模块，比如【ESM;CJS;UMD】模块、CSS、图片等。它可以将许多松散的模块按照依赖和规则打包成符合生产环境部署的前端资源。还可以将按需加载的模块进行代码分隔(Code Splitting)，等到实际需要的时候再异步加载。
- Rollup：下一代 ES6 模块化工具，最大的亮点是利用 ES6 模块设计，利用 tree-shaking 生成更简洁、更简单的代码。

各构建打包工具有各自的特点，例如:

✨ **Gulp:**
Gulp 更倾向于被归类为工作流管理工具，它本身是不提供具体的功能，所有的构建、打包等功能都要由对应的插件来完成，例如需要编译功能，我们可以选用 gulp-babel, gulp-typescript 等插件，需要打包，我们可以选用 gulp-concat 等。

- 使用 Gulp 可以便于项目各个环节工作流程的控制，基于文件流可以保证我们构建(编译)的产物目录结构与源码保持一致，特别适合组件库等构建场景。
- 但与此同时，极高的自由度也导致它在应对大型项目的构建打包时需要及其庞大和复杂的流程管理，此外，它基于文件流本身的特性也导致其无法识别和处理模块化，无法进行 tree-shaking 或者 code-splitting 等打包优化。

✨ **Webpack:**
Webpack 具备编译和打包的能力，它的优势在于:

1. 能够将各种类型的资源视为模块，并通过 loader 全部编译为 js 代码。在项目中，往往存在多种资源类型，例如 js、ts 文件，css 文件，图片，json 等，Webpack 的特性使它能够为这些资源文件统一建立依赖关系，并打包成一个符合生产环境部署的 js 资源文件。
2. 其还为开发环境提供了 HMR 热更新功能。(3).在打包文件优化方面，采用了 code-splitting 处理按需加载的模块等。

尽管 Webpack 提供了强大的编译和打包功能，但是其仍存在不足: 

1. 打包产物中包含较多 Webpack 注入的代码。
2. Webpack 打包遵循 CommonJS 模块规范而非 ESM 规范，无法支持 tree-shaking。
3. Webpack 配置复杂，开发负担重等原因。

✨ **Rollup**
Rollup 遵循 ESM 模块规范进行设计，天然地支持 tree-shaking 特性。

1. 因此在打包遵循 ESM 规范的源代码时，能够在打包阶段就过滤掉未被使用的代码，减小打包产物的体积。
2. 其能够输出 CJS, ESM, UMD, IIFE 等各种模块规范或格式的产物文件。
3. 通过设定 `output.preserveModules: true`，Rollup 还可支持只输出构建产物，而不输出打包产物。这使得其特别适合于组件库构建的应用场景。具体参考[rollup.js-output.preserveModules](https://www.rollupjs.com/guide/big-list-of-options#outputpreservemodules)。
   > 该选项将使用原始模块名作为文件名，为所有模块创建单独的 chunk。

与 Webpack 相对的，Rollup 自身无法打包多种类型资源，例如图片等静态资源。但是可通过 rollup 插件补足这些功能，参考[文章](https://juejin.cn/post/7210684943252045882)。

🔥 一般而言，对于应用使用 Webpack，对于类库使用 Rollup；需要代码拆分(Code Splitting)，或者很多静态资源需要处理，再或者构建的项目需要引入很多 Commonjs 模块的依赖时，使用 webpack。代码库是基于 ES6 模块，而且希望代码能够被其他人直接使用，使用 Rollup，例如 React 源码库，组件库等。Gulp 常用于处理具有规范项目结构的代码库构建，它能保证输入输出的项目结构一致，此外，丰富的 Gulp 插件也为我们提供了更多构建或打包的选项，更适合定制化构建打包场景需求。

### 组件库打包

比对各种工具的特性，我们发现 Gulp 和 Rollup 在组件库构建打包上具有优势。原因在于:

1. 组件库开发过程中，项目结构规范。Gulp 和 Rollup 能够仅输出编译产物，很好地保留了各组件在项目中的目录结构。
2. 不存在多种类型的静态资源，用不到 Webpack 中丰富的资源转换功能。
3. 为了支持 tree-shaking，构建的产物要遵循 ESM 模块规范，Webpack 输出 ESM 模块规范的打包产物仍在试验阶段，而 Rollup 可对输出产物模块规范进行选择，Gulp 同样能够引入合适的编译插件自由选择产物的模块规范。

因此，Gulp 和 Rollup 更适合组件库的构建和输出，本文后续分别实现了基于 Gulp 和基于 Rollup 的组件库构建，请继续往下阅读。

### Gulp 编译插件与打包插件的选择

我们已知，Gulp 作为一个工作流管理工具，它本身不提供具体的功能，所有的构建、打包等功能都要由对应的插件来完成。在编译插件的选择上，由于我们源码基于 TS 进行开发，因此可以用 TypeScript 的编译器作为源码的编译工具，[gulp-typescript](https://www.npmjs.com/package/gulp-typescript) 插件为我们提供了这一功能。当然，ESBuild 也提供了 TS -> JS 的功能，我们可以采用 [gulp-esbuild](https://www.npmjs.com/package/gulp-esbuild) 插件来完成编译这一步工作。在打包插件上，我们可以选择 [gulp-concat](https://www.npmjs.com/package/gulp-concat) 合并文件流。

✨ **Gulp 的整体工作流如下:**

```js
// 读取文件流 => 编译成 js => 输出构建产物 => 合并文件流 => 输出打包产物
gulp.src() => gulp-typescript => gulp.dest() => gulp-concat => gulp.dest()
```

在许多知名的 React 组件库中，例如 Ant-Design, Arco-Design 等，都采用了 Gulp+TypeScript+Webpack 这一构建打包流程。我们可以学习和借鉴 Ant-Design 的开源构建打包工具 [antd-tools:gulpfile.js](https://github.com/ant-design/antd-tools/blob/master/lib/gulpfile.js#L187) 中的部分代码，实现一套简易的组件库构建打包工具。

注：这里采用 Webpack 是用于输出 UMD 产物，因为其天然输出 CommonJS 模块规范的代码，且具备打包压缩的功能。当然也可以用 Rollup 替换，因为其也支持输出 UMD 规范的打包产物。

## gulp+tsc+webpack 实践

参考:

- [antd-tools:gulpfile.js](https://github.com/ant-design/antd-tools/blob/master/lib/gulpfile.js#L187)
- [webpack+gulp+typescript实现组件库打包](https://juejin.cn/post/7172910457894731789)
- [Gulp](https://www.gulpjs.com.cn/)

🔥 **目标:**

- 导出 UMD/CJS/ESM 三种模块规范的输出产物
- 导出类型定义
- 导出全局 CSS 文件

### 准备工作

- Gulp 初始化，参照[Gulp-快速入门](https://www.gulpjs.com.cn/docs/getting-started/quick-start/)

  ```js
  // 环境构建
  npm install --global gulp-cli
  npm init
  npm install --save-dev gulp

  // 依赖安装(本例中后续需要的依赖)
  npm install --save-dev gulp-typescript // TS -> JS
  npm install --save-dev gulp-babel // JS -> ES6/ES5/...
  npm install --save-dev gulp-less // less -> css
  npm install --save-dev less-plugin-autoprefix // css添加厂商前缀
  npm install --save-dev gulp-concat // 合并文件流
  npm install --save-dev gulp-clean-css // 压缩 css
  npm install --save-dev gulp-rename // 文件重命名
  npm install --save-dev merge2 // 合并操作
  npm install --save-dev rimraf // 删除文件的命令
  ```

- Webpack 初始化，参照[webpack5配置与环境搭建](https://github.com/jtwang7/mind-palace/blob/main/notes/%5B%E9%85%8D%E7%BD%AE%5D%20webpack5%E9%85%8D%E7%BD%AE%E4%B8%8E%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA/README.md)

### 基于 gulp-typescript + gulp-babel 的 TS 编译阶段

新建 `gulpfile.js` 文件，gulp 任务需要定义在这个文件中，其会被 gulp 命令识别并运行。具体 gulp 使用参考 [Gulp 入门文档](https://www.gulpjs.com.cn/docs/getting-started/creating-tasks/)。Gulp 使用并不复杂，其主要就是通过 `gulp.src()`，`gulp.dest()` 等组织和管理文件流，并利用各种插件处理文件流的过程。

我们的目标是利用 gulp-typescript 插件将 TS 转为 JS 并生成相应的 `.d.ts` 类型声明文件，参照 gulp task 最新的编写规范，我们可以写出如下代码:

```js
// gulpfile.js
function compile(modules) {
  // 指定产物输出地址
  const targetDir = modules === true ? esDir : libDir;
  // 同步删除上一次构建的产物
  rimraf.sync(targetDir);

  // 构建 css
  const cssStream = compileCss(modules);

  // 读取 source, 文件流经过 ts 编译, 返回 js 和 dts 文件流
  const stream = gulp.src(source).pipe(tsConfig());
  const dtsStream = stream.dts.pipe(gulp.dest(targetDir));
  const jsStream = (modules ? stream.js : babelify(stream.js)).pipe(
    gulp.dest(targetDir)
  );

  return merge2([jsStream, dtsStream, cssStream].filter((s) => s));
}
```

其中 `params:modules` 定义了输出的模块规范，若 `modules === true` 则输出 ESM 模块，反之则输出 CJS 模块。
我们按照不同模块规范分别指定不同的输出地址 `targetDir`，并且同步删除上一次构建输出的目录。
> 构建 css 的步骤我们先忽略。

进一步的，我们需要指定读取的文件流目录路径，Gulp 支持使用 glob 通配符批量读取符合匹配规则的文件，因此非常适合用于构建例如组件库这类项目结构规范的场景。我们定义需要构建的源码 glob 通配规则如下:

```js
function getProjectPath(name) {
  // process.cwd() 返回 Node.js 进程的当前工作目录。
  return path.resolve(process.cwd(), name);
}

// 不同模块规范的输出目录
const libDir = getProjectPath("lib");
const esDir = getProjectPath("es");

const source = [
  "components/**/*.tsx",
  "components/**/*.ts",
  "!node_modules/**/*.*",
  "!components/**/styles/*.tsx",
];
```

所有的组件开发均放在 components 文件夹下，且遵循以下目录结构:

```js
├─ library
│  ├─ components
│  │  ├─ Input
│  │  │  ├─ index.tsx
│  │  │  └─ styles
│  │  │     ├─ index.less
│  │  │     └─ index.tsx
│  │  ├─ InputTag
│  │  │  ├─ index.tsx
│  │  │  └─ styles
│  │  │     ├─ index.less
│  │  │     └─ index.tsx
│  │  ├─ Tag
│  │  │  ├─ index.tsx
│  │  │  └─ styles
│  │  │     ├─ index.less
│  │  │     └─ index.tsx
│  │  ├─ hooks
│  │  │  └─ useControllableValue.ts
│  │  └─ index.tsx
```

其中各组件的样式均放在各自的 styles 文件夹下，但不在组件内部做相应的引用，后续对 styles 文件夹下的 less 样式做单独的构建打包处理。具体原因参考[React 组件库 CSS 样式方案分析](https://juejin.cn/post/7097100515535765534#heading-7)，我们选择分离 JS 与 CSS，通过全局导入 CSS 的方式加载组件样式。

我们创建完源码示例后，即可通过 `gulp.src(source)` 读取文件流，然后调用 gulp-typescript 插件对其进行编译。gulp-typescript 方法具体使用参考[gulp-typescript:Basic Use](https://www.npmjs.com/package/gulp-typescript#basic-usage) 以及 [gulp-typescript:Using tsconfig.json](https://www.npmjs.com/package/gulp-typescript#using-tsconfigjson)。本例采用 `tsconfig.json` 配置相应的编译选项，并通过 `ts.createProject()` 方法加载配置文件:
> gulp-typescript 中基于 Options 的加载方式，貌似不支持 {compilerOptions, exclude, ...} 这种形式，原因未知。
> tsconfig.json 中不能添加备注，否则构建会报错。

```js
// gulp-typscript using tsconfig.json
// https://www.npmjs.com/package/gulp-typescript#using-tsconfigjson
const tsConfig = ts.createProject("tsconfig.json");
```

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": "./",
    "strictNullChecks": true,
    "module": "esnext",
    "moduleResolution": "node",
    "esModuleInterop": true,
    "experimentalDecorators": true,
    "jsx": "react",
    "jsxFactory": "React.createElement",
    "jsxFragmentFactory": "React.Fragment",
    "noUnusedParameters": false,
    "noUnusedLocals": false,
    "noImplicitAny": true,
    "target": "es6",
    "lib": ["dom", "es2017"],
    "skipLibCheck": true,
    "stripInternal": true,
    "allowSyntheticDefaultImports": true,
    "declaration": true
  },
  "exclude": ["node_modules", "lib", "es"]
}
```

上述 `tsconfig.json` 配置参考了 [antd-tsconfig.json](https://github.com/ant-design/ant-design/blob/master/tsconfig.json)。其中:

- `module` 字段指定了输出的模块规范为 `esnext` (即 ESM 规范)，因此默认编译为 ESM 模块。
- jsx 指定 react，支持 tsx 识别和编译。
- 开启 declaration，在编译阶段生成对应的 `.d.ts` 类型声明文件。此处未指定 declarationDir，类型声明文件将默认输出在对应 ts 文件的同级目录下，并以对应 ts 文件名命名。

现在我们已经完成了 TS -> JS 的编译工作，gulp-typescript 会返回一个 `{js, dts}` 对象，其中:

- `dts` 属性字段包含编译后的 dts 文件流。直接通过 `gulp.dest()` 输出构建的 `.d.ts` 类型声明文件。
  > `gulp.dest()` 输出的文件路径结构默认对应源码项目路径结构。即在指定的目录下按照原有的项目结构排布输出。
- `js` 属性字段包含编译后的 js 文件流。对于 js 文件流，我们需要根据自定义的 `modules` 字段判断是否需要借助 `gulp-babel` 进一步编译 CJS 模块:
  - 若 `modules === true`，则直接输出构建产物。
  - 若 `modules === false`，则调用 `babelify()` 中的 `gulp-babel`，将 JS 进一步编译成低版本代码，再输出构建产物。

关于 `babelify()` 的实现如下:

```js
function babelify(stream) {
  const babelConfig = {
    presets: ["@babel/preset-env", "@babel/preset-react"],
    plugins: ["@babel/transform-runtime"],
  };
  return stream.pipe(babel(babelConfig));
}
```

通过引入一些预设的 loader 和插件，对 JS 代码版本进行转换，返回转换后的文件流。
至此，我们已经大致完成了 TS -> JS 的编译构建工作，接下来将对组件样式进行构建打包。

### 基于 gulp-less + gulp-concat 的 CSS 编译打包阶段

为了分离 CSS 样式和 JS 文件，我们不在组件中通过 `import xxx.css` 的方式引入样式。那么如何正确的抽取样式并打包成一个 css 文件就显得格外重要。Rollup 或 Webpack 抽取和打包 CSS 样式，均需要在模块中显式地引入样式，这就破坏了我们的上述原则。Gulp 可以通过 glob 通配符直接读取对应目录下的 css 文件，不需要 `import xxx.css` 导入，符合我们组件库抽取打包 css 样式的需求。具体代码实现如下:

```js
function compileCss(modules) {
  const dir = path.resolve(modules === true ? esDir : libDir, "./style");
  const stream = gulp
    .src(["components/**/styles/*.less"])
    .pipe(less({ plugins: [autoprefix] }))
    .pipe(gulp.dest(dir))
    .pipe(concat("index.css"))
    .pipe(gulp.dest(dir))
    .pipe(cleanCss())
    .pipe(rename({ suffix: ".min" }))
    .pipe(gulp.dest(dir));
  return stream;
}
```

我们利用 `gulp-less` 编译 less 样式文件，并通过 `less-plugin-autoprefix` 插件添加厂商样式前缀。得到编译好的 css 文件流后，首先输出对应项目目录结构的 css 构建产物。然后利用 `gulp-concat` 合并所有 css 文件到一个 css 文件中并输出。最后通过 `gulp-clean-css` 插件压缩 css 文件，通过 `gulp-rename` 添加文件名后缀 `.min` 后，输出压缩后的 css 文件 `xxx.min.css`。

### 配置 package.json

开发者使用我们的构建产物，需要通过 package.json 配置文件进行访问。因此，我们参照[package.json配置解析](https://github.com/jtwang7/mind-palace/blob/main/notes/%5B%E9%85%8D%E7%BD%AE%5D%20package.json%E9%85%8D%E7%BD%AE%E8%A7%A3%E6%9E%90/README.md)对 package.json 进行配置。本例中配置如下:

```json
{
  "name": "@tian/ui", // 库名
  "version": "1.0.0", // 版本号
  "description": "组件库",
  // 包导出
  "exports": {
    ".": {
      "require": "./lib/index.js", // CJS 模块访问入口
      "import": "./es/index.js" // ESM 模块访问入口
    },
    "./es/style/index.css": "./es/style/index.css" // 通过 [name]/es/style/index.css 访问项目路径下的 "./es/style/index.css"
  },
  "typings": "./es/index.d.ts", // 声明文件入口
  "scripts": {
    "build": "webpack -c webpack.config.js",
    "compile": "gulp compile",
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "devDependencies": {
    "@babel/core": "^7.21.8",
    "@babel/plugin-transform-runtime": "^7.21.4",
    "@babel/preset-env": "^7.21.5",
    "@babel/preset-react": "^7.18.6",
    "@babel/preset-typescript": "^7.21.5",
    "@babel/runtime": "^7.21.5",
    "@types/react": "^18.2.6",
    "@types/react-dom": "^18.2.4",
    "@types/uuid": "^9.0.1",
    "babel-loader": "^9.1.2",
    "core-js": "^3.30.2",
    "gulp": "^4.0.2",
    "gulp-babel": "^8.0.0",
    "gulp-clean-css": "^4.3.0",
    "gulp-concat": "^2.6.1",
    "gulp-less": "^5.0.0",
    "gulp-rename": "^2.0.0",
    "gulp-typescript": "6.0.0-alpha.1",
    "less-plugin-autoprefix": "^2.0.0",
    "merge2": "^1.4.1",
    "rimraf": "^5.0.1",
    "through2": "^4.0.2",
    "typescript": "^5.0.4",
    "webpack": "^5.83.1",
    "webpack-cli": "^5.1.1"
  },
  "dependencies": {
    "classnames": "^2.3.2",
    "uuid": "^9.0.0"
  },
  "peerDependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "sideEffects": [
    "dist/*",
    "es/**/style/*",
    "lib/**/style/*",
    "*.less"
  ]
}
```

## rollup 实践

### 准备工作

✨ **参考:**

- 关于 `rollup.config.js` 各字段配置项可参考: [rollup.js options lists](https://www.rollupjs.com/guide/big-list-of-options#%E6%A0%B8%E5%BF%83%E5%8A%9F%E8%83%BD-core-functionality)
- 关于 `@rollup/plugin-node-resolve` 和 `@rollup/plugin-commonjs` 的应用目的，参考[我如何在使用 CommonJS 模块的 Node.js 中使用 Rollup?](https://www.rollupjs.com/guide/faqs#%E6%88%91%E5%A6%82%E4%BD%95%E5%9C%A8%E4%BD%BF%E7%94%A8-commonjs-%E6%A8%A1%E5%9D%97%E7%9A%84-nodejs-%E4%B8%AD%E4%BD%BF%E7%94%A8-rollup)
- 关于 Rollup 插件库可参考 [rollup.js-awesome](https://github.com/rollup/awesome)

✨ **Rollup 依赖:**

  ```js
  // 环境构建
  npm install --global rollup
  npm init

  // 依赖安装(本例中后续需要的依赖)
  npm install --save-dev @rollup/plugin-node-resolve // 处理第三方依赖导入
  npm install --save-dev @rollup/plugin-commonjs // 处理 commonjs 模块的导入
  npm install --save-dev @rollup/plugin-typescript // 处理 ts
  npm install --save-dev @rollup/plugin-json // 处理 json
  ```

### ESM-rollup.config.js 配置

导出 ESM 模块规范的构建产物的 `rollup.config.js` 配置如下:

```js
// rollup.esm.config.mjs
import path from "node:path";
import { fileURLToPath } from "node:url";
import resolve from "@rollup/plugin-node-resolve";
import commonjs from "@rollup/plugin-commonjs";
import typescript from "@rollup/plugin-typescript";
import json from "@rollup/plugin-json";

const __dirname = path.dirname(fileURLToPath(import.meta.url));

export default {
  input: path.resolve(__dirname, "./components/index.tsx"),
  output: [
    {
      dir: path.resolve(__dirname, "./es"),
      format: "esm",
      sourcemap: true,
      preserveModules: true,
    },
  ],
  plugins: [
    resolve(),
    commonjs(),
    json(),
    typescript({
      compilerOptions: {
        declaration: true,
        declarationDir: path.resolve(__dirname, "./es/library/rollup/types"),
      },
    }),
  ],
  external: ["react", "react-dom"],
};
```

我们定义 `output.format = "esm"`，模块最终会被导出为 ESM 模块类型，`output.preserveModules = true` 设置后将使用原始模块名作为文件名，为所有模块创建单独的 chunk，即 Rollup 可以只输出编译产物，不输出打包产物。`output.preserveModules = true` 需要与 `output.dir` 配合使用，构建产物最终会输出到 `output.dir` 目录路径下。
> 不指定 `output.preserveModules` 时，Rollup 默认输出打包产物，此时需要设置 `output.file` 指定打包文件的输出路径。

此处需要用到 `@rollup/plugin-typescript` 编译 ts 代码，同时指定 `.d.ts` 文件的输出位置。

### CJS-rollup.config.js 配置

导出 CJS 模块规范的构建产物的 `rollup.config.js` 配置如下:

```js
// rollup.commonjs.config.mjs
import path from "node:path";
import { fileURLToPath } from "node:url";
import resolve from "@rollup/plugin-node-resolve";
import commonjs from "@rollup/plugin-commonjs";
import typescript from "@rollup/plugin-typescript";
import json from "@rollup/plugin-json";

const __dirname = path.dirname(fileURLToPath(import.meta.url));

export default {
  input: path.resolve(__dirname, "./components/index.tsx"),
  output: [
    {
      dir: path.resolve(__dirname, "./lib"),
      format: "cjs",
      sourcemap: true,
      preserveModules: true,
    },
  ],
  plugins: [
    resolve(),
    commonjs(),
    json(),
    typescript({
      compilerOptions: {
        declarationDir: path.resolve(__dirname, "./lib/library/rollup/types"),
      },
    }),
  ],
  external: ["react", "react-dom"],
};
```

与 `rollup.esm.config.mjs` 唯一不同点在于 `output.format = "cjs"`。

### UMD-rollup.config.js 配置

导出 UMD 模块规范的构建产物的 `rollup.config.js` 配置如下:

```js
import path from "node:path";
import { fileURLToPath } from "node:url";
import resolve from "@rollup/plugin-node-resolve";
import commonjs from "@rollup/plugin-commonjs";
import typescript from "@rollup/plugin-typescript";
import json from "@rollup/plugin-json";

const __dirname = path.dirname(fileURLToPath(import.meta.url));

export default {
  input: path.resolve(__dirname, "./components/index.tsx"),
  output: [
    {
      file: path.resolve(__dirname, "./dist/index.min.js"),
      format: "umd",
      name: "@tian/ui-rollup", // umd 需要提供 name，将作为全局名称挂在到 window 对象上。
      sourcemap: true,
    },
  ],
  plugins: [
    resolve(),
    commonjs(),
    typescript({ compilerOptions: { declaration: false } }),
    json(),
  ],
};
```

在 UMD 中，我们不需要 TS 的类型声明文件，因此设置 `declaration: false`。此外，我们还需要设置 `output.name = "xxx"`，在 Rollup 中当产物输出格式为 iife 或 umd 时，需要指定 [output.name](https://www.rollupjs.com/guide/big-list-of-options#outputname)，它将作为全局变量名使用。
