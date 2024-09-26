---
title: Next.js文档 - 2.数据获取-SSR
toc: true
date: 2023-3-1 18:36:44
tags: [Next.js, TypeScript, Node.js]
categories:
  - [前端技术, Next.js]
---
## getServerSideProps（SSR使用）

当你在一个Page中导出了`getServerSideProps`函数，那么Next将会在每次页面请求前使用`getServerSideProps`返回的数据进行预渲染。

```js
export async function getServerSideProps(context) {
  return {
    props: {}, // will be passed to the page component as props
  }
}
```

请注意，无论渲染类型是什么，任何`props`都将被传递到页面组件上，并可以在客户端得到的初始HTML中被查看，这是为了让页面能够被正确地水合。请确保你没有将任何不应在客户端上出现的敏感信息传递给`props`。

### getServerSideProps何时运行

`getServerSideProps`仅在服务器端运行，从不在浏览器上运行。如果页面使用`getServerSideProps`，则：

- 当你直接请求此页面时，`getServerSideProps`在请求时运行，并且此页面将使用返回的 props 进行预渲染

- 当你从客户端侧请求一个从`next/link`或`next/router`转变而来的页面时，Next会执行getServerSideProps，向服务器发送一个API请求。

  When you request this page on client-side page transitions through [`next/link`](https://nextjs.org/docs/api-reference/next/link) or [`next/router`](https://nextjs.org/docs/api-reference/next/router), Next.js sends an API request to the server, which runs `getServerSideProps`

`getServerSideProps`返回用于渲染页面的 JSON。所有这些工作都将由Next自动处理，所以只要你已经声明好了`getServerSideProps`，那就不需要做任何额外的事情。

您可以使用[next-code-elimination](https://next-code-elimination.vercel.app/)来验证 Next.js 从客户端侧产物中删除了什么。

`getServerSideProps`只能从**Page组件（页面文件）**导出。您不能从非页面文件中导出它。

请注意，您必须导出`getServerSideProps`为独立函数——如果您将其添加为页面组件的属性，它将**无法工作。**

### 何时应该使用getServerSideProps

`getServerSideProps`仅当您需要渲染*必须在每次请求时预先获取数据*的页面时才应使用。这可能是由于请求的数据或属性（如`authorization`标头或地理位置）决定的。使用`getServerSideProps`的页面将在请求时在服务器端渲染，并且只有在[配置了缓存控制标头](https://nextjs.org/docs/going-to-production#caching)时才会被缓存。

如果您不需要在请求页面期间渲染数据，那么您应该考虑在[客户端](https://nextjs.org/docs/basic-features/data-fetching/get-server-side-props#fetching-data-on-the-client-side)请求数据或使用[`getStaticProps`](https://nextjs.org/docs/basic-features/data-fetching/get-static-props).

#### getServerSideProps 或 API 路由

当您想从服务器获取数据时，可能很想去访问[API 路由](https://nextjs.org/docs/api-routes/introduction)，然后从`getServerSideProps`调用该 API 路由的逻辑。这是一种不必要且低效的方法，因为它会由于调用`getServerSideProps`以及访问服务器上运行的 API 路由而导致发出额外的请求。

以下面的例子为例。API 路由用于从 CMS 获取一些数据。然后直接从`getServerSideProps`调用该 API 路由。这会产生额外的调用，从而降低性能。相反，应该直接将 API 路由中使用的逻辑导入到`getServerSideProps`。这意味着直接从`getServerSideProps`内部调用 CMS、数据库或其他 API 。

#### getServerSideProps与边缘路由

`getServerSideProps`可以同时与Serverless 和 Edge Runtimes结合使用，你可以在两者中设置 props。但是，目前在 Edge Runtime 中，您无权访问响应对象。这意味着您不能——例如——在`getServerSideProps`中添加cookie。要访问响应对象，您应该**继续使用**Node.js 运行时（即默认运行时）。

你可以为通过自定义`config`来为每个Page显式地设置运行时，例如：

```javascript
export const config = {
  runtime: 'nodejs',
}
```

## 在客户端侧获取数据（CSR）

如果你的页面包含频繁更新的数据，并且你不需要预渲染数据，你可以在客户端获取数据。这方面的一个例子是用户特定的数据：

- 首先，立即显示没有数据的页面。页面的某些部分可以使用静态生成进行预呈现。您可以显示缺失数据的加载状态
- 首先，立即显示没有数据的页面。页面的某些部分可以使用SSG进行预渲染。您可以显示缺失数据的加载状态
- 然后，在客户端获取数据并在准备好时显示它

例如，此方法适用于用户仪表板页面。由于仪表板是私有的、特定于用户的页面，因此与 SEO 无关，并且不需要预渲染该页面。数据更新频繁，需要请求时再获取数据。

## 使用 getServerSideProps 在请求时获取数据

以下示例显示了如何在请求时获取数据并预渲染结果。

```js
function Page({ data }) {
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

export default Page
```

## 使用服务端渲染（SSR）进行缓存

你可以在`getServerSideProps`内部使用缓存标头(`Cache-Control`)来缓存动态响应，如：使用[`stale-while-revalidate`](https://web.dev/stale-while-revalidate/)

```js
// This value is considered fresh for ten seconds (s-maxage=10).
// If a request is repeated within the next 10 seconds, the previously
// cached value will still be fresh. If the request is repeated before 59 seconds,
// the cached value will be stale but still render (stale-while-revalidate=59).
//
// In the background, a revalidation request will be made to populate the cache
// with a fresh value. If you refresh the page, you will see the new value.
export async function getServerSideProps({ req, res }) {
  res.setHeader(
    'Cache-Control',
    'public, s-maxage=10, stale-while-revalidate=59'
  )

  return {
    props: {},
  }
}
```

## getServerSideProps是否会渲染错误页（500）

如果`getServerSideProps`内部抛出错误，它将显示`pages/500.js`文件。查看[500 页](https://nextjs.org/docs/advanced-features/custom-error-page#500-page)的文档以了解有关如何创建它的更多信息。在开发环境下不会使用此文件，而是会显示dev遮罩层。
