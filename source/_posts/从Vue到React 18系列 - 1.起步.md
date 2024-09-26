---
title: 从Vue到React 18系列 - 1.起步
toc: true
date: 2023-4-12 17:36:44
tags: [React.js, TypeScript, CookBook]
categories:
  - [前端技术, 从Vue到React 18系列]
---
以Vue2为主要参考对比，辅以部分Vue3概念，向React 18迁移的CookBook

## 创建Vite-React工程

用点新东西吧。

```shell
yarn create vite react-study --template react-ts && cd ./react-study && yarn install
yarn add @types/node --dev
```

配置vite.config.ts

```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import path from 'path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  server: {
    host: '0.0.0.0',
    open: false, 
  }
})
```

在tsconfig.json的complierOptions下增加节点

```json
"baseUrl": ".",
"paths": {
	"@/*": [
		"src/*"
	]
}
```

## npm和yarn的区别

[命令对照](https://www.jianshu.com/p/56c6b171134c)

## 起步：简单的Hello World

App.tsx

```tsx
import React from 'react'

function App() {
  const clickEvent = (e: React.MouseEvent<HTMLElement>) => {
    console.log(e)
  }
  const helloWorld = `Hello world! Now timestamp is: ${new Date().getTime()}`
  const btnDom = <button onClick={clickEvent}>This is a JSX rendered button</button>
  const helloWorldTitle = <h1>{helloWorld}</h1>
  return (
    <div className="App">
      {helloWorldTitle}
      <p>{helloWorld}</p>
      {btnDom}
    </div>
  )
}

export default App
```

## Mustache模板与JSX

不同于Vue中常见的模板语法，React使用JSX语法来描述DOM元素。

JSX在react中实质上只是React.createElement的语法糖。与Vue template相同的是，render函数返回的JSX有且只能有一个根节点。

**想直接返回多个子元素？试试[<React.Fragment>](https://react.docschina.org/docs/fragments.html)**

在Vue模板中，我们使用形如{{ 表达式 }}的Mustache语法以在模板中取得Vue实例中的响应式变量（或计算属性）；而在React JSX中则可以使用形如{ 表达式 }的语法来声明一个变量或vDOM。

与Vue不同的是，我们可以直接以变量的形式声明一个DOM元素，然后在组件实例的Render函数中以{}来引用。比如上述的App.tsx实例代码，实际上会输出一行以h1标签包裹的字符串、一行以p标签包裹的相同字符串以及一个绑定了clickEvent事件的按钮。

在使用Vue的模板语法时，我们通常利用v-bind指令（或它的语法糖冒号:）用来给DOM元素（或组件）传递一个动态值而非字符串字面量。如：

```html
<template>
  <button :id="koji" @click="shouting" type="button" :style="{marginLeft: '50px'}" :class="'btn-'+new Date().getTime()">
    {{ name }}
  </button>
</template>
<script>
  export default {
    data(){
      return {
        btnId: 'koji',
        name: 'Yajuu senpai'
      }
    },
    methods: {
      shouting(){
        alert('114514')
      }
    }
  }
</script>
```

而在React中应该使用{}代替""书写：

```jsx
function App() {
  const shouting = () => {
    alert('114514')
  }
  const name = 'Yajuu senpai'
  const btnId = 'koji'
  const btnDom = <button id={btnId} onClick={shouting} type="button" style={{marginLeft: '50px'}} className={`btn-${new Date().getTime()}`}>{name}</button>
  return (
    <div className="App">
      {btnDom}
    </div>
  )
}

export default App
```

上述对比示例中，按钮拥有几个属性

* type="button" 是一个字面量属性。
* id="koji" 绑定了btnId的值。
* 一个onClick事件，点击后执行函数shouting。在Vue中我们通常使用v-on:click的语法糖@click，而React中则会将HTML DOM Event转为小驼峰方式声明，即onclick变为onClick。
* style绑定了一个JavaScript对象（CSS Property）
* 在JSX中因为保留字的关系，class需要变为className。

**React函数式组件的return()和类组件中的render()都不能直接修改组件的状态（states及props）。**

**在React中父组件更新子组件的props，在子组件接收到新的props时会通过[componentWillReceiveProps()](https://reactjs.org/docs/react-component.html#componentwillreceiveprops)生命周期来执行this.setState()来更新视图，但不会引起第二次渲染。**

### onClick等事件的小坑

如果有这样一个组件：

```jsx
const Test: FC = () => {
  function shout(text: string) {
    console.log(text)
  }
  const showNow = () => {
    console.log("一针见血的")
  }
  return (
    <div>
      <button onClick={shout("错误用法")}>错误的</button>
      <button onClick={() => { shout("hello") }}>正确的</button>
      <button onClick={showNow}>一针见血的</button>
    </div>
  )
}
```

这个组件将会在渲染的时候首先打印"错误用法"，而后才会在点击按钮的时候打印"hello"，并且第一个按钮点击无反应。原因是组件渲染时直接将shout()当做一个函数执行了，而它返回了undefined，点击时实际上是onClick=undefined。

如果为shout增加返回值，比如return "哼哼哼"，那么"错误的"按钮将会导致页面报错。因为此时onClick="哼哼哼"，不是一个函数。

为什么"一针见血的"按钮就可以正常使用呢？因为它实际上就是onClick=()=>{}，onClick执行函数当然没问题。同理，如果写onClick={showNow()}就不行了。

当然，也可以进行柯里化：

```tsx
const shout = (text: string) => () => console.log(text);
return (
	<button onClick={shout("hello")}>柯里化的</button>
)
```
