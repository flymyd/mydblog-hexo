---
title: Next.js文档 - 1.页面
toc: true
date: 2023-3-1 17:36:44
tags: [Next.js, TypeScript, Node.js]
categories:
  - [前端技术, Next.js]
---
# 页面（Pages）

Next.js 13 采用了beta版的目录结构（/app）。新结构默认支持layouts（布局）、嵌套路由、服务器端渲染组件。与此同时你可以在layout中获取整个应用的数据，包括支持更精细粒度的嵌套布局。

在Next中，page是从pages文件夹下的`js、jsx、ts、tsx`文件中导出的React组件。每个页面都根据其文件名与路由相关联。

例：这是一个对应`/about`路由的组件`pages/about.js`

```jsx
export default function About() {
  return <div>About</div>
}
```

## 动态路由页面

Next支持动态路由。比如你创建了一个名为`pages/posts/[id].js`的文件，那么你可以通过形如 `posts/1`、`posts/2`的路由来访问它。

## 预渲染

默认情况下，Next会对每个页面进行预渲染，这意味着Next是提前为每个页面生成HTML，而不是由客户端的js来完成所有工作的。预渲染可以带来更好的性能和 SEO。

每个生成的 HTML 都与该页面所需的最简 JavaScript 代码相关联。当浏览器加载页面时，其 JavaScript 代码会运行并使页面可交互。（这个过程称为水合）

*PS：在阅读下一节后更有助于理解水合的概念*。

### 预渲染的两种形式

Next有两种预渲染方式：静态生成（SSG, Static Server Generation）和服务器端渲染（SSR, Server Side Rendering），区别在于它何时为页面生成HTML。

- 静态生成（推荐）：HTML在构建时生成，并将在每个请求中重复使用。
- 服务器端渲染：在每次请求时生成HTML。

Next允许你决定用于每个页面的预渲染形式。比如，你可以选择对大多数页面使用静态生成，而对特定页面使用SSR的方式来创建“混合”的Next.js 应用。

出于性能原因考虑，建议尽量使用SSG。静态生成的页面可以由 CDN 缓存，无需额外配置即可提高性能。但是，在某些情况下，SSR可能是唯一的选择。

*PS：比如说你要构建一个响应式页面，这里面有一个导航栏需要在移动端和PC端上有不同的表现形式。由于需要访问window对象，你无法生成一个纯静态的页面。这时就需要SSR进行脱水生成。*

您还可以将**客户端数据获取**与SSG或SSR一起使用。这意味着页面的某些部分可以完全由客户端JavaScript呈现。

## 静态生成（SSG）

如果一个页面使用了SSG，这个页面的HTML将会在build时生成。这意味着在生产环境下该页面的HTML在你启动应用服务的时候就已经被生成好了。这个HTML会在每个请求中被复用，它可以被CDN缓存。

在Next中你既可以静态生成有（外部）数据的页面，也可以生成一个纯粹的简单页面。

### 无外部数据的静态生成

默认情况下，Next将会预渲染一个无外部数据的页面。例如：

```jsx
function About() {
  return <div>About</div>
}

export default About
```

此页面不需要获取任何用于预渲染的外部数据。在这种情况下，Next 在build时会生成一个单独的 HTML 文件。

### 有外部数据的静态生成

有的页面需要在预渲染时请求额外的数据，这通常有两种场景，并且可能同时出现：

* 该页面的内容依赖于外部数据（如http请求）：使用`getStaticProps`
* 该页面的路径依赖于外部数据（如分页）：使用`getStaticPaths`

#### 场景一：页面内容依赖于外部数据

博客页面可能需要从 CMS（内容管理系统）获取博客文章列表。

```jsx
// TODO: Need to fetch `posts` (by calling some API endpoint)
//       before this page can be pre-rendered.
export default function Blog({ posts }) {
  return (
    <ul>
      {posts.map((post) => (
        <li>{post.title}</li>
      ))}
    </ul>
  )
}
```

为了在预渲染时获取此数据，Next 允许你通过在该文件里调用`export async getStaticProps()`函数。此函数在构建时被调用，并允许您将获取的数据通过`props`传递给将被预渲染的页面。

```jsx
export default function Blog({ posts }) {
  // Render posts...
}

// This function gets called at build time
export async function getStaticProps() {
  // Call an external API endpoint to get posts
  const res = await fetch('https://.../posts')
  const posts = await res.json()

  // By returning { props: { posts } }, the Blog component
  // will receive `posts` as a prop at build time
  return {
    props: {
      posts,
    },
  }
}
```

#### 场景二：页面路径依赖于外部数据

Next允许你创建动态路由页面。比如，你可以创建一个名为`pages/posts/[id].js`的文件来展示基于id的blog页。当你访问`posts/1`的时候，`[id].js`页将会展示id为1的博文。

但是，有时候这个id是依赖于外部数据的。比如说你的数据库里目前只有id为1的博文。这时候你在构建时只会预渲染`post/1`。但过后你可能添加了id为2、3、4...的博文。这时，你想要预渲染所有已经被添加到数据库里的博文。

所以你预渲染的页面路径依赖于外部数据。为了处理这个问题，Next允许您从动态路由页面调用`export async getStaticPaths()`函数。此函数在构建时被调用，并允许您指定要预渲染的路径。例如原本的`pages/posts/[id].js`形如：

```jsx
// This function gets called at build time
export async function getStaticPaths() {
  // Call an external API endpoint to get posts
  const res = await fetch('https://.../posts')
  const posts = await res.json()

  // Get the paths we want to pre-render based on posts
  const paths = posts.map((post) => ({
    params: { id: post.id },
  }))

  // We'll pre-render only these paths at build time.
  // { fallback: false } means other routes should 404.
  return { paths, fallback: false }
}
```

根据上述改造后：

```jsx
export default function Post({ post }) {
  // Render post...
}

export async function getStaticPaths() {
  // ...
}

// This also gets called at build time
export async function getStaticProps({ params }) {
  // params contains the post `id`.
  // If the route is like /posts/1, then params.id is 1
  const res = await fetch(`https://.../posts/${params.id}`)
  const post = await res.json()

  // Pass post data to the page via props
  return { props: { post } }
}
```

### 我什么时候应该使用SSG？

我们建议尽可能使用SSG，因为您的页面可以只构建一次并由 CDN 提供服务，这比让服务器在每次请求时呈现页面要快得多**。**

您可以为多种类型的页面使用SSG，包括：

- 营销页面
- 博客文章和投资组合
- 电子商务产品列表
- 帮助和文档

您应该问问自己：“我可以在用户请求之前**预**呈现此页面吗？” 如果答案是肯定的，那么您应该选择SSG。

另一方面，如果您不能在用户请求之前预呈现页面，那么静态生成不是一个好主意**。**也许您的页面显示经常更新的数据，并且页面内容会在每次请求时发生变化。

在这种情况下，您可以执行以下操作之一：

- 将SSG与**客户端数据获取**结合使用：您可以跳过预呈现页面的某些部分，然后使用客户端 JavaScript 来填充它们。
- 使用**SSR：** Next.js 会根据每个请求预呈现一个页面。它会更慢，因为页面无法被 CDN 缓存，但预渲染页面将始终是最新的。我们将在下面讨论这种方法。

## 服务器渲染（SSR） 

如果页面使用**SSR**，则每次请求时都会生成页面 HTML 。

要为页面使用SSR，您需要在该页面中调用`export async getServerSideProps()`. 服务器将在每次请求时调用此函数。

例如，假设您的页面需要预渲染经常更新的数据（从外部 API 获取），您可以在`getServerSideProps`中获取数据并将其通过props传递给页面，如下所示：

```jsx
export default function Page({ data }) {
  // Render data...
}

// This gets called on every request
export async function getServerSideProps() {
  // Fetch data from external API
  const res = await fetch(`https://.../data`)
  const data = await res.json()

  // Pass data to the page via props
  return { props: { data } }
}
```

`getServerSideProps`使用方法很像`getStaticProps`， 但`getServerSideProps`会在每个请求时被执行，而非仅在build时执行一次。

## 总结

我们已经讨论了 Next.js 的两种预渲染形式。

- **静态生成 SSG（推荐）：** HTML 在**构建时**生成，并将在每个请求中重复使用。要使页面使用静态生成，请导出页面组件，或导出`getStaticProps`（或`getStaticPaths`，如果需要）。它非常适合可以在用户请求之前预渲染的页面。您还可以将它与客户端渲染（CSR）一起使用以引入其他数据。
- **服务器端渲染 SSR：** HTML 是在**每个请求**时生成的。要使页面使用服务器端渲染，请使用`getServerSideProps`. 因为服务器端渲染导致性能低于静态生成，所以只有在绝对必要时才使用它。

*下面是文档之外的理解部分*：

* CSR是在执行 JavaScript 脚本的时候，HTML页面已经开始解析并且构建DOM树了，JavaScript 脚本只是动态的改变 DOM 树的结构，使得页面成为希望成为的样子
* SSG导出的内容就是在浏览器请求页面URL的时候，服务端将build时已经组装好的HTML文本直接返回给浏览器。这个HTML文本被浏览器解析之后，不需要经过 JavaScript 脚本的执行，即可直接构建出希望的 DOM 树并展示到页面中。
* SSR在接收到URL请求之后，前端服务器向后端服务器请求数据。请求完成后，前端服务器会组装一个携带了具体数据的HTML文本（脱水），并且返回给浏览器，浏览器得到HTML之后开始渲染页面。同时，浏览器加载并执行 JavaScript 脚本，给页面上的元素绑定事件，让页面变得可交互（水合）。
