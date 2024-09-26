---
title: 从Vue到React 18系列 - 5.Vue全家桶的迁移
toc: true
date: 2023-4-12 17:36:48
tags: [React.js, TypeScript, CookBook]
categories:
  - [前端技术, 从Vue到React 18系列]
---
以Vue2为主要参考对比，辅以部分Vue3概念，向React 18迁移的CookBook

## 路由管理：从Vue-Router到React-Router

在本文编写时，React-Router实为React-Router-Dom V6，Vue-Router为Vue-Router V4（为了易于接受，文中大部分例子使用V3举例说明，V4的例子会特别标注出）。下文简称“React中”及“Vue中”。

部分参考自[React-Router-Dom6 最佳实践](https://zhuanlan.zhihu.com/p/488571812)、[React-router 路由的使用及配置](https://juejin.cn/post/6844904031857410062#heading-1)、[React-Router官方文档（写的不容易懂）](https://reactrouter.com/en/main/getting-started/overview)、[React-Router官方文档Examples](https://v5.reactrouter.com/web/example/basic)、[vue-router和react-router使用的异同点](https://juejin.cn/post/6844904164531634190#heading-10)

首先安装依赖

```shell
yarn add react-router-dom
```

### Hash模式与History模式

无论是React中还是Vue中，路由的实现模式都基于这两种模式。

* Hash模式

  它的实现原理主要是基于window.onhashchange事件：

  ```js
  window.onhashchange = () => {
      let hash = location.hash
      console.log(hash)
  }
  ```

  hash 模式下，发起的请求也不会被 hash 值影响（http请求中），不会重新加载页面。

* History模式

  它基于 window.onpopstate 事件：

  ```js
  window.onpopstate = function(event) {
    alert("location: " + document.location + ", state: " + JSON.stringify(event.state))
  }
  ```

  通过浏览器提供的 history api，url 更加好看了，但刷新时如果服务器中没有相应的资源就可能会报 404，这是因为刷新后又去请求了服务器。

Vue若要切换路由模式需要在router.js（即路由声明文件）中修改，而React则是以不同组件包裹（类似于router-view，不过React版本的router-view自带了路由模式）来进行区分的。

```jsx
// Vue Router v3 ( Vue 2.x使用的路由 )
const router = new VueRouter({
  mode: 'history',  //或'hash'
  routes: [...]
})
// Vue Router v4 ( Vue 3.x使用的路由 )
import { createRouter, createWebHistory } from 'vue-router'

const router = createRouter({
  history: createWebHistory(),  // Hash模式为createWebHashHistory
  routes: [],
})
// React Router Dom - main.tsx
// Hash模式为<HashRouter></HashRouter>
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App'
import { BrowserRouter } from 'react-router-dom'

ReactDOM.createRoot(document.getElementById('root') as HTMLElement).render(
  <React.StrictMode>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </React.StrictMode>
)
```

### React Router Dom V6常用路由组件

| 组件名     | 作用           | 说明                                                        |
| ---------- | -------------- | ----------------------------------------------------------- |
| <Routes>   | 一组路由       | 代替原来的<Switch>，所有子路由都用基础的Router Children表示 |
| <Route>    | 基础路由       | V6中可以嵌套                                                |
| <Link>     | 导航组件       | 在页面中跳转使用                                            |
| <Outlet /> | 自适应渲染组件 | 根据实际路由URL自动选择组件，一般用来实现嵌套路由           |

### &lt;router-link&gt;与&lt;Router&gt;组件

在Vue中我们可以使用router-link组件进行声明式路由跳转，如：

```html
<script src="https://unpkg.com/vue/dist/vue.js"></script>
<script src="https://unpkg.com/vue-router/dist/vue-router.js"></script>

<div id="app">
  <h1>Hello App!</h1>
  <p>
    <!-- 使用 router-link 组件来导航. -->
    <!-- 通过传入 `to` 属性指定链接. -->
    <!-- <router-link> 默认会被渲染成一个 `<a>` 标签 -->
    <router-link to="/foo">Go to Foo</router-link>
    <router-link to="/bar">Go to Bar</router-link>
  </p>
  <!-- 路由出口 -->
  <!-- 路由匹配到的组件将渲染在这里 -->
  <router-view></router-view>
</div>
```

而在React中则可以使用Link标签达到类似效果：

```jsx
import * as React from "react";
import { Link } from "react-router-dom";

function UsersIndexPage({ users }) {
  return (
    <div>
      <h1>Users</h1>
      <ul>
        {users.map((user) => (
          <li key={user.id}>
            <Link to={user.id}>{user.name}</Link>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

Vue中router-link默认是a标签。若想替换成span标签，可使用`<router-link tag='span'>`。点击后默认会给该标签添加.router-link-active类，若要自定义链接激活时使用的css类名，可使用`<router-link active-class="激活时的类名">`

React中Link没有激活属性。若要使用激活属性可用NavLink替代：

```jsx
import {NavLink} from "react-router-dom"
function App(props){
  return (
    <NavLink to='path' activeClassName='onActiveClz'></NavLink>
    <NavLink to='path' activeStyle={{color:'red'}}></NavLink>
  )
}
// 在React Router DOM v6中有变化：
<NavLink to={nav.route} style={({isActive}) =>
        isActive ? {color: "white"} : {color: 'rgba(245,245,247,0.8)'}
      }>{nav.name}</NavLink>
// 或：
<NavLink to={node.route} className={({isActive}) =>
              isActive ? HeaderStyle["blog-header-cell-active"] : HeaderStyle["blog-header-cell"]
            }>
              {node.name}
            </NavLink>
```

### 视图组件

Vue中router-view会渲染出路径所匹配到的视图组件：

```html
//App.vue
<template>
  <div>
    App组件
    <router-view></router-view>
  </div>
</template>
// user.vue
<template>
  <div>
    user组件
    <router-view></router-view>
  </div>
</template>
// name.vue
<template>
  <div>
    name组件
  </div>
</template>
// routes.js
<script>
const router = new VueRouter({
    routes:[
      {
        path:'/user',
        component:user,
        children:[
          path:'/user/name',
          component:name
        ]
      }
    ]
})
</script>
```

* 路由为/user时，user组件渲染到App.vue中的router-view的位置
* 路由为/user/name时，user组件渲染到App.vue中的router-view的位置，name组件渲染到user.vue中的router-view的位置

React中Route组件负责匹配路由并渲染。`<Route>`在哪，组件就渲染在哪。

Router组件有几个参数用法：

* path: 指定路由的跳转路径

* exact: 精确匹配路由

* component: 路由对应的组件

  ```jsx
  import About from './pages/about';
  <Route path='/about' exact component={About}></Route>
  ```

* render： 通过写render函数返回具体的dom

  ```jsx
  <Route path='/about' exact render={() => (<div>about</div>)}></Route>
  ```

  render 也可以直接返回 About 组件

  ```jsx
  <Route path='/about' exact render={() => <About /> }></Route>
  ```

  使用render的好处是可以向组件传递自定义属性

  ```jsx
  <Route path='/about' exact render={(props) => {
      return <About {...props} name={'田所浩二'} />
  }}></Route>
  ```

  然后，就可在 About 组件中获取 props 和 name 属性：

  ```jsx
  componentDidMount() {
      console.log(this.props) 
  }
  
  // history: {length: 9, action: "POP", location: {…}, createHref: ƒ, push: ƒ, …}
  // location: {pathname: "/home", search: "", hash: "", state: undefined, key: "ad7bco"}
  // match: {path: "/home", url: "/home", isExact: true, params: {…}}
  // name: "cedric"
  ```

  render 方法也可用来进行权限认证：

  ```jsx
  <Route path='/user' exact render={(props) => {
      // isLogin 从 redux 中拿到, 判断用户是否登录
      return isLogin ? <User {...props} name={'田所浩二'} /> : <div>请先登录</div>
  }}></Route>
  ```

* location: 将与当前历史记录位置以外的位置相匹配，在路由过渡动效中很有用。

* sensitive: 区分路由大小写

注意：如果路由 Route 外部包裹 Routes 时，路由匹配到对应的组件后，就不会继续渲染其他组件了。但是如果外部不包裹 Switch 时，所有路由组件会先渲染一遍，然后选择到匹配的路由进行显示。

### 路由传参

该部分Hooks使用方法参考[【React Router v6】路由组件传参](https://blog.csdn.net/Svik_zy/article/details/126203649)

#### 路径参数——动态路由匹配

例子中的页面路径为`http://localhost:5173/Detail/114/Yajuu`

```tsx
// router/index.tsx
{
  path: "/Detail/:id/:name",
  element: <Detail />
}

// Header.tsx
import { FC } from "react";
import { useNavigate } from "react-router-dom";
export const Header: FC = () => {
  const navigate = useNavigate();
  return (
    <div>
      <div onClick={() => { navigate('/Detail/114/Yajuu') }}>详情</div>
    </div>
  )
}

// Detail.tsx
import { FC } from "react";
import { useParams } from "react-router-dom";

const Detail: FC = () => {
  const {id, name} = useParams();
  return (
    <>
      <div>id:{id}</div>
      <div>name:{name}</div>
    </>
  )
}
export default Detail;
```

#### URL Search Params

例子中的页面路径为`http://localhost:5173/Detail?age=24`

```tsx
// router/index.tsx
{
  path: "/Detail",
  element: <Detail />
}

// Header.tsx
import { FC } from "react";
import { useNavigate } from "react-router-dom";
export const Header: FC = () => {
  const navigate = useNavigate();
  return (
    <div>
      <div onClick={() => { navigate('/Detail?age=24') }}>详情</div>
    </div>
  )
}

// Detail.tsx
import { FC } from "react";
import { useSearchParams } from "react-router-dom";

const Detail: FC = () => {
  let [searchParams, setSearchParams] = useSearchParams();
  return (
    <>
      <div>age:{searchParams.get('age')}</div>
      <button onClick={() => { setSearchParams({age: "114514"})}}>便乘王爷</button>
    </>
  )
}
export default Detail;
```

#### Component State Params

参数不显示在URL中，类似Vue中的{ name, params }跳转方法

```tsx
// router/index.tsx
{
  path: "/Detail",
  element: <Detail />
}

// Header.tsx
import { FC } from "react";
import { useNavigate } from "react-router-dom";
export const Header: FC = () => {
  const navigate = useNavigate();
  return (
    <div>
      <div onClick={() => {
        navigate('/Detail', {
          state: {
            age: 24,
            name: '李田所',
            company: 'COAT'
          }
        })
      }}>详情</div>
    </div>
  )
}

// Detail.tsx
import { FC } from "react";
import { useLocation } from "react-router-dom";

const Detail: FC = () => {
  const { state: { age, name, company } }: any = useLocation();
  return (
    <>
      <ul>
        <li>年龄：{age}</li>
        <li>姓名：{name}</li>
        <li>公司：{company}</li>
      </ul >
    </>
  )
}
export default Detail;
```

## React-Router-Dom V6 搭建博客框架实战

#### 文件结构

```javascript
- src
	- App.tsx
	- main.tsx
  - Framework
	  - Header.tsx
		- MainBody.tsx
	- router
		- index.tsx
	- views
		- Index.tsx
		- About.tsx
		- History.tsx
		- Detail.tsx
```

#### 实现目标

首页默认加载`views/Index.tsx`，点击About可以离开该页面，显示单独的About.tsx页面；点击History可以携带路径参数跳转历史列表页面；点击Detail可以携带ID跳转文章详情页面。

### 代码配置

main.tsx

```tsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App'
import { BrowserRouter } from 'react-router-dom'

ReactDOM.createRoot(document.getElementById('root') as HTMLElement).render(
  <React.StrictMode>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </React.StrictMode>
)
```

App.tsx

```tsx
import { useRoutes } from 'react-router-dom';
import './App.css'
import router from './router/index';

function App() {
  let element = useRoutes(router);
  return <div className="App">{element}</div>;
}

export default App;
```

router/index.tsx

```tsx
import { lazy, Suspense } from "react";
import { Framework } from "@/views/Index";
import Categories from "@/views/Categories";
import Tags from "@/views/Tags";
import Detail from "@/views/Detail";
import History from "@/views/History";
/**
 * 懒加载组件
 * @param path 
 * @returns 
 */
function LazyWrapper(path: string) {
  // React.lazy接受的组件必须为export default形式
  const Component = lazy(() => import(`../views${path}`))
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Component />
    </Suspense>
  );
}

let router = [
  {
    path: "/About",
    element: LazyWrapper("/About"),
  },
  {
    path: "/",
    element: <Framework />,
    children: [
      {
        path: "/Detail",
        element: <Detail />
      },
      {
        path: "/History/:page",
        element: <History />
      },
    ],
  },
];

export default router;

```

Header.tsx

```tsx
// 用NavLink组件也可以。它甚至可以根据当前路由变色！
import { FC } from "react";
import { useNavigate } from "react-router-dom";
export const Header: FC = () => {
  const navigate = useNavigate();
  return (
    <div>
      Header
      <div onClick={() => { navigate('/About') }}>关于我</div>
      <div onClick={() => { navigate('/History/893') }}>历史</div>
      <div onClick={() => { navigate('/Detail?id=114514') }}>详情</div>
      <NavLink to='/History/893' className={({isActive}) =>
              isActive ? "blog-header-cell-active" : "blog-header-cell"
      }>
              历史-组件版
      </NavLink>
    </div>
  )
}
```

MainBody.tsx

```tsx
import { FC } from "react"
import { Outlet, useRoutes } from "react-router-dom"

export const MainBody: FC = () => {
  return (
    <div style={{height: '50vh', backgroundColor: 'yellow'}}>
      <Outlet/>
    </div>
  )
}
```

views/About.tsx

```tsx
import { FC } from "react";

const About: FC = ()=>{
  return (
    <div>我是田所浩二</div>
  )
}
export default About;
```

views/History.tsx

```tsx
import { FC } from "react";
import { useParams } from "react-router-dom";

const History: FC = () => {
  const {page} = useParams();
  return (
    <div>
      历史记录页面，当前页码{ page }
    </div>
  )
}
export default History;
```

views/Detail.tsx

```tsx
import { FC } from "react";
import { useSearchParams } from "react-router-dom";

const Detail: FC = () => {
  let [searchParams, setSearchParams] = useSearchParams();
  return (
    <>
      <div>文章ID:{searchParams.get('id')}</div>
      <button onClick={() => { setSearchParams({id: "1919810"})}}>文章有很多啊</button>
    </>
  )
}
export default Detail;
```

## 扩展：为博客管理后台使用路由守卫

router/RouterGuard.tsx

```jsx
import {useRoutes, RouteObject, Navigate, useLocation} from 'react-router-dom';
import React, {Suspense} from "react";

interface FunctionRule {
  (): any
}

//meta规则
type MetaRule = {
  auth?: boolean, //是否需要登录验证
  title?: string, //页面title
  [name: string]: string | boolean | undefined, //其他参数
}

//单个路由规则
type RouteObjectRule = RouteObject & {
  children?: RouteObjectRule[], //子路由
  page?: FunctionRule, //route导入页面的对象
  path?: string, //页面路径
  redirect?: string, //重定向地址 ，常用于设置页面默认地址
  meta?: MetaRule, //页面参数
}

interface onRouteBeforeRule<meta = MetaRule, to = string> {
  (meta: meta, to: to): any | never
}

type LoadingEleRule = React.ReactNode;

interface GuardRule {
  routers: RouteObjectRule[],
  onRouterBefore?: onRouteBeforeRule,
  loading?: LoadingEleRule,
}

let onRouterBefore: onRouteBeforeRule | undefined;
let RouterLoading: FunctionRule;

//路由导航，设置redirect重定向 和 auth权限
function Guard({element, meta}: any) {
  const {pathname} = useLocation();
  const nextPath = onRouterBefore ? onRouterBefore(meta, pathname) : pathname;
  if (nextPath && nextPath !== pathname) {
    element = <Navigate to={nextPath} replace={true}/>;
  }
  return element;
}


// 路由懒加载
function lazyLoadRouters(page: any, meta: {}) {
  meta = meta || {};
  const LazyElement = React.lazy(page);
  // const Component = lazy(() => import(`../views${path}`))
  const GetElement = () => {
    return (
      <Suspense fallback={<RouterLoading/>}>
        <LazyElement/>
      </Suspense>
    );
  };
  return <Guard element={<GetElement/>} meta={meta}/>;
}

function transRoutes(routes: RouteObjectRule[]) {
  const list: any = [];
  routes.forEach(route => {
    const obj = {...route} as any;
    if (obj.redirect) {
      obj.element = <Navigate to={obj.redirect} replace={true}/>
    }
    if (obj.page) {
      obj.element = lazyLoadRouters(obj.page, obj.meta)
    }
    if (obj.children) {
      obj.children = transRoutes(obj.children)
    }
    ['redirect', 'page', 'meta'].forEach(name => delete obj[name]);
    list.push(obj)
  });
  return list
}

export type {
  RouteObjectRule,
  MetaRule,
  FunctionRule,
  onRouteBeforeRule,
  LoadingEleRule,
}

function RouterGuard(params: GuardRule) {
  onRouterBefore = params.onRouterBefore;
  RouterLoading = () => params.loading || <></>;
  return useRoutes(transRoutes(params.routers));
}

export default RouterGuard;
```

router/index.tsx

```jsx
import {Framework} from "@/views/Index";
import {onRouteBeforeRule, RouteObjectRule} from "@/router/RouterGuard";
import {AppState} from "@/store";
const routes: RouteObjectRule[] = [
  {
    //TODO 404
    path: '*',
    redirect: '/',
  },
  {
    path: "/",
    element: <Framework/>,
    children: [
      {
        path: "",
        // element: <Home></Home>,
        page: () => import('../views/Home'),
        meta: {
          title: 'Blog管理'
        }
      },
      {
        path: "Articles",
        page: () => import('../views/Articles'),
        meta: {
          title: '文章管理'
        }
      },
      {
        path: "Categories",
        page: () => import('@/views/Categories'),
        meta: {
          title: '分类管理'
        }
      },
      {
        path: "Tags",
        page: () => import('@/views/Tags'),
        meta: {
          title: 'Tag管理'
        }
      },
      {
        path: "Files",
        page: () => import('@/views/Files'),
        meta: {
          title: '文件管理'
        }
      },
    ],
  },
  {
    path: '/login',
    page: () => import('@/views/Login'),
    meta: {
      title: '登录',
      auth: false
    }
  }
];
//根据路径获取路由
const checkAuth: any = (routers: Array<RouteObjectRule>, path: string) => {
  for (const data of routers) {
    if (data.path === path) return data;
    if (data.children) {
      const res = checkAuth(data.children, path);
      if (res) return res;
    }
  }
  return null
};

//全局路由守卫
const onRouteBefore: onRouteBeforeRule = (meta, to) => {
  const {auth, title} = meta;
  if (title) {
    document.title = title || 'Blog管理';
  }
  return (auth !== false && !AppState.token) ? '/login' : to;
  // return (auth !== false && !localStorage.getItem('token')) ? '/login' : to;
};

export default routes;
export {
  onRouteBefore
}
```

App.tsx

```jsx
import './App.css'
import routes, {onRouteBefore} from './router/index';
import RouterGuard from "@/router/RouterGuard";

function App() {
  return <div className="App">
    <RouterGuard
      routers={routes}
      onRouterBefore={onRouteBefore}
      loading={<span>加载中...</span>}
    />
  </div>
}

export default App;
```

main.tsx

```jsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App'
import './index.css'
import {BrowserRouter, HashRouter} from "react-router-dom";
import {Provider} from "mobx-react";
import * as store from './store'

const baseUrl = `/${import.meta.env.BASE_URL}` || '/';
ReactDOM.createRoot(document.getElementById('root') as HTMLElement).render(
  <HashRouter basename={baseUrl}>
    <Provider {...store}>
      <App/>
    </Provider>
  </HashRouter>
)
```

## 状态管理：从VueX到MobX

Vue官方推荐使用的状态管理工具是VueX，而在React中状态管理则有许多选择，比如Redux、MobX或Dva等；这里选择使用MobX作为入门介绍。

### 安装

`yarn add mobx mobx-react`

### 配置

router/AppState.ts

```jsx
import {makeAutoObservable} from 'mobx'

class AppState {
  constructor() {
    makeAutoObservable(this)
  }

  token = localStorage.getItem("token") ? localStorage.getItem("token") : "";

  setToken = (token: string) => {
    this.token = token;
    localStorage.setItem("token", token);
  }
}

export default new AppState()
```

router/index.tsx

```jsx
import AppState from "@/store/AppState";

export {AppState}
```

main.tsx

```jsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App'
import './index.css'
import {BrowserRouter, HashRouter} from "react-router-dom";
import {Provider} from "mobx-react";
import * as store from './store'

const baseUrl = `/${import.meta.env.BASE_URL}` || '/';
ReactDOM.createRoot(document.getElementById('root') as HTMLElement).render(
  <HashRouter basename={baseUrl}>
    <Provider {...store}>
      <App/>
    </Provider>
  </HashRouter>
)
```

### 在组件中使用

views/Login.tsx

```jsx
import '@/assets/css/Login.css'
import {FC} from "react";
import {observer, inject} from "mobx-react";
import {Button, Card, Form} from "antd";
import {post} from "@/utils/request";

// 重点：在组件实例中注入状态
const Login: FC = ({AppState}: any) => {
    
  const onFinish = (values: any) => {
    post('/user', JSON.stringify(values)).then((res: any) => {
      if (res.data) {
        // 调用状态管理器内的函数
        AppState.setToken(res.data);
      }
    })
  };
  return (
    <div className="blog-login">
      <Card title="登录" bordered={false} style={{width: '30%'}}>
        <Form
          name="login"
          onFinish={onFinish}
          autoComplete="on"
          labelCol={{span: 3}}
          labelAlign={'right'}
        >
          <Form.Item>
            <Button type="primary" htmlType="submit" block>登录</Button>
          </Form.Item>
        </Form>
      </Card>
    </div>
  )
}
// 重点：在组件实例中注入状态
export default inject("AppState")(observer(Login))
```

