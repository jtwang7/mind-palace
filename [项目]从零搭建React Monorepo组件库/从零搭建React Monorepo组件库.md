# 从零搭建 React Monorepo 组件库

## 目录

- 基于 pnpm 的 Monorepo 环境搭建
  - 初始化 pnpm workspace
  - 创建 Monorepo 项目目录结构
  - 附: Monorepo 中 pnpm 的常见命令
- 搭建 React 组件库

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

进入 `packages/aurora-design` 项目路径进行开发。

### package.json

package.json 描述了项目的具体信息，脚本配置，项目依赖等。有以下几项关键配置：

- name: 项目发布在 npm 上的包名
- version: 版本号
- type: 文件默认的模块类型 (commonjs / module)
- module: ESM 模块下的项目访问入口文件
- main: CJS 模块下的项目访问入口文件
- typings: 类型声明文件的访问入口文件
- files: npm 发包导出

```json
{
  "name": "aurora-design",
  "version": "1.0.0",
  "description": "Aurora Design React UI Library.",
  "type": "module",
  "module": "dist/es/index.js",
  "main": "dist/lib/index.js",
  "typings": "dist/es/index.d.ts",
  "files": [
    "dist",
    "es",
    "lib"
  ],
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
  "peerDependencies": {
    "react": ">=16",
    "react-dom": ">=16"
  },
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
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "rollup": "^3.21.5",
    "rollup-plugin-css-only": "^4.3.0",
    "rollup-plugin-peer-deps-external": "^2.2.4",
    "rollup-plugin-postcss": "^4.0.2",
    "rollup-plugin-typescript2": "^0.34.1",
    "storybook": "^7.0.9"
  },
  "sideEffects": [
    "dist/*",
    "es/**/style/*",
    "lib/**/style/*",
    "*.less"
  ]
}
```

### tsconfig.json

### rollup.config.js