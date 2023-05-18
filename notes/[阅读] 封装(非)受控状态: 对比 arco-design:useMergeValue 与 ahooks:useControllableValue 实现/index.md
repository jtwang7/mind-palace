# 封装(非)受控状态: 对比 arco-design:useMergeValue 与 ahooks:useControllableValue 实现

## 受控&非受控封装思路

参考文章：[React 组件的受控与非受控](https://zhuanlan.zhihu.com/p/536322574)

在组件库实现中，我们通常会将某类型组件封装**既支持受控模式，也支持非受控模式**的功能(例如 antd Input)。
> 尽管在业务项目中，我们写的组件都是明确的受控或者非受控，但对于组件库来说，有非常多的组件需要做到既支持受控模式，又支持非受控模式。

非受控和受控组件的区别，简单而言可以理解为：

1. 非受控组件：组件状态不受外部控制，而是封闭在组件内部。
2. 受控组件：组件状态受外部控制，状态存储在组件外，而不是封闭在组件内。

最简单的实现思路：**内外两个状态，手动同步。**
考虑到实现成本的复杂度，我们需要让组件逻辑在两种模式下，尽可能的保持一致，减少逻辑分支意味着更好的可维护性和可读性：Child 组件内部始终存在一个状态，不管它处于哪种模式，它都直接使用自己内部的状态。而当它处于受控模式时，我们让它的内部状态和 Parent 组件中的状态手动保持同步。

### Demo1

基于上述实现思路，我们写出了初版的 Demo 示例:
> CodeSandbox 示例地址：[useRef+forceUpdate实现受控组件: Demo1](https://codesandbox.io/s/useref-forceupdateshi-xian-shou-kong-zu-jian-c31p65?file=/src/Demo1.tsx)

```js
import React, { useState, useEffect } from "react";
import { DemoProps } from "./types";
import { useRerenderCounter } from "./useRerenderCounter";

const Demo1: React.FC<DemoProps> = (props) => {
  const { defaultValue, value, onChange } = props;
  // 判断是否存在 value 属性字段
  // 注意：这里用 Object.property.hasOwnProperty() 的好处在于，它不会去判断对象继承类型上的属性字段。
  // Reflect.has() 以及 in 方法判断对象字段会追溯到其继承类型。
  const isControlled = props.hasOwnProperty("value");

  // 组件始终维护并使用内部状态
  // 若组件受控，则手动同步内外状态
  const [innerValue, setInnerValue] = useState(
    () => (isControlled ? value : defaultValue) ?? 0
  );

  // 组件受控时，组件内部状态同步外部 value
  useEffect(() => {
    if (isControlled) {
      setInnerValue(value ?? 0);
    }
  }, [isControlled, value]);

  const handleClick = () => {
    if (!isControlled) {
      // 组件非受控时，点击 button 默认 +1
      setInnerValue((prev) => prev + 1);
    }
    // 组件受控时，change 规则由外部控制
    onChange?.(innerValue);
  };

  const count = useRerenderCounter();
  useEffect(() => {
    console.log(`demo1: 组件重渲染${count}次`);
  });

  return (
    <div>
      <span style={{ marginRight: 10 }}>Demo1: {innerValue}</span>
      <button onClick={handleClick}>+</button>
    </div>
  );
};

export default Demo1;
```

尽管 Demo1 同时支持了组件的受控和非受控模式，但是当前实现的受控方式存在以下问题：

1. 受控状态下，组件内部状态更新始终比外部状态更新晚一个渲染周期。**(原因在于 useEffect)**
2. 受控状态下，存在重复渲染的情况: 同步内外状态调用 setState 触发 re-render，props.value 变动时导致 re-render。**(原因在于 setState)**

✨ **小Tips**
代码中，对于对象内是否存在某字段的判断逻辑：使用的是 `Object.property.hasOwnProperty()`
其好处在于: 它不会去判断对象继承类型上的属性字段。`Reflect.has()` 以及 `in` 方法判断对象字段会追溯到其继承类型。

### Demo2

为了解决上述问题，我们实现了 Demo2，通过 `useRef + forceUpdate` 的形式，替代 `useState + useEffect` 实现内外状态的实时同步。
> CodeSandbox 示例地址：[useRef+forceUpdate实现受控组件: Demo2](https://codesandbox.io/s/useref-forceupdateshi-xian-shou-kong-zu-jian-c31p65?file=/src/Demo2.tsx)

```js
import React, { useEffect, useState, useRef } from "react";
import { DemoProps } from "./types";
import { useRerenderCounter } from "./useRerenderCounter";

const Demo2: React.FC<DemoProps> = (props) => {
  const { defaultValue, value, onChange } = props;
  // 判断是否存在 value 属性字段
  const isControlled = props.hasOwnProperty("value");

  // 内部状态
  const innerValue = useRef((isControlled ? value : defaultValue) ?? 0);
  // 函数组件中强制触发组件重渲染的操作。
  const [, _forceUpdate] = useState({});
  const forceUpdate = () => {
    _forceUpdate({});
  };

  // 组件受控时，组件内部状态基于外部 value 同步更改。
  if (isControlled) {
    innerValue.current = value ?? 0;
  }

  const handleClick = () => {
    if (!isControlled) {
      innerValue.current += 1;
      forceUpdate();
    }
    // 组件受控时，change 规则由外部控制
    onChange?.(innerValue.current);
  };

  const count = useRerenderCounter();
  useEffect(() => {
    console.log(`demo2: 组件重渲染${count}次`);
  });

  return (
    <div>
      <span style={{ marginRight: 10 }}>Demo2: {innerValue.current}</span>
      <button onClick={handleClick}>+</button>
    </div>
  );
};

export default Demo2;
```

具体实现思路:

1. 组件受控时: 内部状态通过 ref 与外部保持完全同步，`props.value === innerValue`，同时只触发一次渲染(来自父组件)。
2. 组件非受控时: 内部通过 ref + forceUpdate 模拟 useState，实现数据更新与组件重渲染。

虽然 `useRef + forceUpdate` 这种方式可以有效处理(非)受控模式，但是这种实现思路与 React 提倡的设计模式是相违背的。React 并不建议在 render 过程中对 useRef 的 ref 对象进行读写操作，因为 ref 对象的可变形会破坏 React Function Component 的纯函数特性，导致相同输入时，输出结果可能存在无法预期的情况。
> 详见[Best practices for refs](https://react.dev/learn/referencing-values-with-refs#best-practices-for-refs)

当然，在当前实现中，`useRef + forceUpdate` 的作用是始终保持内外状态同步，因此不会出现无法预期的情况 (相同的输入，其渲染结果均保持一致)。

### Demo3

针对 Demo1 存在的两点问题，还有另一种解决方案。分析 Demo1 问题产生的原因，其主要是同步内部状态导致的。我们并非一定需要同步内部状态，可以通过判别当前所处的模式，进而决定是使用 `props.value` 还是 `innerValue`。
> CodeSandbox 示例地址：[useRef+forceUpdate实现受控组件: Demo3](https://codesandbox.io/s/useref-forceupdateshi-xian-shou-kong-zu-jian-c31p65?file=/src/Demo3.tsx)

```js
import React, { useEffect, useState } from "react";
import { DemoProps } from "./types";
import { useRerenderCounter } from "./useRerenderCounter";

const Demo3: React.FC<DemoProps> = (props) => {
  const { defaultValue, value, onChange } = props;
  // 判断是否存在 value 属性字段
  const isControlled = props.hasOwnProperty("value");

  // 内部状态
  const _default = (isControlled ? value : defaultValue) ?? 0;
  const [innerValue, setInnerValue] = useState(_default);

  // 若受控，则直接使用 props.value；反之，使用内部状态。
  // 不再同步内外状态，只有在非受控的时候管理内部状态；
  const _value = isControlled ? value! : innerValue;

  const handleClick = () => {
    if (!isControlled) {
      setInnerValue((prev) => prev + 1);
    }
    onChange?.(_value);
  };

  const count = useRerenderCounter();
  useEffect(() => {
    console.log(`demo3: 组件重渲染${count}次`);
  });

  return (
    <div>
      <span style={{ marginRight: 10 }}>Demo3: {_value}</span>
      <button onClick={handleClick}>+</button>
    </div>
  );
};

export default Demo3;
```

但是上述实现也存在一定的问题:
在极端情况下，用户可能会从受控模式切换到非受控模式。不对内外状态实时进行同步，会导致切换前后状态不一致。
> 详见：[[疑问] useControllableValue 为什么要在有 props.value 的情况下将 props.value 同步到 state](https://github.com/alibaba/hooks/issues/984)

针对上述问题，[arco-design: useMergeValue](https://github.com/arco-design/arco-design/blob/main/components/_util/hooks/useMergeValue.ts#L5) 对这一场景单独做了逻辑判断：

```js
import React, { useState, useEffect, useRef } from 'react';
import { isUndefined } from '../is';
import usePrevious from './usePrevious';

export default function useMergeValue<T>(
  defaultStateValue: T,
  props?: {
    defaultValue?: T;
    value?: T;
  }
): [T, React.Dispatch<React.SetStateAction<T>>, T] {
  const { defaultValue, value } = props || {};
  const firstRenderRef = useRef(true);
  const prevPropsValue = usePrevious(props.value);

  const [stateValue, setStateValue] = useState<T>(
    !isUndefined(value) ? value : !isUndefined(defaultValue) ? defaultValue : defaultStateValue
  );

  useEffect(() => {
    // 第一次渲染时候，props.value 已经在useState里赋值给stateValue了，不需要再次赋值。
    if (firstRenderRef.current) {
      firstRenderRef.current = false;
      return;
    }

    // @tian: 此段逻辑识别了组件从受控模式切换到非受控模式的过程。此时需要同步内外状态。
    // 外部value等于undefined，也就是一开始有值，后来变成了undefined（
    // 可能是移除了value属性，或者直接传入的undefined），那么就更新下内部的值。
    // 如果value有值，在下一步逻辑中直接返回了value，不需要同步到stateValue
    /**
     *  prevPropsValue !== value: https://github.com/arco-design/arco-design/issues/1686
     *  react18 严格模式下 useEffect 执行两次，可能出现 defaultValue 不生效的问题。
     */
    if (value === undefined && prevPropsValue !== value) {
      setStateValue(value);
    }
  }, [value]);

  const mergedValue = isUndefined(value) ? stateValue : value;

  return [mergedValue, setStateValue, stateValue];
}
```

## 旧版本 useControlledValue 存在的问题

ant-design 与 arco-design 针对受控与非受控的 hooks 封装，分别采取了不同的方式:

1. antd 采用了 Demo1 -> Demo2 的封装模式
2. arco-design 则采取了 Demo3 的封装模式。

在[旧版本 useControllableValue](https://github.com/alibaba/hooks/blob/release/v2.x/packages/hooks/src/useControllableValue/index.ts) 中，采用了 Demo1 的封装方式，因此存在多次渲染以及内外同步延迟的问题:

```js
import { useCallback, useState, useEffect, useRef } from "react";
import useUpdateEffect from "../useUpdateEffect";

export interface Options<T> {
  defaultValue?: T;
  defaultValuePropName?: string;
  valuePropName?: string;
  trigger?: string;
}

export interface Props {
  [key: string]: any;
}

export default function useControllableValue<T>(
  props: Props = {},
  options: Options<T> = {}
) {
  const {
    defaultValue,
    defaultValuePropName = "defaultValue",
    valuePropName = "value",
    trigger = "onChange",
  } = options;

  const value = props[valuePropName];

  // @tian: 2.x版本中，采用了 useState + useEffect 模式同步内外状态。
  const [state, setState] = useState<T | undefined>(() => {
    if (valuePropName in props) {
      return value;
    }
    if (defaultValuePropName in props) {
      return props[defaultValuePropName];
    }
    return defaultValue;
  });

  // 存在二次渲染
  useUpdateEffect(() => {
    if (valuePropName in props) {
      setState(value);
    }
  }, [value, valuePropName]);

  const handleSetState = useCallback(
    (v: T | undefined) => {
      if (!(valuePropName in props)) {
        setState(v);
      }
      if (props[trigger]) {
        props[trigger](v);
      }
    },
    [props, valuePropName, trigger]
  );

  return [state, handleSetState] as const;
}
```

## 当前 useControllableValue 实现

在[ahooks(3.x)-useControllableValue](https://github.com/alibaba/hooks/blob/master/packages/hooks/src/useControllableValue/index.ts)中，antd 采用了 Demo2 `useRef + forceUpdate` 的形式同步内外状态。从[ahooks-useControllableValue 更新记录](https://github.com/alibaba/hooks/commit/d0ebab6923f09f172288dbb27cd8ffc2722647f8)中我们可以看到对应的代码变动情况。

✨ 有意思的一点是：useControllableValue 更新记录中，有一天更新 commit:

```text
- 受控模式下，不再维护内部状态，避免额外的 rerender
+ 优化了代码逻辑，避免了不必要的 rerennde
```

也就是说 useControllableValue 实际上也考虑了 Demo3 的实现方式，只是在对于受控与非受控模式切换上，与 arco-design 分别使用了不同的实现方式。

useControllableValue 的代码实现:

```js
import { useMemo, useRef } from 'react';
import type { SetStateAction } from 'react';
import { isFunction } from '../utils';
import useMemoizedFn from '../useMemoizedFn';
import useUpdate from '../useUpdate';

export interface Options<T> {
  defaultValue?: T;
  defaultValuePropName?: string;
  valuePropName?: string;
  trigger?: string;
}

export type Props = Record<string, any>;

export interface StandardProps<T> {
  value: T;
  defaultValue?: T;
  onChange: (val: T) => void;
}

function useControllableValue<T = any>(
  props: StandardProps<T>,
): [T, (v: SetStateAction<T>) => void];
function useControllableValue<T = any>(
  props?: Props,
  options?: Options<T>,
): [T, (v: SetStateAction<T>, ...args: any[]) => void];
function useControllableValue<T = any>(props: Props = {}, options: Options<T> = {}) {
  const {
    defaultValue,
    defaultValuePropName = 'defaultValue',
    valuePropName = 'value',
    trigger = 'onChange',
  } = options;

  // @tian: 识别是否受控
  const value = props[valuePropName] as T;
  const isControlled = props.hasOwnProperty(valuePropName);

  const initialValue = useMemo(() => {
    if (isControlled) {
      return value;
    }
    if (props.hasOwnProperty(defaultValuePropName)) {
      return props[defaultValuePropName];
    }
    return defaultValue;
  }, []);

  // @tian: 内部状态用 ref 存储，并且保证实时的内外同步
  const stateRef = useRef(initialValue);
  if (isControlled) {
    stateRef.current = value;
  }

  // @tian: ahooks 的 forceUpdate 实现
  const update = useUpdate();

  function setState(v: SetStateAction<T>, ...args: any[]) {
    const r = isFunction(v) ? v(stateRef.current) : v;

    // @tian: 非受控时，更新内部状态，同时触发重渲染
    if (!isControlled) {
      stateRef.current = r;
      update();
    }
    if (props[trigger]) {
      props[trigger](r, ...args);
    }
  }

  return [stateRef.current, useMemoizedFn(setState)] as const;
}

export default useControllableValue;
```
