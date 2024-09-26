---
title: 从Vue2到Vue3
toc: true
date: 2022-04-22 17:24:04
tags: [Vite, Vue2, Vue3]
categories:
  - [前端技术, 学习笔记]
---
## 前言

这是一份写给项目组内部的基础普及文档。
在下文中，Vue 2.x将会简称为v2。如无特别说明，Vue或v3特指Vue 3.x。

## 常用全局API

* Vue.prototype

  在v2中，我们经常使用形如`Vue.prototype.$xxx = xxx`的全局挂载形式以添加所有组件都能访问的 property。但在v3中，该api迁移为`app.config.globalProperties.$xxx = xxx`。

* Vue.directive

  v3中迁移为`app.directive`

* Vue.extend

  移除。

* Vue.mixin

  迁移为`app.mixin`。然而，v3中更建议使用hooks来代替mixins。

* 其它：参阅[全局API应用实例](https://v3-migration.vuejs.org/zh/breaking-changes/global-api.html)

## 全局API摇树

v3中为了减少打包体积，被暴露在Vue对象上的全局API将会在显式导入后才被打包。如：

```html
<script>
import { get } from '@/utils/request';
// v2
export default {
    data() {
        return {
            test: ''
        }
    },
    mounted(){
        get('/test').then(res=>{
            // 一些逻辑
            this.$nextTick(()=>{ this.test = res.data; })
        })
    }
}
</script>

<script setup lang="ts">
// v3
import { get } from '@/utils/request';
import { nextTick, onMounted, ref } from 'vue'
const test = ref();
onMounted(()=>{
	get('/test').then(res=>{
		// 一些逻辑
        nextTick(()=>{ test.value = res.data; })
    })
})
</script>
```

## 模板和常用指令

### 模板

Vue3单文件组件不再仅支持一个根节点。换言之，你可以创建若干个根节点，如：

```html
<!--Children.vue-->
<template>
	<span>This is first root</span>
    <span>This is second root</span>
</template>
```

当一个模板包含多个根节点时，这个模板则在AST时被表示为一个Fragment。但是，如果该组件作为一个子组件出现，而该组件下又有若干个子组件，那么需要显示定义attribute绑定的位置。此处请参阅[多根节点的 Attributes 继承](https://cn.vuejs.org/guide/components/attrs.html#attribute-inheritance-on-multiple-root-nodes)。如果无法很好地了解`$attrs`所代表的意义，那么请使用传统单根节点写法，这有助于减轻心智负担。

### v-model（对于自定义组件的破坏性更改）

在v2中，v-model拥有一组默认的prop及事件默认名称——通常我们称为v-bind及v-on的语法糖。复习一下：

```html
<intput v-model="name" />
等价于
<input type="text" :value="name" @input="name = $event.target.value">

自定义组件：
<script>
// ChildComponent.vue
export default {
  model: {
    prop: 'title',
    event: 'change'
  },
  props: {
    // 这将允许 `value` 属性用于其他用途
    value: String,
    // 使用 `title` 代替 `value` 作为 model 的 prop
    title: {
      type: String,
      default: 'Default title'
    }
  }
}
</script>

在父组件中：
<ChildComponent v-model="pageTitle" />
等价于
<ChildComponent :title="pageTitle" @change="pageTitle = $event" />
```

但在v3中，用于自定义组件时：

* v-model的prop和事件的默认名称已经更改：

  prop: `value -> modelValue`

  事件： `input -> update:modelValue`

* 子组件的`model`选项及`.sync`修饰符也已被移除。
* 现在可以在同一个组件上使用多个 `v-model` 绑定；
* 现在可以自定义 `v-model` 修饰符。

更详细的说明，参考：[v-model](https://v3-migration.vuejs.org/zh/breaking-changes/v-model.html)

### key

* 重要：`<template v-for>` 的 `key` 应该设置在 `<template>` 标签上 (而不是设置在它的子节点上)。
* 对于 `v-if`/`v-else`/`v-else-if` 的各分支项 `key` 将不再是必须的，因为现在 Vue 会自动生成唯一的 `key`。
* 如果你手动提供 `key`，那么每个分支必须使用唯一的 `key`。你将不再能通过故意使用相同的 `key` 来强制重用分支。

### v-if和v-for的优先级（相反）

两者作用于同一个元素上时，`v-if` 会拥有比 `v-for` 更高的优先级。相对地，v2中 `v-for` 会优先作用。

### v-bind合并

v-bind 的绑定顺序会影响渲染结果。简而言之，v3中的属性覆盖规则类似于css，后来者居上。如：

```html
<!-- 模板 -->
<div id="red" v-bind="{ id: 'blue' }"></div>
<!-- 结果 -->
<div id="blue"></div>

<!-- 模板 -->
<div v-bind="{ id: 'blue' }" id="red"></div>
<!-- 结果 -->
<div id="red"></div>
```

### 重要：移除v-on.native修饰符

v2中默认情况下，传递给带有 `v-on` 的组件的事件监听器只能通过 `this.$emit` 触发。要将原生 DOM 监听器添加到子组件的根元素中，可以使用 `.native` 修饰符：

```html
<my-component @close="handleComponentEvent" @click.native="handleNativeClickEvent" />
```

v3中则移除了`.native`修饰符。同时，[新增的 `emits` 选项](https://v3-migration.vuejs.org/zh/breaking-changes/emits-option)允许子组件定义真正会被触发的事件。因此，对于子组件中*未*被定义为组件触发的所有事件监听器，Vue 现在将把它们作为原生事件监听器添加到子组件的根元素中 (除非在子组件的选项中设置了 `inheritAttrs: false`)。

```html
<my-component
  @close="handleComponentEvent"
  @click="handleNativeClickEvent"
/>

//MyComponent.vue
<script>
  export default {
    emits: ['close']
  }
</script>
```

### 事件API（EventBus）

`$on`，`$off` 和 `$once` 实例方法已被移除，组件实例不再实现事件触发接口。因此，忘掉事件总线吧。更多请参阅[事件 API](https://v3-migration.vuejs.org/zh/breaking-changes/events-api.html)

## 其它被移除的常见API或属性
* filter（过滤器）

* this.$children

## 异步组件

请参阅[异步组件](https://v3-migration.vuejs.org/zh/breaking-changes/async-components.html)

## 新增emits选项

Vue 3 现在提供一个 `emits` 选项，和现有的 `props` 选项类似。这个选项可以用来定义一个组件可以向其父组件触发的事件。更多请参阅[emits选项](https://v3-migration.vuejs.org/zh/breaking-changes/emits-option.html)

## Watch一个数组

 当侦听一个数组时，只有当数组被替换时才会触发回调。如果你需要在数组被改变时触发回调，必须指定 `deep` 选项。

## 一些小变化

### [自定义指令](https://v3-migration.vuejs.org/zh/breaking-changes/custom-directives.html)

### Data选项

* 组件选项 `data` 的声明不再接收纯 JavaScript `object`，而是接收一个 `function`。也就是说，现在只接受形如`data() { return { foo: 'bar' } }`的数据了。

* 当合并来自 mixin 或 extend 的多个 `data` 返回值时，合并操作现在是浅层次的而非深层次的 (只合并根级属性)。

### Transition相关

* 过渡类名 `v-enter` 修改为 `v-enter-from`、过渡类名 `v-leave` 修改为 `v-leave-from`。
* 当使用 `<transition>` 作为根结点的组件从外部被切换时将不再触发过渡效果。
* `<transition-group>` 不再默认渲染根元素，但仍然可以用 `tag` attribute 创建根元素。

## 重要：组合式API——快速入门

这里是一组简单的示例，uniapp中将`<div>`换为`<view>`、`<span>`换为`<text>`等即可：

父组件Home.vue

```html
<template>
	<div>
		<button @click="changeColor('green')">变绿色</button>
    	<button @click="changeColor('yellow')">变黄色</button>
	    <button @click="changeColor('red')">变红色</button>
    	<button @click="triggerRef">从ref触发函数</button>
	    <ZoTest title="组件测试" :color="color" ref="zoTest"></ZoTest>
    </div>
</template>
<script setup lang="ts">
import { ref } from "vue";
import ZoTest from '../../../components/ZoTest/ZoTest.vue';
const color = ref();
const changeColor = (val: string)=>{
  color.value = val;
}
const zoTest = ref<InstanceType<typeof ZoTest> | null>(null);
const triggerRef = ()=>{
  zoTest?.value?.onButtonClick();
}
</script>
```

子组件ZoTest.vue

```html
<template>
  <view>
    <text>{{ testText }}</text>
    <text>{{ helloText }}</text>
    <text>props传入标题：{{ props.title }}</text>
    <text :style="{ color: props.color }">{{ propsColorText }}</text>
    <button @click="onButtonClick">点我改变状态</button>
    <text>点击了{{ eventRecorder.count }}次</text>
    <view :style="{ display: 'flex', flexDirection: 'column' }">
      <text v-for="date in eventRecorder.timeStamp">{{ date.toLocaleString('zh') }}</text>
    </view>
  </view>
</template>

<script lang="ts" setup>
import { computed, reactive, ref, toRefs, watch, watchEffect } from 'vue';
const props = withDefaults(
  defineProps<{
    title: string | number;
    color?: string;
  }>(),
  {
    color: 'skyblue'
  }
);
const emits = defineEmits<{
  (e: 'counter', count: number): void;
}>();
const helloText = ref('Hello world!');
const testText = 'test';
const testArray = reactive<Array<number>>();
testArray.push(123);
const eventRecorder = reactive<{ count: number; timeStamp: Array<Date> }>({
  count: 0,
  timeStamp: []
});
const eventToRefs = () => {
  return toRefs(eventRecorder);
};
const { count, timeStamp } = eventToRefs();
const onButtonClick = () => {
  helloText.value = '我被点击了';
  eventRecorder.count += 1;
};
const doWarn = () => {
  console.log('我是一个未被暴露的函数');
};
defineExpose({
  onButtonClick
});
const colorWatcher = watchEffect(() => {
  if (props.color !== 'red') {
    console.log('颜色改变为', props.color);
  } else {
    console.log('遇到红色，停止监听');
    colorWatcher();
  }
});

// watch可以监听浅层的改变了，但如果需要监听对象套对象这种深层结构则依然要deep。
watch(
  () => eventRecorder.count,
  (newVal, oldVal) => {
    console.log('计数器最新值为', newVal, '上次的值为', oldVal);
    eventRecorder.timeStamp.push(new Date());
  },
  { immediate: false }
);

const propsColorText = computed(() => `props传入文字颜色：${props.color}`);
</script>
```

为了加速学习，这里直接使用了script setup语法糖。用法很简单，template部分写法和v2一样，script部分从以前的：

```html
<script>
export default {
    
}
</script>
```

方式初始化变为：

```html
<script setup lang="ts"></script>
```

当使用setup写法时，任何在 `<script setup>` 声明的顶层的绑定 (包括变量，函数声明，以及 import 导入的内容) 都能在模板中直接使用。import 导入的内容也会以同样的方式暴露。这意味着我们可以在模板表达式中直接使用导入的 helper 函数，而不需要通过 `methods` 选项来暴露它。

如上述代码中的`helloText`是一个非响应式常量，而`helloText`是一个响应式数据。它们在模板中都可以被直接访问。

响应式数据（也就是v2中的`data(){ return {} }`）需要明确地使用响应式API创建。常用的响应式有`ref()`和`reactive()`。

`ref()`常用于基础数据类型的响应式声明，如`number`、`string`等。而`reactive`则用于声明响应式对象，比如Object或Array。

在v3中，响应式对象的追踪原理从基于`Object.defineProperty() + getters/setters`变成了`Proxy`，这意味着像数组之类的JavaScript对象也能被直接监听修改了。响应式转换是“深层”的：它会影响到所有嵌套的属性。一个响应式对象也将深层地解包任何ref属性，同时保持响应性。

`ref`在模板中访问时会被自动解包。可以在上述代码看到，`const helloText = ref('Hello world!')`被直接读取了。然而，当在script中尝试访问或修改由`ref`创建的响应式变量时，则需要访问`ref.value`，如上述代码中的`helloText.value = '我被点击了'; `

由`reactive`创建的响应式对象则只需要直接访问。如`eventRecorder.count +=1`或`testArray.push(123);`等。

`ref()`与`reactive()`均会根据初始值进行类型推导。若要声明其它可能的类型，则需要在泛型中传入类型，如：

```typescript
const eventRecorder = reactive<{ count: number; timeStamp: Array<Date> }>({
  count: 0,
  timeStamp: []
});
```

响应式对象若直接解构则会丢失响应性。比如`const {count} = eventRecorder`后再`count++`则不会被追踪。这时应该使用`toRefs()`进行响应式转换。监听props时也同样建议先进行响应式解构再watch。这里是一个将`reactvice`安全解构的例子：

```typescript
const { count, timeStamp } = toRefs(eventRecorder);
```

与v2不同的是，v3的props与emit需要用define宏声明类型，使用时使用define的返回值进行访问。声明props的各种属性则需要在`defineProps`的泛型中传入一组类型定义。若想为props声明初始值则需要在`defineProps`外包装一层`withDefaults`。

```typescript
// title为必传, color为选传. color的默认值为'skyblue'
const props = withDefaults(
  defineProps<{
    title: string | number;
    color?: string;
  }>(),
  {
    color: 'skyblue'
  }
);
// 不为color声明默认值
defineProps<{
  title: string | number;
  color?: string;
}>()
```

setup中引入组件也同样简单，直接import后就可以将其当做组件名使用，如：

```html
<script setup>
import MyComponent from './MyComponent.vue'
</script>

<template>
  <MyComponent />
</template>
```

当然， 在HbuilderX中的uni-app项目同样可以使用easycom方式加载组件，也就是在component目录下创建一个以组件名命名的目录，在其中放入对应的组件，就可以免去import直接引用：

```
-- components
   -- ZoTest
      -- ZoTest.vue
```

当我们尝试从父组件用ref的方式访问子组件内部方法或数据时，有如下几点是和v2不同的：

* 访问方式不同

  首先在父组件中给子组件添加`ref`属性，然后声明一个与`ref`同名的变量，该变量即为`ref`的引用：

  `const zoTest = ref<InstanceType<typeof ZoTest> | null>(null); `

  `ref`中有一个较为复杂的泛型，它主要是表明了该变量的类型为子组件暴露属性的类型或null。当DOM尚未渲染（或被v-if隐藏）时该变量的值可能为null。换言之，与v2相似的是，你依然要保证在DOM渲染完成后再去尝试访问`ref`内部的属性。至于暴露属性这个概念，见下一条。

* 子组件需要声明向外部暴露的属性

  ```typescript
  const onButtonClick = () => {
    helloText.value = '我被点击了';
    eventRecorder.count += 1;
  };
  const doWarn = () => {
    console.log('我是一个未被暴露的函数');
  };
  defineExpose({
    onButtonClick  //只有它能被父组件访问
  });
  ```

  在v3的script setup中，子组件的属性默认无法通过`ref`直接访问到。我们可以使用`defineExpose`宏来手动声明该组件允许向外部暴露的属性列表。在上述代码示例中，我们只暴露了`onButtonClick`方法。

在v3中新增了很多钩子。常用的有`watchEffect()`、`watch()`、`computed()`。

* computed()的使用方法和v2基本一致，只是函数化了。例如，我们想要为props中的color属性绑定一个计算属性：

  ```typescript
  const propsColorText = computed(() => `props传入文字颜色：${props.color}`);
  ```

* watchEffect钩子的作用是立即运行一个函数，同时响应式地追踪其依赖，并在依赖更改时重新执行。换言之，watchEffect内部的任何一项依赖改变都会触发它，**回调函数的触发就是因为依赖改变而导致的副作用**。

  第一个参数就是要运行的副作用函数。这个副作用函数的参数也是一个函数，用来注册清理回调。清理回调会在该副作用下一次执行前被调用，可以用来清理无效的副作用，例如等待中的异步请求。

  第二个参数是一个可选的选项，可以用来调整副作用的刷新时机或调试副作用的依赖。

  返回值是一个用来停止该副作用的函数。

  在上述代码的例子中：

  ```typescript
  const colorWatcher = watchEffect(() => {
    if (props.color !== 'red') {
      console.log('颜色改变为', props.color);
    } else {
      console.log('遇到红色，停止监听');
      colorWatcher();
    }
  });
  ```

  我们监听了`props.color`，当它不是红色时持续监听，只要改变了就在控制台里打印一次。而当它变为红色时，销毁该副作用监听。

* watch钩子也是和v2基本一致的，三个参数分别为source、callback和options。

  第一个参数是监听器的源，这个来源可以是以下几种：

  - 一个函数，返回一个值
  - 一个 ref
  - 一个响应式对象
  - ...或是由以上类型的值组成的数组

  第二个参数是在发生变化时要调用的回调函数。这个回调函数接受三个参数：新值、旧值，以及一个用于注册副作用清理的回调函数。该回调函数会在副作用下一次重新执行前调用，可以用来清除无效的副作用，例如等待中的异步请求。

  当侦听多个来源时，回调函数接受两个数组，分别对应来源数组中的新值和旧值。

  第三个可选的参数是一个对象，支持以下这些选项：

  - （常用）**`immediate`**：在侦听器创建时立即触发回调。第一次调用时旧值是 `undefined`。
  - （常用）**`deep`**：如果源是对象，强制深度遍历，以便在深层级变更时触发回调。参考[深层侦听器](https://cn.vuejs.org/guide/essentials/watchers.html#deep-watchers)。
  - **`flush`**：调整回调函数的刷新时机。参考[回调的刷新时机](https://cn.vuejs.org/guide/essentials/watchers.html#callback-flush-timing)及 [`watchEffect()`](https://cn.vuejs.org/api/reactivity-core.html#watcheffect)。
  - **`onTrack / onTrigger`**：调试侦听器的依赖。参考[调试侦听器](https://cn.vuejs.org/guide/extras/reactivity-in-depth.html#watcher-debugging)。

  与 [`watchEffect()`](https://cn.vuejs.org/api/reactivity-core.html#watcheffect) 相比，`watch()` 使我们可以：

  - 懒执行副作用；
  - 更加明确是应该由哪个状态触发侦听器重新执行；
  - 可以访问所侦听状态的前一个值和当前值。

  在上述代码例子中：

  ```typescript
  watch(
    () => eventRecorder.count,
    (newVal, oldVal) => {
      console.log('计数器最新值为', newVal, '上次的值为', oldVal);
      eventRecorder.timeStamp.push(new Date());
    },
    { immediate: false }
  );
  ```

  我们监听了`eventRecorder`的`count`属性，并在`count`被修改时执行对应的回调操作。

## 在script setup中使用组件的高级用法

动态组件、递归组件、命名空间组件请参阅[使用组件](https://cn.vuejs.org/api/sfc-script-setup.html#using-components)

## 插槽

在v3中，具名插槽的使用方法变成了`v-slot:header`

```html
<BaseLayout>
  <template v-slot:header>
    <!-- header 插槽的内容放这里 -->
  </template>
</BaseLayout>
```

