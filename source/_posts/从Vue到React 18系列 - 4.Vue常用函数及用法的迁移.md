---
title: 从Vue到React 18系列 - 4.Vue常用函数及用法的迁移
toc: true
date: 2023-4-12 17:36:47
tags: [React.js, TypeScript, CookBook]
categories:
  - [前端技术, 从Vue到React 18系列]
---
以Vue2为主要参考对比，辅以部分Vue3概念，向React 18迁移的CookBook

## 在React中实现Computed、Watch

### 实现Watch

类组件中通过componentDidUpdate访问变化前的props和states

```tsx
class Test extends React.Component<any, any> {
  constructor(props: any) {
    super(props)
    this.state = {
      count: 0
    }
  }
  componentDidUpdate(prevProps: any, prevState: any) {
    console.log(prevState.count)
    console.log(this.state.count)
  }
  clickCount() {
    this.setState((state: any) => ({
      count: state.count + 1
    }));
  }
  render() {
    return (
      <div>
        <h1>{this.state.count}</h1>
        <button onClick={this.clickCount.bind(this)}>点我</button>
      </div>
    )
  }
}
```

函数式组件中写个Hook就好。

useWatch.js

```javascript
import { useEffect, useRef } from 'react'

const useWatch = (value, fn, config = { immediate: false }) => {
  const oldValue = useRef()
  const isFirst = useRef(false)
  useEffect(() => {
    if (isFirst.current) {
      fn(value, oldValue.current)
    } else {
      isFirst.current = true
      if (config.immediate) {
        fn(value, oldValue.current)
      }
    }
    oldValue.current = value
  }, [value])
}

export default useWatch
```

组件中使用：

```jsx
import { FC } from 'react'
import { useState } from 'react'
import useWatch from "./useWatch";
const Test: FC = () => {
  const [count, setCount] = useState(0)
  const clickCount = () => {
    setCount(count + 1);
  }
  useWatch(count, (newVal, oldVal) => {
    console.log(newVal, oldVal)
  })
  return (
    <div>
      <h1>{count}</h1>
      <button onClick={clickCount}>点我</button>
    </div>
  )
}
```

### 实现Computed

类组件还是在componentDidUpdate里操作，都是一回事。当然，使用get收集依赖也行：

```jsx
class App extends React.Component {
    constructor(props) {
        super(props)
        this.state = {
            msg: '田所浩二'
        }
    }
    get koji_computed() {
        return this.state.msg
    }
    componentDidMount() {
        setTimeout(() => {
            this.setState({
                msg: '使用软手机'
            })
        }, 2000)
    }
    render() {
        return (
            <div>
                { this.koji_computed }
            </div>
        )
    }
    
}
```

函数式组件用**useMemo**这个hook：

```jsx
import { useState, useMemo } from 'react'
function Demo() {
  const [name, setName] = useState('田所浩二')
  const [food, setFood] = useState('红茶')
  const msg = useMemo(() => `${name}享用${food}`, [name, food])
  const handleClick = (type: number) => {
    if (type === 1) {
      setName('淳平')
      setFood('迎宾酒')
    } else if (type === 2) {
      setName('我修院')
      setFood('雪')
    }
  }

  return (
    <div>
      <button onClick={() => handleClick(1)}>淳平Type</button>
      <button onClick={() => handleClick(2)}>我修院Type</button>
      <h1>{msg}</h1>
    </div>
  )
}
```



## 从v-if、v-show、v-model迁移

在Vue中，v-if的本质是控制组件/DOM的渲染，而v-show只是控制它的display属性是否为none。所以在React中想要实现类似逻辑，只需要在render时使用三目运算符决定是否返回该DOM即可。

如果想通过props传递控制组件来实现类似于v-if/v-show的效果，则需要在render时判断props内的相关属性并填充相关逻辑。

```jsx
import React, { FC, useState } from 'react'

const Test = (props: { [x: string]: any; }) => {
  if (props['v-if']) {
    let isShow = props['v-show'] ? 'none' : 'block';
    return (
      <ul style={{ display: isShow }} >
        <li>Yajuu Senpai</li>
        <li>24</li>
      </ul>
    );
  } else {
    return (<></>);
  }
}
const App: FC = () => {
  const [vif, setVif] = useState(false)
  const clickIf = () => {
    setVif((state) => (!state))
  }
  const [vshow, setVshow] = useState(false)
  const clickShow = () => {
    setVshow((state) => (!state))
  }
  return (
    <div className='App'>
      <button onClick={clickIf}>v-if:{vif.toString()}</button>
      <button onClick={clickShow}>v-show:{vshow.toString()}</button>
      <Test v-if={vif} v-show={vshow}></Test>
    </div>
  )
}

export default App
```

以React写法来实现v-if的效果更是简单：

```jsx
import { useState } from 'react'
function Demo() {
  const [show, setShow] = useState(false)
  const changeShow = () => {
    setShow(!show)
  }
  return (
    <div>
      {show && <h1>哼哼哼啊啊啊啊啊</h1>}
      <button onClick={changeShow}>点我</button>
    </div>
  )
}
```

v-model本质上是利用名为value的prop和名为input的事件来实现数据的双向绑定。

```jsx
import React, { FC, useState } from 'react'

function Input(props: any) {
  const handleInput = (e: any) => {
    props['v-model'](e.target.value)
  }
  return (
    <>
      <input onChange={handleInput}></input>
    </>
  )
}

const App: FC = () => {
  const [value, setValue] = useState('')
  return (
    <div className='App'>
      <p>输入的值为：{value}</p>
      <Input v-model={setValue}></Input>
    </div>
  )
}

export default App
```

## 从v-for、v-on:event迁移

### v-for

React里直接使用map在JSX中进行循环即可。当然，如果想要用for，也可以尝试在JSX中插入一段立即执行的匿名函数。

Vue示例：

```html
<div>
	<PostItem v-for="(item, i) in postItems" 
            :key="item.id" 
            :post="item" 
            @onVote="handleVote"></PostItem>
</div>
```

React示例：

```jsx
render(){
  return (
  	<div>
    	{this.state.postItems.map((item,index)=>
      	return (
					<PostItem key={item.id} post={item} onVote={this.handleVote}></PostItem>
      	)
      )}
    </div>
  )
}
```

使用立即执行函数：

```jsx
const data = [{
    foo: 111,
    bar: 222
  },{
    foo: 333,
    bar: 444
  }]
  return (
    <div className="App">
      {
         (() => {
          let domArr = [];
          for(const item of data) {
              domArr.push(<li>{item.foo},{item.bar}</li>)
          }
          return domArr;
        })()
      }
    </div>
  );
```

### v-on

参见上节中的: React中的父子组件通信、自定义事件、事件处理

## React中的CSS方案

### 内联样式

优点：官方推荐，直接使用CSS Property Class描述样式；组件私有的样式不会导致冲突；可以动态获取当前state中的状态

缺点：写法上都需要使用驼峰标识；会导致代码混乱；某些样式无法编写(比如伪类/伪元素)

```tsx
import { FC } from "react";

export const Header: FC = () => {
  return (
    <div style={{color: 'red'}}>114514</div>
  )
}
```

### 使用第三方库实现style scoped

使用起来最接近于Vue中`<style scoped>`的实现：[react-scoped-css](https://github.com/gaoxiaoliangz/react-scoped-css)

### 导入外部CSS文件

优点：写法和普通HTML开发相似，易于理解。

缺点：导入的CSS是全局样式，可能导致冲突。

```tsx
import { FC } from "react";
import './assets/css/Header.css'  // .title { color: red; }
export const Header: FC = () => {
  return (
    <div className="title">114514</div>
  )
}
```

### CSS Modules

优点：实现原理类似于`<style scoped>`，属于组件私有样式；

缺点：

* 引用的类名不能使用连接符(如 .header-title )，因为JS中不识别。尽管loader可以转换驼峰和连接符，但这将会出现如下情况：

  ```jsx
  .header-title {
    color: #999;
  }
  <div className="style['header-title']">第一种写法</div>
  <div className="style.headerTitle">第二种写法</div>
  ```

  这将会产生一些理解成本，而且可能因粗心而导致一些意料之外的问题。

* 动态修改样式时依然需要依赖于内联样式。

以xxx.module.css（或.scss / .less等等）命名的样式文件在被组件导入后会被加载为一个CSS Module。

示例：

```tsx
import { FC } from "react";
import HeaderStyle from '@/assets/css/Header.module.scss';
export const Header: FC = () => {
  return (
    <>
      <div className={HeaderStyle.blogHeaderTitle}>第一种写法</div>
      <div className={HeaderStyle['blog-header-title']}>第二种写法</div>
    </>
  )
}
```

#### Vite配置CSS Modules

详细说明参见[Vite CSS](https://cn.vitejs.dev/guide/features.html#css)

首先安装一个喜欢的CSS预处理器

```shell
# .scss and .sass
yarn add -D sass
# .less
yarn add -D less
# .styl and .stylus
yarn add -D stylus
```

然后配置vite.config.ts，使其支持驼峰转换：

```js
css:{
    modules:{
      localsConvention: 'camelCase'
    }
}
```

#### VSCode配置CSS Modules提示

```shell
# 安装依赖
yarn add -D typescript-plugin-css-modules
```

配置tsconfig.json

```json
{
  "compilerOptions": {
    ...
    "plugins": [{"name": "typescript-plugin-css-modules"}]
  }
  ...
}
```

配置VSCode settings.json

```json
{
	...
	"typescript.tsserver.pluginPaths": ["typescript-plugin-css-modules"],
	...
}

```

### CSS in JS

该节部分via：[React中的CSS](https://blog.csdn.net/qq_27009517/article/details/122743311)

实际上，官方文档也有提到过CSS in JS这种方案：

- “CSS-in-JS” 是指一种模式，其中 CSS 由 JavaScript 生成而不是在外部文件中定义；
- 注意此功能并不是 React 的一部分，而是由第三方库提供。 React 对样式如何定义并没有明确态度；

在传统的前端开发中，我们通常会将结构（HTML）、样式（CSS）、逻辑（JavaScript）进行分离。

- 但是在前面的学习中，我们就提到过，React的思想中认为逻辑本身和UI是无法分离的，所以才会有了JSX的语法。
- 样式呢？样式也是属于UI的一部分；
- 事实上CSS-in-JS的模式就是一种将样式（CSS）也写入到JavaScript中的方式，并且可以方便的使用JavaScript的状态；
- 所以React有被人称之为 All in JS；

当然，这种开发的方式也受到了很多的批评：

- Stop using CSS in JavaScript for web development
- [https://hackernoon.com/stop-using-css-in-javascript-for-web-development-fa32fb873dcc](https://link.zhihu.com/?target=https%3A//hackernoon.com/stop-using-css-in-javascript-for-web-development-fa32fb873dcc)

批评声音虽然有，但是在我们看来很多优秀的CSS-in-JS的库依然非常强大、方便：

- CSS-in-JS通过JavaScript来为CSS赋予一些能力，包括类似于CSS预处理器一样的样式嵌套、函数定义、逻辑复用、动态修改状态等等；
- 依然CSS预处理器也具备某些能力，但是获取动态状态依然是一个不好处理的点；
- 所以，目前可以说CSS-in-JS是React编写CSS最为受欢迎的一种解决方案；

目前比较流行的CSS-in-JS的库有哪些呢？

- styled-components
- emotion
- glamorous

styled-components目前是社区最流行的CSS-in-JS库，但这里推荐使用emotion。它拥有相当全面的功能，体积小巧，而且还是为数不多的支持source-map的css-in-js框架之一。[Emotion文档](https://emotion.sh/docs/install)

#### Vite配置Emotion && Typescript支持

```shell
yarn add @emotion/react
yarn add -D @emotion/babel-plugin
yarn add -D @emotion/babel-preset-css-prop
```

配置vite.config.ts

```typescript
export default defineConfig({
  plugins: [
    react({
      jsxImportSource: "@emotion/react",
      babel: {
        plugins: ["@emotion/babel-plugin"],
        babelrc: true
      },
    }),
  ],
});
```

配置tsconfig.json

```json
{
  "compilerOptions": {
    "jsxImportSource": "@emotion/react"
  }
}
```

新建.babelrc

```json
{
  "presets": [
    [
      "@emotion/babel-preset-css-prop",
      {
        "autoLabel": "dev-only",
        "labelFormat": "[local]"
      }
    ]
  ]
}
```

示例：

```tsx
import { FC, Fragment } from "react";
import { css } from '@emotion/react'

const blogHeaderTitle = css`
  color: red;
  font-size: 114px;
`;
export const Header: FC = () => {
  return (
    <div>
      <div css={blogHeaderTitle}>123123123</div>
    </div>
  )
}
```

