---
title: Vue3项目初始化搭建
toc: true
date: 2022-05-22 00:00:00
tags: [Vue3, Vite]
categories:
  - [前端技术, 工程搭建]
---
## 前言

俗话说的好，万事开头难。这篇文章主要介绍一下Vue 3项目的初始化所需工作。

## 主要配置

技术选型：

* 全家桶：Vue3 + TypeScript + Vite + Pinia + Vue Router 4
* CSS相关：Sass + PostCSS + UnoCSS

## 开始吧

#### 1、使用Vite构建项目并安装插件

```shell
npm init vite@latest vue3-project -- --template vue-ts && cd ./vue3-project && npm i
```

##### 配置基本的vite.config.ts

```typescript
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
const path = require('path');

export default defineConfig({
  server: {
    host: '0.0.0.0',
  },
  resolve: {
    alias: {
      '@': path.resolve(__dirname, "src")
    }
  },
  plugins: [
    vue(),
  ]
})
```

##### 配置tsconfig.json

```json
{
  "compilerOptions": {
    "target": "esnext",
    "useDefineForClassFields": true,
    "module": "esnext",
    "moduleResolution": "node",
    "strict": true,
    "jsx": "preserve",
    "sourceMap": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "esModuleInterop": true,
    "lib": [
      "esnext",
      "dom"
    ],
    "skipLibCheck": true,
    "baseUrl": ".",
    "allowJs": true,
    "paths": {
      "@/*": [
        "src/*"
      ],
      "#/*": [
        "types/*"
      ]
    }
  },
  "include": [
    "src/**/*.ts",
    "src/**/*.d.ts",
    "src/**/*.tsx",
    "src/**/*.vue"
  ],
  "references": [
    {
      "path": "./tsconfig.node.json"
    }
  ]
}
```

##### 安装node类型声明及配置TSX

```shell
npm i --save-dev @types/node
npm i @vitejs/plugin-vue-jsx
```

修改vite.config.ts

```typescript
import vueJsx from '@vitejs/plugin-vue-jsx'
export default defineConfig ({
  plugins: [
    vue(),
    vueJsx(),
  ]
})
```

##### 安装插件vite-plugin-vue-setup-extend（在setup标签上声明name）

```shell
npm i vite-plugin-vue-setup-extend -D
```

修改vite.config.ts

```typescript
import VueSetupExtend from 'vite-plugin-vue-setup-extend'
export default defineConfig({
  plugins: [
    VueSetupExtend()
  ]
})
```

##### 安装插件unplugin-auto-import（无需import即可使用Vue的API）

```shell
npm i unplugin-auto-import -D
npm i vue-global-api -D
```

修改vite.config.ts

```typescript
import AutoImport from 'unplugin-auto-import/vite'
export default defineConfig({
  plugins: [
    AutoImport({
      // dts: 'src/auto-imports.d.ts', // 可以自定义文件生成的位置，默认是根目录下
      imports: ['vue']
    })
  ]
})
```

#### 2、配置Vue Router

```shell
npm install vue-router -S
```

新建/src/router/index.ts

```typescript
import { createRouter, createWebHashHistory, RouteRecordRaw } from 'vue-router';
const routes: Array<RouteRecordRaw> = [
  {
    path: '/',
    name: 'Index',
    component: () => import("@/views/Index.vue"),
    meta: {
      title: '首页'
    }
  },
  {
    path: '/About',
    name: 'About',
    component: () => import("@/views/About.vue"),
    meta: {
      title: '关于'
    }
  },
];

const router = createRouter({
  //hash模式. 历史模式为createWebHistory
  history: createWebHashHistory(),
  routes
});
export default router;
```

修改main.ts

```typescript
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'

createApp(App)
  .use(router)
  .mount('#app')
```

#### 3、配置Pinia

```shell
npm i pinia
```

修改main.ts

```typescript
import { createPinia } from 'pinia'
const pinia = createPinia()
createApp(App)
  .use(pinia)
  .mount('#app')
```

新建/src/store/index.ts

```typescript
import { defineStore } from 'pinia'

export const useStore = defineStore('default', {
  state: () => ({
    counter: 0
  }),
  getters: {},
  actions: {
    increase(num: number) {
      this.counter += num;
    }
  },
})
```

#### 4、安装Sass、PostCSS及一些插件

```shell
npm i sass -D
npm i postcss postcss-modules postcss-flexbugs-fixes autoprefixer -D
```

vite.config.ts添加节点

```typescript
  css: {
    postcss: {
      plugins: [
        require('autoprefixer')({
          overrideBrowserslist: [
            'Android 4.1',
            'iOS 7.1',
            'Chrome > 31',
            'ff > 31',
            'ie >= 8',
            '> 1%',
          ],
          grid: true,
        }),
        require('postcss-flexbugs-fixes'),
      ],
    }
  },
```

##### 配置iconify

```shell
npm i @iconify/iconify
npm i vite-plugin-purge-icons @iconify/json -D
```

修改vite.config.ts

```typescript
import PurgeIcons from 'vite-plugin-purge-icons'

export default defineConfig({
  plugins: [
    PurgeIcons({ content: ['**/*.html', '**/*.ts', '**/*.js', '**/*.vue'] })
  ]
})
```

修改main.ts

```typescript
import '@purge-icons/generated';
import FIcon from '@/components/FIcon';
createApp(App)
  .component("f-icon", FIcon)
  .mount('#app')
```

封装方便的组件

```tsx
// @/components/FIcon.tsx
import { defineComponent } from 'vue';

export default defineComponent({
  name: 'FIcon',
  props: {
    icon: { type: String, required: true },
    size: { type: [String, Number], default: '18' },
  },
  setup(props) {
    return () => {
      return (
        <span style={{ fontSize: props.size + 'px' }}>
          <span class="iconify" style="vertical-align: middle;" data-icon={props.icon}></span>
        </span>
      )
    }
  },
});
```

在模板中使用

```html
<f-icon icon="bi:alarm-fill" size="24"></f-icon>
```

##### 配置UnoCSS

```shell
npm i -D @unocss/vite
npm i -D @unocss/preset-wind
npm i @unocss/reset
```

修改vite.config.ts

```typescript
import Unocss from '@unocss/vite'
import presetWind from '@unocss/preset-wind'
export default {
  plugins: [
    Unocss({
      presets: [
        presetWind(),
      ],
    }),
  ],
}
```

修改main.ts

```typescript
import 'uno.css'
import '@unocss/reset/tailwind.css'
```
