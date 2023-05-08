# pnpm + Monorepo 项目环境搭建

参考文章：

- [5W1H 带你入门 Monorepo](https://juejin.cn/post/7207689082184974394)
- [Monorepo 下的模块包设计实践](https://juejin.cn/post/7052271542000074782#heading-11)
- [Element Plus 组件库相关技术揭秘：2. 组件库工程化实战之 Monorepo 架构搭建](https://juejin.cn/post/7146183222425518093#heading-6)
- [使用 pnpm 构建 Monorepo 项目](https://zhuanlan.zhihu.com/p/373935751)

## 什么是 Monorepo ?

首先了解 **Polyrepo (multiple repostories)**  是什么：
Polyrepo 指多 Git 仓库结构的多项目(包)管理方式。各个可以独立运行命令的项目成为一个单独的 git 仓库进行管理。

Polyrepo 的 **缺点** 是什么：

1. 大型项目被割裂成多个 Git 仓库：
   - 需要 git clone 多个仓库，并运行不同的命令来共同跑起完整的服务（典型：前端+后端+ SDK 项目分离）。
2. 代码复用率严重下降：
   - 公共函数或组件只能通过 duplicate 或 npm 方式共享。
   - 当某个公用组件（对应 npm 包）修改时，需要人为确认该 NPM 包所依赖的各个项目升级情况
3. 相同类型项目却有着千奇百怪的开发工具和周边库：
   - jest、java 的版本不一致，导致 API 写法不一
   - React 组件库以及状态管理库不一致，新人接手学习成本提高
   - ESlint 、CommitLint、prettier 配置 和 Changelogs 生成规则不一致
4. 新建项目时有额外心智负担：
   - 如果有需要，可能要复制多套 CI 文件
   - 出于好奇，可能会找更符合当下的 cli 来新建项目（耗时且可能有坑）
   - 完美主义的人可能会配置更多的 eslint、prettier（耗时）
   - 需要在其他项目中 Readme 中添加新项目的 git 地址，在表面上报保持 Connection

🔥 **Monorepo (Monolithic Repository)** 是什么：
Monorepo 指将许多相关但独立的项目统一包含在单个 Git 仓库中进行管理的多项目(包)管理模式。Monorepo 的项目结构通常如下：

```js
// monorepo root
.
├── node_modules
├── package.json
├── packages // monorepo 的项目(包)统一管理路径
│   ├── ui // 独立的子项目
│   │   ├── node_modules
│   │   └── package.json
│   ├── utils
│   └── web
├── pnpm-lock.yaml
├── pnpm-workspace.yaml // pnpm 管理 monorepo 的必要文件 (暂时不关注)
├── readme.md
└── tsconfig.json
```

Monorepo 项目结构特点

- 顶层 (root)：package.json 管理公共的依赖 (dependencies) 以及子项目的一些公用配置文件 (例如 webpack.config.js 等)
- 项目管理路径 (packages)：项目总管理路径，其路径下放置各类项目(包)进行统一管理。
- 子项目：独立的项目/包/应用。有自己的 package.json / node_modules 以及编译打包工具等。

> - packages 文件夹中的就是原本每个独立的项目，现在放在一起用 workspace 去管理。
> - 在 root 路径下的所有配置 (例如 package.json dependencies) 是所有子 package 共用的。

Monorepo 子项目间可独立运行、共享代码，但没有强依赖，剥离出来照样能运行，遵循高内聚，低耦合理念。

## pnpm 搭建 Monorepo 项目环境

> pnpm 是新一代 node 包管理器。这里不做过多的介绍，具体可参考 [为什么现在我更推荐 pnpm 而不是 npm/yarn?](https://link.zhihu.com/?target=https%3A//jishuin.proginn.com/p/763bfbd3bcff)

### 搭建 Monorepo 项目结构

Monorepo 本质是一个多项目管理的概念，具体实现依赖于 npm **workspaces**。

🔥 **workspaces 的作用**

- 开发多个互相依赖的 package 时，workspace 会自动对 package 的 **引用设置软链接(symlink)** ，链接仅局限在当前 workspace 中，不会对整个系统造成影响。
- 所有 package 的 **通用依赖统一管理** ，安装在根目录的 node_modules 下， **节省磁盘空间** 。
- 所有 package 使用同一个 **yarn.lock or package-lock.json or ...** ，**维护同一套版本信息** ，减少因依赖版本造成的冲突且易于审查。

1️⃣ 在 **yarn** 和 **npm** 包管理器中，主要基于 **package.json : [workspaces](https://docs.npmjs.com/cli/v8/configuring-npm/package-json#workspaces)** 这一字段。

```js
{
  "name": "mono-root",
  "version": "1.0.0",
  "private": true, // monorepo 根目录无需发布
  "workspaces": [
    "packages/*" // 社区常见写法，指定工作区包含 packages 路径下的所有项目。也可以通过 ["package-a", "package-b"] 枚举。
  ],
}
```

yarn 或 npm 包管理器下的 monorepo 搭建具体可参考：

- [Yarn Workspace使用指南](https://juejin.cn/post/6974967455114362888)。
- [Yarn Workspaces: Organize Your Project’s Codebase Like A Pro](https://www.smashingmagazine.com/2019/07/yarn-workspaces-organize-project-codebase-pro/)
- ...

2️⃣ **pnpm** 管理 workspaces 没有直接使用 package.json 的 workspaces 字段。在 **pnpm** 包管理器下，我们需要 root 目录新建 **pnpm-workspace.yaml** ，在里面写入 workspaces 所要管理的项目路径：

```js
packages:
  // all packages in subdirs of packages/
  - 'packages/**'

or

packages:
  - 'packages/ui'
  - 'packages/utils'
  - 'packages/web'
  // - ...
```

我们通常会使用到以下命令安装第三方 npm 包：

```js
pnpm install react react-dom -w
```

```js
pnpm i dayjs -r --filter @test/web
```

- `-w` 表示把包安装在 root 下，该包会放置在 `<root>/node_modules` 下。
- `-r` 表示将包安装在所有 package 中。
- `--filter` 表示该行命令只作用于 filter 参数之后的对象(项目)。

> `@test/web` 是 web 项目的 package.json name，即该项目包的名称。

👉 **创建完工作区(workspaces)后** ，将独立的子项目建在对应的 **{packages}** 路径下就可以了，各个子项目的具体开发同独立项目开发一样，进入到对应的项目路径即可。

```js
├── packages
│   ├── ui
│   ├── utils
│   └── web
```

至此，我们基于 pnpm 初步搭建了 monorepo 项目结构。

## 补充

### workspaces 协议：workspaces 管理路径下的模块间依赖

Polyrepo 各项目间互相引用是需要从 npm registry 独立安装的，这也就意味着各项目都要先经历发布这一过程。而在 Monorepo 中各项目间的引用在本地 workspaces 进行，各项目(包)之间可以直接引用，不需要发布到线上。

在依赖 **package.json : workspaces** 字段的包管理器中，其 package.json dependencies 引用如下：

```js
// Monorepo 项目下 packages/antd 对 packages/arco 的引用
{
  "name": "@tian/antd"
  // ...
  "dependencies": {
    "@tian/arco": "^1.0.0"
  }
}

// 其中 packages/arco 目录下的 package.json 如下
{
  "name": "@tian/arco"
  "version": "1.0.0",
  // ...
}
```

**Monorepo workspaces 结构** 如下所示：

```js
{
  "@tian/antd": {
    "location": "packages/antd",
    "workspaceDependencies": [
      "@tian/arco"
    ],
    "mismatchedWorkspaceDependencies": []
  },
  "@tian/arco": {
    "location": "packages/arco",
    "workspaceDependencies": [],
    "mismatchedWorkspaceDependencies": []
  }
}
```

**Monorepo 项目依赖结构** 如下所示：

```js
/package.json
/yarn.lock

/node_modules
// root package.json workspaces 字段将各项目文件夹统一软链到 root node_modules中
/node_modules/@tian/arco -> /packages/arco 

/packages/arco/package.json
/packages/antd/package.json
```

其中 packages/antd 会优先查找本地 package.json 的  `@tian/arco` 包依赖，若没有找到则查找 root package.json 的包依赖，由于设置了 workspaces，因此项目顶层的依赖结构会将 `@tian/arco` 软链到对应的 package 上，packages/antd 最终会命中 packages/arco。

> 值得注意的是，包名 + 版本号全部命中才能使用对应的包。假设 package/arco 版本变动但 package/antd 中依赖的 arco 版本没有及时更新，那么最终会因为无法命中正确的版本号导致报错。

**在 package.json dependencies 项目内部依赖的表现上，pnpm 与 yarn 或其他 node 包管理工具有所不同。** 上述的依赖关系在 pnpm 的 package.json dependencies 中表现形式如下：

```js
{
  "name": "@tian/antd"
  // ...
  "dependencies": {
    "@tian/arco": "workspace:*" // 前缀:版本号
  }
}
```

其中 workspace 是前缀，表示该包由 workspaces 管理，`*` 表示任意版本号，除此之外还有 `~` / `^` 等通用的版本管理符号。

`workspace:*` 会在打包阶段被相关的打包工具自动转换成具体的版本号信息，从而保证了内部依赖版本的一致性。

> 在实际内部项目引用的时候，我们不需要到 package.json 中手动添加依赖，像引用外部 npm 包一样使用 pnpm 命令安装即可：
>
> `pnpm install @tian/arco -w`
>
> pnpm 会自动在 root package.json dependencies 中添加 `"@tian/arco": "workspace:*"`

### TypeScript 的 Monorepo 设置

> 参考 [TypeScript 的 Monorepo 设置](https://juejin.cn/post/7146183222425518093#heading-13)

假如说 **workspaces** 字段是项目 monorepo 设置的关键，那么 tsconfig.json 的 **references** 字段则是项目 TypeScript 声明的 monorepo 关键。其作用同 workspaces 类似， **references 字段中设置的 tsconfig.json 路径均会被统一管理，项目引用时会自动软链接到对应项目的 ts 声明。** 

```js
// root tsconfig.json
{
  "files": [],
  // 在 ts 层面把组件库类型声明分成了两部分
  "references": [
    { "path": "./tsconfig.arco.json" },
    { "path": "./tsconfig.antd.json" },
  ]
}
```

**每个引用的 path 属性可以指向包含 tsconfig.json 文件的目录，也可以指向配置文件本身。** 我们通过 **具体配置文件** 进行具体每个部分的 TypeScript 编译项设置， **而每个部分的公共配置项统一抽离到一个公众配置文件中，再通过 extends 进行引用** ，这样一来就可以大大减少相同的配置代码。

```js
// 公共配置文件示例 tsconfig.base.json
{
  "compilerOptions": {
    "outDir": "dist", // 指定输出目录
    "target": "es2018", // 目标语言的版本
    "module": "esnext", // 生成代码的模板标准
    "baseUrl": ".", // 解析非相对模块的基地址，默认是当前目录
    "sourceMap": false, // 是否生成相应的Map映射的文件，默认：false
    "moduleResolution": "node", // 指定模块解析策略，node或classic
    "allowJs": false, // 是否允许编译器编译JS，JSX文件
    "strict": true, // 是否启动所有严格检查的总开关，默认：false，启动后将开启所有的严格检查选项
    "noUnusedLocals": true, // 是否检查未使用的局部变量，默认：false
    "resolveJsonModule": true, // 是否解析 JSON 模块，默认：false
    "allowSyntheticDefaultImports": true, // 是否允许从没有默认导出的模块中默认导入，默认：false
    "esModuleInterop": true, // 是否通过为所有导入模块创建命名空间对象，允许CommonJS和ES模块之间的互操作性，开启改选项时，也自动开启allowSyntheticDefaultImports选项，默认：false
    "removeComments": false, // 删除注释
    "rootDir": ".", // 指定输出文件目录(用于输出)，用于控制输出目录结构
    "types": [],
    "paths": { // 路径映射，相对于baseUrl
      "@tian/*": ["packages/*"]
    }
  }
}
```

```js
// 子项目 tsconfig.json 配置
{
  "extends": "./tsconfig.base.json",
  "compilerOptions": {
    "composite": true, // 是否开启项目编译，开启该功能，将会生成被编译文件所在的目录，同时开启declaration、declarationMap和incremental，默认：false
    "jsx": "preserve", // 指定JSX代码生成用于的开发环境
    "lib": ["ES2018", "DOM", "DOM.Iterable"], // 指定项目运行时使用的库
    "types": ["unplugin-vue-define-options"], // 用来指定需要包含的模块，并将其包含在全局范围内
    "skipLibCheck": true // 是否跳过声明文件的类型检查，这可以在编译期间以牺牲类型系统准确性为代价来节省时间，默认：false
  },
  "include": ["packages",],// 使用 include 来指定应从绝对类型中使用哪些类型
  "exclude": [ // 提供用于禁用 JavaScript 项目中某个模块的类型获取的配置
    "node_modules",
    "**/dist",
    "**/__tests__/**/*",
    "**/gulpfile.ts",
    "**/test-helper",
    "packages/test-utils",
    "**/*.md"
  ]
}
```
