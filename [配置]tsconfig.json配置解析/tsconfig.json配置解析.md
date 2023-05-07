# tsconfig.json配置解析

参考文章:

- [掌握 tsconfig.json](https://juejin.cn/post/6844904178234458120#heading-13)

## 目录

tsconfig.json 顶层属性：

- compileOnSave
- compilerOptions
- exclude
- extends
- files
- include
- references
- typeAcquisition

tsconfig.json 作用：编译工具或命令行编译 `.ts` 文件时所需遵循的配置文件。

## 顶层属性介绍及使用场景

| 顶层属性名称    | 作用                                                        | 常见使用场景                                                 |
| --------------- | ----------------------------------------------------------- | ------------------------------------------------------------ |
| compileOnSave   | 设置为`true`时，保存`.ts`文件将会自动编译，需要编译器支持。 | 让 IDE 在保存文件时根据 `tsconfig.json` 重新生成新文件。     |
| compilerOptions | 配置编译选项                                                | 🔥 一系列编译选项，编译器需要根据这些规则进行编译。           |
| exclude         | 不需要编译的文件/文件夹                                     | 默认排除 `node_modules` 文件夹下文件                         |
| extends         | 引入其他配置文件(继承配置)                                  | 把基础配置抽离成 `tsconfig.base.json` 并引入                 |
| files           | 表示编译器需要编译的单个文件列表                            | 指定具体需要编译的单个`.ts`文件                              |
| include         | 需要编译的文件或目录                                        | `src`：编译 `src` 目录下的所有文件和子目录；<br /> `src/*`：只编译 `src` 一级目录下的所有文件和子目录；<br /> `src/*/*`：只编译 `src` 二级目录下的所有文件和子目录。 |
| references      | 指定依赖的工程                                              |                                                              |
| typeAcquisition | 与自动引入库类型定义文件`.d.ts`的设置项相关                 | 包含3个属性：<br /> `enable`：布尔类型，是否开启自动引入库类型定义文件`.d.ts`，默认为 `false`。<br /> `include`：数组类型，允许自动引入类型文件的库名，例如 `["jquery", "lodash"]`。<br /> `exclude`：数组类型，排除的库名 |

## 配置项

```json
{
	// 是否设置保存文件的时候自动编译(需要编译器支持)
  "compileOnSave": false,
  // 编译选项：此处只列出了一部分
  "compilerOptions": {
    // TS编译器在第一次编译之后会生成一个存储编译信息的文件，第二次编译会在第一次的基础上进行增量编译，可以提高编译的速度
    "incremental": true, 
    // 增量编译文件的存储位置
    "tsBuildInfoFile": "./buildFile",
    // 打印诊断信息  
    "diagnostics": true, 
    // 🔥目标语言的版本
    "target": "ES5", 
    // 🔥生成代码的模块标准
    "module": "CommonJS", 
    // 将多个相互依赖的文件生成一个文件，可以用在AMD模块中，即开启时应设置"module": "AMD"
    "outFile": "./app.js", 
    // 🔥TS需要引用的库，即声明文件，es5 默认引用dom、es5、scripthost
    // 如需要使用es的高级版本特性，通常都需要配置，如es8的数组新特性需要引入"ES2019.Array"
    "lib": ["DOM", "ES2015", "ScriptHost", "ES2019.Array"], 
    // 🔥允许编译器编译JS，JSX文件
    "allowJS": true, 
    // 允许在JS文件中报错，通常与allowJS一起使用
    "checkJs": true, 
    // 🔥指定输出目录
    "outDir": "./dist", 
    // 🔥指定输出文件目录(用于输出)，用于控制输出目录结构
    "rootDir": "./", 
    // 🔥生成声明文件，开启后会自动生成声明文件
    "declaration": true, 
    // 🔥指定生成声明文件存放目录
    "declarationDir": "./file", 
    // 🔥只生成声明文件，而不会生成js文件
    "emitDeclarationOnly": true, 
    // 生成目标文件的sourceMap文件
    "sourceMap": true, 
    // 生成目标文件的inline SourceMap，inline SourceMap会包含在生成的js文件中
    "inlineSourceMap": true, 
    // 为声明文件生成sourceMap
    "declarationMap": true, 
    // 声明文件目录，默认时node_modules/@types
    "typeRoots": [], 
    // 加载的声明文件包
    "types": [], 
    // 删除注释 
    "removeComments":true, 
    // 不输出文件,即编译后不会生成任何文件
    "noEmit": true, 
    // 发送错误时不输出任何文件
    "noEmitOnError": true, 
    // 不生成helper函数，减小体积，需要额外安装，常配合importHelpers一起使用
    "noEmitHelpers": true, 
    // 通过tslib引入helper函数，文件必须是模块
    "importHelpers": true, 
    // 降级遍历器实现，如果目标源是es3/5，那么遍历器会有降级的实现
    "downlevelIteration": true, 
    // 开启所有严格的类型检查
    "strict": true, 
    // 在代码中注入'use strict'
    "alwaysStrict": true, 
    // 不允许隐式的any类型
    "noImplicitAny": true, 
    // 不允许把null、undefined赋值给其他类型的变量
    "strictNullChecks": true, 
    // 不允许函数参数双向协变
    "strictFunctionTypes": true, 
    // 类的实例属性必须初始化
    "strictPropertyInitialization": true, 
    // 严格的bind/call/apply检查
    "strictBindCallApply": true, 
    // 不允许this有隐式的any类型
    "noImplicitThis": true, 
    // 检查只声明、未使用的局部变量(只提示不报错)
    "noUnusedLocals": true, 
    // 检查未使用的函数参数(只提示不报错)
    "noUnusedParameters": true, 
    // 防止switch语句贯穿(即如果没有break语句后面不会执行)
    "noFallthroughCasesInSwitch": true, 
    //每个分支都会有返回值
    "noImplicitReturns": true, 
    // 允许export=导出，由import from 导入
    "esModuleInterop": true, 
    // 允许在模块中全局变量的方式访问umd模块
    "allowUmdGlobalAccess": true, 
    // 🔥模块解析策略，ts默认用node的解析策略，即相对的方式导入
    "moduleResolution": "node", 
    // 🔥解析非相对模块的基地址，默认是当前目录
    "baseUrl": "./", 
    // 🔥路径映射，相对于baseUrl
    "paths": { 
      // 如使用jq时不想使用默认版本，而需要手动指定版本，可进行如下配置
      "jquery": ["node_modules/jquery/dist/jquery.min.js"]
    },
    // 将多个目录放在一个虚拟目录下，用于运行时，即编译后引入文件的位置可能发生变化，这也设置可以虚拟src和out在同一个目录下，不用再去改变路径也不会报错
    "rootDirs": ["src","out"], 
    // 打印输出文件
    "listEmittedFiles": true, 
    // 打印编译的文件(包括引用的声明文件)
    "listFiles": true
  },
  // 🔥指定编译器需要排除的文件或文件夹, 默认排除 node_modules 文件夹下文件。
  "exclude": [
      "src/lib" // 排除src目录下的lib文件夹下的文件不会编译
  ],
  // 🔥引入其他配置文件，继承配置, 默认包含当前目录和子目录下所有 TypeScript 文件。
  // 通常把基础配置抽离成tsconfig.base.json文件，然后引入
  "extends": "./tsconfig.base.json", 
  // 指定需要编译的单个文件列表, 默认包含当前目录和子目录下所有 TypeScript 文件。
  "files": [
      "scr/leo.ts" // 指定编译文件是src目录下的leo.ts文件
  ],
  // 🔥指定编译需要编译的文件或目录
  "include": [
      // "scr" // 编译src目录下的所有文件，包括子目录
      // "scr/*" // 编译scr一级目录下的文件
      "scr/*/*" // 编译scr二级目录下的文件
  ],
  "references": [ 
      {"path": "./common"}
  ],
  "typeAcquisition": {
      "enable": false,
      "exclude": ["jquery"],
      "include": ["jest"]
  },
}
```

> `exclude` 和 `include` 支持 glob 通配符：
>
> - `*` 匹配0或多个字符（不包括目录分隔符）
> - `?` 匹配一个任意字符（不包括目录分隔符）
> - `**/` 递归匹配任意子目录