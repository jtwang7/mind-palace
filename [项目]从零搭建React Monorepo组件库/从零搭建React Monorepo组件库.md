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

参考文章：[Creating a React Component Library using Rollup, Typescript, Sass and Storybook](https://blog.harveydelaney.com/creating-your-own-react-component-library/)

项目搭建介绍：

- 项目路径: `packages/aurora-design`
- 打包工具: rollup
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
  "type": "module",
  // esm 模块访问入口
  "module": "dist/es/index.js",
  // cjs 模块访问入口
  "main": "dist/lib/index.js",
  // 声明文件访问入口
  "typings": "dist/es/index.d.ts",
  // 需要发布的文件
  "files": ["dist"],
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
    "jsx": "react",
    "jsxFactory": "React.createElement",
    "jsxFragmentFactory": "React.Fragment",
    "noUnusedParameters": true,
    "noUnusedLocals": true,
    "noImplicitAny": true,
    "target": "es6",
    "lib": ["dom", "es2017"],
    "skipLibCheck": true,
    "stripInternal": true,
    "declaration": true,
    "declarationDir": "dist/es/types"
  },
  "exclude": ["node_modules", "lib", "es"]
}
```

### rollup.config.js