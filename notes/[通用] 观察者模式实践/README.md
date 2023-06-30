# 观察者模式实践

## 介绍

**概念:**

1. 定义对象间的一种一对多的依赖关系。
2. 当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新。

在观察者模式中，由一个目标物件管理所有相依于它的观察者物件，并且在它本身的状态改变时主动发出通知。通知发布通常通过调用各观察者所提供的方法来实现。

> 摘自百度百科: [观察者模式](https://baike.baidu.com/item/%E8%A7%82%E5%AF%9F%E8%80%85%E6%A8%A1%E5%BC%8F/5881786)

**使用场景:**

适用于“一对多”的应用场景，用于应对各观察者对象间可能存在的复杂关联关系。
为了避免维护各观察者间复杂的依赖关系，我们需要一个顶层的管理者存储全局的状态，并在相应状态变动时，通知依赖该状态的观察者对象执行它的逻辑。

这样做的好处在于:

1. 不再需要关心各观察者对象间的依赖关系，不需要知道具体有多少对象要被改变。
2. 各观察者对象间是松散的，添加或移除观察者对象，对整体逻辑不会产生较大的影响。

**模式角色:**

在观察者模式中，需要实现两大角色: 管理者(Subject) 和 观察者(Observer)，它们通常需要包含一些特定的功能。

- Subject
  - 作为顶层容器存储状态。
  - 存储所有观察者对象(Observer)。
  - 提供可增加和删除观察者对象的接口。
  - 在内部状态改变时，给所有登记过的观察者发出通知。

- Observer
  - 调用 Subject 的注册方法，将自身注册为一个 Observer。
  - 提供一个供管理者通知自身的方法。

**观察者模式实现思路:**

遵循 “注册—通知—撤销注册” 的流程。Observer 将自己注册到 Subject 的管理容器 Container 中。Subject 发生了某种变化，从容器中得到所有注册过的 Observer 并通知其变化。当要 Observer 要注销时，调用 Subject 相应的注销方法，Subject 将从容器中移除对应的 Observer。

## 代码样例

根据上述概念及实现思路，我们简单实现一个观察者模式，该观察者模式符合最基本的几个条件:

1. 有明确的 Subject 管理者和 Observer 观察者划分。
2. Subject 提供注册、注销以及通知观察者的功能。
3. Subject 维护注册的观察者列表及顶层的状态。
4. Observer 提供 Subject 通知自己的方法(接口)。

CodeSandbox 地址: [观察者模式](https://codesandbox.io/s/guan-cha-zhe-mo-shi-pctyw9)

**Subject 实现:**

```js
class Subject {
  constructor() {
    this.observers = []; // 存放观察者
    this.state = {
      1: "小红",
      2: "小明",
      3: "小华"
    }; // 存放状态
  }

  // 添加观察者
  addObserver(observer) {
    this.observers.push(observer);
  }

  // 移除观察者
  removeObserver(observer) {
    this.observers = this.observers.filter((ob) => {
      return ob.id !== observer.id;
    });
  }

  // 通知观察者: 遍历注册的观察者所提供的通知接口(例如 xx.update())
  notify() {
    this.observers.forEach((ob) => {
      ob.update(this.state[ob.id]);
    });
  }

  // 若全局管理了 state，则还可以为 Observer 提供相应的 state 更改接口
  setState() {
    // ...
    this.notify()
  }
}

export default Subject;
```

**Observer 实现:**

```js
class Observer {
  constructor(id) {
    this.id = id;
  }

  // 👇 Subject 通知 Observer 的方式
  update(name) {
    console.log(`${name}接收到通知!`);
  }
}

export default Observer;
```

**测试:**

```js
import Observer from "./Observer";
import Subject from "./Subject";

// 实例化
const subject = new Subject();
const ob1 = new Observer(1);
const ob2 = new Observer(2);
const ob3 = new Observer(3);

// 添加观察者，并触发通知
subject.addObserver(ob1);
subject.addObserver(ob2);
subject.addObserver(ob3);
subject.notify();

console.log("------")

// 移除某个观察者，并触发通知
subject.removeObserver(ob1);
subject.notify();

// 👇 最终返回结果
// 小红接收到通知! 
// 小明接收到通知! 
// 小华接收到通知! 
// ------
// 小明接收到通知! 
// 小华接收到通知!
```
