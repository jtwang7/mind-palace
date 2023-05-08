# 关于React组件二次封装的那些事儿

参考：

- [React通用解决方案——组件二次包装](https://juejin.cn/post/7064119685708513287)

## Questions

🧐 **为什么要进行组件二次封装？**

原生组件面向大众，它是对公共场景的抽象，若要契合实际业务往往需要对其进行二次包装。

> 简而言之，就是对组件进行个性化定制。

🧐 **组件二次封装通常需要做什么？**

1. 类型声明二次封装 **( 通常采用类型继承方式 )**
2. 自定义组件逻辑
3. 向原生组件透传属性

## Contents

### 继承式类型声明

类型声明是组件二次包装过程中的第一步。常见的类型声明方式有：

- 直接进行类型声明
- 类型继承

在组件二次封装场景中，我们推荐使用类型继承的方式。因为我们组件开发的载体是原生组件，使用类型继承可以：

- 减少不必要的类型声明工作
- 保留原生组件提供的所有功能
- 不会因为原生组件参数变动，导致二次封装的组件失效。

```js
import React from "react";
import { Button, ButtonProps } from "@arco-design/web-react";

// 继承类型
type IProps = ButtonProps & {};

const MyButton: React.FC<IProps> = (props) => {
  // 此处为自定义逻辑

  // 继承属性
  return <Button {...props} />;
};

export default MyButton;
```

### 定制默认属性

开源组件库提供的组件满足了日常项目的开发需求，但开源组件的默认值可能并不能匹配实际的业务场景，因此我们通常需要在二次包装自定义组件的时候**重置默认值**。

通常我们会在二次封装的组件中直接重置默认值：

```js
import React from "react";
import { Button, ButtonProps } from "@arco-design/web-react";

type IProps = ButtonProps & {};

const MyButton: React.FC<IProps> = (props) => {
  // 通过解构方式，直接重置默认值
  const { size = "large", type = "primary" } = props;
  return <Button size={size} type={type} {...props} />;
};

export default MyButton;
```

这是其中一种重置默认值的方式，通常我们都采用这种方式。但是，假如某些默认值在我们许多的二次封装组件中都需要被重置，那么这种只针对一个组件的重置方式就显得不太优雅了，对于上述情况，我们通常把默认值重置的逻辑抽离到一个高阶组件中：

```js
import React from "react";

/**
 * 组件默认属性Hoc
 * @param defaultProps 默认属性
 * @returns
 */
export function withDefaultProps<P = {}>(defaultProps?: Partial<P>) {
  return function (Component: any) {
    const WrappedComponent: React.FC<P> = (props) => {
      return React.createElement(Component, {
        ...defaultProps,
        ...props,
      });
    };

    WrappedComponent.displayName = `withDefaultProps(${getDisplayName(
      Component
    )})`;

    return WrappedComponent;
  };
}
```

我们会这样使用这个包含默认值重置逻辑的 `HOC`：

```js
import React from "react";
import { Button, ButtonProps } from "@arco-design/web-react";

type IProps = ButtonProps & {};

export default withDefaultProps<IProps>({
  size: "large",
  type: "primary",
})(Button);
```

### 自定义参数 & 透传原生组件参数

通常我们会为二次封装的组件添加一些新的功能逻辑，其可能依赖于新的用户传参。同时，我们又要保证原生组件的功能不被影响，这就涉及到了**参数隔离/过滤**的问题。

**我们需要通过某些参数过滤的方法，将自定义参数与原生组件参数隔离。并将原生组件的参数原封不动 (透传) 的传递给原生组件。** 过滤方式有以下几种：

- 通过拓展运算符
- 通过omit函数
- ...

```js
import React, { useEffect } from "react";
import { Button, ButtonProps } from "@arco-design/web-react";

type IProps = ButtonProps & {
  hello?: string;
};


const MyButton: React.FC<IProps> = (props) => {
  // 通过拓展运算符过滤 自定义属性'hello' 和 原生组件属性集合{...restProps}
  const {hello, ...restProps} = props
  
  // ...
  
  // 透传原生组件属性
  return <Button {...restProps} />;
};

export default withDefaultProps<IProps>({
  hello: "world",
})(MyButton);
```
