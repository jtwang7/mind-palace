# 7 Architectural Attributes of a Reliable React Component

👉 [原文链接](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#61-case-study-testable-means-well-designed)

> **RULES**
>
> - 主要是基于原文的学习及注解，一切解释以原文为主。
> - 所有个人标注均带有 🚀 标签。

---

I like how React embraces component-based architecture. You can compose complex user interfaces from smaller pieces, take advantage of components reusability and abstracted DOM manipulations.

Component-based development is productive: a complex system is built from specialized and easy to manage pieces. Yet only well designed components ensure composition and reusability benefits.

> 🚀 基于组件的开发是富有成效的：一个复杂的系统是由专门的、易于管理的片段构建的。然而，只有设计良好的组件才能确保组合和重用性的好处。

Despite the application complexity, hurry to meet the deadlines and unexpectedly changing requirements, you must constantly walk on *the thin line of architectural correctness*. Make your components decoupled, focused on a single task, well tested.

Unfortunately, it's tempting to follow the wrong path: write big components with many responsibilities, tightly couple components, forget about unit tests. These increase [technical debt](https://www.nczonline.net/blog/2012/02/22/understanding-technical-debt/), making progressively hard to modify existing or create new functionality.

> 🚀 编写有很多职责的大组件，紧密耦合的组件，忘记单元测试。这些都会增加技术债务，使修改现有功能或创建新功能变得越来越困难。我们必须不断地走在架构正确性的细线上。让组件解耦，专注于单一任务，并经过良好的测试。

When writing a React application, I regularly ask myself:

- How *to correctly structure the component*?
- At what point a big component should *split into smaller components*?
- How to design *a communication between components that prevents tight coupling*?

Luckily, reliable components have common characteristics. Let's study these 7 useful criterias, and detail into case studies.

> 🚀 可靠的组件有以下几个共同的特点，遵循组件开发的 7 个标准，或许可以帮助我们开发更可靠、更易维护的组件。

### Table of Contents

- "Single responsibility"
  - [1.1 The pitfall of multiple responsibilities](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#11-the-pitfall-of-multiple-responsibilities)
  - [1.2 Case study: make component have one responsibility](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#12-case-study-make-component-have-one-responsibility)
  - [1.3 Case study: HOC favors single responsibility principle](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#13-case-study-hoc-favors-single-responsibility-principle)
- "Encapsulated"
  - [2.1 Information hiding](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#21-information-hiding)
  - [2.2 Communication](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#22-communication)
  - [2.3 Case study: encapsulation restoration](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#23-case-study-encapsulation-restoration)
- "Composable"
  - 3.1 Composition benefits
    - [Single responsibility](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#single-responsibility)
    - [Reusability](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#reusability)
    - [Flexibility](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#flexibility)
    - [Efficiency](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#efficiency)
- "Reusable"
  - [4.1 Reuse across application](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#41-reuse-across-application)
  - [4.2 Reuse of 3rd party libraries](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#42-reuse-of-3rd-party-libraries)
- "Pure" or "Almost-pure"
  - [5.1 Case study: purification from global variables](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#51-case-study-purification-from-global-variables)
  - [5.2 Case study: purification from network requests](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#52-case-study-purification-from-network-requests)
  - [5.3 Transform almost-pure into pure](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#53-transform-almost-pure-into-pure)
- "Testable" and "Tested"
  - [6.1 Case study: testable means well designed](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#61-case-study-testable-means-well-designed)
- "Meaningful"
  - 7.1 Component naming
    - [Pascal case](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#pascal-case)
    - [Specialization](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#specialization)
    - [One word - one concept](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#one-word---one-concept)
    - [Code comments](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#code-comments)
  - [7.2 Case study: write self-explanatory code](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#72-case-study-write-self-explanatory-code)
  - [7.3 Expressiveness stairs](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#73-expressiveness-stairs)
- [Do continuous improvement](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#8-do-continuous-improvement)
- [Reliability is important](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#9-reliability-is-important)
- [Conclusion](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#10-conclusion)

## 1. "Single responsibility"

> A component has a **single responsibility** when it has one reason to change.

A fundamental rule to consider when writing React components is the [single responsibility principle](https://en.wikipedia.org/wiki/Single_responsibility_principle).

Single responsibility principle (abbreviated SRP) requires a component to have *one reason to change*.

A component has *one reason to change* when it implements one responsibility, or simpler when it does one thing.

A responsibility is either to *render a list of items*, or *to show a date picker*, or *to make an HTTP request*, or *to draw a chart*, or *to lazy load an image*, etc. Your component should pick only one responsibility and implement it. When you modify the way component implements its responsibility (e.g. a change to limit the number of items for *render a list of items* responsibility) - it has one reason to change.

Why is it important to have only *one reason to change*? Because component's modification becomes isolated and under control.

Having one responsibility restricts the component size and makes it focused on one thing. A component focused on one thing is convenient to code, and later modify, reuse and test.

> 🚀 组件开发最基本的一项原则：单一责任原则 (Single Responsibility Principle, SRP)
>
> 我们在开发组件时要时时刻刻思考并尽可能确保组件单一职责，即一个组件只负责一项功能。
>
> WHY SRP IMPORTANT? 单一职责限制了组件的大小，使其专注于一件事。一个专注于一件事的组件很容易编写代码，然后进行修改、重用和测试。

Let's follow a few examples.

**Example 1** A component fetches remote data, correspondingly it has *one reason to change when fetch logic changes*.
A reason to change happens when:

- The server URL is modified
- The response format is modified
- You want to use a different HTTP requests library
- Or any modification related to fetch logic *only*.

**Example 2** A table component maps an array of data to a list of row components, as result having *one reason to change when mapping logic changes*.
A reason to change occurs when:

- You have a task to limit the number of rendered row components (e.g. display up to 25 rows)
- You're asked to show a message "The list is empty" when there are no items to display
- Or any modification related to mapping of array to row components *only*.

*Does your component have many responsibilities?* If the answer is *yes*, split the component into chunks by each individual responsibility.

> 🚀 要不断询问自己 “当前组件是否包含多个功能职责”。若不能肯定回答 NO，那么我们就有必要考虑将其拆分成符合 SRP 的多个单一组件。

An alternative reasoning about the single responsibility principle says to create the component around a clearly distinguishable [axis of change](https://stackoverflow.com/questions/2952662/srp-axis-of-change). An axis of change attracts modifications of the same meaning.
In the previous 2 examples, the axis of change were fetch logic and mapping logic.

If you find SRP a bit obscured, check out [this article](https://8thlight.com/blog/uncle-bob/2014/05/08/SingleReponsibilityPrinciple.html).

> 🚀 关于 SRP 的深入讲解 👉 [this article](https://8thlight.com/blog/uncle-bob/2014/05/08/SingleReponsibilityPrinciple.html)

Units written at early project stage *will change often* until reaching the release stage. These *change often* components are required to be *easily modifiable in isolation*: a goal of the SRP.

### 1.1 The pitfall of multiple responsibilities

A common oversight happens when a component has multiple responsibilities. At first glance, this practice seems harmless and requires less work:

- *You start coding right away*: no need to recognize the responsibilities and plan the structure accordingly
- *One big component does it all*: no need to create components for each responsibility
- *No split - no overhead*: no need to create props and callbacks for communication between split components.

Such naive structuring is easy to code at the beginning. Difficulties will appear on later modifications, as the application grows and becomes more complex.

A component that implements simultaneously multiple responsibilities has *many reasons to change*. Now emerges the main problem: changing the component for one reason *unintentionally* influences how other responsibilities are implemented by the same component.

> 🚀 遵循 SRP 开发组件是必要的：一个同时实现多种责任的组件会因为各种各样的因素触发改变，可能会无意中影响了同一组件的其他责任的实现。
>
> - 任何组件开发都必须有清晰的职责规划。
> - 拆分的组件需要遵循 SRP 原则。
> - 拆分的组件通过预留 props 和 callback 实现组件之间的通信。

Such design is fragile. Unintentional side effects of are *hard to predict and control*.

For example, `<ChartAndForm>` implements simultaneously 2 responsibilities to draw a chart, and handle a form that provides data for that chart. `<ChartAndForm>` has *2 reasons to change*: draw the chart and handle the form.

When you change a form field (e.g. transform an `<input>` into a `<select>`), you can unintentionally break how chart is rendered. Moreover the chart implementation is non reusable, because it's coupled with the form details.

Solving multiple responsibilities issue requires to split `<ChartAndForm>` in 2 components: `<Chart>` and `<Form>`. Each chunk has one responsibility: to draw the chart or correspondingly handle the form. The communication between the chunks is done through props.

The worst case of multiple responsibilities problem is so called God component anti-pattern (an analogy of [God object](https://en.wikipedia.org/wiki/God_object)). A God component tends to know and do everything within the application. You might see it named `<Application>`, `<Manager>`, `<BigContainer>` or `<Page>`, having more than 500 lines of code.

Dissolve God components by having them conform to SRP with the [help of composition](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#31-composition-benefits).

### 1.2 Case study: make component have one responsibility

Imagine a component that makes an HTTP request to a specialized server to get the current weather. When data is successfully fetched, the same component uses the response to display the weather:

```
import axios from 'axios';
// Problem: A component with multiple responsibilities 
class Weather extends Component {
   constructor(props) {
     super(props);
     this.state = { temperature: 'N/A', windSpeed: 'N/A' };
   }
 
   render() {
     const { temperature, windSpeed } = this.state;
     return (
       <div className="weather">
         <div>Temperature: {temperature}°C</div>
         <div>Wind: {windSpeed}km/h</div>
       </div>
     );
   }
   
   componentDidMount() {
     axios.get('http://weather.com/api').then(function(response) {
       const { current } = response.data; 
       this.setState({
         temperature: current.temperature,
         windSpeed: current.windSpeed
       })
     });
   }
}
```

When dealing with alike situations, ask yourself: *do I have to split the component into smaller pieces*? The question is best answered by determining how component might change according to its responsibilities.

The weather component has *2 reasons to change*:

- Fetch logic in `componentDidMount()`: server URL or response format can be modified

- Weather visualization in `render()`: the way component displays the weather can change several times

The solution is to divide `<Weather>` in 2 components: each having one responsibility. Let's name the chunks `<WeatherFetch>` and `<WeatherInfo>`.

First component `<WeatherFetch>` is responsible for fetching the weather, extracting response data and saving it to state. It has *one fetch logic reason to change*:

```
import axios from 'axios';
// Solution: Make the component responsible only for fetching
class WeatherFetch extends Component {
   constructor(props) {
     super(props);
     this.state = { temperature: 'N/A', windSpeed: 'N/A' };
   }
 
   render() {
     const { temperature, windSpeed } = this.state;
     return (
       <WeatherInfo temperature={temperature} windSpeed={windSpeed} />
     );
   }
   
   componentDidMount() {
     axios.get('http://weather.com/api').then(function(response) {
       const { current } = response.data; 
       this.setState({
         temperature: current.temperature,
         windSpeed: current.windSpeed
       });
     });
   }
}
```

What benefits brings such structuring?

For instance, you would like to use `async/await` syntax instead of promises to get the response from server. This is a reason to change related to fetch logic:

```
// Reason to change: use async/await syntax
class WeatherFetch extends Component {
   // ..... //
   async componentDidMount() {
     const response = await axios.get('http://weather.com/api');
     const { current } = response.data; 
     this.setState({
       temperature: current.temperature,
       windSpeed: current.windSpeed
     });
   }
}
```

Because `<WeatherFetch>` has one fetch logic reason to change, any modification of this component happens in isolation. Using `async/await` does not affect directly the way weather is displayed.

Then `<WeatherFetch>` renders `<WeatherInfo>`. The latter is responsible only for displaying the weather, having *one visual reason to change*:

```
// Solution: Make the component responsible for displaying the weather
function WeatherInfo({ temperature, windSpeed }) {
   return (
     <div className="weather">
       <div>Temperature: {temperature}°C</div>
       <div>Wind: {windSpeed} km/h</div>
     </div>
   );
}
```

Let's change `<WeatherInfo>` that instead of `"Wind: 0 km/h"` display `"Wind: calm"`. That's a reason to change related to visual display of weather:

```
// Reason to change: handle calm wind  
function WeatherInfo({ temperature, windSpeed }) {
   const windInfo = windSpeed === 0 ? 'calm' : `${windSpeed} km/h`;
   return (
     <div className="weather">
       <div>Temperature: {temperature}°C</div>
       <div>Wind: {windInfo}</div>
     </div>
   );
}
```

Again, this modification of `<WeatherInfo>` happens in isolation and does not affect `<WeatherFetch>` component.

`<WeatherFetch>` and `<WeatherInfo>` have their own one responsibility. A change of one component has small effect on the other one. That's the power of single responsibility principle: modification in isolation that affects lightly and predictability other components of the system.

### 1.3 Case study: HOC favors single responsibility principle

> 🚀 React 实现 SRP 原则的几种途径：
>
> - HOC
> - props render
> - hooks
>
> 他们共同目标都是 ”实现逻辑的分离“，然后通过不同的途径将这些隔离的逻辑通过注入或组合的方式附加给 SRP 组件。

Applying composition with chunking components by responsibilities doesn't always help to conform to single responsibility principle. You can benefit from another efficient practice called Higher order components (abbreviated HOC):

> **Higher order component** is a function that takes one component and returns a new component.

A common usage of HOC is to provide the wrapped component with additional props or modify existing prop values. This technique is called *props proxy*:

```
function withNewFunctionality(WrappedComponent) {
  return class NewFunctionality extends Component {
    render() {
      const newProp = 'Value';
      const propsProxy = {
         ...this.props,
         // Alter existing prop:
         ownProp: this.props.ownProp + ' was modified',
         // Add new prop:
         newProp
      };
      return <WrappedComponent {...propsProxy} />;
    }
  }
}
const MyNewComponent = withNewFunctionality(MyComponent);
```

You can also hook into render mechanism by altering elements that wrapped component renders. This HOC technique is named *render highjacking*:

```
function withModifiedChildren(WrappedComponent) {
  return class ModifiedChildren extends WrappedComponent {
    render() {
      const rootElement = super.render();
      const newChildren = [
        ...rootElement.props.children, 
        // Insert a new child:
        <div>New child</div>
      ];
      return cloneElement(
        rootElement, 
        rootElement.props, 
        newChildren
      );
    }
  }
}
const MyNewComponent = withModifiedChildren(MyComponent);
```

If you want to deep dive into HOCs practice, I recommend to read ["React Higher Order Components in depth"](https://medium.com/@franleplant/react-higher-order-components-in-depth-cf9032ee6c3e).

> 🚀 关于 HOC 使用的深入讲解 👉 ["React Higher Order Components in depth"](https://medium.com/@franleplant/react-higher-order-components-in-depth-cf9032ee6c3e)

Let's follow an example how props proxy HOC technique helps separating responsibilities.

The component `<PersistentForm>` consists of an input field and a button *Save to storage*. Input field value is read/saved to local storage. After changing the input value, hitting *Save to storage* writes it to storage.

On input field change component's state gets updated inside `handleChange(event)` method. On button click the value is saved to local storage in `handleClick()`:

```
class PersistentForm extends Component {  
  constructor(props) {
    super(props);
    this.state = { inputValue: localStorage.getItem('inputValue') };
    this.handleChange = this.handleChange.bind(this);
    this.handleClick = this.handleClick.bind(this);
  }

  render() {
    const { inputValue } = this.state;
    return (
      <div className="persistent-form">
        <input type="text" value={inputValue} 
          onChange={this.handleChange}/> 
        <button onClick={this.handleClick}>Save to storage</button>
      </div>
    );
  }

  handleChange(event) {
    this.setState({
      inputValue: event.target.value
    });
  }

  handleClick() {
    localStorage.setItem('inputValue', this.state.inputValue);
  }
}
```

Unfortunately `<PersistentForm>` has 2 responsibilities: manage form fields and saving the input value to store.

Let's refactor `<PersistentForm>` to have one responsibility: render form fields and attach event handlers. It shouldn't know how to use storage directly:

```
class PersistentForm extends Component {  
  constructor(props) {
    super(props);
    this.state = { inputValue: props.initialValue };
    this.handleChange = this.handleChange.bind(this);
    this.handleClick = this.handleClick.bind(this);
  }

  render() {
    const { inputValue } = this.state;
    return (
      <div className="persistent-form">
        <input type="text" value={inputValue} 
          onChange={this.handleChange}/> 
        <button onClick={this.handleClick}>Save to storage</button>
      </div>
    );
  }

  handleChange(event) {
    this.setState({
      inputValue: event.target.value
    });
  }

  handleClick() {
    this.props.saveValue(this.state.inputValue);
  }
}
```

The component receives the stored input value from a prop `initialValue`, and saves the input value using a prop function `saveValue(newValue)`. These props are provided by `withPersistence()` HOC using props proxy technique.

Now `<PersistentForm>` conforms to SRP. Its only reason to change is the form field modifications.

The responsibility of querying and saving to local storage goes to `withPersistence()` HOC:

```
function withPersistence(storageKey, storage) {
  return function(WrappedComponent) {
    return class PersistentComponent extends Component {
      constructor(props) {
        super(props);
        this.state = { initialValue: storage.getItem(storageKey) };
      }

      render() {
         return (
           <WrappedComponent
             initialValue={this.state.initialValue}
             saveValue={this.saveValue}
             {...this.props}
           />
         );
      }

      saveValue(value) {
        storage.setItem(storageKey, value);
      }
    }
  }
}
```

`withPersistence()` is a HOC which responsibility is persistence. It doesn't know any details about form fields. It has one focused job: provide the wrapped component with `initialValue` string and `saveValue()` function.

Wiring up together `<PersistentForm>` and `withPersistence()` creates a new component `<LocalStoragePersistentForm>`. It is connected with the local storage and ready to be used within the application:

```
const LocalStoragePersistentForm = withPersistence('key', localStorage)(PersistentForm);
const instance = <LocalStoragePersistentForm />;
```

As long as `<PersistentForm>` uses correctly `initialValue` and `saveValue()` props, any modification of this component cannot break the logic of saving to storage that `withPersistence()` holds.

And vice versa: as long as `withPersistence()` provides the correct `initialValue` and `saveValue()`, any modification of the HOC cannot break the way `<PersistentForm>` handles the form fields.

Again emerges the efficiency of SRP: allowing you to make modifications in isolation that less affect other parts of the system.

Moreover the code reusability increases. You can connect any other form `<MyOtherForm>` to local storage:

```
const LocalStorageMyOtherForm = withPersistence('key', localStorage)(MyOtherForm);
const instance = <LocalStorageMyOtherForm />;
```

You can easily change the type of storage to `sessionStorage` (which gets cleared when the page session ends):

```
const SessionStoragePersistentForm = withPersistence('key', sessionStorage)(PersistentForm);
const instance = <SessionStoragePersistentForm />;
```

The isolation of modification and reusability benefits are not possible with the initial version of `<PersistentForm>`, which incorrectly had multiple responsibilities.

In situations when composition is ineffective, props proxy and render highjacking HOC techniques are useful in making component have one responsibility.

## 2. "Encapsulated"

> An **encapsulated** component provides props to control its behavior while not exposing its internal structure.
>
> 🚀 组件封装目的：提供 props 来控制其行为的同时，不暴露其内部结构。

[Coupling](https://en.wikipedia.org/wiki/Coupling_(computer_programming)) is a system characteristic that determines the degree of dependency between components.

Based on the degree of components dependence, 2 coupling types are distinguishable:

- *Loose coupling* happens when the application components have little or no knowledge about other components.
- *Tight coupling* happens when the application components know a lot of details about each other.

Loose coupling is the goal when designing application's structure and the relationship between components.

> 🚀 耦合是一个系统特性，决定了组件之间的依赖程度。
>
> - 低耦合：组件间不关心内部实现，只知道彼此通讯的接口 (props) 及对应的职责。
> - 高耦合：组件间掌握较多彼此内部状态 (state)，甚至可以通过途径 (state, ref) 直接操作组件内部状态。

Loose coupling leads to the following benefits:

- Allow making changes in one area of the application *without affecting others*
- Any component can be *replaced with an alternative* implementation
- Enables components *reusability* across the application, thus favoring [Don't repeat yourself](https://en.wikipedia.org/wiki/Don't_repeat_yourself) principle
- Independent components are *easier to test*, increasing the application code coverage

Contrary, a tightly coupled system looses the benefits described above. The main drawback is the difficulty to modify a component that is highly dependent on other components. Even a single modification might lead to a cascade of dependency *echo* modifications.

> 🚀 松散耦合会带来以下好处：
>
> - 允许在应用程序的一个区域进行修改而不影响其他区域
> - 任何组件都可以用另一种实现来代替
> - 使得组件在整个应用中可以重复使用，从而有利于 "不要重复自己 "的原则
> - 独立的组件更容易测试，增加应用的代码覆盖率
>
> 我们通常很难修改一个高度依赖其他组件的组件。面对高耦合组件，即使是一个单一的修改，也可能导致一连串的依赖性呼应的修改。

**Encapsulation**, or **Information Hiding**, is a fundamental principle of how to design components, and is the key to loose coupling.

> 🚀 封装，其实就是“隐藏组件内部信息/状态”的过程。它是设计低耦合组件的一项基本原则。

### 2.1 Information hiding

A well encapsulated component *hides its internal structure* and provides a set of *props to control its behavior*.

Hiding internal structure is essential. Other components are not allowed to know or rely on the component's internal structure or implementation details.

A React component can be functional or class based, define instance methods, setup refs, have state or use lifecycle methods. These implementation details are encapsulated within the component itself, and other components shouldn't know anything about these details.

Units that precisely hide their internal structure are less dependent on each other. Lowering the dependency degree brings the benefits of loose coupling.

> 🚀 一个封装良好的组件需要满足以下几点：
>
> - 隐藏其内部结构
>   - 一个React组件内部可以定义实例方法，设置 ref，维护 state 或使用生命周期方法。我们要确保这些实现细节被封装在组件本身，其他组件不应该知道任何关于这些细节的事情。
> - 提供一组 props 来控制其行为。
>
> 精确隐藏其内部结构的单元，相互之间的依赖性较低。降低依赖程度带来了松散耦合的好处。

### 2.2 Communication

Details hiding is a restriction that isolates the component. Nevertheless, you need a way to make components communicate. So welcome the props.

> 🚀 组件封装 ≠ 组件孤立。我们推荐通过 props 沟通组件。

Props are meant to be plain, raw data that are component's input.

A prop is recommended to be a *primitive type* (e.g. string, number, boolean):

```
<Message text="Hello world!" modal={false} />;
```

When necessary use a complex data structure like *objects or arrays*:

```
<MoviesList items={['Batman Begins', 'Blade Runner']} />
```

Prop as a *function* handles events and async behavior:

```
<input type="text" onChange={handleChange} />
```

A prop can be even a *component constructor*. A component can take care of other component's instantiation:

```
function If({ component: Component, condition }) {  return condition ? <Component /> : null;}<If condition={false} component={LazyComponent} />  
```

To avoid breaking encapsulation, watch out the details passed through props. A parent component that sets child props should not expose any details about its internal structure. For example, it's a bad decision to transmit using props the whole component instance or refs.

> 🚀 虽然 props 可以传递各种类型信息，但是为了避免破坏封装，我们不应该传递一些暴露组件内部信息的方法途径，例如使用props传递整个组件实例或 refs 是一个糟糕的决定。

Accessing global variables is another problem that negatively affects encapsulation.

### 2.3 Case study: encapsulation restoration

> 🚀 本节演示了违背封装原则导致的后果。例中 props 传递组件实例，导致子组件获取了改变父组件内部状态的方法，导致子组件与父组件高度耦合。

Component's instance and state object are implementation details encapsulated inside the component. Thus a certain way to break the encapsulation is to pass the parent instance for state management to a child component.

Let's study such a situation.

A simple application shows a number and 2 buttons. First button increases and second button decreases the number.

The application consists of two components: `<App>` and `<Controls>`.

`<App>` holds the state object that contains the modifiable number as a property, and renders this number:

```
// Problem: Broken encapsulation
class App extends Component {
  constructor(props) {
    super(props);
    this.state = { number: 0 };
  }
  
  render() {
    return (
      <div className="app"> 
        <span className="number">{this.state.number}</span>
        <Controls parent={this} />
      </div>
    );
  }
}
```

`<Controls>` renders the buttons and attaches click event handlers to them. When user clicks a button, parent component state is updated (`updateNumber()` method) by increasing `+1` or decreasing `-1` the displayed number:

```
// Problem: Using internal structure of parent component
class Controls extends Component {
  render() {
    return (
      <div className="controls">
        <button onClick={() => this.updateNumber(+1)}>
          Increase
        </button> 
        <button onClick={() => this.updateNumber(-1)}>
          Decrease
        </button>
      </div>
    );
  }
  
  updateNumber(toAdd) {
    this.props.parent.setState(prevState => ({
      number: prevState.number + toAdd       
    }));
  }
}
```

What is wrong with the current implementation? The demo at the beginning of the chapter shows sort of working application.

The first problem is `<App>`'s *broken encapsulation*, since its internal structure spreads across the application. `<App>` incorrectly permits `<Controls>` to update its state directly.

Consequently, the second problem is that `<Controls>` knows too many details about its parent `<App>`. It has access to parent instance, knows that parent is a stateful component, knows the state object structure (`number` property) and knows how to update the state.

The broken encapsulation couples `<App>` and `<Controls>` components.

A troublesome outcome is that `<Controls>` would be complicated to test (see [6.1 Case study](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#61-case-study-testable-means-well-designed)) and reuse. A slight modification to structure of `<App>` leads to cascade of modifications to `<Controls>` (and to alike coupled components in case of a bigger application).

The solution is to design a convenient communication interface that respects *loose coupling* and *strong encapsulation*. Let's improve the structure and props of both components in order to restore the encapsulation.

Only the component itself should know its state structure. The state management of `<App>` should move from `<Controls>` (`updateNumber()` method) in the right place: `<App>` component.

Later, `<App>` is modified to provide `<Controls>` with props `onIncrease` and `onDecrease`. These are simple callbacks that update `<App>` state:

```
// Solution: Restore encapsulation
class App extends Component {  
  constructor(props) {
    super(props);
    this.state = { number: 0 };
  }

  render() {
    return (
      <div className="app"> 
        <span className="number">{this.state.number}</span>
        <Controls 
          onIncrease={() => this.updateNumber(+1)}
          onDecrease={() => this.updateNumber(-1)} 
        />
      </div>
    );
  }

  updateNumber(toAdd) {
    this.setState(prevState => ({
      number: prevState.number + toAdd       
    }));
  }
}
```

Now `<Controls>` receives callbacks for increasing and decreasing the number. Notice the decoupling and encapsulation restoration moment: `<Controls>` has no longer the need to access parent instance and modify `<App>` state directly.

Moreover `<Controls>` is transformed into a functional component:

```
// Solution: Use callbacks to update parent state
function Controls({ onIncrease, onDecrease }) {
  return (
    <div className="controls">
      <button onClick={onIncrease}>Increase</button> 
      <button onClick={onDecrease}>Decrease</button>
    </div>
  );
}
```

`<App>` encapsulation is now restored. The component manages its state by itself, as it should be.

Furthermore `<Controls>` no longer depends on `<App>` implementation details. `onIncrease` and `onDecrease` prop functions are called when corresponding button is clicked, and `<Controls>` does not know (and *should not know*) what happens inside those functions.

`<Controls>` reusability and testability significantly increased.

The reuse of `<Controls>` is convenient because it requires only callbacks, without any other dependencies. Testing is also handy: just verify whether callbacks are executed on buttons click (see [6.1 Case study](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#61-case-study-testable-means-well-designed)).

## 3. "Composable"

> A **composable** component is created from the composition of smaller specialized components.

Composition is a way to combine components to create a bigger (composed) component. [Composition is the heart of React](https://medium.com/@dan_abramov/youre-missing-the-point-of-react-a20e34a51e1a).

Fortunately, composition is easy to understand. Take a set of small pieces, combine them, and create a bigger thing.

Let's look at a common frontend application composition pattern. The application is composed of a header at the top, footer at the bottom, sidebar on the left and payload content in the middle.

The application skeleton demonstrates how well composition builds the application. Such organization is expressive and open for understanding.

React composes components expressively and naturally. The library uses a [declarative paradigm](https://en.wikipedia.org/wiki/Declarative_programming) that doesn't suppress the expressiveness of composition. The following components render the described application:

const app = (   <Application>     <Header />     <Sidebar>       <Menu />     </Sidebar>     <Content>       <Article />     </Content>     <Footer />   </Application> );

`<Application>` is composed of `<Header>`, `<Sidebar>`, `<Content>` and `<Footer>`.
`<Sidebar>` has one component `<Menu>`, as well as `<Content>` has one `<Article>`.

How does composition relate with single responsibility and encapsulation? Let's see:

> **Single responsibility principle** describes how to split requirements into components, **encapsulation** describes how to organize these components, and **composition** describes how to glue the whole system back.
>
> 🚀 SRP => encapsulation => composition 章节流程介绍了从单一职责组件逐步构建成一个项目的流程。
>
> - 单一责任原则描述了如何将需求分割成组件
> - 封装描述了如何组织这些组件
> - 组合描述了如何将整个系统粘合在一起
>
> 我们可以把 SRP 组件理解为项目搭建的原材料，封装则是对原材料进行加工，在遵循低耦合原则的前提下，定制成功能更加丰富的公用道具，组合则使用这些道具合理构建出一个完整的项目。

### 3.1 Composition benefits

> 🚀 通过组合 (composition) 的方式搭建大型应用有以下优势：
>
> - Single responsibility：充分利用并维护了组件的单一职责。
>   - SRP 原则在 composition 模式下的优势被更好的放大了。项目中每个组件功能职责清晰，我们拆卸/更换组件，并不会对项目整体造成太大影响。
> - Reusability：组合的方式可以让我们在组件级别操作各种组合方式，我们可以提取并重用公共逻辑或组件集。
> - Flexibility：利用 children props 等技术方法，可以实现组件级别的灵活组合/注入。
> - Efficiency：通过组件组合成一个大型项目是高效的，我们不需要花费更多精力去从零设计项目。

#### Single responsibility

An important aspect of composition is the ability *to compose complex components from smaller specialized components*. This [divide and conquer](https://en.wikipedia.org/wiki/Divide_and_rule) approach helps an authority component conform to single responsibility principle.

Recall the previous code snippet. `<Application>` has the responsibility to render the header, footer, sidebar and main regions.

Makes sense to divide this responsibility into four sub-responsibilities, each of which is implemented by specialized components `<Header>`, `<Sidebar>`, `<Content>` and `<Footer>`. Later composition glues back `<Application>` from these specialized components.

Now comes up the benefit. Composition makes `<Application>` conform to single responsibility principle, by allowing its children to implement the sub-responsibilities.

#### Reusability

Components using composition can reuse common logic. This is the benefit of *reusability*.

For instance, components `<Composed1>` and `<Composed2>` share common code:

```
const instance1 = (
  <Composed1>
    /* Specific to Composed1 code... */
    /* Common code... */
  </Composed1>
);
const instance2 = (
  <Composed2>
    /* Common code... */
    /* Specific to Composed2 code... */
  </Composed2>
);
```

Since code duplication is a bad practice, how to make components reuse common code?

Firstly, encapsulate common code in a new component `<Common>`. Secondly, `<Composed1>` and `<Composed2>` should use composition to include `<Common>`, fixing code duplication:

```
const instance1 = (
  <Composed1>
    <Piece1 />
    <Common />
  </Composed1>
);
const instance2 = (
  <Composed2>
    <Common />
    <Piece2 />
  </Composed2>
);
```

Reusable components favor [Don't repeat yourself](https://en.wikipedia.org/wiki/Don't_repeat_yourself) (DRY) principle. This beneficial practice saves efforts and time.

#### Flexibility

In React a composable component can control its children, usually through `children` prop. This leads to another benefit of *flexibility*.

For example, a component should render a message depending on user's device. Use composition's flexibility to implement this requirement:

```
function ByDevice({ children: { mobile, other } }) {
  return Utils.isMobile() ? mobile : other;
}

<ByDevice>{{
  mobile: <div>Mobile detected!</div>,
  other:  <div>Not a mobile device</div>
}}</ByDevice>
```

`<ByDevice>` composed component renders the message `"Mobile detected!"` for a mobile, and `"Not a mobile device"` for other devices.

#### Efficiency

User interfaces are composable hierarchical structures. Thus composition of components is an *efficient* way to construct user interfaces.

## 4. "Reusable"

> A **reusable** component is written once but used multiple times.
>
> 🚀 DRY (Don't repeat yourself) 原则：当一个组件或逻辑被重复使用时，我们就可以考虑将它提取并加工成公共物料，这可以大大提高项目搭建的效率。

Imagine a fantasy world where software development is mostly reinventing the wheel.

When coding, you can't use any existing libraries or utilities. Even across the application you can't use code that you already wrote.

In such environment, would it be possible to write an application in a reasonable amount of time? Definitely not.

Welcome reusability. Make things work, not reinvent how they work.

### 4.1 Reuse across application

According to *Don't repeat yourself* (DRY) principle, every piece of knowledge must have a single, unambiguous, authoritative representation within a system. The principle advises to avoid repetition.

Code repetition increases complexity and maintenance efforts without adding significant value. An update of the logic forces you to modify all its clones within the application.

Repetition problem is solved with reusable components. Write once and use many times: efficient and time saving strategy.

However you don't get reusability property for free. A component is *reusable* when it conforms to *single responsibility principle* and has correct *encapsulation*.

Conforming to single responsibility is essential:

> **Reuse** of a component actually means the reuse of its **responsibility implementation**.

Components that have only one responsibility are the easiest to reuse.

But when a component incorrectly has multiple responsibilities, its reusage adds a heavy overhead. You want to reuse only one responsibility implementation, but also you get the unneeded implementation of out of place responsibilities.

You want a banana, and you get a banana, plus all the jungle with it.

Correct encapsulation creates a component that doesn't stuck with dependencies. Hidden internal structure and focused props enable the component to fit nicely in multiple places where it's about to be reused.

> 🚀 可重用性依赖于 SRP 单一职责原则以及良好的封装策略。

## 5. "Pure" or "Almost-pure"

> A **pure** component always renders same elements for same prop values.
> An **almost-pure** component always renders same elements for same prop values, and can produce a side effect.

In functional programming terms, a *pure* function always returns the same output for given the same input. Let's see a simple pure function:

```
function sum(a, b) {
  return a + b;
}
sum(5, 10); // => 15
```

For given two numbers, `sum()` function always returns the same sum.

A function becomes *impure* when it returns different output for same input. It can happen because the function relies on global state. For example:

```
let said = false;

function sayOnce(message) {
  if (said) {
    return null;
  }
  said = true;
  return message;
}

sayOnce('Hello World!'); // => 'Hello World!'
sayOnce('Hello World!'); // => null
```

`sayOnce('Hello World!')` on first call returns `'Hello World!'`.

Even when using same argument `'Hello World!'`, on later invocations `sayOnce()` returns `null`. That's the sign of an impure function that relies on a global state: `said` variable.

`sayOnce()` body has a statement `said = true` that modifies the global state. This produces a *side effect*, which is another sign of impure function.

Consequently, pure functions have no side effects and don't rely on global state. Their single source of truth are parameters. Thus pure functions are predictable and determined, are reusable and straightforward to test.

> 🚀 "Pure" 的重要性：
>
> - 纯函数：若接收相同输入，则始终返回相同的输出。
>
> - 纯组件：若接受相同的 props，则始终返回相同的渲染结果。
>
> 纯函数没有副作用，不依赖全局状态。它们的唯一来源是参数。因此，纯函数是可预测的，是确定的，是可重复使用的，是直接测试的。
>
> 使用纯函数可以减少我们使用组件(函数)时的心智负担，我们可以明确知道使用该组件(函数)时会达到什么样的效果，从而避免了无法预判的输出结果。

> 🚀 破坏 "Pure" 的因素，我们将其统称为 "Side Effect" 副作用。例如一些全局变量，随机值，逻辑注入 (例如 react 生命周期) 等。
>
> 更广义的，我们可以将所有**不直接依赖于参数，但与输出变化相关联**的逻辑定义为副作用。

React components should benefit from pure property. Given the same prop values, a pure component (not to be confused with [React.PureComponent](https://facebook.github.io/react/docs/react-api.html#react.purecomponent)) always renders the same elements. Let's take a look:

```
function Message({ text }) {
  return <div className="message">{text}</div>;
}

<Message text="Hello World!" /> 
// => <div class="message">Hello World</div>
```

You are guaranteed that `<Message>` for the same `text` prop value renders the same elements.

It's not always possible to make a component pure. Sometimes you have to ask the environment for information, like in the following case:

```
class InputField extends Component {
  constructor(props) {
    super(props);
    this.state = { value: '' };
    this.handleChange = this.handleChange.bind(this);
  }

  handleChange({ target: { value } }) {
    this.setState({ value });
  }

  render() {
    return (
      <div>
         <input 
           type="text" 
           value={this.state.value} 
           onChange={this.handleChange} 
         />
         You typed: {this.state.value}
      </div>
    );
  }
}
```

`<InputField>` stateful component doesn't accept any props, however renders different output depending on what user types into the input. `<InputField>` has to be impure, because it accesses the environment through input field.

Impure code is a necessary evil. Most of the applications require global state, network requests, local storage and alike. What you can do is *isolate impure code from pure*, a.k.a. apply purification on your components.

Isolated impure code explicitly shows it has side effects, or rely on global state. Being in isolation, impure code has less unpredictability effect on the rest of the system.

> 在组件中，保持 Pure 原则是比较困难的，大多数应用程序都需要全局状态、网络请求、本地存储和类似的东西。
>
> 我们能做的就是将不纯的代码与纯的代码隔离开来，尽可能对组件进行 Pure 处理。

Let's detail into purification examples.

### 5.1 Case study: purification from global variables

I don't like global variables. They break encapsulation, create unpredictable behavior and make testing difficult.

Global variables can be used as mutable or immutable (read-only) objects.

Mutating global variables create uncontrolled behavior of components. Data is injected and modified at will, confusing [reconciliation](https://facebook.github.io/react/docs/reconciliation.html) process. This is a mistake.

If you need a mutable global state, the solution is a predictable application state management. Consider using [Redux](http://redux.js.org/).

An immutable (or read-only) usage of globals is often application's configuration object. This object contains the site name, logged-in user name or any other configuration information.

The following statement defines a configuration object that holds the site name:

```
export const globalConfig = {
  siteName: 'Animals in Zoo'
};
```

Next, `<Header>` component renders the header of an application, including the display of site name `"Animals in Zoo"`:

```
import { globalConfig } from './config';

export default function Header({ children }) {
  const heading = 
    globalConfig.siteName ? <h1>{globalConfig.siteName}</h1> : null;
  return (
     <div>
       {heading}
       {children}
     </div>
  );
}
```

`<Header>` component uses `globalConfig.siteName` to render site name inside a heading tag `<h1>`. When site name is not defined (i.e. `null`), the heading is not displayed.

The first to notice is that `<Header>` is impure. Given same value of `children`, the component returns different results because of `globalConfig.siteName` variations:

```
// globalConfig.siteName is 'Animals in Zoo'
<Header>Some content</Header>
// Renders:
<div>
  <h1>Animals in Zoo</h1>
  Some content
</div>
```

or:

```
// globalConfig.siteName is `null`
<Header>Some content</Header>
// Renders:
<div>
  Some content
</div>
```

The second problem is testing difficulties. To test how component handles `null` site name, you have to modify the global variable `globalConfig.siteName = null` manually:

```
import assert from 'assert';
import { shallow } from 'enzyme';
import { globalConfig } from './config';
import Header from './Header';

describe('<Header />', function() {
  it('should render the heading', function() {
    const wrapper = shallow(
      <Header>Some content</Header>
    );
    assert(wrapper.contains(<h1>Animals in Zoo</h1>));
  });

  it('should not render the heading', function() {
    // Modification of global variable:
    globalConfig.siteName = null;
    const wrapper = shallow(
      <Header>Some content</Header>
    );
    assert(appWithHeading.find('h1').length === 0);
  });
});
```

The modification of global variable `globalConfig.siteName = null` for sake of testing is hacky and uncomfortable. It happens because `<Heading>` has a tight dependency on globals.

To solve such impurities, rather than injecting globals into component's scope, make the global variable an input of the component.

Let's modify `<Header>` to accept one more prop `siteName`. Then wrap the component with [defaultProps()](https://github.com/acdlite/recompose/blob/master/docs/API.md#defaultprops) higher order component (HOC) from [recompose](https://github.com/acdlite/recompose) library. `defaultProps()` ensures fulfilling the missing props with default values:

```
import { defaultProps } from 'recompose';
import { globalConfig } from './config';

export function Header({ children, siteName }) {
  const heading = siteName ? <h1>{siteName}</h1> : null;
  return (
     <div className="header">
       {heading}
       {children}
     </div>
  );
}

export default defaultProps({
  siteName: globalConfig.siteName
})(Header);
```

`<Header>` becomes a pure functional component, and does not depend directly on `globalConfig` variable. The pure version is a named export: `export function Header() {...}`, which is useful for testing.

At the same time, the wrapped component with `defaultProps({...})` sets `globalConfig.siteName` when `siteName` prop is missing. That's the place where impure code is separated and isolated.

Let's test the pure version of `<Header>` (remember to use a named import):

```
import assert from 'assert';
import { shallow } from 'enzyme';
import { Header } from './Header'; // Import the pure Header

describe('<Header />', function() {
  it('should render the heading', function() {
    const wrapper = shallow(
      <Header siteName="Animals in Zoo">Some content</Header>
    );
    assert(wrapper.contains(<h1>Animals in Zoo</h1>));
  });

  it('should not render the heading', function() {
    const wrapper = shallow(
      <Header siteName={null}>Some content</Header>
    );
    assert(appWithHeading.find('h1').length === 0);
  });
});
```

This is great. Unit testing of pure `<Header>` is straightforward. The test does one thing: verify whether the component renders the expected elements for a given input. No need to import, access or modify global variables, no side effects magic.

Well designed components are easy to test (statement detailed in [chapter 6](https://dmitripavlutin.com/7-architectural-attributes-of-a-reliable-react-component/#6-testable-and-tested)), which is visible in case of pure components.

> 🚀 上述例子演示了如何利用 HOC 将副作用与纯组件隔离。
>
> 1. create pure component：组件不再依赖于全局变量，所有输入均从 props 接口获取，遵循了 Pure principle。
> 2. hoc: 将 Global Variable (全局变量 ; 副作用) 从直接注入组件的方式变为通过 props 注入组件。注入逻辑放在 HOC 函数中，实现了 impure logic 与 pure component 的隔离。

### 5.2 Transform almost-pure into pure

> 🚀 在实际应用场景中，我们只需要尽可能的让组件/函数是纯净的。过于纯净的组件/函数，反而会增加开发的心智负担。
>
> 虽然 pure component 相比于 almost-pure component 在可预测性和简单性方面很好，但它需要像 compose() 和 lifecycle() 这样的 HOC 来增加开销。通常情况下，将一个 impure component 转化为一个 almost-pure component 是一个不错的权衡。

Practically at this step you would end isolating impurities. Almost-pure component has a good level of predictability and is easy to test.

But... let's see how deep the rabbit hole goes. The almost-pure version of `<WeatherFetch>` can be transformed into an ideally pure component.

Let's extract the call of `fetch()` into [lifecycle()](https://github.com/acdlite/recompose/blob/master/docs/API.md#lifecycle) HOC from `recompose` library:

```
import { connect } from 'react-redux';  
import { compose, lifecycle } from 'recompose';
import { fetch } from './action';

export function WeatherFetch({ temperature, windSpeed }) {  
   return (
     <WeatherInfo temperature={temperature} windSpeed={windSpeed} />
   );
}

function mapStateToProps(state) {  
  return {
    temperature: state.temperate,
    windSpeed: state.windSpeed
  };
}

export default compose(
  connect(mapStateToProps, { fetch }),
  lifecycle({
    componentDidMount() {
      this.props.fetch();
    }
  })
)(WeatherFetch);
```

`lifecycle()` HOC accepts an object with lifecycle methods. `componentDidMount()` is handled by HOC, which calls `this.props.fetch()`. This way the side effect is extracted from `<WeatherFetch>`.

Now `<WeatherFetch>` is a *pure* component. It doesn't have side effects, and always renders same output for same `temperature` and `windSpeed` prop values.

While the pure version of `<WeatherFetch>` is great in terms of predictability and simplicity, it adds an overhead by requiring HOCs like `compose()` and `lifecycle()`. Usually transforming an *impure* component to an *almost-pure* is a decent trade-off.

## 6. "Testable" and "Tested"

> A **tested** component is verified whether it renders the expected output for a given input.
> A **testable** component is easy to test.

How to be sure that a component works as expected? You can say: "I manually verify how it works."

If you plan to manually verify *every* component modification, sooner or later you're going to skip this tedious task. Sooner or later small defects are going to make through.

That's why is important to automate the verification of components: do unit testing. Unit tests make sure that your components are working correctly every time you make a modification.

Unit testing is *not only about early bugs detection*. Another important aspect is the ability to verify how well components are built architecturally.

The following statement I find especially important:

> A component that is **untestable** or **hard to test** is most likely **badly designed**.

A component is hard to test because it has a lot of props, dependencies, requires mockups and access to global variables: that's the sign of a bad design.

When the component has *weak architectural design*, it becomes *untestable*. When the component is untestable, you simply skip writing unit tests: as result it remains *untested*.

In conclusion, the reason why many applications are untested is incorrectly designed components. Even if *you want* to test such an application, *you can't*.

## 7. "Meaningful"

> A **meaningful** component is easy to understand what it does.

It's hard overstate the importance of readable code. How many times did you stuck with obscured code? You see the characters, but don't see the meaning.

Developer spends most of the time reading and understanding code, than actually writing it. Coding activity is 75% of time understanding code, 20% of time modifying existing code and only 5% writing new ([source](https://blog.codinghorror.com/when-understanding-means-rewriting/)).

A slight additional time spent on readability reduces the understanding time for teammates and yourself in the future. The naming practice becomes important when the application grows, because understanding efforts increase with volume of code.

*Reading* meaningful code is easy. Nevertheless *writing* meaningfully requires clean code practices and constant effort to express yourself clearly.

> 🚀 相比于注释，形象的命名往往更重要。

### 7.1 Component naming

> 🚀 Meaningful component naming RULEs
>
> - 组件名称用一个或多个单词（主要是名词）连接，并使用普通大小写。
> - 一个组件越是专业，它的名字就可能包含越多的词。有时候我们为了描述清楚该组件的职责，不得不用更多的词汇，但这是不可避免的，因为更多的表达总比不清楚的要好得多。
> - 一个词代表一个概念：我们应该为每个概念选一个词，然后在整个应用程序中保持一致的关系。当同一个概念由许多词来表示时，可读性就会受到影响。

#### Pascal case

Component name is a concatenation of one or more words (mostly nouns) in [pascal case](https://en.wikipedia.org/wiki/PascalCase). For instance `<DatePicker>`, `<GridItem>`, `<Application>`, `<Header>`.

#### Specialization

The more specialized a component is, the more words its name might contain.

A component named `<HeaderMenu>` suggests a menu in the header. A name `<SidebarMenuItem>` indicates a menu item located in sidebar.

A component is easy to understand when the name meaningfully implies the intent. To make this happen, often you have to use verbose names. That's fine: more verbose is better than less clear.

Suppose you navigate some project files and identify 2 components: `<Authors>` and `<AuthorsList>`. Based on names only, can you conclude the difference between them? Most likely not.

To get the details, you have to open `<Authors>` source file and explore the code. After doing that, you realize that `<Authors>` fetches authors list from server and renders `<AuthorsList>` presentational component.

A more specialized name instead of `<Authors>` doesn't create this situation. Better names are `<FetchAuthors>`, `<AuthorsContainer>` or `<AuthorsPage>`.

[Favor clarity over brevity](https://signalvnoise.com/posts/3250-clarity-over-brevity-in-variable-and-method-names).

#### One word - one concept

A word represents a concept. For example, a *collection of rendered items* concept is represented by *list* word.

Pick one word per concept, then keep the relation consistent within the whole application. The result is a predicable mental mapping of *words - concepts* that you get used to.

Readability suffers when the same concept is represented by many words. For example, you define a component that renders a list of orders `<OrdersList>`, and another that renders a list of expenses `<ExpensesTable>`.

The same concept of a *collection of rendered items* is represented by 2 different words: *list* and *table*. There's no reason to use different words for the same concept. It adds confusion and breaks consistency in naming.

Name the components `<OrdersList>` and `<ExpensesList>` (using *list* word) or `<OrdersTable>` and `<ExpensesTable>` (using *table* word). Use whatever word you feel is better, just keep it consistent.

#### Code comments

Meaningful names for components, methods and variables are enough for making the code readable. Thus, comments are mostly redundant.

## 8. Do continuous improvement

At the same time of composing this article, I was reading an interesting book by William Zinsser named ["On Writing Well: The Classic Guide to Writing Nonfiction"](https://www.amazon.com/Writing-Well-Classic-Guide-Nonfiction/dp/0060891548). It's an amazing piece for improving the writing skills.

The book is important for me because I don't have any writing studies. I'm an avid master of computer science who usually writes scripts that feed computers only.

While most of the book makes you better at [non-fiction writing](https://en.wikipedia.org/wiki/Non-fiction), one moment drove my attention. William Zinsser states that:

> I then said that **rewriting** is the essence of writing. I pointed out that professional writers **rewrite** their sentences over and over and then **rewrite** what they have rewritten.

To produce a quality text, you have to rewrite your sentence multiple times. Read the written, simplify confusing places, use more synonyms, remove clutter words - then repeat until you have an enjoyable piece of text.

Interestingly that the same concept of rewriting applies to designing the components.

Sometimes it's hardly possible to create the right components structure at the first attempt. It happens because:

- A tight deadline doesn't allow spending enough time on system design
- The initially chosen approach appears to be wrong
- You've just found an open source library that solves the problem better
- or any other reason.

Finding the right organization is a series of trials and reviews. The more complex a component is, the more often it requires verification and refactoring.

*Does the component implement a single responsibility, is it well encapsulated, is it enough tested?* If you can't answer a certain *yes*, determine the weak part (by comparing against presented above 7 attributes) and refactor the component.

Pragmatically, development is a never stopping process of reviewing previous decisions and making improvements.

> 🚀 好的组件代码是不断重构的过程：
>
> 有时，在第一次尝试时，几乎不可能创造出正确的组件结构。发生这种情况是因为：
>
> - 时间紧迫，不允许在系统设计上花足够的时间
> - 最初选择的方法似乎是错误的
> - 你刚刚发现一个开源库能更好地解决这个问题
> - 或任何其他原因。
>
> 找到正确的组织是一系列的试验和审查。一个组件越复杂，越经常需要验证和重构。你需要不断询问自己：
>
> - 该组件是否实现了单一的责任
> - 它是否被很好地封装
> - 它是否被充分地测试
>
> 如果你不能肯定地回答 "是"，那就确定薄弱的部分（通过与上述7个属性进行比较），并重构该组件。

## 10. Conclusion

> 🚀 7 大原则总结
>
> - 🔥 SRP 单一职责原则
> - 🔥 低耦合封装原则
> - Composition
> - Reusable
> - Pure
> - 遵循有意义的命名规则
> - 测试
>
> 个人认为：
>
> - 合理的职责划分及封装是关键
>   - 单一职责
>   - 所有通信均通过 props 传递，隐藏组件内部信息
> - 组件保持可重用和纯净的特性，是检验是否达到上述两大原则的标准
>   - 隔离 side effect 和 pure logic
> - 组合则是使用这些组件搭建复杂大型项目的一种模式
>
> 🌈 实现逻辑隔离/提取/封装等所用到的技术方法：HOC ; props render ; hooks 

The presented 7 characteristics suggest the same idea from different angles:

> A **reliable** component implements one responsibility, hides its internal structure and provides effective set of props to control its behavior.

Single responsibility and encapsulation are the base of a solid design.

*Single responsibility* suggests to create a component that implements only one responsibility and has one reason to change.

*Encapsulated* component hides its internal structure and implementation details, and defines props to control the behavior and output.

*Composition* structures big and authority components. Just split them into smaller chunks, then use composition to glue the whole back, making complex simple.

*Reusable* components are the result of a well designed system. Reuse the code whenever you can to avoid repetition.

Side effects like network requests or global variables make components depend on environment. Make them *pure* by returning same output for same prop values.

*Meaningful* component naming and expressive code are the key to readability. Your code must be understandable and welcoming to read.

*Testing* is not only an automated way of detecting bugs. If you find a component difficult to test, most likely it's incorrectly designed.

A quality, extensible and maintainable, thus successful application stands on shoulders of reliable components.