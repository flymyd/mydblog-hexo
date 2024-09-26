---
title: Next.js文档 - 3.数据获取-SSG
toc: true
date: 2023-3-1 19:36:44
tags: [Next.js, TypeScript, Node.js]
categories:
  - [前端技术, Next.js]
---
## getStaticProps（SSG使用）

当你在一个Page中导出了`getStaticProps `函数，那么Next将会在构建时使用`getStaticProps`返回的数据预生成这个页面。

```js
export async function getStaticProps(context) {
  return {
    props: {}, // will be passed to the page component as props
  }
}
```

请注意，无论渲染类型是什么，任何`props`都将被传递到页面组件上，并可以在客户端得到的初始HTML中被查看，这是为了让页面能够被正确地水合。请确保你没有将任何不应在客户端上出现的敏感信息传递给`props`。

*PS：SSG页面的生命为：声明getStaticProps => Next构建（build） => 用户请求该页面（可被复用）*

### 何时应该使用getStaticProps

如果出现以下情况，您应该使用`getStaticProps`：

- 渲染页面所需的数据在用户请求该页面之前的构建时可用
- 数据来自无头 CMS
- 页面必须预渲染（用于 SEO）并且需要非常快的速度——`getStaticProps`生成`HTML`和`JSON`文件，两者都可以由 CDN 缓存以提高性能
- 数据可以被公开缓存（非面向特定用户）。在某些特定情况下，可以通过使用中间件重写路由来绕过这种情况。

### getStaticProps何时运行

`getStaticProps`始终在服务器上运行，从不在客户端上运行。[您可以使用此工具](https://next-code-elimination.vercel.app/)验证`getStaticProps`内部编写的代码是否已从客户端包中删除。

* `getStaticProps`总在 `next build`执行期间运行
* `getStaticProps`当使用 [`fallback: true`](https://nextjs.org/docs/api-reference/data-fetching/get-static-paths#fallback-true)时，在后台运行
* `getStaticProps`当使用 [`fallback: blocking`](https://nextjs.org/docs/api-reference/data-fetching/get-static-paths#fallback-blocking)时，在初始渲染之前调用
* `getStaticProps`当使用 `revalidate`时，在后台运行
* `getStaticProps`当使用[`revalidate()`](https://nextjs.org/docs/basic-features/data-fetching/incremental-static-regeneration#on-demand-revalidation)时，在后台按需运行

当与[增量静态再生](https://nextjs.org/docs/basic-features/data-fetching/incremental-static-regeneration)（ISG）结合使用时，`getStaticProps`将在后台运行，同时重新验证陈旧页面，并将新页面提供给浏览器。

`getStaticProps`无法访问传入请求（例如查询参数或 HTTP 标头），因为它会生成静态 HTML。如果您需要访问您页面的请求，请考虑在`getStaticProps`之外结合使用[Middleware](https://nextjs.org/docs/middleware)

### 使用 getStaticProps 从 CMS 获取数据

这是一个从 CMS 获取博客文章列表的例子：

```jsx
// posts will be populated at build time by getStaticProps()
function Blog({ posts }) {
  return (
    <ul>
      {posts.map((post) => (
        <li>{post.title}</li>
      ))}
    </ul>
  )
}

// This function gets called at build time on server-side.
// It won't be called on client-side, so you can even do
// direct database queries.
export async function getStaticProps() {
  // Call an external API endpoint to get posts.
  // You can use any data fetching library
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

export default Blog
```

### 直接写服务端代码

由于`getStaticProps`只在服务器端运行，它永远不会在客户端运行。它甚至不会包含在浏览器的 JS 包中，因此您可以编写直接的数据库查询，而无需将它们发送到浏览器。

这意味着你可以直接在`getStaticProps`里写服务端代码以代替在`getStaticProps`中访问API路由来获取数据的行为。

如下所示，这里有一个用于从CMS获取一些数据的API路由。这个API路由直接在`getStaticProps`中被调用，这产生了额外的开销，降低了性能。应该取而代之的是，把从CMS获取数据的逻辑放在`lib/`文件夹下共享。这段逻辑可以同时共享给`getStaticProps`。

```js
// lib/load-posts.js

// The following function is shared
// with getStaticProps and API routes
// from a `lib/` directory
export async function loadPosts() {
  // Call an external API endpoint to get posts
  const res = await fetch('https://.../posts/')
  const data = await res.json()

  return data
}

// pages/blog.js
import { loadPosts } from '../lib/load-posts'

// This function runs only on the server side
export async function getStaticProps() {
  // Instead of fetching your `/api` route you can call the same
  // function directly in `getStaticProps`
  const posts = await loadPosts()

  // Props returned will be passed to the page component
  return { props: { posts } }
}
```

如果你不使用API路由来获取数据，那么也可以直接使用`fetch()`API在`getStaticProps`里获取数据。

### 静态生成HTML和JSON

当一个使用了`getStaticProps`的页面在构建时被预渲染，除了页面 HTML 文件之外，Next.js 还会生成一个 JSON 文件来保存`getStaticProps`运行的结果。

此 JSON 文件将用于通过[`next/link`](https://nextjs.org/docs/api-reference/next/link)或[`next/router`](https://nextjs.org/docs/api-reference/next/router)访问的客户端路由。当您导航到使用`getStaticProps`预渲染的页面时，Next.js 会获取此 JSON 文件（在构建时预生成）并将其用作页面组件的`props`。这意味着客户端侧页面转换时不会调用`getStaticProps`，因为该页面仅使用导出的 JSON。

使用增量静态生成（ISR）时，`getStaticProps`将在后台执行以生成客户端导航所需的 JSON。您可能会以针对同一页面发出多个请求的形式看到这一点，但是，这是有意为之的，不会影响最终的性能。

### 何处可以使用getStaticProps

`getStaticProps`只能在页面组件里被导出。你不能从一个非页面文件或`_app`、`_document`、`_error`等页面导出该函数。

这种限制的原因之一是 React 需要在页面渲染之前拥有所有必需的数据。

此外，您必须将`export getStaticProps`作为独立函数使用——如果您将其添加为页面组件的属性，它将**无法工作。**

*注意：如果您已经创建了[自定义app](https://nextjs.org/docs/advanced-features/custom-app)，请确保将`pageProps`传递给链接文档中所示的页面组件，否则props将为空。*

### 开发模式下每次请求都会执行

在开发模式下(`next dev`)，每个请求都会调用`getStaticProps`

### 预览模式

您可以使用[**预览模式**](https://nextjs.org/docs/advanced-features/preview-mode)暂时绕过SSG并在请求时渲染页面而不是构建时。例如，您可能正在使用无头 CMS 并希望在发布之前预览草稿。
