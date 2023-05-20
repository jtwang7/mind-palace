# 阅读清单

## React

- [x] [Render and Commit](https://react.dev/learn/render-and-commit)
  > 组件渲染流程
- [x] [Rendering Lists](https://react.dev/learn/rendering-lists#rules-of-keys)
- **组件 key 的作用及应用场景**
  - [x] [Keeping list items in order with key](https://react.dev/learn/rendering-lists#keeping-list-items-in-order-with-key)
  - [x] [Resetting state with a key](https://react.dev/reference/react/useState#resetting-state-with-a-key)
  - [x] [Preserving and Resetting State](https://react.dev/learn/preserving-and-resetting-state)
- [x] [useState](https://react.dev/reference/react/useState#storing-information-from-previous-renders)
  - [x] [Queueing a Series of State Updates](https://react.dev/learn/queueing-a-series-of-state-updates)
    > Batch Update 原理
  - [ ] [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect)
    > setState 直接在 render 函数中使用的应用场景
- [x] [useRef](https://react.dev/reference/react/useRef)
  - [x] [Referencing Values with Refs](https://react.dev/learn/referencing-values-with-refs#differences-between-refs-and-state)

## Bundlers

- [x] [Webpack and Rollup: the same but different](https://medium.com/webpack/webpack-and-rollup-the-same-but-different-a41ad427058c)
  - [x] [[译] 同中有异的 Webpack 与 Rollup](https://juejin.cn/post/6844903473700405261)
- [x] [webpack、gulp、rollup、tsc/babel 使用对比](https://segmentfault.com/a/1190000037638760)
  - [ ] [用 rollup + gulp 造个轮子，别说还挺香](https://juejin.cn/post/7081998643657605127#heading-7)
  - [ ] [antd-tools: gulp + tsc/babel 打包组件](https://github.com/ant-design/antd-tools/blob/master/lib/gulpfile.js)

## 开发

- [ ] [为什么说 90% 的前端不会调试 Ant Design 源码？](https://juejin.cn/post/7158430758070140942)
- **受控组件与非受控组件**
  - [ ] [一次手写Antd Form的经历，让我受益匪浅](https://juejin.cn/post/7038099720400535582)
  - [ ] [React受控&非受控组件——我想都要！](https://juejin.cn/post/7051855761588092958)
  
- **[封装(非)受控状态: 对比 arco-design:useMergeValue 与 ahooks:useControllableValue 实现](https://github.com/jtwang7/mind-palace/blob/main/notes/%5B%E9%98%85%E8%AF%BB%5D%20%E5%B0%81%E8%A3%85(%E9%9D%9E)%E5%8F%97%E6%8E%A7%E7%8A%B6%E6%80%81%3A%20%E5%AF%B9%E6%AF%94%20arco-design%3AuseMergeValue%20%E4%B8%8E%20ahooks%3AuseControllableValue%20%E5%AE%9E%E7%8E%B0/index.md)**
  - [x] [React 组件的受控与非受控](https://zhuanlan.zhihu.com/p/536322574)
  - [x] [arco-design: useMergeValue](https://github.com/arco-design/arco-design/blob/main/components/_util/hooks/useMergeValue.ts#L5)
  - [ ] [ahooks(2.x)-useControllableValue](https://github.com/alibaba/hooks/blob/release/v2.x/packages/hooks/src/useControllableValue/index.ts)
  - [x] [[疑问] useControllableValue 为什么要在有 props.value 的情况下将 props.value 同步到 state](https://github.com/alibaba/hooks/issues/984)
  - [x] [ahooks(3.x)-useControllableValue](https://github.com/alibaba/hooks/blob/master/packages/hooks/src/useControllableValue/index.ts)
  - [x] [ahooks-useControllableValue 更新记录](https://github.com/alibaba/hooks/commit/d0ebab6923f09f172288dbb27cd8ffc2722647f8)

## Github

- [x] [.gitignore template](https://github.com/github/gitignore/blob/main/Node.gitignore)
- **新建 ssh key 并链接到 github 仓库:**
  - [x] [生成新的 SSH 密钥并将其添加到 ssh-agent](https://docs.github.com/zh/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)
  - [x] [新增 SSH 密钥到 GitHub 帐户](https://docs.github.com/zh/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)
