---
title: 从Vue到React 18系列 - 2.组件的基本知识
toc: true
date: 2023-4-12 17:36:45
tags: [React.js, TypeScript, CookBook]
categories:
  - [前端技术, 从Vue到React 18系列]
---
以Vue2为主要参考对比，辅以部分Vue3概念，向React 18迁移的CookBook

## React类组件/Vue Options组件

上述示例中均以函数式组件为示例。但是在学习React的过程中，类组件依然是一个绕不开的话题。React的类组件之于函数式组件就类似Vue3中的script
setup之于Vue2的选项式API。

来自React官方文档的Clock组件：Clock.jsx

```jsx
import React from "react";

class Clock extends React.Component {
  constructor(props) {
    super(props);
    this.state = {date: new Date()};
  }

  componentDidMount() {
    this.timerID = setInterval(
      () => this.tick(),
      1000
    );
  }

  componentWillUnmount() {
    clearInterval(this.timerID);
  }

  tick() {
    this.setState({
      date: new Date()
    });
  }

  render() {
    return (
      <div>
        <h1>Hello, world!</h1>
        <h2>It is {this.state.date.toLocaleTimeString()}.</h2>
      </div>
    );
  }
}

export default Clock;
```

用Vue Options Api改写：Clock.vue

```html

<template>
    <div>
        <h1>Hello, world!</h1>
        <h2>It is {date.toLocaleTimeString()}.</h2>
    </div>
</template>
<script>
    export default {
        data() {
            return {
                date: new Date(),
                timerID: null
            }
        },
        methods: {
            tick() {
                this.date = new Date()
            }
        },
        mounted() {
            this.timerId = setInterval(() => this.tick(), 1000)
        },
        beforeDestroy() {
            clearInterval(this.timerID);
        }
    }
</script>
```

简单地来说，React类组件中的state大概相当于Vue2中的data()
，state是属于当前组件私有的数据。以上述代码为例，在Vue中，若要使响应式数据的变化反馈到模板上，我们只需要为this.date赋新值即可；但在React中需要使用this.setState()
方法进行赋值操作。直接向state赋值是无法重新渲染组件的，如：

```jsx
this.state.date = new Date();   //这样不会触发DOM更新
```

关于setState（或useStates）的更多信息，参见`第6节：深入组件`

### 生命周期对比

详细的React[生命周期图](https://projects.wojtekmaj.pl/react-lifecycle-methods-diagram/)

```jsx
class HelloWorld extends React.Component {
  // 继承react的props，和设置state的初始化
  constructor(props) {
    super(props); //不能缺少
    this.state = {
      color: props.initialColor
    };
  }

  // 类似Vue的beforeMoute，只mount前调用一次，在 render 之前调用
  componentWillMount();

  /* 类似Vue的template 该方法会创建一个虚拟DOM，用来表示组件的输出。
  render方法需要满足下面几点: 
    1.只能通过 this.props 和 this.state 访问数据（不能修改）
    2.可以返回 null,false 或者任何React组件 
    3.只能出现一个顶级组件，不能返回一组并列元素（**react16也支持返回数组了**） 
    4.不能改变组件的状态 
    5.不能修改DOM的输出 
  */
  render() {
    return <h1>Hello World!</h1>;
  }

  // 类似Vue的mounted, 在 render 之后调用,从这里开始可以通过 ReactDOM.findDOMNode(this) 获取到组件的 DOM 节点。
  componentDidMount() {
    //添加事件订阅，额外的DOM处理
  };

  // 通过改变props或state来驱动视图的更改，会触发以下钩子
  componentWillReceiveProps();

  // 在react中这是一个性能优化的关键点，当父组件改变，全部子组件都会重新渲染
  // 可以通过该钩子返回false来阻止渲染，此处还有另外一种PureComponent实例
  // 参见https://reactjs.org/docs/react-api.html#reactpurecomponent
  // Vue对此做了优化，参见https://v2.cn.vuejs.org/v2/guide/comparison.html#运行时性能
  shouldComponentUpdate(nextProps, nextState) {
    //return boolean
  };

  // 类似Vue的beforeUpdate
  componentWillUpdate();

  render();  //再次触发render，实际上Class Component中只会有一个render函数

  // 类似Vue的updated 
  componentDidUpdate();

  // 类似Vue的beforeDestroy,组件销毁之前被调用,在此钩子中，出于性能的考虑需移除在componentDidMount添加的事件订阅，网络请求等。
  componentWillUnmount() {
    //移除在componentDidMount添加的事件订阅，网络请求等
  };

}
```

### 类组件的事件绑定

React在JSX中传递的事件不是一个字符串，而是一个函数。比如在onClick中，它所绑定的函数会丢失this指向，所以要记得使用箭头函数或通过手动bind来将事件处理函数的this一直指向当前实例化对象。常见的绑定方式有三种：

```jsx
class Test extends React.Component {
  constructor(props: any) {
    super(props)
    this.handleClick = this.handleClick.bind(this);  //构造器预先bind方式要这样写
  }

  handleClick(text: string) {
    console.log(text)
  }

  noParams() {
    console.log("hello")
  }

  render(): React.ReactNode {
    return (
      <div>
        <button onClick={() => {
          this.handleClick("Yajuu 114514")
        }}>箭头函数方式
        </button>
        {/* render中bind的函数不可以传参，因为bind是在函数的引用上使用的，因为它是Function.prototype.bind() */}
        <button onClick={this.noParams.bind(this)}>render中bind方式</button>
        <button onClick={this.handleClick("Yajuu 114514")}>构造器预先bind方式</button>
      </div>
    )
  }
}
```

### 函数式组件的事件绑定

参见[onClick等事件的小坑](###onClick等事件的小坑)

## React函数式组件/Vue Composition组件

**Function Component 是更彻底的状态驱动抽象，甚至没有 Class Component 生命周期的概念，只有一个状态，而 React 负责同步到
DOM。**

### 写法

总的来说，Vue Compotision的写法很像React函数式组件。以`<script setup>`写法举例：

```html

<template>
    <div>
        <h1>{{hello}}</h1>
        <button @click="hello='哼哼哼'">点我</button>
    </div>
</template>
<script setup lang="ts">
    import {ref} from "vue";

    const hello = ref("Hello World!")
</script>
```

下面是React函数式组件写法：

```tsx
import {FC} from 'react'
import {useState} from 'react'

const Test: FC = () => {
  const [hello, setHello] = useState("Hello World!")
  return (
    <div>
      <h1>{hello}</h1>
      <button onClick={() => setHello("哼哼哼")}>点我</button>
    </div>
  )
}
```

在函数式组件中要使用useState hook进行state的声明和修改。useState的参数即为该state的初始值，使用数组解构可以得到一个访问state的引用以及用于更新该state的函数。在Vue
Composition中，以ref()声明的响应式变量需要使用.value进行访问及更新，而React中的useState则是使用解构出的引用及更新函数分别进行访问/更新。

### 从生命周期钩子迁移到useEffect()

在Composition Api中访问生命周期可以使用形如onMounted(()=>{})
的钩子，而在函数式组件中则可以使用[useEffect](https://zh-hans.reactjs.org/docs/hooks-effect.html)这个较为通用的hook。

useEffect的简单使用可以看下面的例子。若要进一步了解，参见[精读《useEffect完全指南》](https://zhuanlan.zhihu.com/p/60277120)

```javascript
import {useState, useEffect} from 'react'

function App() {
  const [num, setNum] = useState(1)
  const [count, setCount] = useState(1)
  useEffect(() => {
    console.log('哈哈哈哈')
  })
  return (
    <div>
      <button onClick={() => setNum(num + 1)}>点我修改num</button>
      <button onClick={() => setCount(count + 1)}>点我count</button>
    </div>
  )
}
```

第二个参数不传

```javascript
 useEffect(() => {
  console.log('哈哈哈哈')
})
```

当` useEffect `第二个参数不传时，`页面初始`和`数据更新`
的时候，第一个参数函数都会执行，所以此时初始页面时会输出一次` 哈哈哈哈 `
，然后无论你点修改num或者修改count的按钮时，也都会输出`哈哈哈哈`

第二个参数传空数组

```javascript
 useEffect(() => {
  console.log('哈哈哈哈')
}, [])
```

当` useEffect `第二个参数传` [] `时，那么第一个参数函数只有在` 页面初始 `的时候才会执行，也就是只执行一次，无论你点修改num或者修改count的按钮，都不会执行这个函数

第二个参数传非空数组

```javascript
 // ①
useEffect(() => {
  console.log('哈哈哈哈')
}, [num])

// ②
useEffect(() => {
  console.log('哈哈哈哈')
}, [count])

// ③
useEffect(() => {
  console.log('哈哈哈哈')
}, [num, count])
```

当` useEffect `第二个参数传非空数组时，` 页面初始 `和` 依赖的数据发生更新 `的时候，第一个参数函数都会执行。比如上方例子：

- 只有按修改num按钮时，才会再次输出` 哈哈哈哈`
- 只有按修改count按钮时，才会再次输出` 哈哈哈哈`
- 无论按哪个按钮都会再次输出` 哈哈哈哈`

return清除操作

```javascript
useEffect(() => {
  const timeId = setTimeout(() => console.log('我是定时器'), 1000)
  return () => clearTimeout(timeId)
})
```

React 会在组件卸载的时候执行清除操作。effect 在每次渲染的时候都会执行。React 会在执行当前 effect 之前对上一个 effect 进行清除。

## 一些组件的相关知识

尽管上文中已经介绍过部分要点，不过还是有必要再总结一遍：

1. 无论是使用函数或是类来声明一个组件，它决不能修改它自己的 props。
2. 所有 React 组件都必须是纯函数，并禁止修改其自身 props 。
3. React是单项数据流，父组件改变了属性，那么子组件视图会更新。
4. 属性 props 是外界传递过来的，状态 state 是组件本身的，状态可以在组件中任意修改组件的属性和状态改变都会更新视图。

### 注册及使用组件

React组件没有全局注册和局部注册的概念，我们可以使用ES6的class或function来创建组件，调用通过import导入组件实例。

类组件：

```jsx
import React from "react";

class Hello extends React.Component {
  constructor(props) {
    super(props);
    // 这里类似于Vue Options中声明一组data(){return {}}
    this.state = {helloText: 'Welcome to Shimokita'};
  }

  // 类似于Vue的模板
  render() {
    return (
      <div>
        <h1>Hello, world!</h1>
        <h2>{this.state.helloText}</h2>
      </div>
    );
  }
}

export default Hello;
```

函数式组件：

```jsx
import {FC, useState} from "react";

const Test: FC = () => {
  const [helloText] = useState("Yajuu Senpai")
  return (
    <div>{helloText}</div>
  )
}
export default Test;
```

在父组件中使用：

```jsx
import {FC, useState} from 'react'
import Test from './Test'
import Hello from './Hello';

const App: FC = () => {
  const [value, setValue] = useState('')
  return (
    <div className='App'>
      <Test></Test>
      <Hello></Hello>
    </div>
  )
}

export default App
```

### 组件的生命周期（八股文限定版）

**还别说，除开constructor真的有八种**

重要的事要强调：详细的React[生命周期图](https://projects.wojtekmaj.pl/react-lifecycle-methods-diagram/)

constructor（调用React.Component的构造器，进行state的初始化及绑定事件处理方法等工作）

componentWillMount（类似beforeMounted，在这里调用this.setState不会引起组件的重新渲染）

render：根据props和state返回一个React Element。在这里不能调用this.setState，否则会改变组件的状态

componentDidMount：组件挂载到DOM后调用。此时已经可以获取到DOM，类似于Mounted。在这里调用this.setState也会引起组件的重新渲染。

componentWillReceiveProps(newProps)
：父组件传给该组件新的props时触发的钩子，一般需要比较新props与this.props来决定是否执行props变化后的逻辑，例如根据新props调用this.setState触发重新渲染。

shouldComponentUpdate(newProps, newState)
：该方法决定组件是否继续更新。一般通过比较新props、state与组件当前的props、state来决定该方法返回true还是false，从而减少组件不必要的渲染，优化性能。

componentWillUpdate(newProps, newState)：该方法在render调用前执行。

componentDidUpdate(oldProps, oldState)：组件更新后的钩子，可以在此处操作更新后的DOM

componentWillUnmount：卸载组件前调用，类似于beforeDestory。

### 类组件、函数式组件的一些区别

若要学习更多，参见[20. 从类组件迁移到函数式组件](##20. 从类组件迁移到函数式组件)

- 类组件可以获取实例化的 this，并且基于 this 做各种操作，函数组件不可以。
- 类组件有生命周期，函数组件没有。
- 类组件内部可以定义并维护 state， 函数组件为无状态组件（可以通过hooks useState实现）
- 类组件需要继承 Class，函数组件不需要。所以函数组件的性能比类组件的性能要高，因为类组件使用的时候要实例化，而函数组件直接执行函数取返回结果即可。

函数组件相比较类组件，优点是更轻量与灵活，便于逻辑的拆分复用。

### 类组件、函数式组件在TypeScript中

#### 函数式组件React.FC<>

* React.FC是函数式组件，是在TypeScript使用的一个泛型，FC就是FunctionComponent的缩写，事实上React.FC可以写成React.FunctionComponent：

  ```jsx
  const App: FunctionComponent<{ message: string }> = ({ message }) => (
    <div>{message}</div>
  );
  ```

* React.FC 包含了 PropsWithChildren 的泛型，不用显式的声明 props.children 的类型。React.FC<>
  对于返回类型是显式的，而普通函数版本是隐式的（否则需要附加注释）*注意，React 18与低版本在此处有所不同，参见下文*

* React.FC提供了类型检查和自动完成的静态属性：displayName，propTypes和defaultProps（注意：defaultProps与React.FC结合使用会存在一些问题）。

* 使用React.FC来写 React 组件的时候，是不能用setState的，取而代之的是useState()、useEffect等 Hook API。

##### React 18和低版本的FC定义区别

* React 18不再预定义PropsWithChildren

  ```typescript
  type FC<P = {}> = FunctionComponent<P>;
  interface FunctionComponent<P = {}> {
    (props: P, context?: any): ReactElement<any, any> | null;
    propTypes?: WeakValidationMap<P> | undefined;
    contextTypes?: ValidationMap<any> | undefined;
    defaultProps?: Partial<P> | undefined;
    displayName?: string | undefined;
  }
  ```

* React < 18则有

  ```typescript
  type FC<P = {}> = FunctionComponent<P>;
  interface FunctionComponent<P = {}> {
    (props: PropsWithChildren<P>, context?: any): ReactElement<any, any> | null;
    propTypes?: WeakValidationMap<P> | undefined;
    contextTypes?: ValidationMap<any> | undefined;
    defaultProps?: Partial<P> | undefined;
    displayName?: string | undefined;
  }
  ```

#### 类组件class xx extends React.Component

* 如需定义 class 组件，需要继承类组件 React.Component。
* 在TypeScript中，React.Component是通用类型（aka React.Component<PropType, StateType>），因此要为其提供（可选）prop和state类型参数。

### 定义组件的Props属性及默认值（参数类型检查）

#### 使用TypeScript的类型检查

```tsx
import React from 'react'
import {FC} from 'react'

// 声明一个人（大嘘）
// TS类型声明的精华都在接口上，可以方便地实现联合类型、子对象的类型检测等
interface People {
  pName: string,   //必选项 姓名
  age?: number,    //可选项 年龄
  company?: string //可选项 公司
}

// 类组件，可能是看起来最像Vue的写法
class TestClass extends React.Component<People, {}> {
  // 不加People类型的话可以声明任意个甚至零个默认值
  static defaultProps: People = {
    pName: '李田所',
    age: 24,
    company: 'COAT',
  }

  constructor(props: any) {
    super(props)
  }

  render(): React.ReactNode {
    return (
      <div>
        <p>{this.props.pName}</p>
        <p>{this.props.age}</p>
        <p>{this.props.company}</p>
      </div>
    )
  }
}

// 不需要设置默认值的FC组件，这样写很简单。
// 你是一个一个一个TypeScript啊啊啊啊啊啊
const TestFC: FC<People> = (props: People) => {
  return (
    <div>
      <p>{props.pName}</p>
      <p>{props.age}</p>
      <p>{props.company}</p>
    </div>
  )
}

// 如果想在FC组件上设置默认值，试试解构+默认参数似乎不错。不要跳类型体操，好吗？
const TestWithDefaultProps: FC<People> = ({pName, age = 24, company = "NicoNico"}) => {
  return (
    <div>
      <p>{pName}</p>
      <p>{age}</p>
      <p>{company}</p>
    </div>
  )
}

const App: FC = () => {
  return (
    <div className='App'>
      <TestClass pName="张三浦" age={114514}></TestClass>
      <TestFC pName="哼哼哼"></TestFC>
      <TestWithDefaultProps pName="赵木村"></TestWithDefaultProps>
    </div>
  )
}

export default App
```

#### 使用JavaScript的类型检查

可以不要开历史的倒车吗？还要记得安装一个库```npm install prop-types --save```

```jsx
import React from 'react'
import {FC} from 'react'
import PropTypes from 'prop-types';

class Greeting extends React.Component {
  render() {
    return (
      <div>
        <h1>Hello, {this.props.pName}</h1>
        <h2>I'm {this.props.age} years old</h2>
        <h3>{this.props.company}爆破</h3>
      </div>
    );
  }
}

Greeting.propTypes = {
  pName: PropTypes.string.isRequired,  //必选项需要在该属性上调用isRequired
  age: PropTypes.number,
  company: PropTypes.string
};
Greeting.defaultProps = {
  pName: '李田所',
  company: 'COAT'
}

// 函数式组件像上文TS写法中TestWithDefaultProps组件一样直接解构就行，懒得写了

const App: FC = () => {
  return (
    <div className='App'>
      <Greeting age={24}></Greeting>
    </div>
  )
}

export default App
```

更多用法，如声明联合类型、检查对象由特定的类型值组成等请参考[使用PropTypes类型检查](https://react.docschina.org/docs/typechecking-with-proptypes.html)

## 使用state的一些要点

如果尝试使用`setState()`更新的数据依赖于this.props或this.state，那么应该向`setState()`传入一个函数而非一个对象，因为React
可能会把多个 `setState()`调用合并成一个调用，而`this.props` 和 `this.state` 可能会异步更新。

简而言之，第二次更新state时的逻辑如果是基于上一次的state，如state.counter += 1，那么它可能不会得到你预期中的值。

这个函数用上一个 state 作为第一个参数，将此次更新被应用时的 props 做为第二个参数：

```jsx
// Correct
this.setState((state, props) => ({
  counter: state.counter + props.increment
}));
```

阅读官网的setState函数有助于理解它与Vue的响应式数据修改的区别。

setState()有两个参数，分别是上述的updater函数以及可选的callback函数。如：

```jsx
this.setState((state) => ({
  date: new Date()
}), () => {
  console.log(114)
});
```

每次时间更新后都会调用一次`console.log(114)`。

### 类似于Vue2中操作响应式对象或数组时需要注意的点

在修改state中的引用对象或数组时需要先进行浅拷贝再修改。

```jsx
import React from 'react'
import {FC, useState} from 'react'

function Demo() {
  const [obj, setObj] = useState({
    name: '我修院',
    lines: ['OC']
  })
  const onClickName = () => {
    setObj({...obj, name: '淳平'})
  }
  const onClickLines = () => {
    setObj({...obj, lines: [...obj.lines, '嗯', '嘛', '啊']})
  }
  return (
    <div>
      <button onClick={onClickName}>{obj.name}</button>
      <button onClick={onClickLines}>上菜</button>
      {obj.lines.map(v => <p>{v}</p>)}
    </div>
  )
}
```

##  
