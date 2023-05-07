# 从零搭建 React Monorepo 组件库

## 基于 pnpm 的 Monorepo 环境搭建

参考文章:

- [Setup a Monorepo with PNPM workspaces and speed it up with Nx!](https://blog.nrwl.io/setup-a-monorepo-with-pnpm-workspaces-and-speed-it-up-with-nx-bc5d97258a7e)

### 初始化 pnpm workspace

1. 创建一个新的文件夹目录，重命名为 "pnpm-mono"
2. 再改目录下创建顶层的 pacakge.json 文件，它将作为 Monorepo 项目的根 package.json

```javascript
mkdir pnpm-mono
cd pnpm-mono
pnpm init
```

如果你希望将该项目托管到 github 上，那么你需要：

1. 初始化一个 git 仓库
2. 创建一个 ".gitignore" 文件，排除不需要托管的路径

```javascript
git init
```

```text
# .gitignore
node_modules
dist
build
```

### 创建 Monorepo 项目目录结构

在 Monorepo 项目目录下创建 apps ; packages 等多个文件夹存储不同类型的项目。

- apps: 存储应用类型项目
- packages: 存储组件库/包类型

配置 pnpm workspace 以正确识别 Monorepo 工作区。我们必须在版本库的根目录下创建一个 **pnpm-workspace.yaml** 文件，定义我们的 monorepo 结构：

```text
# pnpm-workspace.yaml

packages:
  - 'apps/**'
  - 'packages/**'
```

上述 pnpm-workspace.yaml 文件 packages 字段下定义了 Monorepo 所接管的工作区，即 apps 和 packages 文件夹下的所有目录均为工作区。
至此，我们已经完成了 pnpm + Monorepo 项目的初步搭建，我们可以进入对应的 workspace 下进行项目开发，开发过程同正常开发流程一致。

```text
cd packages/antd
pnpm init
...
```

> pnpm 创建 Monorepo 的关键就是 pnpm-workspace.yaml 文件的配置，我们只需要将正确的项目路径添加到 pnpm workspace (packages 字段) 中即可。

### 附: Monorepo 中 pnpm 的常见命令

由于 Monorepo 目录下托管了多个项目，因此在 Monorepo 根目录下运行 pnpm 命令时，我们需要通过额外的指令明确 pnpm 所要作用的工作区。常见的额外指令如下：

- `-w`: 指令作用于 root ，包会放置在 `<root>/node_modules` 下。
- `-r`: 指令作用于所有托管的 workspace 中。
- `--filter <package.json-name>`: 指定作用的工作区，只作用于 filter 参数之后的 workspace。

> `--filter` 指令优先级最高。

## 搭建 React 组件库

参考文章：

- [Creating a React Component Library using Rollup, Typescript, Sass and Storybook](https://blog.harveydelaney.com/creating-your-own-react-component-library/)
- [Creating a Reusable Component Library with React, Storybook, and Webpack](https://levelup.gitconnected.com/creating-a-reusable-component-library-with-react-storybook-and-webpack-c0a30076aa54)

项目搭建介绍：

- 项目路径: `packages/aurora-design`
- 打包工具: rollup / webpack / ...
- 组件开发工具: storybook
- 主要技术栈: react + react-dom + typescript

### 前期准备

进入 `packages/aurora-design` 目录，运行如下命令：

```js
// 生成 package.json
pnpm init
// 生成 tsconfig.json
tsc --init
// 添加主要依赖
pnpm add react react-dom
pnpm add @types/react -D
pnpm dlx storybook@latest init
```

### package.json

参照 [antd4.x-stable package.json](https://github.com/ant-design/ant-design/blob/4.x-stable/package.json)

```js
{
  "name": "aurora-design",
  "version": "1.0.0",
  "description": "Aurora Design React UI Library.",
  // esm 模块访问入口
  "module": "es/index.js",
  // cjs 模块访问入口
  "main": "lib/index.js",
  // 声明文件访问入口
  "typings": "es/types/index.d.ts",
  // 需要发布的文件
  "files": ["es", "lib"],
  // 脚本
  "scripts": {
    "build": "rimraf dist/* && rollup -c",
    "storybook": "storybook dev -p 6006",
    "build-storybook": "storybook build"
  },
  "keywords": [
    "react",
    "component",
    "ui",
    "aurora",
    "AuroraDesign"
  ],
  "author": "jtwang7",
  "license": "ISC",
  // 🔥使用组件库的宿主通常包含 react 和 react-dom，因此将依赖放在 peerDependencies 中，不安装。
  "peerDependencies": {
    "react": ">=16.8.0",
    "react-dom": ">=16.8.0"
  },
  // 🔥开发环境下的依赖
  "devDependencies": {
    "@babel/preset-env": "^7.21.5",
    "@rollup/plugin-commonjs": "^24.1.0",
    "@rollup/plugin-node-resolve": "^15.0.2",
    "@storybook/addon-essentials": "^7.0.9",
    "@storybook/addon-interactions": "^7.0.9",
    "@storybook/addon-links": "^7.0.9",
    "@storybook/blocks": "^7.0.9",
    "@storybook/react": "^7.0.9",
    "@storybook/react-webpack5": "^7.0.9",
    "@storybook/testing-library": "^0.0.14-next.2",
    "@types/react": "^18.2.6",
    "less": "^4.1.3",
    "prop-types": "^15.8.1",
    "rollup": "^3.21.5",
    "rollup-plugin-css-only": "^4.3.0",
    "rollup-plugin-peer-deps-external": "^2.2.4",
    "rollup-plugin-postcss": "^4.0.2",
    "rollup-plugin-typescript2": "^0.34.1",
    "storybook": "^7.0.9"
  },
  // 🔥tree-shaking忽略项
  // 宿主环境调用组件库时，会 tree-shaking 掉未被真实引用(关联)的代码。
  // 组件库中组件的样式 "import xxx.css" 会被识别为未被引用(导入的内容没有在项目中被显式调用)，因此会被宿主编译器在编译阶段移除。
  // 指定 package.json sideEffects 可以在宿主编译器调用组件库 npm 包、访问 package.json 时，告知编译器禁止 tree-shaking sideEffects 中包含的引用路径，防止样式导入丢失。
  "sideEffects": [
    "dist/*",
    "es/**/style/*",
    "lib/**/style/*",
    "*.less"
  ]
}
```

**注意**

- 通过 "main", "module" 等字段定义不同规范的入口文件位置。
- 通过 "typings" 字段定义声明文件的入口。
- 将 react, react-dom 依赖放在 peerDependencies 中。
- 将打包输出的文件路径放入 files 字段中，跟随 npm 发布。
- 将样式文件路径定义到 sideEffects 中，避免被 tree-shaking。

### tsconfig.json

参照 [antd4.x-stable tsconfig.json](https://github.com/ant-design/ant-design/blob/4.x-stable/tsconfig.json)

```js
{
  "compilerOptions": {
    "baseUrl": "./",
    // 别名配置
    "paths": {
      "aurora": ["components/index.tsx"],
      "aurora/es/*": ["components/*"],
      "aurora/lib/*": ["components/*"]
    },
    "strictNullChecks": true,
    "module": "esnext",
    "moduleResolution": "node",
    "esModuleInterop": true,
    "experimentalDecorators": true,
    // 🔥指定JSX代码生成用于的开发环境
    "jsx": "react",
    "jsxFactory": "React.createElement",
    "jsxFragmentFactory": "React.Fragment",
    "noUnusedParameters": true,
    "noUnusedLocals": true,
    "noImplicitAny": true,
    // 🔥目标语言的版本
    "target": "es6",
    // 🔥ts引用的库
    "lib": ["dom", "esnext"],
    "skipLibCheck": true,
    "stripInternal": true,
    // 🔥生成类型声明
    "declaration": true,
    "declarationDir": "es/types"
  },
  "exclude": ["node_modules", "lib", "es"]
}
```

**注意**

- 开启类型声明导出，并指定 .d.ts 文件的输出路径，确保与 package.json typings 字段定义的类型声明文件入口路径一致。

### rollup.config.js

首先安装 rollup 以及相应的插件：

```js
npm i -D rollup rollup-plugin-typescript2 @rollup/plugin-commonjs @rollup/plugin-node-resolve rollup-plugin-peer-deps-external rollup-plugin-postcss node-sass
```

然后在项目根路径下创建 rollup.config.js 配置文件：

```js
import peerDepsExternal from "rollup-plugin-peer-deps-external";
import resolve from "@rollup/plugin-node-resolve";
import commonjs from "@rollup/plugin-commonjs";
import typescript from "rollup-plugin-typescript2";
import postcss from "rollup-plugin-postcss";
// esm 模块加载 json 文件的方式：
import packageJson from "./package.json" assert {type: 'json'};

export default {
  // rollup 打包的入口文件
  input: "components/index.ts",
  // rollup 输出路径：分别输出 esm 模块和 cjs 模块
  output: [
    {
      file: packageJson.main,
      format: "cjs",
      sourcemap: true
    },
    {
      file: packageJson.module,
      format: "esm",
      sourcemap: true
    }
  ],
  plugins: [
    peerDepsExternal(),
    resolve(),
    commonjs(),
    typescript({ useTsconfigDeclarationDir: true }),
    postcss({
      extract: 'css/index.css'
    })
  ]
};
```

#### 插件解析

- rollup-plugin-peer-deps-external: prevents Rollup from bundling the peer dependencies we've defined in package.json (react and react-dom)
- @rollup/plugin-node-resolve: efficiently bundles third party dependencies we've installed and use in node_modules
- @rollup/plugin-commonjs: enables transpilation into CommonJS (CJS) format
- rollup-plugin-typescript2: transpiles our TypeScript code into JavaScript. This plugin will use all the settings we have set in tsconfig.json. We set "useTsconfigDeclarationDir": true so that it outputs the .d.ts files in the directory specified by in tsconfig.json
- [rollup-plugin-postcss](https://www.npmjs.com/package/rollup-plugin-postcss): transforms our Sass into CSS. In order to get this plugin working with Sass, we've installed node-sass. It also supports CSS Modules, LESS and Stylus.
  - 配置 extract 可以提取 js 文件中的 css 导入，并在指定位置生成 .css 文件。不指定输出目录，则在对应输出的 js 文件路径下生成 .css 文件。

### webpack.config.js

待更新...
