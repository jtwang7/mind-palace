# InputTag 进阶功能: Tag 折叠

## 概述

在 [InputTag 简版实现](../%5B%E9%A1%B9%E7%9B%AE%5D%20InputTag%20%E7%AE%80%E7%89%88%E5%AE%9E%E7%8E%B0/README.md) 中，我们已经初步实现了 InputTag 组件的基础功能。但落地到实际应用场景时发现，当固定 InputTag 组件宽度后，随着 Tag 数量的增加，会自动发生换行，从而挤压其他页面布局空间。因此，我们需要为 InputTag 新增 Tag 标签折叠的功能，当标签数量或者宽度超过设定阈值时，自动折叠。

## 实现目标

Tag 自动折叠的关键在于计算 Tag 宽度(数量)是否超出设定阈值。为了获取 Tag 的宽度，React 需要生成相应的 Tag DOM 实例并调用相应的样式获取方法，同时，为了避免频闪现象的发生，我们需要在计算完 Tag 宽度之后，立刻重置样式并正确渲染可被展示的 Tag 标签，折叠剩余的 Tag 标签。

此外，为了尽量避免破坏原有 InputTag 组件的基础逻辑，同时实现逻辑的抽离与复用，本文尝试借鉴 Headless UI 的设计思想，尽可能地抽离了 Tag 宽度计算及阈值判断的逻辑。同时，还为折叠标签的弹出组件预留了相应的插槽，为使用者提供了最大可能的自由度。
> (什么是 Headless UI? 可阅读 [全新的 React 组件设计理念 Headless UI](https://juejin.cn/post/7160223720236122125) 了解)

总结而言，我们需要实现以下几个目标:

- [x] Tag 宽度计算
- [x] 宽度计算逻辑抽离

## useTags

我们将 Tags 的宽度收集与计算抽离到了 useTags 中，在 useTags 中，我们接受三个参数:

- tags: 标签内容。(TagType `{id: string; content: string}`)
- maxWidth: 最大宽度阈值
- maxCount: 最大数量阈值

为了获取 `<Tag />` 组件数组的所有实例，我们暴露了 `getRefs()` 方法，通过 ref 回调的方式获取:

```js
const refs = []
array.map(item => (
  <Component ref={(ref) => {refs.push(ref)}} />
))
```

获取到组件实例后，调用 `ELEMENT.getBoundingClientRect()` 方法获取 DOM 元素的空间占用对象，然后依次累加计算宽度并判断是否超出阈值。并给超出阈值的索引位置添加标记。代码如下所示:

```js
import { useLayoutEffect, useState } from "react";
import { TagType } from ".";

const TAG_MARGIN = 10;
const MIN_INPUT_WIDTH = 100;
export const useTags = <T extends HTMLElement>(
  tags: TagType[],
  maxWidth?: number,
  maxCount?: number
) => {
  // @tian: 始终获取最新的 tag refs
  const refs: T[] = [];
  // @tian: 暴露获取 tag refs 的接口
  const getRefs = (ref: T) => {
    refs.push(ref);
  };

  // @tian: 标记被隐藏的 tags 索引起始位置。(初始设为 Infinity，即全部展示)
  const [indexOfStartHidden, setIndexOfStartHidden] = useState(Infinity);

  // @tian: 在 dom 渲染到页面前执行宽度计算，判断是否超出设定阈值；同时，标记索引位置并触发重渲染。
  useLayoutEffect(() => {
    if (maxWidth || maxCount) {
      let width = 0;
      refs.forEach((ref, idx: number) => {
        width += ref.getBoundingClientRect().width + TAG_MARGIN;
        if (maxCount) {
          if (idx >= maxCount) {
            setIndexOfStartHidden(idx);
          }
        }
        if (maxWidth) {
          if (width + MIN_INPUT_WIDTH + 150 > maxWidth) {
            setIndexOfStartHidden(idx);
          }
        }
      });
    }
  }, [tags]);

  return [getRefs, indexOfStartHidden] as const;
};
```

值得关注的地方有几点:

1. 对于 ref 实例的存储，我们并没有将其用 useRef/useState/... 等记忆化保留。而是在每次组件重渲染时重新获取，保证 ref 实例对象始终最新，能有效避免很多潜在的 Bug。
2. 宽度计算放在 useLayoutEffect 中执行。useLayoutEffect 阶段，组件实例已生成，且此时还未被渲染到页面上 (React 维护了视图树与内存树，useLayoutEffect 阶段内存树最新，但还没有被切换到视图中)，此时我们可以获取到最新的页面布局及组件宽度。
3. 需要隐藏的 Tag 索引位置标记我们采用 useState 存储，同时结合 useLayoutEffect 可以实现无缝、无频闪的 Tag 折叠渲染过程。我们已知在 useLayoutEffect 阶段视图并不会发生更新，此时我们完成宽度计算并触发重渲染，React 会中断本次渲染，直接进行最新的渲染操作。useEffect 就不能达到上述效果，因为在 useEffect 阶段，视图已经被更新了。

## 整体代码

本文 InputTag 进阶版实现是基于 antd 的二次封装方案。我们基于 `<Input />` + `<Tag />` 实现基础的功能，同时注入 `useTags` 方法，实现标签宽度的自动计算。
在 `<Tag />` 标签展示部分，我们以 `useTags` 计算出的标记索引为界限，分割 Tags 标签的可视和不可视部分。

我们为每个 `<Tag />` 都注入了 ref 回调 `getRefs()` 方法，确保我们在每次重渲染过程中，都能收集到完整的 ref 实例数组，从而正确计算宽度及临界阈值。

为了给予使用者最大化定制组件的自由度，我们使用了 `React.cloneElement()` 以及 children props 等方法，将需要被折叠的 `<Tag />` 标签数组作为参数传递给 children props 接口，使用者可根据实际情况，引用不同的组件 (例如 `<Popover />` 或 `<Tooltip />` 定制不同的样式)。
> 此处 `React.cloneElement()` 的主要作用是为了传递 props 参数。

```js
import React, {
  useImperativeHandle,
  useLayoutEffect,
  useMemo,
  useRef,
  useState,
} from "react";
import { Input, Tag, InputRef, Popover } from "antd";
import { v4 as uuid } from "uuid";
import _ from "lodash";
import { CloseSquareOutlined } from "@ant-design/icons";
import "antd/dist/reset.css";
import "./index.scss";
import cs from "classnames";
import { useTags } from "./useTags";

export type TagType = { id: string; value: string };
type InputTagProps = {
  style?: React.CSSProperties;
  width?: number;
  maxCount?: number; // 最多tag展示数
  autoRest?: boolean; // 超出范围的tag自动折叠
  wrapper?: (Tags: React.ReactNode) => React.ReactElement;
};
const InputTag: React.ForwardRefRenderFunction<any, InputTagProps> = (
  $props,
  ref
) => {
  const props = _.defaults($props, { width: 400 });
  const MIN_INPUT_WIDTH = 100;

  const inputRef = useRef<InputRef>(null!);
  useImperativeHandle(
    ref,
    () => {
      return {
        focus() {
          inputRef.current.focus();
        },
        blur() {
          inputRef.current.blur();
        },
      };
    },
    []
  );

  const [tags, setTags] = useState<TagType[]>([]);
  const [tag, setTag] = useState<string>();

  const addTag = (tag: string) => {
    if (tags.find((t) => t.value === tag)) {
      return;
    }
    setTags((prev) => [...prev, { id: uuid(), value: tag }]);
    setTag(undefined);
  };
  const removeTag = (tag: TagType) => {
    setTags((prev) => prev.filter((pt) => pt.id !== tag.id));
  };
  const clearTags = () => {
    setTags([]);
  };

  // 宽度计算逻辑
  const [getRefs, index] = useTags(tags, props.width, props.maxCount);

  return (
    <div
      ref={ref}
      className={"tag-container"}
      style={{
        width: props.width,
        ...props.style,
      }}
    >
      {/* tag 展示部分 */}
      {tags.slice(0, index).map((tag) => (
        <Tag
          key={uuid()}
          ref={getRefs}
          bordered
          closable
          onClose={(e) => {
            e.preventDefault();
            removeTag(tag);
          }}
        >
          {tag.value}
        </Tag>
      ))}
      {/* tag 隐藏部分 */}
      {/* 此处用 React.cloneElement 设置插槽，可传入 Popover，Tooltip 等组件 */}
      {props.wrapper && tags.slice(index).length
        ? React.cloneElement(
            props.wrapper(
              tags.slice(index).map((tag) => (
                <Tag
                  key={uuid()}
                  ref={getRefs}
                  bordered
                  closable
                  onClose={(e) => {
                    e.preventDefault();
                    removeTag(tag);
                  }}
                >
                  {tag.value}
                </Tag>
              ))
            )!,
            {
              children: (
                <Tag bordered closable={false}>{`+${tags.length - index}`}</Tag>
              ),
            }
          )
        : null}
      <Input
        ref={inputRef}
        placeholder="请输入..."
        style={{ minWidth: MIN_INPUT_WIDTH, flex: "1 1" }}
        bordered={false}
        value={tag}
        onChange={(e) => {
          setTag(e.target.value);
        }}
        onPressEnter={() => {
          tag && addTag(tag);
        }}
      />
      <CloseSquareOutlined
        rev={undefined}
        style={{ marginLeft: "auto" }}
        onMouseDown={(e) => {
          e.preventDefault();
          clearTags();
        }}
      />
    </div>
  );
};

export default React.forwardRef(InputTag);
```

**[项目地址](https://github.com/jtwang7/UI-Library/tree/master/src/components/input-tag)**

**✨ 欢迎关注我的 [github](https://github.com/jtwang7)，🥳个人学习总结、项目开发、前端代码手撕等好物多多！**