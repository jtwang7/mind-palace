# InputTag 简版实现

## 简介

✨ [CodeSandbox: InputTag](https://codesandbox.io/s/inputtag-med39g?file=/src/components/InputTag/index.tsx)

**实现功能:**

- [x] allowClear: 一键清空
- [x] autoFocus: 自动聚焦
- [x] disabled: 允许禁用;
- [x] saveOnBlur: 失焦时生成tag;
- [x] placeholder: 输入框默认提示词;
- [x] onBlur: 监听失焦;
- [x] onFocus: 监听聚焦;
- [x] defaultValue & value & onChange: 实现组件受控/非受控;
- [x] onChange: 监听input输入变动;
- [x] onPressEnter: 监听回车事件;
- [x] onRemove: 监听tag移除事件;
- [x] onClear: 监听tags清空;

## React.forwardRef + useImperativeHandle

自定义组件实现时，除了暴露必要的 props，还需要向调用者或开发者提供一些必要的 ref instance 方法。我们可以通过 [React.forwardRef()](https://react.dev/reference/react/forwardRef) 暴露整个组件实例，这意味着外部拥有控制当前组件内部状态的能力。虽然暴露全部的实例方法可以为开发者提供足够大的自由度，但是也会带来一些无法预期的问题。因此，我们通常会结合 [useImperativeHandle](https://react.dev/reference/react/useImperativeHandle) 限制实例暴露的方法。

```ts
// Input.tsx
import React, { useImperativeHandle, useRef } from "react";
import { InputProps } from "./types";

const Input = (props, ref) => {
  // 暴露实例
  const inputRef = useRef<HTMLInputElement>(null!);
  useImperativeHandle(
    ref,
    () => {
      // ❌ 错误的写法 (原因未知)
      // return {
      //   focus: inputRef.current.focus,
      //   blur: inputRef.current.blur,
      // }

      // ✅ 正确的写法
      return {
        focus() {
          inputRef.current.focus();
        },
        blur() {
          inputRef.current.blur();
        }
      } as any;
    },
    []
  );

  return (
    <input
      ref={inputRef}
      // ...props
    />
  );
};

export default React.forwardRef<HTMLInputElement, InputProps<string>>(Input);
```

我们将要向外暴露的方法放置在一个对象中返回，`useImperativeHandle` 会将这个 `createHandle` 挂载到接收的 `ref` 上，外部访问这个 ref 对象时，能访问且仅能访问到我们自定义暴露出的方法。
> `useImperativeHandle()` 第一个参数接受 `React.forwardRef()` 返回的 ref 对象。

## useControllableValue

Input 组件和 InputTag 组件均支持【受控/非受控模式】，因此我们将组件受控的逻辑封装成一个单独的 hooks 方便共用。具体的封装细节以及这么封装的原因，参考[封装(非)受控状态: 对比 arco-design:useMergeValue 与 ahooks:useControllableValue 实现](https://github.com/jtwang7/mind-palace/tree/main/notes/%5B%E9%98%85%E8%AF%BB%5D%20%E5%B0%81%E8%A3%85(%E9%9D%9E)%E5%8F%97%E6%8E%A7%E7%8A%B6%E6%80%81%3A%20%E5%AF%B9%E6%AF%94%20arco-design%3AuseMergeValue%20%E4%B8%8E%20ahooks%3AuseControllableValue%20%E5%AE%9E%E7%8E%B0)。

```js
import { SetStateAction, useCallback, useMemo, useRef, useState } from "react";

interface Props<T> {
  defaultValue?: T;
  value?: T;
  onChange?: (v: T, ...args: any[]) => void;
}

export const useControllableValue = <T>(props: Props<T>) => {
  const isControlled = props.hasOwnProperty("value");
  const initialState = useMemo(() => {
    if (Reflect.has(props, "value")) {
      return props.value;
    }
    if (Reflect.has(props, "defaultValue")) {
      return props.defaultValue;
    }
    return undefined;
  }, []);

  // 内部状态：受控状态下，实时保持与外部状态同步
  const innerValue = useRef<T | undefined>(initialState);
  if (isControlled) {
    innerValue.current = props.value;
  }

  const [, update] = useState({});
  const forceUpdate = useCallback(() => {
    update({});
  }, []);

  // 封装可同时控制内外状态更新的 set function
  const setState = (v: SetStateAction<T | undefined>, ...args: any[]) => {
    const _r =
      typeof v === "function" ? (v as Function)(innerValue.current) : v;
    if (!isControlled) {
      innerValue.current = _r as T;
      forceUpdate();
    }
    props.onChange?.(_r, ...args);
  };

  return [innerValue.current!, setState] as const;
};
```

`useControllableValue()` 接受 defaultValue, value, onChange 3个参数:

- 若传入 defaultValue，则启用非受控模式。状态由 `useControllableValue` 内部声明的状态变量控制。
  > `useControllableValue()` 中通过 `useRef` 实时同步内外状态，确保模式发生变化时(例如受控 -> 非受控)状态保持一致。
- 若传入 value，则启用受控模式。状态由外部状态变量控制，并且通过 onChange 改变。

`useControllableValue` 始终用内部状态控制组件渲染，其返回内部状态以及改变状态的 set 方法 (以 useState 模板设计)。

## InputTag 实现以及一些技术细节

InputTag 实现思路: 维护一个存储 tags 内容的数组，`<Tag />` 渲染均依赖于这个 tags 数组。`<input />` 的作用是向 tagsList 中推入创建的 tag 内容。任何增删本质上都是对 tagsList 的操作。
> 这符合 React 数据驱动视图渲染的核心思想。

✨ **完整代码示例:**

```js
import React, { useEffect, useImperativeHandle, useRef, useState } from "react";
import cls from "classnames";
import { v4 as uuidv4 } from "uuid";
import Input from "../Input";
import Tag from "../Tag";
import { InputTagProps } from "./types";
import { useControllableValue } from "../../hooks/useControllableValue";

const InputTag = (props: InputTagProps<string>, ref: any) => {
  const inputRef = useRef<HTMLInputElement>(null!);
  useImperativeHandle(
    ref,
    () => {
      return {
        focus() {
          inputRef.current.focus();
        },
        blur() {
          inputRef.current.blur();
        }
      };
    },
    []
  );

  const [value, setValue] = useControllableValue<string>(props);

  const [tagLists, setTagLists] = useState<string[]>([]);
  const isTagListsEmpty = tagLists.length === 0;

  const increaseTag = (val: string) => {
    setTagLists((t) => {
      const _t = structuredClone(t);
      _t.push(val);
      return _t;
    });
  };
  const decreaseTag = (tagPosition: number) => {
    setTagLists((t) => {
      const _t = [...t];
      _t.splice(tagPosition, 1);
      return _t;
    });
  };

  const handleTagClear = (tag: string, tagIndex: number, e: any) => {
    decreaseTag(tagIndex);
    props.onRemove?.(tag, tagIndex, e);
  };
  const handleChange = (v: string, e: any) => {
    setValue(v);
  };
  const handlePressEnter = () => {
    // 检测是否已存在 tag
    const isTagExisted = tagLists.find((tag) => tag === value);
    if (value && !isTagExisted) {
      increaseTag(value);
      setValue("");
    }
  };
  // 点击清除按钮时，onClear 和 onBlur 会发生冲突，原因是因为:
  // <input /> 会先失去焦点，触发 onBlur，然后才会触发 onClick。但因为 onBlur 触发后，关闭按钮的位置发生了变化，导致 onClick 执行失败。
  // 我们可以将 onClick 替换为 onMouseDown，并添加 e.preventDefault() 组织 mouseDown 本身的 focus 行为来避免上述情况。
  // 原因: 1. onMouseDown 优先于 onBlur 执行。2. 阻止 mouseDown 原生的 focus 行为可以避免触发 onBlur。
  const handleClear = () => {
    if (!isTagListsEmpty) {
      setTagLists([]);
      props.onClear?.();
    }
  };
  const handleBlur = (e: any) => {
    if (props.saveOnBlur) {
      handlePressEnter();
    }
    props.onBlur?.(e);
  };
  const handleFocus = (e?: any) => {
    props.onFocus?.(e);
  };

  useEffect(() => {
    if (props.autoFocus && isTagListsEmpty) {
      inputRef.current.focus();
    }
  }, [props.autoFocus, isTagListsEmpty]);

  return (
    <div className={cls("input-tag-container", props.className)}>
      {tagLists.map((tag, idx) => (
        <Tag
          bordered
          closable={!props.disabled}
          onClose={(e) => handleTagClear(tag, idx, e)}
          key={uuidv4()}
          className="input-tag-tagger"
        >
          {tag}
        </Tag>
      ))}
      <Input
        ref={inputRef}
        className="input-tag-inputer"
        placeholder="请输入数据"
        disabled={props.disabled}
        value={value}
        onChange={handleChange}
        onPressEnter={handlePressEnter}
        onBlur={handleBlur}
        onFocus={handleFocus}
        allowClear={props.allowClear && !isTagListsEmpty}
        onClear={handleClear}
      />
    </div>
  );
};

export default React.forwardRef(InputTag);
```

### event:blur 与 event:click 冲突的问题

当同时设置 `InputTag:allowClear` 和 `InputTag:saveOnBlur` 为 true 时，发现:
![404](images/problem-click%20with%20blur.gif)

当我们点击清除时，InputTag 执行了 saveOnBlur 的逻辑，并没有正确清除所有 tags。
产生上述问题的原因是: **事件触发存在一定的优先级，event:blur 优先于 event:click 触发，导致 clear button 在新的 tag 被添加进来后，位置变动，丢失 click 操作。**

🌈 **解决方法:**
将 `event:click` 替换为 `event:mousedown`，并禁用 mousedown 的原生逻辑 `e.preventDefault()`。mousedown 事件优先于 blur 执行，保证了执行逻辑的准确性。

```js
// Input.tsx
const Input = React.forwardRef<HTMLInputElement, InputProps<string>>(
  (props, ref) => {
    return (
      <div style={style} className={cls("input-container", className)}>
        <input
          // ...props
        />
        <span
          role="button"
          className="input-close-button"
          onMouseDown={(e) => {
            // onMouseDown 优先于 onBlur 执行
            // 我们只利用其优先级高的特性，因此需要避免 mouseDown 原生里的 focus:
            // 通过 e.preventDefault() 取消 mouseDown 的原生事件，焦点就不会因为 mouseDown 而转移到 closeButton 上。
            e.preventDefault();
            props.onClear?.();
          }}
        >
          // ...
        </span>
      </div>
    );
  }
);

export default Input;
```

🔥 值得注意的是：我们调用 `e.preventDefault()` 禁用 mousedown 原生是必要的。当点击某 dom 时，mousedown 会将焦点定位到当前点击对象上。这就导致我们 mousedown 逻辑执行完后，会触发 blur 事件并执行相应的逻辑。对于当前应用场景而言，我们不希望清除 tags 后又重新存入还未输入完成的 input content。因此我们需要禁用 mousedown 的原生逻辑。
未禁用 mousedown 原生逻辑，实际渲染效果如下:
![404](images/problem-mousedown%20with%20default.gif)

正确的渲染效果如下:
![404](images/correct%20render.gif)

### 其他细节

- 为了保证 tagsList 列表渲染时，各渲染子项 key 的唯一性，使用了 [uuid](https://www.npmjs.com/package/uuid) 第三方库。
- 深拷贝使用目前 JS 原生提供的深拷贝 API [structuredClone](https://developer.mozilla.org/en-US/docs/Web/API/structuredClone)。
