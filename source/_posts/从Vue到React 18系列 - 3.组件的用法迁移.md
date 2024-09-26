---
title: 从Vue到React 18系列 - 3.组件的用法迁移
toc: true
date: 2023-4-12 17:36:46
tags: [React.js, TypeScript, CookBook]
categories:
  - [前端技术, 从Vue到React 18系列]
---
以Vue2为主要参考对比，辅以部分Vue3概念，向React 18迁移的CookBook

## React中的父子组件通信、自定义事件、事件处理

### 父子组件通信

#### 父传子

1、使用props传递属性

2、使用ref

父组件通过`React.createRef()`创建`Ref`，保存在实例属性`myRef`上。父组件中，渲染子组件时，定义一个`Ref`属性，值为刚创建的`myRef`。

父组件调用子组件的`myFunc`函数，传递一个参数，子组件接收到参数，打印出参数。

参数从父组件传递给子组件，完成了父组件向子组件通信。

```javascript
import React, { Component, Fragment } from 'react';

class Son extends Component {
    myFunc(name) {
        console.log(name);
    }
    render() {
        return <></>;
    }
}

// 父组件
export default class Father extends Component {
    constructor(props) {
        super(props);
        // 创建Ref，并保存在实例属性myRef上
        this.myRef = React.createRef();
    }

    componentDidMount() {
        // 调用子组件的函数，传递一个参数
        this.myRef.current.myFunc('Jack');
    }
    render() {
        return (
            <>
                <Son ref={this.myRef} />
            </>
        );
    }
}
```

#### 子传父

1、使用回调函数，参见下文自定义事件

2、事件冒泡

点击子组件的`button`按钮，事件会冒泡到父组件身上，触发父组件的`onClick`函数，打印出`Jack`。点击的是子组件，父组件执行函数，完成了子组件向父组件通信。

```javascript
const Son = () => {
    return <button>点击</button>;
};

const Father = () => {
    const sayName = name => {
        console.log(name);
    };
    return (
        <div onClick={() => sayName('Jack')}>
            <Son />
        </div>
    );
};

export default Father;
```

### 自定义事件

在Vue中，我们通常会声明一组v-on与emit来进行自定义事件的传递。而在React中则可以通过props将事件处理器本身传递进子组件中用以调用。

Vue示例：

```html
<!-- 父组件 -->
<template>
	<Coat @rape="rape"></Coat>
</template>
<script>
	export default {
    methods: {
      rape(e){
        console.log(e)
      }
    }
  }
</script>

<!-- 子组件 -->
<template>
	<button @click="$emit('rape','哼哼哼啊啊啊啊啊')">子组件雷普父组件</button>
</template>
```

React示例

```tsx
const Test: FC<{ rape: Function }> = (props) => {
  return (
    <button onClick={() => { props.rape('哼哼哼啊啊啊啊啊') }}>子组件雷普父组件</button>
  )
}

const App: FC = () => {
  const onRape = (text: string) => {
    console.log(text)
  }
  return (
    <div className='App'>
      <Test rape={onRape}></Test>
    </div>
  )
}
```

### 事件处理

详解参见[React的事件处理](https://blog.csdn.net/Han_Zhou_Z/article/details/123338807)

## React中的祖孙组件通信：Provider-Consumer

当组件嵌套层级过多时，使用props一层层地传递事件不是一个好主意。除了使用第三方状态库以外，还可以使用Provider-Consumer设计模式来解决这个问题。在Vue中，类似的模式是Provide-Inject。

相关阅读：[React组件设计模式-Provider-Consumer](https://juejin.cn/post/6844903861161820167)

### 类组件示例

```tsx
import React, { Component, createContext } from 'react'
import './index.css'

const COLOR = ['#B5E61D', '#ED1C24', '#00A2E8', '#A349A4', '#B97A57', '#A349A4']
const Theme = createContext(COLOR[1])

export default class grandfather extends Component {
  state = {
    theme: COLOR[0]
  }
  changeColor = () => {
    this.setState({
      theme: COLOR[Math.ceil(Math.random() * (COLOR.length - 1))]  // 随机获取颜色
    })
  }
  render() {
    return (
        <div className="grandfather">
          <div>当前主题为：{this.state.theme}</div>
          <div style={{ color: this.state.theme }}>grandfather</div>
          <Theme.Provider value={this.state.theme}>
            <Father></Father>
          </Theme.Provider >
          <button className="fixed2" onClick={this.changeColor}>换肤</button>
        </div>
      
    )
  }
}

function Father(props) {
  return (
    <div className="father">
      <Theme.Consumer>
        {
          (theme) => {
            return <div style={{ color: theme }}>father</div>
          }
        }
      </Theme.Consumer>
      <Son></Son>
    </div>
  )
}

function Son(props) {
  return (
    <div className="son">
      <Theme.Consumer>
        {
          (theme) => {
            return <div style={{ color: theme }}>son</div>
          }
        }
      </Theme.Consumer>
    </div>
  )
}
```

### 函数式组件示例（使用hooks）

```tsx
import React, { useState ,useContext, createContext } from 'react'
import './index.css'

const COLOR = ['#B5E61D', '#ED1C24', '#00A2E8', '#A349A4', '#B97A57', '#A349A4']
const Theme = createContext('#B5E61D')

export default function Grandfather() {
  const [ theme, setTheme ] = useState(COLOR[0])
  function changeColor() {
    setTheme(COLOR[Math.ceil(Math.random() * (COLOR.length - 1))])
  }
  return (
    <div className="grandfather">
      <div>当前主题为：{theme}</div>
      <div style={{ color: theme }}>grandfather</div>
      <Theme.Provider value={theme}>
        <Father></Father>
      </Theme.Provider>
      <button className="fixed3" onClick={changeColor}>换肤</button>
    </div>
  )
}
function Father(props) {
  return (
    <div className="father">
    // 在函数组件中一样可以使用Context.Consumer语法来拿数据
      <Theme.Consumer>
        {
          (theme) => <div style={{ color: theme }}>father</div>
        }
      </Theme.Consumer>
      <Son></Son>
    </div>
  )
}

function Son(props) {
  const theme = useContext(Theme) // 注意此处的Theme是开头createContext('#B5E61D')的返回值Theme
  return (
    <div className="son">
      <div style={{ color: theme }}>son</div>
    </div>
  )
}
```

### 实战：在文章分类页面中自定义筛选器

预期效果：Categories.tsx页面传入文章分类，点击筛选器的具体文章类别时可以动态修改Categories.tsx中的参数

文件结构：

```text
- src
  - components
    - CategoryList
      - CategoryList.tsx
      - CategoryListItem.tsx
      - Context.tsx
	- views
		- Categories.tsx
```

views/Categories.tsx

```tsx
import {FC, useState} from "react";
import FluidWrapper from "../Framework/FluidWrapper";
import CategoryList from "@/components/CategoryList/CategoryList";
import CategoryContext from "@/components/CategoryList/Context";

const listOpts = [{
  id: 1,
  name: "iPhone",
  children: [{id: 10, name: "iPhone 13 mini"}, {id: 11, name: "iPhone 13"}, {id: 12, name: "iPhone 13 Pro"}, {
    id: 13,
    name: "iPhone 13 Pro Max"
  }]
}, {
  id: 2,
  name: "MacBook Air",
  children: [{id: 20, name: "MacBook Air M1"}, {id: 21, name: "MacBook Air M2"},]
}]


const Categories: FC = () => {
  const [choseId, setCategory] = useState(Number());
  let setChoseId = (id: number) => {
    setCategory(id)
  }
  return (
    <FluidWrapper>
      <CategoryContext.Provider value={{choseId, setChoseId}}>
        <CategoryList options={listOpts}/>
      </CategoryContext.Provider>
      <span>已选择的类别ID：{choseId}</span>
    </FluidWrapper>
  )
}

export default Categories;
```

components/CategoryList/Context.tsx

```tsx
import React from "react";

export default React.createContext({
  choseId: 0,
  setChoseId: (id: number): void => {
  }
})
```

components/CategoryList/CategoryList.tsx

```tsx
import {FC} from "react";
import CategoryListItem from "@/components/CategoryList/CategoryListItem";

interface CategoryListType {
  options: Array<CategoryListItemType>
}

export interface CategoryListItemType {
  id: number,
  name: number | string,
  children?: Array<CategoryListItemType>
}

const CategoryList: FC<CategoryListType> = ({options}) => {
  return (
    <ul>
      {options.map(row => {
        return <li key={row.id}>
          <h3>{row.name}</h3>
          {row.children ? <CategoryListItem children={row.children}></CategoryListItem> : ""}
        </li>
      })}
    </ul>
  )
}

export default CategoryList;
```

components/CategoryList/CategoryListItem.tsx

```tsx
import {FC, useContext} from "react";
import CategoryContext from "@/components/CategoryList/Context";
import {CategoryListItemType} from "@/components/CategoryList/CategoryList";
import {animated} from "react-spring";

const CategoryListItem: FC<{ children: Array<CategoryListItemType> }> = ({children}) => {
  const context = useContext(CategoryContext)
  return (
    <animated.ul>
      {children.map(item => (
        <li key={item.id} onClick={() => {
          context.setChoseId(item.id)
        }}>{item.name}</li>
      ))}
    </animated.ul>
  )
}
export default CategoryListItem;
```

## 在React组件中实现插槽

向组件的props里传一个DOM进去就可以了，这里是JSX！

在Vue中，我们可以像这样通过插槽来实现在自定义组件ComponentA中插入组件ComponentB：

```html
// ComponentA.vue
<template>
	<div>
    <span>This is Component Yajuu</span>
    <slot></slot>
  </div>
</template>

// Test.vue
<template>
	<div>
    <ComponentA>
      <ComponentB></ComponentB>
  	</ComponentA>
  </div>
</template>
```

而在React中有一个props.children的概念，被父子件所包裹的子组件可以使用这个属性来获取。

扩展阅读：[React组件设计模式-组合组件](https://juejin.cn/post/6844903861166014477)

```tsx
import React, { FC } from "react";

const ChildComponent: FC = () => (
  <div>
    <span>This is ChildComponent</span>
  </div>
)

// 在React18中，FC不再内置PropsWithChildren的定义，需要手动定义一下，否则TS会报错
// 坑死了哼哼哼啊啊啊啊
const ParentComponent: FC<{ children?: React.ReactNode }> = (props) => (
  <div>
    <span>This is ParentComponent</span>
    {props.children}
  </div>
)

const Test: FC = () => {
  return (
    <>
      <ParentComponent>
        <ChildComponent></ChildComponent>
        <ChildComponent></ChildComponent>
      </ParentComponent>
    </>
  )
}
export default Test;
```

### 
