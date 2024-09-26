---
title: 从Vue到React 18系列 - 6.深入组件
toc: true
date: 2023-4-12 17:36:49
tags: [React.js, TypeScript, CookBook]
categories:
  - [前端技术, 从Vue到React 18系列]
---
以Vue2为主要参考对比，辅以部分Vue3概念，向React 18迁移的CookBook

## React Hook

React Hook既强大又灵活，它在有状态的函数式组件中十分重要。由于本人学习React的时间很短，尚不敢打包票说自己对Hooks有如何深刻的了解，故在此分享一些读过的文章以作参考学习之用。

* [React官方文档-Hooks](https://zh-hans.reactjs.org/docs/hooks-intro.html#motivation)
* [理解 React Hooks](https://cloud.tencent.com/developer/article/1360473?from=15425)
* [useState和useRef的区别](https://cloud.tencent.com/developer/article/1884542)
* [精读《React Hooks》](https://zhuanlan.zhihu.com/p/49408348)
* [精读《怎么用 React Hooks 造轮子》](https://zhuanlan.zhihu.com/p/50274018)
* [精读《React Hooks 最佳实践》](https://zhuanlan.zhihu.com/p/81752821)
* [精读《React Hooks 数据流》](https://zhuanlan.zhihu.com/p/126476910)

## React中使用异步组件（懒加载组件）

参考官网文档[React.lazy](https://zh-hans.reactjs.org/docs/code-splitting.html#reactlazy)

*注意！你的组件必须是export default形式！否则会报类型转换错误！*

## React和Vue中的Key、Diff

[为什么 React 的 Diff 算法不采用 Vue 的双端对比算法？](https://juejin.cn/post/7116141318853623839)

## Render的Capture Value特性

本节参考[精读《Function VS Class 组件》](https://zhuanlan.zhihu.com/p/59558396)及[精读《useEffect 完全指南》](https://zhuanlan.zhihu.com/p/60277120)，部分内容有删改。

### 每次 Render 都有自己的 Props 与 State

可以认为每次 Render 的内容都会形成一个快照并保留下来，因此当状态变更而 Rerender 时，就形成了 N 个 Render 状态，而每个 Render 状态都拥有自己固定不变的 Props 与 State。

看下面的 `count`：

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>Click me</button>
    </div>
  );
}
```

在每次点击时，`count` 只是一个不会变的常量，而且也不存在利用 `Proxy` 的双向绑定，只是一个常量存在于每次 Render 中。

初始状态下 `count` 值为 `0`，而随着按钮被点击，在每次 Render 过程中，`count` 的值都会被固化为 `1`、`2`、`3`：

```jsx
// During first render
function Counter() {
  const count = 0; // Returned by useState()
  // ...
  <p>You clicked {count} times</p>;
  // ...
}

// After a click, our function is called again
function Counter() {
  const count = 1; // Returned by useState()
  // ...
  <p>You clicked {count} times</p>;
  // ...
}

// After another click, our function is called again
function Counter() {
  const count = 2; // Returned by useState()
  // ...
  <p>You clicked {count} times</p>;
  // ...
}
```

其实不仅是对象，函数在每次渲染时也是独立的。这就是 **Capture Value** 特性。

### 每次 Render 都有自己的事件处理

解释了为什么下面的代码会输出 `5` 而不是 `3`:

```jsx
const App = () => {
  const [temp, setTemp] = React.useState(5);

  const log = () => {
    setTimeout(() => {
      console.log("3 秒前 temp = 5，现在 temp =", temp);
    }, 3000);
  };

  return (
    <div
      onClick={() => {
        log();
        setTemp(3);
        // 3 秒前 temp = 5，现在 temp = 5
      }}
    >
      xyz
    </div>
  );
};
```

在 `log` 函数执行的那个 Render 过程里，`temp` 的值可以看作常量 `5`，**执行** `**setTemp(3)**` **时会交由一个全新的 Render 渲染**，所以不会执行 `log` 函数。**而 3 秒后执行的内容是由** `**temp**` **为** `**5**` **的那个 Render 发出的**，所以结果自然为 `5`。

原因就是 `temp`、`log` 都拥有 Capture Value 特性。

### 每次 Render 都有自己的 Effects

`useEffect` 也一样具有 Capture Value 的特性。

`useEffect` 在实际 DOM 渲染完毕后执行，那 `useEffect` 拿到的值也遵循 Capture Value 的特性：

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    document.title = `You clicked ${count} times`;
  });

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>Click me</button>
    </div>
  );
}
```

上面的 `useEffect` 在每次 Render 过程中，拿到的 `count` 都是固化下来的常量。

### 如何绕过 Capture Value

利用 `useRef` 就可以绕过 Capture Value 的特性。**可以认为** `**ref**` **在所有 Render 过程中保持着唯一引用，因此所有对** `**ref**` **的赋值或取值，拿到的都只有一个最终状态**，而不会在每个 Render 间存在隔离。

```jsx
function Example() {
  const [count, setCount] = useState(0);
  const latestCount = useRef(count);

  useEffect(() => {
    // Set the mutable latest value
    latestCount.current = count;
    setTimeout(() => {
      // Read the mutable latest value
      console.log(`You clicked ${latestCount.current} times`);
    }, 3000);
  });
  // ...
}
```

也可以简洁的认为，`ref` 是 Mutable 的，而 `state` 是 Immutable 的。

## 从类组件迁移到函数式组件

本节参考[精读《Function VS Class 组件》](https://zhuanlan.zhihu.com/p/59558396)，部分内容有删改。

### 怎么替代 shouldComponentUpdate

说实话，Function Component 替代 `shouldComponentUpdate` 的方案并没有 Class Component 优雅，代码是这样的：

```jsx
const Button = React.memo(props => {
  // your component
});
```

或者在父级就直接生成一个自带 `memo` 的子元素：

```jsx
function Parent({ a, b }) {
  // Only re-rendered if `a` changes:
  const child1 = useMemo(() => <Child1 a={a} />, [a]);
  // Only re-rendered if `b` changes:
  const child2 = useMemo(() => <Child2 b={b} />, [b]);
  return (
    <>
      {child1}
      {child2}
    </>
  );
}
```

相比之下，Class Component 的写法通常是：

```jsx
class Button extends React.PureComponent {}
```

这样就自带了 `shallowEqual` 的 `shouldComponentUpdate`。

### 怎么替代 componentDidUpdate

由于 `useEffect` 每次 Render 都会执行，因此需要模拟一个 `useUpdate` 函数：

```jsx
const mounting = useRef(true);
useEffect(() => {
  if (mounting.current) {
    mounting.current = false;
  } else {
    fn();
  }
});
```

更多可以查看 [精读《怎么用 React Hooks 造轮子》](https://link.zhihu.com/?target=https%3A//github.com/dt-fe/weekly/blob/master/80.%E7%B2%BE%E8%AF%BB%E3%80%8A%E6%80%8E%E4%B9%88%E7%94%A8%20React%20Hooks%20%E9%80%A0%E8%BD%AE%E5%AD%90%E3%80%8B.md%23componentdidupdate)

### 怎么替代 forceUpdate

React 官方文档提供了一种方案：

```jsx
const [ignored, forceUpdate] = useReducer(x => x + 1, 0);

function handleClick() {
  forceUpdate();
}
```

每次执行 `dispatch` 时，只要 `state` 变化就会触发组件更新。当然 `useState` 也同样可以模拟：

```jsx
const useUpdate = () => useState(0)[1];
```

我们知道 `useState` 下标为 1 的项是用来更新数据的，而且就算数据没有变化，调用了也会刷新组件，所以我们可以把返回一个没有修改数值的 `setValue`，这样它的功能就仅剩下刷新组件了。

更多可以查看 [精读《怎么用 React Hooks 造轮子》](https://link.zhihu.com/?target=https%3A//github.com/dt-fe/weekly/blob/master/80.%E7%B2%BE%E8%AF%BB%E3%80%8A%E6%80%8E%E4%B9%88%E7%94%A8%20React%20Hooks%20%E9%80%A0%E8%BD%AE%E5%AD%90%E3%80%8B.md%23force-update)

### state 拆分过多

`useState` 目前的一种实践，是将变量名打平，而非像 Class Component 一样写在一个 State 对象里：

```jsx
class ClassComponent extends React.PureComponent {
  state = {
    left: 0,
    top: 0,
    width: 100,
    height: 100
  };
}

// VS

function FunctionComponent {
  const [left,setLeft] = useState(0)
  const [top,setTop] = useState(0)
  const [width,setWidth] = useState(100)
  const [height,setHeight] = useState(100)
}
```

实际上在 Function Component 中也可以聚合管理 State：

```jsx
function FunctionComponent() {
  const [state, setState] = useState({
    left: 0,
    top: 0,
    width: 100,
    height: 100
  });
}
```

只是更新的时候，不再会自动 merge，而需要使用 `...state` 语法：

```jsx
setState(state => ({ ...state, left: e.pageX, top: e.pageY }));
```

可以看到，更少的黑魔法，更可预期的结果。

### 获取上一个 props

虽然不怎么常用，但是毕竟 Class Component 可以通过 `componentWillReceiveProps` 拿到 `previousProps` 与 `nextProps`，对于 Function Component，最好通过自定义 Hooks 方式拿到上一个状态：

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  const prevCount = usePrevious(count);
  return (
    <h1>
      Now: {count}, before: {prevCount}
    </h1>
  );
}

function usePrevious(value) {
  const ref = useRef();
  useEffect(() => {
    ref.current = value;
  });
  return ref.current;
}
```

通过 `useEffect` 在组件渲染完毕后再执行的特性，再利用 `useRef` 的可变特性，让 `usePrevious` 的返回值是 “上一次” Render 时的。

可见，合理运用 `useEffect` `useRef`，可以做许多事情，而且封装成 CustomHook 后使用起来仍然很方便。

> 未来 `usePrevious` 可能成为官方 Hooks 之一。

### 性能注意事项

`useState` 函数的参数虽然是初始值，但由于整个函数都是 Render，因此每次初始化都会被调用，如果初始值计算非常消耗时间，建议使用函数传入，这样只会执行一次：

```jsx
function FunctionComponent(props) {
  const [rows, setRows] = useState(() => createRows(props.count));
}
```

> `useRef` 不支持这种特性，需要[写一些冗余的函判定是否进行过初始化](https://link.zhihu.com/?target=https%3A//reactjs.org/docs/hooks-faq.html%23how-to-create-expensive-objects-lazily)。

掌握了这些，Function Component 使用起来与 Class Component 就几乎没有差别了！

