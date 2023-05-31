# 源码解析 useEffect 在父子组件中的执行顺序

## Demo 初探

在通读整篇文章之前，你需要先试着运行一下 [Demo](https://codesandbox.io/s/fu-zi-zu-jian-useeffectde-zhi-xing-shun-xu-qbpllf?file=/src/index.tsx)，观察控制台的 log 结果。

```js
// Parent
const Parent: React.FC<ParentProps> = () => {
  debugger;
  console.log(`${rerenderCount}: Parent Sync`);

  const [rerenderCount, setRerenderCount] = useState(1);

  useEffect(() => {
    debugger;
    console.log(`${rerenderCount}: Parent Mount`);

    return () => {
      debugger;
      console.log(`${rerenderCount}: Parent UnMount`);
    };
  });

  return (
    <>
      <Child rerenderCount={rerenderCount} />
      <button
        onClick={() => {
          setRerenderCount((rc) => rc + 1);
        }}
      >
        重渲染
      </button>
    </>
  );
};

// Child
const Child: React.FC<ChildProps> = (props) => {
  debugger;
  console.log(`${props.rerenderCount}: Child Sync`);

  useEffect(() => {
    debugger;
    console.log(`${props.rerenderCount}: Child Mount`);

    return () => {
      debugger;
      console.log(`${props.rerenderCount}: Child UnMount`);
    };
  });

  return null;
};
```

在这个 Demo 中，我们设计了一个父子组件嵌套的应用场景，并在 rendering 阶段，useEffect create 阶段以及 useEffect destroy 阶段均标记了 `console.log`。控制台在 mount 和 update 中的打印结果如下所示:

```js
// mount
1: Parent Sync 
1: Child Sync 
1: Child Mount 
1: Parent Mount 

// update
2: Parent Sync 
2: Child Sync 
1: Child UnMount 
1: Parent UnMount 
2: Child Mount 
2: Parent Mount 
```

我们可以发现几点关键:

1. 函数组件作用域内的同步代码最先运行，且遵循父组件先运行，子组件后运行的顺序。
2. useEffect 遵循子组件先运行，父组件后运行的顺序。
3. mount 过程中不运行 useEffect destroy 函数，它将在 update 阶段优先于第2轮 useEffect create 函数运行。

通过上述 Demo 我们可以了解到 React useEffect 在父子组件中大致的执行顺序，但仍存在疑问：为什么会优先执行子组件的 useEffect 逻辑？接下来，我们将从源码层面逐步调试并解析对应的执行流程。

## 源码初探

**注意:**

1. 在代码解析过程中，我们将隐藏一部分与当前目的无关的代码(例如错误校验，优先级调度等)。
2. 部分注释将根据解读进度，以备注的形式写在源码中。
3. 源码解析顺序按照 render -> commit 过程排布，且本文中只保留与 useEffect 相关的逻辑。

**调试环境搭建:**

本文 fork 了 [react-source-code-debug](https://github.com/neroneroffy/react-source-code-debug) 搭建的 React 18 源码的调试环境作为本地的源码调试平台。此外，借用 vscode 的 Debug 功能，我们可以很方便的在 vscode 上逐句跳转访问各个调用栈信息。

### render 阶段

render 阶段开始于 performSyncWorkOnRoot 或 performConcurrentWorkOnRoot 方法的调用。这取决于本次更新是同步更新还是异步更新。因此，我们调试的时候，首先将调用栈定位到 performConcurrentWorkOnRoot。

由于我们只关心 useEffect 在 render 阶段是怎么被执行的，因此我们不会过多的赘述其他方法的细节及流程，所有相关流程均会以树表形式简略展示:

```js
- performConcurrentWorkOnRoot // 入口
  - renderRootSync
    - workLoopSync
      - performUnitOfWork
        - 👉 beginWork
          - mountIndeterminateComponent
            - 👉 renderWithHooks
              - 👉 Component // 这个阶段调用或注册相应的 hooks
            - 👉 reconcileChildren
        - 👉 completeUnitOfWork
          - completeWork
```

在上述流程中，我们重点关注 👉 指向的几处函数调用的具体执行逻辑。

#### beginWork

```js
function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}
```

进入到 workLoopSync 作用域中，发现该函数主要循环调用了 performUnitOfWork 直到 workInProgress 为 null，跳出循环，其中:

- workInProgress: 指针，指向当前正在构建的 Fiber 节点。
- workInProgress.alternate: 指针，指向当前构建节点对应视图中正被展示的 Fiber 节点位置。

当 `workInProgress === null` 时，意味着已完成最新 Fiber 树的构建。

在 performUnitOfWork 中，有两个相对重要的出入口:

1. beginWork
2. completeUnitOfWork

```js
function performUnitOfWork(unitOfWork: Fiber): void {
  const current = unitOfWork.alternate;

  let next;
  if (enableProfilerTimer && (unitOfWork.mode & ProfileMode) !== NoMode) {
    // ......
  } else {
    next = beginWork(current, unitOfWork, subtreeRenderLanes);
  }

  unitOfWork.memoizedProps = unitOfWork.pendingProps;
  if (next === null) {
    // If this doesn't spawn new work, complete the current work.
    completeUnitOfWork(unitOfWork);
  } else {
    workInProgress = next;
  }
  // ......
}
```

beiginWork 接收了三个参数，其中 workInProgress 表示当前内存中正在处理的 Fiber 节点指针位置，current 表示 workInProgress 指针位置在视图中所对应的 Fiber 节点(可以理解为上一轮渲染的 Fiber 记录)。

通过 `current !== null` 我们可以确定当前的渲染是 mount 还是 update。

- `current === null` 时，说明上一轮渲染中不存在相应的 Fiber 节点，因此此次渲染属于创建挂载阶段(mount)。将 didReceiveUpdate 赋值为 false，表明不需要采用 update 的逻辑进行渲染。
- `current !== null` 时，意味着该 Fiber 节点在上一轮渲染中已存在，属于节点更新的范畴。此处对比该节点位置上一轮的 props(oldProps) 和 这一轮的 props(newProps) 是否一致，判断是否需要对该节点进行更新。若需要更新，则将 didReceiveUpdate 赋值为 true。

接着，我们根据当前需要构建的 Fiber 节点类型(workInProgress.tag)，调用合适的组件构建方法。

```js
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  if (current !== null) {
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;

    if (
      oldProps !== newProps ||
      hasLegacyContextChanged() ||
      // Force a re-render if the implementation changed due to hot reload:
      (__DEV__ ? workInProgress.type !== current.type : false)
    ) {
      // If props or context changed, mark the fiber as having performed work.
      // This may be unset if the props are determined to be equal later (memo).
      didReceiveUpdate = true;
    } else {
      // ......
    }
  } else {
    didReceiveUpdate = false;
    // ......
  }
  // ......

  switch (workInProgress.tag) {
    case IndeterminateComponent: {
      return mountIndeterminateComponent(
        current,
        workInProgress,
        workInProgress.type,
        renderLanes,
      );
    }
    case FunctionComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      return updateFunctionComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderLanes,
      );
    }
    case ClassComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      return updateClassComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderLanes,
      );
    }
    // ... more cases ...
  }
}
```

这里我们匹配到了 IndeterminateComponent 组件类型，因此执行了 mountIndeterminateComponent 方法，其内部核心方法为 renderWithHooks 和 reconcileChildren。其中:

- renderWithHooks 的作用是调用函数组件，执行组件内部的方法，并最终返回相应的 react element 对象。
- reconcileChildren 的作用是基于 react element 对象，生成相应的 FiberNode 对象。

```js
function mountIndeterminateComponent(
  _current,
  workInProgress,
  Component,
  renderLanes,
) {
  const props = workInProgress.pendingProps;

  // 👇 标记组件渲染的起点
  if (enableSchedulingProfiler) {
    markComponentRenderStarted(workInProgress);
  }
  if (__DEV__) {
    // ......
  } else {
    // 👇 main entry point
    value = renderWithHooks(
      null,
      workInProgress,
      Component,
      props,
      context,
      renderLanes,
    );
    hasId = checkDidRenderIdHook();
  }

  // 👇 标记组件渲染的终点
  if (enableSchedulingProfiler) {
    markComponentRenderStopped();
  }
  // ......

  // 👇 确定组件类型，并调用 reconcileChildren 生成 Fiber 节点。
  if (
    !disableModulePatternComponents &&
    typeof value === 'object' &&
    value !== null &&
    typeof value.render === 'function' &&
    value.$$typeof === undefined
  ) {
    workInProgress.tag = ClassComponent;
    // 类组件相关逻辑
    // ......
  } else {
    workInProgress.tag = FunctionComponent;
    // 函数组件相关逻辑
    // ......
    reconcileChildren(null, workInProgress, value, renderLanes);
    return workInProgress.child;
  }
}
```

#### renderWithHooks

renderWithHooks 接收 current, workInProgress, Component, props, secondArgs 等众多参数:

- current: 当前指针位置对应的上一轮 Fiber 节点对象。
- workInProgress: 当前指针指向的正在构建的 Fiber 节点对象。
- Component: 定义的组件函数，被存储在 workInProgress.type 中。

> Component(props, secondArg) 即为函数组件的执行过程。

```js
function renderWithHooks<Props, SecondArg>(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: (p: Props, arg: SecondArg) => any,
  props: Props,
  secondArg: SecondArg,
  nextRenderLanes: Lanes,
): any {
  renderLanes = nextRenderLanes;
  currentlyRenderingFiber = workInProgress;

  // 👇 初始化 workInProgress 各字段
  workInProgress.memoizedState = null;
  workInProgress.updateQueue = null;
  workInProgress.lanes = NoLanes;

  if (__DEV__) {
    // ......
  } else {
    // 👇 根据 mount 或 update 获取不同逻辑的 hooks 函数，挂载到全局 ReactCurrentDispatcher 中使用。
    ReactCurrentDispatcher.current =
      current === null || current.memoizedState === null
        ? HooksDispatcherOnMount
        : HooksDispatcherOnUpdate;
  }

  // 🚀 调用函数组件的 function 生成对应的 Fiber 节点。
  let children = Component(props, secondArg);

  // 👇 此处处理在 render 阶段引发的重渲染。
  // Check if there was a render phase update
  if (didScheduleRenderPhaseUpdateDuringThisPass) {
    // Keep rendering in a loop for as long as render phase updates continue to
    // be scheduled. Use a counter to prevent infinite loops.
    let numberOfReRenders: number = 0;
    do {
      didScheduleRenderPhaseUpdateDuringThisPass = false;
      localIdCounter = 0;

      if (numberOfReRenders >= RE_RENDER_LIMIT) {
        throw new Error(
          'Too many re-renders. React limits the number of renders to prevent ' +
            'an infinite loop.',
        );
      }

      numberOfReRenders += 1;
      // Start over from the beginning of the list
      currentHook = null;
      workInProgressHook = null;
      workInProgress.updateQueue = null;

      ReactCurrentDispatcher.current = __DEV__
        ? HooksDispatcherOnRerenderInDEV
        : HooksDispatcherOnRerender;

      children = Component(props, secondArg);
    } while (didScheduleRenderPhaseUpdateDuringThisPass);
  }
  // ......
  return children;
}
```

可以发现，renderWithHooks 执行初期，保存了 workInProgress 的指针副本，并及时的清空了 workInProgress 指针各属性字段。随后，根据当前 Fiber 节点属于 mount 还是 update 调用不同的逻辑获取 hooks 函数，挂载到全局 ReactCurrentDispatcher 中方便后续调用。一切准备就绪后，React 调用了 `Component(props, secondArg)`，触发了组件 render 函数(函数组件即执行其内部作用域内的代码)。

**注意: 此处执行了第一次 console.log("Parent Sync")，表明执行到了父组件作用域内的代码逻辑。**

#### Component

在 Component 函数调用中，将会运行处理 hooks 相关的注册和调用逻辑。React 遇到 useState 时，会进入到 ReactHooks.js 这个文件中。可以发现，useState 实际是一个封装，它接收一个 initialState，并返回 state 和相应的 dispatch 方法。真正的 useState 方法从全局的 ReactCurrentDispatcher 对象中获取。

```js
// ReactHooks.js
function useState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  const dispatcher = resolveDispatcher();
  return dispatcher.useState(initialState);
}

function resolveDispatcher() {
  const dispatcher = ReactCurrentDispatcher.current;
  return ((dispatcher: any): Dispatcher);
}
```

由于当前处于 mount 阶段，所以之前赋值 ReactCurrentDispatcher 的时候调用的是 HooksDispatcherOnMount。

```js
ReactCurrentDispatcher.current =
  current === null || current.memoizedState === null
    ? HooksDispatcherOnMount
    : HooksDispatcherOnUpdate;
```

mount 阶段中 `dispatcher.useState(initialState)` 会对应到 mounState 逻辑。它首先调用了 mountWorkInProgressHook 方法，创建了一个 Hook 对象，并将其链接到 workInProgressHook 这一对象上。从 Hook 类型中我们可知，Hook 对象本质上是一个链表结构，包含 next 字段指向后一个 Hook 链表对象。此外，当前节点中所涉及的 Hook 对象都会被链接到 workInProgressHook 中保存，并存储在 currentlyRenderingFiber 的 memoizedState 字段上。

> 结合前文可知，currentlyRenderingFiber 实际就是 workInProgress 对象。

```js
function mountWorkInProgressHook(): Hook {
  const hook: Hook = {
    memoizedState: null,

    baseState: null,
    baseQueue: null,
    queue: null,

    next: null,
  };

  if (workInProgressHook === null) {
    // This is the first hook in the list
    currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
  } else {
    // Append to the end of the list
    workInProgressHook = workInProgressHook.next = hook;
  }
  return workInProgressHook;
}
```

hook 对象初始化完成后，workInProgressHook 会把当前指针指向该新创建的 hook，并返回。接着执行 useState 的初始化操作，若传入的 initialState 参数为函数类型，则执行该 initialState 初始化函数，并将返回值保存到 hook 对象的 memoizedState 和 baseState 属性字段上，否则则直接执行赋值操作。

```js
function mountState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  const hook = mountWorkInProgressHook();
  if (typeof initialState === 'function') {
    // $FlowFixMe: Flow doesn't like mixed types
    initialState = initialState();
  }
  hook.memoizedState = hook.baseState = initialState;
  const queue: UpdateQueue<S, BasicStateAction<S>> = {
    pending: null,
    interleaved: null,
    lanes: NoLanes,
    dispatch: null,
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: (initialState: any),
  };
  hook.queue = queue;
  const dispatch: Dispatch<
    BasicStateAction<S>,
  > = (queue.dispatch = (dispatchSetState.bind(
    null,
    currentlyRenderingFiber,
    queue,
  ): any));
  return [hook.memoizedState, dispatch];
}
```

同样的，React 处理代码中的 useEffect，也会通过 resolveDispatcher 从全局的 ReactCurrentDispatcher 中获取。

```js
function useEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null,
): void {
  const dispatcher = resolveDispatcher();
  return dispatcher.useEffect(create, deps);
}

function mountEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null,
): void {
  if (
    __DEV__ &&
    enableStrictEffects &&
    (currentlyRenderingFiber.mode & StrictEffectsMode) !== NoMode
  ) {
    return mountEffectImpl(
      MountPassiveDevEffect | PassiveEffect | PassiveStaticEffect,
      HookPassive,
      create,
      deps,
    );
  } else {
    return mountEffectImpl(
      PassiveEffect | PassiveStaticEffect,
      HookPassive,
      create,
      deps,
    );
  }
}
```

可以看到，在 mount 阶段中，useEffect 内部调用了 mountEffectImpl 这个函数逻辑。mountEffectImpl 初始化了一个 hook 对象，并在 currentlyRenderingFiber (workInProgress) 指针对象中添加了一个 flags 标记。接着，内部调用了 pushEffect 方法，生成了一个 Effect 对象保存本次 useEffect 定义的副作用(即 create 和 destroy 函数)。最后将这个 Effect 副作用对象保存到 hook 对象的 memoizedState 字段中。

```js
function mountEffectImpl(fiberFlags, hookFlags, create, deps): void {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  currentlyRenderingFiber.flags |= fiberFlags;
  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags,
    create,
    undefined,
    nextDeps,
  );
}

function pushEffect(tag, create, destroy, deps) {
  const effect: Effect = {
    tag,
    create,
    destroy,
    deps,
    // Circular
    next: (null: any),
  };
  let componentUpdateQueue: null | FunctionComponentUpdateQueue = (currentlyRenderingFiber.updateQueue: any);
  if (componentUpdateQueue === null) {
    componentUpdateQueue = createFunctionComponentUpdateQueue();
    currentlyRenderingFiber.updateQueue = (componentUpdateQueue: any);
    componentUpdateQueue.lastEffect = effect.next = effect;
  } else {
    const lastEffect = componentUpdateQueue.lastEffect;
    if (lastEffect === null) {
      componentUpdateQueue.lastEffect = effect.next = effect;
    } else {
      const firstEffect = lastEffect.next;
      lastEffect.next = effect;
      effect.next = firstEffect;
      componentUpdateQueue.lastEffect = effect;
    }
  }
  return effect;
}

function createFunctionComponentUpdateQueue(): FunctionComponentUpdateQueue {
  return {
    lastEffect: null,
    stores: null,
  };
}
```

不难发现，effect 对象其实也是一个链表结构，更特别的，它是一个环形链表。pushEffect 方法除了生成 effect 对象，将其保存在 hook.memoizedState 之外，还会在 `currentlyRenderingFiber.updateQueue` 更新队列中推入相应的副作用，其 lastEffect 属性字段始终指向 effect 环形链表对象的最后一个节点。
> 环形链表中，指针指向最后一个 effect 链表节点的好处在于: 可以直接通过 `effect.next` 访问到链表的首个节点，确保可以快速获取到链表的首尾节点指针位置。

执行完所有 hooks 以及作用域内代码后，Function Component 就会执行到 `return JSX` 这部分语句，React 在这里会调用 createElement 方法并返回相应的 Fiber Node 节点。关于 createElement 及其内部的实现细节在本文就不过多赘述，React 具体构建 Fiber Tree 并渲染 DOM Tree 的过程将在其他文章中展开讲解。

至此！`Component(props, secondArg)` 逻辑全部执行完毕，在本文 Demo 中，它主要做了以下几件事情:

1. 按顺序正常执行作用域内的代码
2. 遇到 hooks 时，调用 ReactCurrentDispatcher 对象中对应的 hooks 方法，主要涉及初始化和副作用注册等。
3. 返回当前组件的 element 对象(JSX 元素的解析结果，包含多个属性字段的对象)。

#### reconcileChildren

我们将 `Component(props, secondArg)` 返回的 element 对象传入 reconcileChildren 中，reconcileChildren 内部会根据 mount / update 以及 element 对象元素类型等生成并返回相应的 FiberNode 对象。并挂载到 workInProgress.child 属性字段下。
> 本文不具体介绍 reconcileChildren 内部是如何根据 element 对象生成相应的 FiberNode 。

```js
function reconcileChildren(
  current: Fiber | null,
  workInProgress: Fiber,
  nextChildren: any,
  renderLanes: Lanes,
) {
  if (current === null) {
    // If this is a fresh new component that hasn't been rendered yet, we
    // won't update its child set by applying minimal side-effects. Instead,
    // we will add them all to the child before it gets rendered. That means
    // we can optimize this reconciliation pass by not tracking side-effects.
    workInProgress.child = mountChildFibers(
      workInProgress,
      null,
      nextChildren,
      renderLanes,
    );
  } else {
    // If the current child is the same as the work in progress, it means that
    // we haven't yet started any work on these children. Therefore, we use
    // the clone algorithm to create a copy of all the current children.

    // If we had any progressed work already, that is invalid at this point so
    // let's throw it out.
    workInProgress.child = reconcileChildFibers(
      workInProgress,
      current.child,
      nextChildren,
      renderLanes,
    );
  }
}
```

至此，beginWork - mountIndeterminateComponent 方法调用完毕，并将 workInProgress.child 返回。
返回 workInProgress 的子 FiberNode 节点的原因: React 需要递归构建 Fiber Tree，workInProgress 作为一个指针，需要保证指向最新构建的 FiberNode 节点对象，方便后续生成的 FiberNode 对象能够正确地链接到 workInProgress 链表中，返回 workInProgress.child，其本质和链表中 `p = p.next` 是一样的。在 performUnitOfWork 中也可以看到:

```js
function performUnitOfWork(unitOfWork: Fiber): void {
  // ......
  const current = unitOfWork.alternate;
  let next = = beginWork(current, unitOfWork, subtreeRenderLanes);
  unitOfWork.memoizedProps = unitOfWork.pendingProps;

  if (next === null) {
    // If this doesn't spawn new work, complete the current work.
    completeUnitOfWork(unitOfWork);
  } else {
    workInProgress = next;
  }

  ReactCurrentOwner.current = null;
}
```

- 当 `next === null` 时，意味着遍历到叶子结点了，则进入 completeUnitOfWork 逻辑。
- 当 `next !== null` 时，意味着还没有遍历到叶子结点，需要将 workInProgress 指针指向当前最新构建的节点，即 `workInProgress = workInProgress.child`，进而推动 Fiber Tree 的构建。

#### completeUnitOfWork

当 workInProgress 遍历到叶子结点后，react 就会开始回溯调用 completeUnitOfWork 构建真实的 DOM 实例。其核心逻辑在 completeWork 中，completeWork 会根据传入的 workInProgress 调用 createInstance 方法生成具体的 DOM 实例。当一轮 completeWork 完成并跳出后，会移动 workInProgress 指针指向兄弟节点，开启新一轮 completeWork。若同级兄弟节点均完成了 DOM 实例构建任务，则 workInProgress 将返回父节点。

```js
function completeUnitOfWork(unitOfWork: Fiber): void {
  // 👇 获取当前 FiberNode 指针对象 (workInProgress)
  let completedWork = unitOfWork;
  do {
    // 👇 获取视图中展示节点对应的 FiberNode(current) 以及 workInProgress 的父节点 FiberNode
    const current = completedWork.alternate;
    const returnFiber = completedWork.return;

    // Check if the work completed or if something threw.
    if ((completedWork.flags & Incomplete) === NoFlags) {
      setCurrentDebugFiberInDEV(completedWork);
      let next;
      if (
        !enableProfilerTimer ||
        (completedWork.mode & ProfileMode) === NoMode
      ) {
        // 👇 核心逻辑
        next = completeWork(current, completedWork, subtreeRenderLanes);
      } else {
        // ......
      }
      // ......

      if (next !== null) {
        // Completing this fiber spawned new work. Work on that next.
        workInProgress = next;
        return;
      }
    } else {
      // ......
    }

    // 👇 将当前指针切换到兄弟节点。若存在兄弟节点，则遍历兄弟节点，否则则回溯到父组件。
    const siblingFiber = completedWork.sibling;
    if (siblingFiber !== null) {
      // If there is more work to do in this returnFiber, do that next.
      // 👇 workInProgress 切换到兄弟节点后，需要跳出 completeUnitOfWork，继续执行 performUnitOfWork 构建兄弟组件的 Fiber，回溯过程中调用 completeUnitOfWork 创建 DOM 实例。
      workInProgress = siblingFiber;
      return;
    }
    // Otherwise, return to the parent
    completedWork = returnFiber;
    // Update the next thing we're working on in case something throws.
    workInProgress = completedWork;
  } while (completedWork !== null);

  // We've reached the root.
  if (workInProgressRootExitStatus === RootInProgress) {
    workInProgressRootExitStatus = RootCompleted;
  }
}
```

completeWork 中，基于当前 FiberNode 节点的 tag 标签，匹配不同的逻辑生成相应的真实 DOM 节点。除了生成当前 FiberNode 节点的 DOM 实例外，completeWork 中还调用了 appendAllChildren 方法，将 workInProgress 下所有的 children 节点已生成的 DOM Instance 添加到当前 DOM Instance 之下。
> 值得注意的是，这些 DOM 实例并没有直接被渲染到页面上，而是保存在了当前 FiberNode 对象的 stateNode 属性字段中 (`workInProgress.stateNode = instance`)。

```js
function completeWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  const newProps = workInProgress.pendingProps;
  popTreeContext(workInProgress);
  switch (workInProgress.tag) {
    case HostComponent: {
      popHostContext(workInProgress);
      const rootContainerInstance = getRootHostContainer();
      const type = workInProgress.type;
      if (current !== null && workInProgress.stateNode != null) {
        // ......
      } else {
        // ......
        if (wasHydrated) {
          // ......
        } else {
          // 👇 根据 FiberNode 节点生成 DOM 实例
          const instance = createInstance(
            type,
            newProps,
            rootContainerInstance,
            currentHostContext,
            workInProgress,
          );

          // 👇 将 workInProgress 下所有的 children 节点已生成的 DOM Instance 添加到当前 DOM Instance 之下
          appendAllChildren(instance, workInProgress, false, false);

          // 👇 当前 FiberNode 的 DOM 实例保存在 stateNode 属性字段下
          workInProgress.stateNode = instance;

          // ......
        }

        if (workInProgress.ref !== null) {
          // If there is a ref on a host node we need to schedule a callback
          markRef(workInProgress);
        }
      }
      bubbleProperties(workInProgress);
      return null;
    }
    // ... more cases ...
  }
}
```

### commit 阶段

在 performConcurrentWorkOnRoot 中，当我们执行完 renderRootConcurrent 或 renderRootSync 后，会返回最终跳出渲染后的状态。通常当我们成功完成渲染后，`exitStatus === RootCompleted === 5`。
> React 会对不同的 exitStatus 返回值调用不同的逻辑，我们将重点关注 `exitStatus === RootCompleted` 下的执行逻辑。

```js
function performConcurrentWorkOnRoot(root, didTimeout) {
  // ......

  let exitStatus = shouldTimeSlice
    ? renderRootConcurrent(root, lanes)
    : renderRootSync(root, lanes);
  if (exitStatus !== RootInProgress) {
    if (exitStatus === RootErrored) {
      // ......
    }
    if (exitStatus === RootFatalErrored) {
      // ......
    }
    if (exitStatus === RootDidNotComplete) {
      // ......
    } else {
      // The render completed.
      const renderWasConcurrent = !includesBlockingLane(root, lanes);
      const finishedWork: Fiber = (root.current.alternate: any);
      if (
        renderWasConcurrent &&
        !isRenderConsistentWithExternalStores(finishedWork)
      ) {
        // A store was mutated in an interleaved event. Render again,
        // synchronously, to block further mutations.
        exitStatus = renderRootSync(root, lanes);

        // We need to check again if something threw
        if (exitStatus === RootErrored) {
          const errorRetryLanes = getLanesToRetrySynchronouslyOnError(root);
          if (errorRetryLanes !== NoLanes) {
            lanes = errorRetryLanes;
            exitStatus = recoverFromConcurrentError(root, errorRetryLanes);
            // We assume the tree is now consistent because we didn't yield to any
            // concurrent events.
          }
        }
        if (exitStatus === RootFatalErrored) {
          const fatalError = workInProgressRootFatalError;
          prepareFreshStack(root, NoLanes);
          markRootSuspended(root, lanes);
          ensureRootIsScheduled(root, now());
          throw fatalError;
        }
      }

      // We now have a consistent tree. The next step is either to commit it,
      // or, if something suspended, wait to commit it after a timeout.
      root.finishedWork = finishedWork;
      root.finishedLanes = lanes;
      finishConcurrentRender(root, exitStatus, lanes);
    }
  }

  ensureRootIsScheduled(root, now());
  if (root.callbackNode === originalCallbackNode) {
    // The task node scheduled for this root is the same one that's
    // currently executed. Need to return a continuation.
    return performConcurrentWorkOnRoot.bind(null, root);
  }
  return null;
}
```

在 `exitStatus === RootCompleted` 时，其内部还要根据当前渲染是否并行，分别调用不同的逻辑。若并行渲染，则还需要额外的进行同步渲染保证正确同步未来的变化。
分析代码可知，`root.finishedWork === root.current.alternate === workInProgress`，也就是最新的内存树指针，在已经完成 Fiber 树构建的前提下，workInProgress 此时指向 root 节点。我们将 root 对象和 exitStatus 传入 finishConcurrentRender。在 finishConcurrentRender 中，同样也是根据构建状态结果执行相应的逻辑。

由于我们只关心 useEffect 在 commit 阶段是怎么被执行的，因此我们不会过多的赘述其他方法的细节及流程，所有相关流程均会以树表形式简略展示:

```js
- performConcurrentWorkOnRoot // 入口
// --- render ---
  - renderRootSync
    - workLoopSync
      - performUnitOfWork
        - 👉 beginWork
          - mountIndeterminateComponent
            - 👉 renderWithHooks
              - 👉 Component // 这个阶段调用或注册相应的 hooks
            - 👉 reconcileChildren
        - 👉 completeUnitOfWork
          - completeWork
// --- commit ---
  - finishConcurrentRender
    - commitRoot
      - 👉 commitRootImpl
        - 👉 scheduleCallback -> flushPassiveEffects
          - flushPassiveEffectsImpl
            - commitPassiveUnmountEffects
              - 👉 commitPassiveUnmountEffects_begin
                - commitPassiveUnmountEffects_complete
                  - commitPassiveUnmountOnFiber
                    - 👉 commitHookEffectListUnmount
            - commitPassiveMountEffects
              - 👉 commitPassiveMountEffects_begin
                - commitPassiveMountEffects_complete
                  - commitPassiveMountOnFiber
                    - 👉 commitHookEffectListMount
        - commitBeforeMutationEffects
        - commitMutationEffects
        - commitLayoutEffects

```

在上述流程中，我们重点关注 👉 指向的几处函数调用的具体执行逻辑。

#### commitRootImpl

commitRootImpl 是整个 commit 阶段的核心，它包括了:

1. 上一轮副作用的清空阶段: `do { flushPassiveEffects() } while (rootWithPendingPassiveEffects !== null)`
2. effects 调度阶段: `scheduleCallback(NormalSchedulerPriority, () => { flushPassiveEffects(); return null; })`
3. commitBeforeMutationEffects 阶段
4. commitMutationEffects 阶段
5. `root.current = finishedWork` 阶段: 切换组件树指针
6. commitLayoutEffects 阶段
7. requestPaint: 请求重绘。
8. ......

```js
function commitRootImpl(
  root: FiberRoot,
  recoverableErrors: null | Array<mixed>,
  transitions: Array<Transition> | null,
  renderPriorityLevel: EventPriority,
) {
  do {
    // `flushPassiveEffects` will call `flushSyncUpdateQueue` at the end, which
    // means `flushPassiveEffects` will sometimes result in additional
    // passive effects. So we need to keep flushing in a loop until there are
    // no more pending effects.
    flushPassiveEffects();
  } while (rootWithPendingPassiveEffects !== null);
  const finishedWork = root.finishedWork;
  // ......

  // If there are pending passive effects, schedule a callback to process them.
  // Do this as early as possible, so it is queued before anything else that
  // might get scheduled in the commit phase. (See #16714.)
  if (
    (finishedWork.subtreeFlags & PassiveMask) !== NoFlags ||
    (finishedWork.flags & PassiveMask) !== NoFlags
  ) {
    if (!rootDoesHavePassiveEffects) {
      rootDoesHavePassiveEffects = true;
      pendingPassiveEffectsRemainingLanes = remainingLanes;
      pendingPassiveTransitions = transitions;
      // 👇 useEffect 等异步逻辑调度时机
      scheduleCallback(NormalSchedulerPriority, () => {
        flushPassiveEffects();
        return null;
      });
    }
  }

  // Check if there are any effects in the whole tree.
  const subtreeHasEffects =
    (finishedWork.subtreeFlags &
      (BeforeMutationMask | MutationMask | LayoutMask | PassiveMask)) !==
    NoFlags;
  const rootHasEffect =
    (finishedWork.flags &
      (BeforeMutationMask | MutationMask | LayoutMask | PassiveMask)) !==
    NoFlags;
  if (subtreeHasEffects || rootHasEffect) {
    // ......

    // The commit phase is broken into several sub-phases. We do a separate pass
    // of the effect list for each phase: all mutation effects come before all
    // layout effects, and so on.

    // The first phase a "before mutation" phase. We use this phase to read the
    // state of the host tree right before we mutate it. This is where
    // getSnapshotBeforeUpdate is called.
    const shouldFireAfterActiveInstanceBlur = commitBeforeMutationEffects(
      root,
      finishedWork,
    );
    // ......

    // The next phase is the mutation phase, where we mutate the host tree.
    commitMutationEffects(root, finishedWork, lanes);
    // ......

    // The work-in-progress tree is now the current tree. This must come after
    // the mutation phase, so that the previous tree is still current during
    // componentWillUnmount, but before the layout phase, so that the finished
    // work is current during componentDidMount/Update.
    root.current = finishedWork;
    // ......

    // The next phase is the layout phase, where we call effects that read
    // the host tree after it's been mutated. The idiomatic use case for this is
    // layout, but class component lifecycles also fire here for legacy reasons.
    commitLayoutEffects(finishedWork, root, lanes);
    // ......

    // Tell Scheduler to yield at the end of the frame, so the browser has an
    // opportunity to paint.
    requestPaint();
    // ......
  } else {
    // No effects.
    root.current = finishedWork;
  }
  // ......

  // Always call this before exiting `commitRoot`, to ensure that any
  // additional work on this root is scheduled.
  ensureRootIsScheduled(root, now());

  // If the passive effects are the result of a discrete render, flush them
  // synchronously at the end of the current task so that the result is
  // immediately observable. Otherwise, we assume that they are not
  // order-dependent and do not need to be observed by external systems, so we
  // can wait until after paint.
  if (
    includesSomeLane(pendingPassiveEffectsLanes, SyncLane) &&
    root.tag !== LegacyRoot
  ) {
    flushPassiveEffects();
  }
  // ......

  // If layout work was scheduled, flush it now.
  flushSyncCallbacks();
  return null;
}
```

#### Scheduler_scheduleCallback

Scheduler 调度方法统一分离在 `Scheduler.js` 中，负责任务的管理分配和执行。
ScheduleCallback 方法会创建一个新的 Task 任务对象 (newTask)，并将新的任务对象 push 到 taskQueue 任务队列中。
> newTask 任务对象中包含了任务的优先级、id、任务回调、起止时间等。

延迟的任务将会被推入 timerQueue 队列中等待。

```js
function unstable_scheduleCallback(priorityLevel, callback, options) {
  var currentTime = getCurrentTime();
  var startTime;
  if (typeof options === 'object' && options !== null) {
    var delay = options.delay;
    if (typeof delay === 'number' && delay > 0) {
      startTime = currentTime + delay;
    } else {
      startTime = currentTime;
    }
  } else {
    startTime = currentTime;
  }

  var timeout;
  switch (priorityLevel) {
    case ImmediatePriority:
      timeout = IMMEDIATE_PRIORITY_TIMEOUT;
      break;
    case UserBlockingPriority:
      timeout = USER_BLOCKING_PRIORITY_TIMEOUT;
      break;
    case IdlePriority:
      timeout = IDLE_PRIORITY_TIMEOUT;
      break;
    case LowPriority:
      timeout = LOW_PRIORITY_TIMEOUT;
      break;
    case NormalPriority:
    default:
      timeout = NORMAL_PRIORITY_TIMEOUT;
      break;
  }

  var expirationTime = startTime + timeout;

  var newTask = {
    id: taskIdCounter++,
    callback,
    priorityLevel,
    startTime,
    expirationTime,
    sortIndex: -1,
  };
  if (enableProfiling) {
    newTask.isQueued = false;
  }

  if (startTime > currentTime) {
    // This is a delayed task.
    newTask.sortIndex = startTime;
    push(timerQueue, newTask);
    if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
      // All tasks are delayed, and this is the task with the earliest delay.
      if (isHostTimeoutScheduled) {
        // Cancel an existing timeout.
        cancelHostTimeout();
      } else {
        isHostTimeoutScheduled = true;
      }
      // Schedule a timeout.
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
    newTask.sortIndex = expirationTime;
    push(taskQueue, newTask);
    if (enableProfiling) {
      markTaskStart(newTask, currentTime);
      newTask.isQueued = true;
    }
    // Schedule a host callback, if needed. If we're already performing work,
    // wait until the next time we yield.
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true;
      requestHostCallback(flushWork);
    }
  }

  return newTask;
}

function push(heap: Heap, node: Node): void {
  const index = heap.length;
  heap.push(node);
  siftUp(heap, node, index);
}
```

#### commitPassiveUnmountEffects & commitPassiveMountEffects

在 render 阶段我们已知，useEffect 调用会将回调函数注册成一个 effect 对象，但此时并没有直接执行回调函数。在 commit 阶段初期，react 用 Scheduler 向任务队列中调度推入一个正常优先级的 flushPassiveEffects 函数。flushPassiveEffects 中内部的核心逻辑如下:

```js
function flushPassiveEffectsImpl() {
  // ......
  commitPassiveUnmountEffects(root.current);
  commitPassiveMountEffects(root, root.current, lanes, transitions);
  // ......

  return true;
}
```

它主要执行了以下两项:

1. commitPassiveUnmountEffects
2. commitPassiveMountEffects

其中，commitPassiveUnmountEffects 主要处理组件被卸载时 FiberNode 中挂载的副作用；commitPassiveMountEffects 主要处理组件挂载时 FiberNode 中挂载的副作用。useEffect 中定义的副作用函数 create 和 destroy 分别在 commitPassiveMountEffects 和 commitPassiveUnmountEffects 中被执行。

#### commitPassiveUnmountEffects_begin & commitPassiveMountEffects_begin

在本文开头，我们就抛出了一个疑问: "为什么子组件 useEffect 内的代码会优先于父组件执行，而不是遵循父组件到子组件从上到下按顺序执行?"。这个疑惑将在这一小节被解开。我们分析 commitPassiveUnmountEffects_begin 和 commitPassiveMountEffects_begin 可以发现:

```js
function commitPassiveUnmountEffects_begin() {
  while (nextEffect !== null) {
    const fiber = nextEffect;
    const child = fiber.child;

    if ((nextEffect.flags & ChildDeletion) !== NoFlags) {
      const deletions = fiber.deletions;
      if (deletions !== null) {
        for (let i = 0; i < deletions.length; i++) {
          const fiberToDelete = deletions[i];
          nextEffect = fiberToDelete;
          commitPassiveUnmountEffectsInsideOfDeletedTree_begin(
            fiberToDelete,
            fiber,
          );
        }

        if (deletedTreeCleanUpLevel >= 1) {
          // A fiber was deleted from this parent fiber, but it's still part of
          // the previous (alternate) parent fiber's list of children. Because
          // children are a linked list, an earlier sibling that's still alive
          // will be connected to the deleted fiber via its `alternate`:
          //
          //   live fiber
          //   --alternate--> previous live fiber
          //   --sibling--> deleted fiber
          //
          // We can't disconnect `alternate` on nodes that haven't been deleted
          // yet, but we can disconnect the `sibling` and `child` pointers.
          const previousFiber = fiber.alternate;
          if (previousFiber !== null) {
            let detachedChild = previousFiber.child;
            if (detachedChild !== null) {
              previousFiber.child = null;
              do {
                const detachedSibling = detachedChild.sibling;
                detachedChild.sibling = null;
                detachedChild = detachedSibling;
              } while (detachedChild !== null);
            }
          }
        }

        nextEffect = fiber;
      }
    }

    if ((fiber.subtreeFlags & PassiveMask) !== NoFlags && child !== null) {
      child.return = fiber;
      nextEffect = child;
    } else {
      commitPassiveUnmountEffects_complete();
    }
  }
}

function commitPassiveMountEffects_begin(
  subtreeRoot: Fiber,
  root: FiberRoot,
  committedLanes: Lanes,
  committedTransitions: Array<Transition> | null,
) {
  while (nextEffect !== null) {
    const fiber = nextEffect;
    const firstChild = fiber.child;
    if ((fiber.subtreeFlags & PassiveMask) !== NoFlags && firstChild !== null) {
      firstChild.return = fiber;
      nextEffect = firstChild;
    } else {
      commitPassiveMountEffects_complete(
        subtreeRoot,
        root,
        committedLanes,
        committedTransitions,
      );
    }
  }
}
```

在 commitPassiveUnmountEffects_begin 和 commitPassiveMountEffects_begin 中都存在一个逻辑:

```js
while (nextEffect !== null) {
  if (fiber.child !== null) {
    firstChild.return = fiber;
    nextEffect = firstChild;
  } else {
    // ......
  }
}
```

分析可知，在处理 effect 对象之前，react 会将 nextEffect 指针移动到叶子节点 (即 fiber.child === null) 后再进行 commitPassiveMountEffects_complete 或 commitPassiveUnmountEffects_complete 操作。
当然，上述逻辑中只展示了 nextEffect 不断更改指向到叶子结点的过程，而 nextEffect 指针从叶子结点向上回溯执行副作用的过程由 commitPassiveMountEffects_complete 或 commitPassiveUnmountEffects_complete 完成。

```js
function commitPassiveMountEffects_complete(
  subtreeRoot: Fiber,
  root: FiberRoot,
  committedLanes: Lanes,
  committedTransitions: Array<Transition> | null,
) {
  while (nextEffect !== null) {
    const fiber = nextEffect;

    if ((fiber.flags & Passive) !== NoFlags) {
      setCurrentDebugFiberInDEV(fiber);
      try {
        commitPassiveMountOnFiber(
          root,
          fiber,
          committedLanes,
          committedTransitions,
        );
      } catch (error) {
        captureCommitPhaseError(fiber, fiber.return, error);
      }
      resetCurrentDebugFiberInDEV();
    }

    if (fiber === subtreeRoot) {
      nextEffect = null;
      return;
    }

    // 👇 nextEffect 指针回溯的过程
    const sibling = fiber.sibling;
    if (sibling !== null) {
      sibling.return = fiber.return;
      nextEffect = sibling;
      return;
    }

    nextEffect = fiber.return;
  }
}
```

分析如下这段代码可以发现:

```js
if (fiber === subtreeRoot) {
  nextEffect = null;
  return;
}

const sibling = fiber.sibling;
if (sibling !== null) {
  sibling.return = fiber.return;
  nextEffect = sibling;
  return;
}

nextEffect = fiber.return;
```

当 nextEffect (也就是 fiber) 存在同级兄弟节点时，他会将 nextEffect 指针指向该兄弟节点，同时跳出 commitPassiveMountEffects_complete 的循环，继续由 commitPassiveMountEffects_begin 递归到该兄弟节点的叶子结点，从下往上执行副作用。
若同级不存在兄弟节点了，则将 nextEffect 指针指向当前 FiberNode 的父节点，无需跳出循环，直接从父节点位置开启下一轮 commitPassiveMountEffects_complete 执行副作用，直到 nextEffect 指向根目录 `fiber === subtreeRoot`，将 nextEffect 置为空，跳出循环。

commitPassiveMountEffects_begin 向下递归和 commitPassiveMountEffects_complete 向上回溯执行的遍历方式决定了 effect 副作用始终是从叶子结点向上执行。这也就是为什么子组件的副作用优先被执行的原因。

**更准确地说，effect 副作用的执行顺序是“自底向上，自左向右”执行的。因为 FiberNode Tree 中同层还存在兄弟节点这一情况。**

#### commitHookEffectListUnmount & commitHookEffectListMount

useEffect 注册的回调最终会在 commitHookEffectListUnmount 和 commitHookEffectListMount 中被执行。在 render 阶段，workInProgress 指向的 FiberNode 节点中的所有 effect 对象都会被挂载在 workInProgress.updateQueue 上，在 commitHookEffectListUnmount 或 commitHookEffectListMount 中，我们从 updateQueue 属性字段中取出所有 effect 对象，并从 firstEffect 开始循环执行到 lastEffect 结束。

**commitHookEffectListMount:**

```js
function commitHookEffectListMount(flags: HookFlags, finishedWork: Fiber) {
  const updateQueue: FunctionComponentUpdateQueue | null = (finishedWork.updateQueue: any);
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    do {
      if ((effect.tag & flags) === flags) {
        // ......
        // Mount
        const create = effect.create;
        effect.destroy = create();
        // ......
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```

在 commitHookEffectListMount 中，我们从 effect 对象的 create 字段中取出先前在 useEffect 中定义的 create 回调函数并运行。Demo 中，此处会在控制台打印出 "Child Mount" 或 "Parent Mount"。此外，react 会将 create 的返回值，也就是 destroy 函数赋值到 `effect.destroy` 中保存，等待下一轮渲染时在 commitHookEffectListUnmount 中执行。
effect 对象链表的遍历执行由这段代码结构控制:

```js
do {
  // do something ...
  effect = effect.next;
} while (effect !== firstEffect);
```

当 effect 中存在 destroy 函数时，需要调用 commitHookEffectListUnmount 来执行。其逻辑与 commitHookEffectListMount 类似，他会取出 effect 对象中存储的 destroy 函数并执行。由于 `effect.destroy` 来自于 `effect.create` 的执行结果，因此 destroy 执行的是上一轮副作用返回的清除函数，这一轮 create 返回的副作用清除函数将在下一轮渲染中执行。

```js
function commitHookEffectListUnmount(
  flags: HookFlags,
  finishedWork: Fiber,
  nearestMountedAncestor: Fiber | null,
) {
  const updateQueue: FunctionComponentUpdateQueue | null = (finishedWork.updateQueue: any);
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    do {
      if ((effect.tag & flags) === flags) {
        // Unmount
        const destroy = effect.destroy;
        effect.destroy = undefined;
        if (destroy !== undefined) {
          // ......
          // 👇 destroy 函数在此执行
          safelyCallDestroy(finishedWork, nearestMountedAncestor, destroy);
          // ......
        }
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```

### 总结

1. render 阶段，useEffect 的回调将被挂载到 effect 对象上，同一 FiberNode 对象中的 effect 按环形链表排布，并保存到当前 FiberNode 的 updateQueue 更新队列中。
2. commit 阶段。react 通过 Scheduler 调度器调度 flushPassiveEffects 方法，在 commit 阶段完成后通过异步方式执行。flushPassiveEffects 方法会深度遍历到叶子结点，并按照“自左向右，自底向上”的顺序执行保存在各个 FiberNode 节点中的副作用。
