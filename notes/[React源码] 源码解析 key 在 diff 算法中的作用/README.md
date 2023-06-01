# 源码解析 key 在 diff 算法中的作用

参考文章:

- [React Diff 算法源码阅读笔记](https://juejin.cn/post/6987714071801888798#heading-5)
- [React 源码- key 有什么作用, 可以省略吗?](https://juejin.cn/post/6998913043077791775)

## Demo

本文源码调试的样例来自于 [React官方:Resetting state with a key](https://react.dev/learn/preserving-and-resetting-state#option-2-resetting-state-with-a-key)。为了探究不同 key 赋值场景下，react 对于组件复用的表现及内部运行流程。本文对调试样例做了相应的调整，样例代码如下:

```js
// Scoreboard.js
import React, { useState } from "react";
import Counter from "./Counter";

export default function Scoreboard() {
  const [isPlayerA, setIsPlayerA] = useState(true);
  return (
    <div>
      {/* 场景一: 进入 reconcileChildrenArray */}
      {/* 
      {isPlayerA ? (
        <Counter key="Taylor" person="Taylor" />
      ) : (
        <Counter key="Sarah" person="Sarah" />
      )} 
      */}

      {/* 场景二: 进入 reconcileSingleElement */}
      <div>
        {isPlayerA ? (
          <Counter key="Taylor" person="Taylor" />
        ) : (
          <Counter key="Sarah" person="Sarah" />
        )}
      </div>
      <button
        onClick={() => {
          setIsPlayerA(!isPlayerA);
        }}
      >
        Next player!
      </button>
    </div>
  );
}
```

```js
// Counter.js
import React, { useState } from "react";

function Counter({ person }) {
  const [score, setScore] = useState(0);

  return (
    <div>
      <h1>
        {person}'s score: {score}
      </h1>
      <button onClick={() => setScore(score + 1)}>Add one</button>
    </div>
  );
}

export default Counter;
```

react diff 算法会针对不同的节点类型应用不同的算法逻辑，其中 reconcileSingleElement 和 reconcileChildrenArray 两种逻辑较为常见。

- 当更新的节点为单一节点，不存在兄弟节点时，react 会调用 reconcileSingleElement 逻辑进行 diff。
- 当更新的节点同层存在兄弟节点时，react 会调用 reconcileChildrenArray 逻辑进行 diff。

为了复现上述两种场景，本文在 `Scoreboard.js` 中，分别设置了两种不同的元素排布:

- 场景一: `<Counter />` 与 `<button />` 处于同一层级，模拟了 `<Counter />` 组件改变，但同层仍存在兄弟节点的情况。
- 场景二: `<Counter />` 被包裹在 `<div />` 中，`<div />` 子节点中只包含单一元素 `<Counter />`。

在运行过程中，两种不同的场景将会进入 react diff 算法的两种不同逻辑中。

## 源码流程

diff 算法的目的是为了复用组件 FiberNode 对象。而组件 FiberNode 对象的创建或更新均发生在 reconcileChildren 逻辑中，因此我们具体关注 reconcileChildren 及其内部的运行流程。主要运行流程如下所示:

```js
- beginWork
  - updateFunctionComponent
    - renderWithHooks
      - Component // 组件函数: (props, secondArgs) => element
    - reconcileChildren
      - reconcileChildFibers
        - 👉 reconcileSingleElement
        - 👉 reconcileChildrenArray
        - ...
```

我们运行样例，并切换组件时，会发生组件的更新。函数组件的更新将会进入到 updateFunctionComponent 中，通过 renderWithHooks 执行函数组件并返回 JSX 的解析结果 react element。reconcileChildren 会根据这些 element 对象生成相应的 FiberNode。
在 reconcileChildren 中，react 会根据当前位置上一轮渲染是否已存在节点 (`current === null`) 判断是否调用挂载逻辑，但实际上 mountChildFibers 和 reconcileChildFibers 内部调用的都是 reconcileChildFibers。

```js
function reconcileChildren(
  current: Fiber | null,
  workInProgress: Fiber,
  nextChildren: any,
  renderLanes: Lanes,
) {
  if (current === null) {
    workInProgress.child = mountChildFibers(
      workInProgress,
      null,
      nextChildren,
      renderLanes,
    );
  } else {
    workInProgress.child = reconcileChildFibers(
      workInProgress,
      current.child,
      nextChildren,
      renderLanes,
    );
  }
}
```

进入到 reconcileChildFibers 中，react 会先对传入的元素(newChild)做一定的处理: 将顶层的 Fragment 元素统一转为数组元素存储在 children 属性中，方便后续处理。接着，react 会根据当前需要处理的 element 元素类型 (element.$$typeof) 调用不同的 FiberNode 生成逻辑。

```js
function reconcileChildFibers(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  newChild: any,
  lanes: Lanes,
): Fiber | null {
  // This function is not recursive.
  // If the top level item is an array, we treat it as a set of children,
  // not as a fragment. Nested arrays on the other hand will be treated as
  // fragment nodes. Recursion happens at the normal flow.

  // Handle top level unkeyed fragments as if they were arrays.
  // This leads to an ambiguity between <>{[...]}</> and <>...</>.
  // We treat the ambiguous cases above the same.
  const isUnkeyedTopLevelFragment =
    typeof newChild === 'object' &&
    newChild !== null &&
    newChild.type === REACT_FRAGMENT_TYPE &&
    newChild.key === null;
  if (isUnkeyedTopLevelFragment) {
    newChild = newChild.props.children;
  }

  // Handle object types
  if (typeof newChild === 'object' && newChild !== null) {
    switch (newChild.$$typeof) {
      case REACT_ELEMENT_TYPE:
        return placeSingleChild(
          reconcileSingleElement(
            returnFiber,
            currentFirstChild,
            newChild,
            lanes,
          ),
        );
      case REACT_PORTAL_TYPE:
        return placeSingleChild(
          reconcileSinglePortal(
            returnFiber,
            currentFirstChild,
            newChild,
            lanes,
          ),
        );
      case REACT_LAZY_TYPE:
        const payload = newChild._payload;
        const init = newChild._init;
        // TODO: This function is supposed to be non-recursive.
        return reconcileChildFibers(
          returnFiber,
          currentFirstChild,
          init(payload),
          lanes,
        );
    }

    if (isArray(newChild)) {
      return reconcileChildrenArray(
        returnFiber,
        currentFirstChild,
        newChild,
        lanes,
      );
    }

    if (getIteratorFn(newChild)) {
      return reconcileChildrenIterator(
        returnFiber,
        currentFirstChild,
        newChild,
        lanes,
      );
    }

    throwOnInvalidObjectType(returnFiber, newChild);
  }

  if (
    (typeof newChild === 'string' && newChild !== '') ||
    typeof newChild === 'number'
  ) {
    return placeSingleChild(
      reconcileSingleTextNode(
        returnFiber,
        currentFirstChild,
        '' + newChild,
        lanes,
      ),
    );
  }

  // Remaining cases are all treated as empty.
  return deleteRemainingChildren(returnFiber, currentFirstChild);
}
```

我们先以场景一为例，探究 reconcileChildrenArray 中的 diff 算法以及 key 在其中发挥的作用。

### reconcileChildrenArray

当 newChild 为数组类型时，react 会调用 reconcileChildrenArray 处理数组类型的 element 并返回相应的 FiberNode 对象。reconcileChildrenArray 具体流程如下所示，其中:

- workInProgress -> returnFiber
- current.child -> currentFirstChild
- elements(当前函数组件return的JSX解析结果) -> newChildren

reconcileChildrenArray 主要被分为了四个阶段，用于处理可能出现的四种不同的使用场景。

**第一阶段:**

第一阶段的作用是: 按位置顺序依次判断对应位置的新元素能否复用旧节点，该遍历过程直到遇到无法复用旧节点停止，并跳出第一阶段。
第一阶段同时从头部开始遍历旧节点链表和新元素数组，调用 updateSlot 生成新的 FiberNode 节点，其会基于 key 判断是否能够复用已有的 FiberNode 对象，若不能复用旧节点，则返回 null，从而跳出循环。

```js
function reconcileChildrenArray(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  newChildren: Array<*>,
  lanes: Lanes,
): Fiber | null {
  let resultingFirstChild: Fiber | null = null;
  let previousNewFiber: Fiber | null = null;

  let oldFiber = currentFirstChild;
  let lastPlacedIndex = 0;
  let newIdx = 0;
  let nextOldFiber = null;
  
  // 👇 第一阶段
  for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
    if (oldFiber.index > newIdx) {
      nextOldFiber = oldFiber;
      oldFiber = null;
    } else {
      nextOldFiber = oldFiber.sibling;
    }
    // 基于 oldFiber 和 newChildren 生成新的 FiberNode 节点
    // 若 key 匹配成功，则复用当前位置的旧节点；反之，则返回 null，即无法复用当前位置旧节点。
    const newFiber = updateSlot(
      returnFiber,
      oldFiber,
      newChildren[newIdx],
      lanes,
    );
    // 若没有匹配到相应可复用的节点，则跳出循环
    if (newFiber === null) {
      if (oldFiber === null) {
        oldFiber = nextOldFiber;
      }
      break;
    }
    if (shouldTrackSideEffects) {
      if (oldFiber && newFiber.alternate === null) {
        // We matched the slot, but we didn't reuse the existing fiber, so we
        // need to delete the existing child.
        deleteChild(returnFiber, oldFiber);
      }
    }
    lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
    if (previousNewFiber === null) {
      resultingFirstChild = newFiber;
    } else {
      previousNewFiber.sibling = newFiber;
    }
    previousNewFiber = newFiber;
    oldFiber = nextOldFiber;
  }

  // ......
  
  return resultingFirstChild;
}
```

分析 updateSlot 可知，它会优先比较新元素的 key 与旧节点的 key 是否一致，若一致则调用 updateElement 方法复用相应的 FiberNode 节点，若不一致则返回 null。

```js
function updateSlot(
  returnFiber: Fiber,
  oldFiber: Fiber | null,
  newChild: any,
  lanes: Lanes,
): Fiber | null {
  // Update the fiber if the keys match, otherwise return null.
  const key = oldFiber !== null ? oldFiber.key : null;

  // 处理文本节点
  if (
    (typeof newChild === 'string' && newChild !== '') ||
    typeof newChild === 'number'
  ) {
    // Text nodes don't have keys. If the previous node is implicitly keyed
    // we can continue to replace it without aborting even if it is not a text
    // node.
    if (key !== null) {
      return null;
    }
    return updateTextNode(returnFiber, oldFiber, '' + newChild, lanes);
  }

  if (typeof newChild === 'object' && newChild !== null) {
    switch (newChild.$$typeof) {
      case REACT_ELEMENT_TYPE: {
        if (newChild.key === key) {
          return updateElement(returnFiber, oldFiber, newChild, lanes);
        } else {
          return null;
        }
      }
      case REACT_PORTAL_TYPE: {
        if (newChild.key === key) {
          return updatePortal(returnFiber, oldFiber, newChild, lanes);
        } else {
          return null;
        }
      }
      case REACT_LAZY_TYPE: {
        const payload = newChild._payload;
        const init = newChild._init;
        return updateSlot(returnFiber, oldFiber, init(payload), lanes);
      }
    }

    if (isArray(newChild) || getIteratorFn(newChild)) {
      if (key !== null) {
        return null;
      }

      return updateFragment(returnFiber, oldFiber, newChild, lanes, null);
    }

    throwOnInvalidObjectType(returnFiber, newChild);
  }

  return null;
}
```

#### updateElement

updateElement 是 diff 算法判断是否复用并返回 FiberNode 对象的核心函数。当 `oldFiber.key === element.key` 匹配到旧节点对象后，传入 updateElement 中进一步判断元素类型 elementType 是否一致，若两者 key 一致的前提条件下，元素类型也一致，则意味着该节点对象可被复用。

```js
function updateElement(
  returnFiber: Fiber,
  current: Fiber | null,
  element: ReactElement,
  lanes: Lanes,
): Fiber {
  const elementType = element.type;
  if (elementType === REACT_FRAGMENT_TYPE) {
    return updateFragment(
      returnFiber,
      current,
      element.props.children,
      lanes,
      element.key,
    );
  }
  if (current !== null) {
    if (
      current.elementType === elementType ||
      // Keep this check inline so it only runs on the false path:
      (__DEV__
        ? isCompatibleFamilyForHotReloading(current, element)
        : false) ||
      // Lazy types should reconcile their resolved type.
      // We need to do this after the Hot Reloading check above,
      // because hot reloading has different semantics than prod because
      // it doesn't resuspend. So we can't let the call below suspend.
      (typeof elementType === 'object' &&
        elementType !== null &&
        elementType.$$typeof === REACT_LAZY_TYPE &&
        resolveLazy(elementType) === current.type)
    ) {
      // Move based on index
      const existing = useFiber(current, element.props);
      existing.ref = coerceRef(returnFiber, current, element);
      existing.return = returnFiber;
      if (__DEV__) {
        existing._debugSource = element._source;
        existing._debugOwner = element._owner;
      }
      return existing;
    }
  }
  // Insert
  const created = createFiberFromElement(element, returnFiber.mode, lanes);
  created.ref = coerceRef(returnFiber, current, element);
  created.return = returnFiber;
  return created;
}
```

#### useFiber

useFiber 接收一个目标 FiberNode 节点对象以及 element 中的 props，基于目标节点对象生成新的 FiberNode 对象。值得注意的是:

1. 若目标对象在 workInProgress 中已存在相应的节点，则会在该节点上同步最新的属性字段；反之，则会创建一个 workInProgress 并赋予相应的属性字段。
2. 并非所有属性字段都是被继承的，例如 index，sibling 会被初始化。
3. 复用 FiberNode 除了保留已有状态，继承副作用等之外，其还能减少 DOM 实例的反复创建。因为 FiberNode 有一个 stateNode 属性字段，它保存了已被创建生成的 DOM 实例。

```js
function useFiber(fiber: Fiber, pendingProps: mixed): Fiber {
  // We currently set sibling to null and index to 0 here because it is easy
  // to forget to do before returning it. E.g. for the single child case.
  const clone = createWorkInProgress(fiber, pendingProps);
  clone.index = 0;
  clone.sibling = null;
  return clone;
}

function createWorkInProgress(current: Fiber, pendingProps: any): Fiber {
  let workInProgress = current.alternate;
  if (workInProgress === null) {
    // We use a double buffering pooling technique because we know that we'll
    // only ever need at most two versions of a tree. We pool the "other" unused
    // node that we're free to reuse. This is lazily created to avoid allocating
    // extra objects for things that are never updated. It also allow us to
    // reclaim the extra memory if needed.
    workInProgress = createFiber(
      current.tag,
      pendingProps,
      current.key,
      current.mode,
    );
    workInProgress.elementType = current.elementType;
    workInProgress.type = current.type;
    workInProgress.stateNode = current.stateNode;
    workInProgress.alternate = current;
    current.alternate = workInProgress;
  } else {
    workInProgress.pendingProps = pendingProps;
    // Needed because Blocks store data on type.
    workInProgress.type = current.type;

    // We already have an alternate.
    // Reset the effect tag.
    workInProgress.flags = NoFlags;

    // The effects are no longer valid.
    workInProgress.subtreeFlags = NoFlags;
    workInProgress.deletions = null;

    if (enableProfilerTimer) {
      // We intentionally reset, rather than copy, actualDuration & actualStartTime.
      // This prevents time from endlessly accumulating in new commits.
      // This has the downside of resetting values for different priority renders,
      // But works for yielding (the common case) and should support resuming.
      workInProgress.actualDuration = 0;
      workInProgress.actualStartTime = -1;
    }
  }

  // Reset all effects except static ones.
  // Static effects are not specific to a render.
  workInProgress.flags = current.flags & StaticMask;
  workInProgress.childLanes = current.childLanes;
  workInProgress.lanes = current.lanes;

  workInProgress.child = current.child;
  workInProgress.memoizedProps = current.memoizedProps;
  workInProgress.memoizedState = current.memoizedState;
  workInProgress.updateQueue = current.updateQueue;

  // Clone the dependencies object. This is mutated during the render phase, so
  // it cannot be shared with the current fiber.
  const currentDependencies = current.dependencies;
  workInProgress.dependencies =
    currentDependencies === null
      ? null
      : {
          lanes: currentDependencies.lanes,
          firstContext: currentDependencies.firstContext,
        };

  // These will be overridden during the parent's reconciliation
  workInProgress.sibling = current.sibling;
  workInProgress.index = current.index;
  workInProgress.ref = current.ref;

  if (enableProfilerTimer) {
    workInProgress.selfBaseDuration = current.selfBaseDuration;
    workInProgress.treeBaseDuration = current.treeBaseDuration;
  }

  return workInProgress;
}
```

**第二阶段:**

若新元素全部遍历完毕生成了相应的 FiberNode，则清理剩余的旧节点。
> 这意味着第一阶段完全执行，所有新元素都复用了旧节点中的 FiberNode 对象。

```js
function reconcileChildrenArray(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  newChildren: Array<*>,
  lanes: Lanes,
): Fiber | null {
  // ......

  if (newIdx === newChildren.length) {
    // We've reached the end of the new children. We can delete the rest.
    deleteRemainingChildren(returnFiber, oldFiber);
    if (getIsHydrating()) {
      const numberOfForks = newIdx;
      pushTreeFork(returnFiber, numberOfForks);
    }
    return resultingFirstChild;
  }
  
  // ......
}
```

**第三阶段:**

若旧节点已遍历完毕，但仍有剩余的新元素没有被处理，则对剩余的 element 创建全新的 FiberNode 节点，返回新创建 FiberNode 的头节点。
> 这意味着第一阶段完全执行，所有旧节点中的 FiberNode 对象均被新元素复用。

```js
function reconcileChildrenArray(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  newChildren: Array<*>,
  lanes: Lanes,
): Fiber | null {
  // ......

  if (oldFiber === null) {
    // If we don't have any more existing children we can choose a fast path
    // since the rest will all be insertions.
    for (; newIdx < newChildren.length; newIdx++) {
      const newFiber = createChild(returnFiber, newChildren[newIdx], lanes);
      if (newFiber === null) {
        continue;
      }
      lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
      if (previousNewFiber === null) {
        resultingFirstChild = newFiber;
      } else {
        previousNewFiber.sibling = newFiber;
      }
      previousNewFiber = newFiber;
    }
    if (getIsHydrating()) {
      const numberOfForks = newIdx;
      pushTreeFork(returnFiber, numberOfForks);
    }
    return resultingFirstChild;
  }
  
  // ......
}
```

**第四阶段:**

当上述三个阶段均判断完成后，仍存在最后一种情况: 在遍历过程中，相应索引位置的旧节点与新元素 key 不想等，无法被复用，导致跳出了循环。此时旧节点仍有部分没有被遍历，元素数组中也仍存在部分元素未被遍历转化为 FiberNode 对象。

考虑到元素数组中剩余的元素所对应的 FiberNode 对象，可在旧节点中找到可复用的 FiberNode 对象。因此，react 将同层所有的旧节点按照各自的 key 值(默认为位置索引)映射为一个索引表，并通过 updateFromMap 优先从索引表中查找可复用的 FiberNode 节点。
索引表中所有已被用于复用的旧节点均会被保留，由于后续 react 会清空索引表中剩余的未被复用的旧节点，因此在 reconcileChildrenArray 中，react 会将已被复用的旧节点从索引表中剔除。

```js
function reconcileChildrenArray(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  newChildren: Array<*>,
  lanes: Lanes,
): Fiber | null {
  // ......

  // Add all children to a key map for quick lookups.
  const existingChildren = mapRemainingChildren(returnFiber, oldFiber);

  // Keep scanning and use the map to restore deleted items as moves.
  for (; newIdx < newChildren.length; newIdx++) {
    const newFiber = updateFromMap(
      existingChildren,
      returnFiber,
      newIdx,
      newChildren[newIdx],
      lanes,
    );
    if (newFiber !== null) {
      if (shouldTrackSideEffects) {
        // 👇 newFiber.alternate === current !== null 意味着当前创建的 FiberNode 是复用生成的，因此要保留原有的 current FiberNode。
        // 后续会将 existingChildren 中所有剩余的节点全部删除 (剩余节点意味着没有被复用，需要重新创建)，因此需要将已被复用的节点从索引表中剔除。
        if (newFiber.alternate !== null) {
          // The new fiber is a work in progress, but if there exists a
          // current, that means that we reused the fiber. We need to delete
          // it from the child list so that we don't add it to the deletion
          // list.
          existingChildren.delete(
            newFiber.key === null ? newIdx : newFiber.key,
          );
        }
      }
      lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
      if (previousNewFiber === null) {
        resultingFirstChild = newFiber;
      } else {
        previousNewFiber.sibling = newFiber;
      }
      previousNewFiber = newFiber;
    }
  }

  // 👇 未被复用的旧节点均被推入删除队列中
  if (shouldTrackSideEffects) {
    // Any existing children that weren't consumed above were deleted. We need
    // to add them to the deletion list.
    existingChildren.forEach(child => deleteChild(returnFiber, child));
  }

  if (getIsHydrating()) {
    const numberOfForks = newIdx;
    pushTreeFork(returnFiber, numberOfForks);
  }
  return resultingFirstChild;
}
```

#### mapRemainingChildren

mapRemainingChildren 的作用是生成一个索引表，通过 key 值可以快速访问到已有的 FiberNode 节点，方便相同 key 值下 FiberNode 节点对象的复用。

```js
function mapRemainingChildren(
  returnFiber: Fiber,
  currentFirstChild: Fiber,
): Map<string | number, Fiber> {
  // Add the remaining children to a temporary map so that we can find them by
  // keys quickly. Implicit (null) keys get added to this set with their index
  // instead.

  // 👇 索引表
  const existingChildren: Map<string | number, Fiber> = new Map();
  // 从旧节点头部开始遍历
  // 若 Fiber 节点存在 key 值，则将 key 作为索引键；
  // 若 Fiber 节点不存在 key 值，则将其节点索引位置 index 作为索引键；
  let existingChild = currentFirstChild;
  while (existingChild !== null) {
    if (existingChild.key !== null) {
      existingChildren.set(existingChild.key, existingChild);
    } else {
      existingChildren.set(existingChild.index, existingChild);
    }
    existingChild = existingChild.sibling;
  }
  return existingChildren;
}
```

#### updateFromMap

可以看到，在 updateFromMap 中，react 会优先根据 key 值从索引表中复用符合条件的 FiberNode 对象。

- 若匹配到可复用的 FiberNode 节点，则会在 updateElement 中调用 useFiber，接收匹配到的可复用 FiberNode 对象和 element.props，生成新的 FiberNode。值得注意的是，复用 FiberNode 对象本质上就是克隆，因此，复用的 FiberNode 节点会保留相应的状态信息。这也就是为什么不正确的 key 赋值会导致状态信息没有被清空或正确更新的原因。
- 若没有匹配到复用的 FiberNode 节点，则会在 updateElement 中调用 createFiberFromElement，创建一个全新的 FiberNode 对象，所有状态都是初始值。

```js
function updateFromMap(
  existingChildren: Map<string | number, Fiber>,
  returnFiber: Fiber,
  newIdx: number,
  newChild: any,
  lanes: Lanes,
): Fiber | null {
  // 文本类型
  if (
    (typeof newChild === 'string' && newChild !== '') ||
    typeof newChild === 'number'
  ) {
    // Text nodes don't have keys, so we neither have to check the old nor
    // new node for the key. If both are text nodes, they match.
    const matchedFiber = existingChildren.get(newIdx) || null;
    return updateTextNode(returnFiber, matchedFiber, '' + newChild, lanes);
  }

  if (typeof newChild === 'object' && newChild !== null) {
    switch (newChild.$$typeof) {
      case REACT_ELEMENT_TYPE: {
        // 优先从索引表中复用 FiberNode
        const matchedFiber =
          existingChildren.get(
            newChild.key === null ? newIdx : newChild.key,
          ) || null;
        return updateElement(returnFiber, matchedFiber, newChild, lanes);
      }
      // ... more cases ...
    }

    if (isArray(newChild) || getIteratorFn(newChild)) {
      const matchedFiber = existingChildren.get(newIdx) || null;
      return updateFragment(returnFiber, matchedFiber, newChild, lanes, null);
    }

    throwOnInvalidObjectType(returnFiber, newChild);
  }

  return null;
}
```

### reconcileChildrenArray 总结

至此，我们便完成了子元素为数组类型时的 diff 过程。该过程主要分为了四个阶段:

- 第一阶段: 从旧节点链表头部以及新元素数组头部开始，按照位置顺序依次比较两者 key 是否相同。若不相同则跳出第一阶段；若相同，则进一步判断 elementType 是否相同，若不相同则生成新的 FiberNode，反之则复用旧节点。
- 第二阶段: 在第一阶段完全执行的前提下，若新节点已全部构建，则删除剩余的旧节点(将节点对象添加进入待删除队列中)。
- 第三阶段: 在第一阶段完全执行的前提下，若旧节点全部被复用，但仍有剩余的新元素没有相应的 FiberNode，则创建节点对象。
- 第四阶段: 若第一阶段未完全执行，意味着存在某些位置旧节点无法复用的情况。此时旧链表以及新元素数组均未完成遍历，考虑到旧节点中仍有能够被新元素复用的节点对象(索引顺序不一定匹配)，因此采用了映射表的形式，通过 key 值索引可能被复用的节点，进一步结合 elementType 做判断。若 key 和 elementType 均符合，则复用旧节点产生 FiberNode，否则为新元素创建新的 FiberNode 对象。
- 未被复用的旧节点将全部推入删除队列中等待删除，被复用的旧节点会被保留。

### reconcileSingleElement

了解了 reconcileChildrenArray 的相关执行流程后，回过头来看 reconcileSingleElement 就相对轻松了。reconcileSingleElement 针对更新的元素为单一元素的情况，不管上一轮该位置的元素有多少，只要当前替换的元素为单一元素，那么就会进入到该逻辑内执行 diff 操作。

在 reconcileSingleElement 中，react 会从同级的头节点开始，依次遍历比较 `FiberNode.key` 和 `element.key`，若 key 值相同且 elementType 一致，则意味着上一轮该位置的 FiberNode 节点可被复用，调用 useFiber 方法基于目标节点生成相应的 FiberNode，保留状态以及相应的 DOM 实例。若 key 值不同或 elementType 不一致，意味着这些节点不会被复用，因此均被推入删除队列中等待删除。

对于复用节点的判断遍历到同级子节点末端结束，若在此期间匹配到了可复用节点，则返回生成的复用节点 existing。反之，则调用 createFiberFromElement 基于 element 对象创建相应的 FiberNode 节点。

```js
function reconcileSingleElement(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  element: ReactElement,
  lanes: Lanes,
): Fiber {
  const key = element.key;
  let child = currentFirstChild;
  while (child !== null) {
    if (child.key === key) {
      const elementType = element.type;
      if (elementType === REACT_FRAGMENT_TYPE) {
        if (child.tag === Fragment) {
          deleteRemainingChildren(returnFiber, child.sibling);
          const existing = useFiber(child, element.props.children);
          existing.return = returnFiber;
          if (__DEV__) {
            existing._debugSource = element._source;
            existing._debugOwner = element._owner;
          }
          return existing;
        }
      } else {
        if (
          child.elementType === elementType ||
          // Keep this check inline so it only runs on the false path:
          (__DEV__
            ? isCompatibleFamilyForHotReloading(child, element)
            : false) ||
          // Lazy types should reconcile their resolved type.
          // We need to do this after the Hot Reloading check above,
          // because hot reloading has different semantics than prod because
          // it doesn't resuspend. So we can't let the call below suspend.
          (typeof elementType === 'object' &&
            elementType !== null &&
            elementType.$$typeof === REACT_LAZY_TYPE &&
            resolveLazy(elementType) === child.type)
        ) {
          deleteRemainingChildren(returnFiber, child.sibling);
          const existing = useFiber(child, element.props);
          existing.ref = coerceRef(returnFiber, child, element);
          existing.return = returnFiber;
          if (__DEV__) {
            existing._debugSource = element._source;
            existing._debugOwner = element._owner;
          }
          return existing;
        }
      }
      // Didn't match.
      deleteRemainingChildren(returnFiber, child);
      break;
    } else {
      deleteChild(returnFiber, child);
    }
    child = child.sibling;
  }

  if (element.type === REACT_FRAGMENT_TYPE) {
    const created = createFiberFromFragment(
      element.props.children,
      returnFiber.mode,
      lanes,
      element.key,
    );
    created.return = returnFiber;
    return created;
  } else {
    const created = createFiberFromElement(element, returnFiber.mode, lanes);
    created.ref = coerceRef(returnFiber, currentFirstChild, element);
    created.return = returnFiber;
    return created;
  }
}
```