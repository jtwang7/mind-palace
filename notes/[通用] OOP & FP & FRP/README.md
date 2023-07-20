# OOP & FP & FRP

参考文档:

- [FP, RP, FRP… What the F…](https://medium.com/@emadalam/fp-rp-frp-what-the-f-71c4949c29d) `理解 FP/RP/FRP 三种编程范式`
- [Object Oriented Programing in Javascript](https://dev.to/bhaveshdaswani93/object-oriented-programing-in-javascript-3bp0) `OOP 面向对象`
- [Functional Programming in javascript](https://dev.to/bhaveshdaswani93/functional-programming-in-javascript-59e2) `FP 面向函数`
- [OOP vs FP with Javascript](https://dev.to/bhaveshdaswani93/oop-vs-fp-with-javascript-39jf) `两者区别以及如何选择合适的编程范式`

## 概念

### OOP（面向对象编程）：

OOP 是一种编程范式，它侧重于使用对象来组织和处理数据。在前端开发中，我们经常使用 JavaScript，它是一种基于原型的面向对象编程语言。在 OOP 中，我们将数据（属性）和操作数据的方法（函数）封装在对象中。这有助于我们更好地组织代码，实现模块化和代码复用。

例如，假设我们要在前端开发中处理用户信息。我们可以使用 OOP 的方式创建一个用户类（User class）：

```javascript
class User {
  constructor(name, age) {
    this.name = name;
    this.age = age;
  }

  greet() {
    console.log(`Hello, my name is ${this.name} and I am ${this.age} years old.`);
  }
}

const user = new User('Alice', 30);
user.greet(); // 输出："Hello, my name is Alice and I am 30 years old."
```

### FP（函数式编程）：

函数式编程是一种编程范式，它将计算视为函数的求值。在 FP 中，函数是一等公民，我们可以将函数作为参数传递给其他函数，或者将函数作为返回值。FP 强调纯函数（无副作用，相同输入总是产生相同输出）和不可变性（避免修改数据，而是创建新的数据）。

在前端开发中，我们可以使用函数式编程来处理数据和操作 DOM。以下是一个简单的 FP 示例：

```javascript
const users = [
  { name: 'Alice', age: 30 },
  { name: 'Bob', age: 25 },
  { name: 'Charlie', age: 35 }
];

const getNames = users => users.map(user => user.name);
const names = getNames(users);
console.log(names); // 输出：['Alice', 'Bob', 'Charlie']
```

### FRP（函数响应式编程）：

函数响应式编程是一种编程范式，它结合了函数式编程和响应式编程。在 FRP 中，我们使用响应式数据流（数据随时间而变化）来处理异步事件和数据变化。这有助于我们更好地处理前端中的用户交互、动画和实时数据更新。

在前端开发中，我们可以使用 FRP 库（如 RxJS）来处理响应式数据流。以下是一个简单的 FRP 示例：

```javascript
import { fromEvent } from 'rxjs';
import { debounceTime, map } from 'rxjs/operators';

const input = document.querySelector('input');
const input$ = fromEvent(input, 'input');

input$
  .pipe(
    debounceTime(300),
    map(event => event.target.value)
  )
  .subscribe(value => {
    console.log('Input value:', value);
  });
```

在这个示例中，我们使用 RxJS 创建了一个响应式数据流，它表示输入框中的值随着用户输入而变化。我们使用 `debounceTime` 来限制事件的触发频率，然后使用 `map` 提取输入框的值。最后，我们订阅这个数据流，并在控制台输出输入框的值。

## 疑问

> 以下回答均来自 GPT

### js 闭包和 FP 编程范式纯函数原则相违背吗?

JavaScript 的闭包和函数式编程范式中的纯函数原则并不一定相违背。实际上，在很多情况下，闭包可以帮助我们实现纯函数和其他函数式编程概念。让我们先回顾一下闭包和纯函数的概念：

1. 闭包：在 JavaScript 中，闭包是一个函数可以访问和操作其外部作用域中的变量，即使该函数在其声明的作用域之外被调用。这使得我们可以创建私有变量和状态，从而实现模块化和封装。

2. 纯函数：纯函数是一种没有副作用、相同输入总是产生相同输出的函数。这有助于我们编写可预测、可测试和可维护的代码。

虽然在某些情况下，闭包可能会导致副作用（如修改外部作用域中的变量），但这并不意味着闭包和纯函数原则总是相互矛盾。实际上，在很多情况下，我们可以使用闭包来实现纯函数，例如：

1. 函数柯里化（Currying）：柯里化是一种将多参数函数转换为一系列单参数函数的技术。闭包可以帮助我们实现柯里化，从而实现更简洁、可复用的函数式代码。以下是一个简单的柯里化示例：

```javascript
function add(a) {
  return function(b) {
    return a + b;
  };
}

const add5 = add(5);
console.log(add5(3)); // 输出：8
```

在这个示例中，`add` 函数返回一个闭包，该闭包可以访问参数 `a`。`add5` 是一个纯函数，因为它没有副作用，且相同输入总是产生相同输出。

2. 私有状态：闭包可以帮助我们实现私有状态，这有助于我们编写更安全、更可控的纯函数。以下是一个简单的私有状态示例：

```javascript
function createCounter() {
  let count = 0;

  return function() {
    count += 1;
    return count;
  };
}

const counter = createCounter();
console.log(counter()); // 输出：1
console.log(counter()); // 输出：2
```

在这个示例中，`createCounter` 函数返回一个闭包，该闭包可以访问私有变量 `count`。虽然 `counter` 函数修改了 `count` 变量，但由于 `count` 是私有的，这并不会导致全局副作用。因此，我们可以认为 `counter` 函数是纯函数。或者换句话说，`createCounter` 函数是一个广义的纯函数 (执行 createCounter 及其后续产物，虽然相同的输入不能产生相同的输出，但是返回结果是可控、可预测的)，那么他所创建出来的函数也可被视为广义的纯函数。

总之，虽然闭包和纯函数原则在某些情况下可能相互矛盾，但在很多情况下，它们可以共同实现函数式编程概念。关键在于我们如何使用闭包来实现可预测、无副作用的代码。

### js 中的副作用场景有哪些?

在 JavaScript 中，副作用（side effect）是指函数在执行过程中对外部环境产生的影响，如修改全局变量、修改函数参数、执行 I/O 操作等。以下是一些可能产生副作用的场景：

1. 修改全局变量：当函数修改全局变量时，会产生副作用。这可能导致其他依赖于该变量的函数的行为发生变化。

```javascript
let globalVar = 0;

function modifyGlobalVar() {
  globalVar = 10;
}
```

2. 修改函数参数：当函数修改其参数时，会产生副作用。这可能导致传入的对象在函数调用之后发生变化。

```javascript
function modifyObject(obj) {
  obj.value = 10;
}

const myObj = { value: 0 };
modifyObject(myObj);
console.log(myObj.value); // 输出：10
```

3. 执行 I/O 操作：当函数执行 I/O 操作（如读写文件、发送网络请求等）时，会产生副作用。这可能导致外部环境（如文件系统、数据库等）发生变化。

```javascript
function fetchData() {
  fetch('https://api.example.com/data')
    .then(response => response.json())
    .then(data => console.log(data));
}
```

4. 修改 DOM：当函数修改 DOM 时，会产生副作用。这可能导致页面内容和样式发生变化。

```javascript
function changeText() {
  const element = document.getElementById('text');
  element.innerHTML = 'New text';
}
```

5. 使用定时器和事件监听器：当函数使用定时器（如 `setTimeout`、`setInterval` 等）或事件监听器（如 `addEventListener` 等）时，会产生副作用。这可能导致函数在未预期的时刻执行，从而影响程序的行为。

```javascript
function delayedLog() {
  setTimeout(() => {
    console.log('This is a delayed log message.');
  }, 1000);
}
```

在编写代码时，我们应尽量减少副作用，以提高代码的可预测性、可测试性和可维护性。函数式编程范式鼓励我们使用纯函数（无副作用，相同输入总是产生相同输出）来编写代码。

然而，在实际开发中，我们可能无法完全避免副作用，因为很多任务（如操作 DOM、发送网络请求等）本身就需要与外部环境进行交互。在这种情况下，我们应尽量将副作用限制在可控范围内，并确保代码易于理解和维护。
