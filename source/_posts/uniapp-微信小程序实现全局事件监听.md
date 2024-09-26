---
title: uniApp-微信小程序实现全局事件监听并进行数据埋点
toc: true
date: 2022-06-17 00:00:00
tags: [uniApp, 代码片段, 剖析精研]
categories:
  - [前端技术, uniApp]
---
## 零、前言

最近接到需求，领导希望使用微信开放平台上免费的We分析进行数据埋点，但又不希望在现有uniapp开发的微信小程序代码上做侵入式修改，笔者奉命进行了技术调研，考虑通过劫持事件的方式来实现捕获特定事件并上传分析平台的功能。

需要特别注意的是，微信小程序是不能得到document对象的，$el上挂载的也是undefined，自然也就不能通过全局addEventListener的方式来监听特定事件。在调研中想到可以通过劫持小程序的自定义组件构造器Component()来实现事件的监听。

为了便于理解，部分数据结构通过TypeScript接口形式进行描述。

## 一、软件环境

* HbuilderX 3.4.7.20220422
* 微信开发者工具 Stable 1.05.2203070
* 小程序基础库版本 2.24.4 [749]

## 二、相关分析及实现

### uniapp编译微信小程序时对于事件的处理分析

部分知识via掘金：https://juejin.cn/post/6968438754180595742#heading-20

uniapp使用了**uni-app runtime**这个运行时将小程序发行代码进行打包，实现了Vue与小程序之间的数据及事件同步。

#### 源Vue模板及编译产物wxml对照

uniapp的模板编译器代码在/Applications/HBuilderX.app/Contents/HBuilderX/plugins/uniapp-cli/node_modules/@dcloudio/uni-template-complier下。

首先以一个简单的Vue模板为例，观察uniapp是如何将Vue template编译为wxml的：

```html
<template>
  <div @click="add();subtract(2)" @touchstart="mixin($event)">{{ num }}</div>
</template>
```

编译结果为：

```html
<view 
    data-event-opts="{{
        [
            ['tap', [['add'],['subtract',[2]]] ],
            ['touchstart', [['mixin',['$event']]] ]
        ]
    }}"
    bindtap="__e" bindtouchstart="__e"
    class="_div">
    {{num}}
</view>

```

可以看到，uniapp将tap和touchstart事件绑定到__e函数上，然后将事件对应的动作放到了名为eventOpts的dataset中。

#### data-event-opts

**data-event-opts**非常重要。`data-event-opts`是一个二维数组，每个子数组代表一个事件类型。事件类型有两个值，第一个表示事件类型名称，第二个表示触发事件函数的个数。事件函数又是一个数组，第一个值表述事件函数名称，第二个是参数表。下面用TypeScript的类型声明方式进行简单描述：

```typescript
//data-event-opts是一个二维数组，每个子数组代表一个事件类型EventTypes
const dataEventOpts: EventTypes;
interface EventTypes {
  [index:number]: EventType;
}
//事件类型的描述为EventType。EventType只有两个元素，也就是说EventType.length===2
interface EventType {
  //EventType的第一个元素是事件类型名称
  //第二个元素是事件函数的数组EventFuncList，数组内元素为被触发的事件函数
  [index:number]: string | EventFuncList;
}
interface EventFuncList {
  //事件函数依旧是一个数组
  [index:number]: EventFunc;
}
//事件函数的元素为1或2个，分别是事件函数名称和参数表Array<any>
interface EventFunc {
  [index:number]: string | Array<any>;
}
```

对照模板，就可以得出如下推论：

`['tap',[['add'],['subtract',[2]]]]`表示事件类型为`tap`，触发函数有两个，一个为`add`函数且无参数，一个为`subtract`且参数为2。 `['touchstart',[['mixin',['$event']]]]`表示事件类型为`touchstart`，触发函数有一个为`mixin`，参数为`$event`对象。

***不难看出，我们在进行事件捕捉时，只需要读取到```data-event-opts[i][0]```就可以得到每个事件的类型。***

#### handleEvent事件：__e

所有的事件都会调用__e事件，也就是handleEvent。在上文的模板中，handleEvent做了如下操作：

1、拿到点击元素上的`data-event-opts`属性:`[['tap',[['add'],['subtract',[2]]]]`,`['touchstart',[['mixin',['$event']]]]]`

2、根据点击类型获取相应数组，比如`bindTap`就取`['tap',[['add'],['subtract',[2]]]]`，`bindtouchstart`就取`['touchstart',[['mixin',['$event']]]]`

3、依次调用相应事件类型的函数，并传入参数，比如`tap`调用`this.add();this.subtract(2)`

uniapp对mp-wx的相关处理在/Applications/HBuilderX.app/Contents/HBuilderX/plugins/uniapp-cli/node_modules/@dcloudio/uni-mp-weixin下。

```javascript
// @dcloudio/uni-mp-weixin/dist/index.js:1302
function handleEvent (event) {
  event = wrapper$1(event);

  // [['tap',[['handle',[1,2,a]],['handle1',[1,2,a]]]]]
  const dataset = (event.currentTarget || event.target).dataset;
  if (!dataset) {
    return console.warn('事件信息不存在')
  }
  const eventOpts = dataset.eventOpts || dataset['event-opts']; // 支付宝 web-view 组件 dataset 非驼峰
  if (!eventOpts) {
    return console.warn('事件信息不存在')
  }

  // [['handle',[1,2,a]],['handle1',[1,2,a]]]
  const eventType = event.type;

  const ret = [];

  eventOpts.forEach(eventOpt => {
    let type = eventOpt[0];
    const eventsArray = eventOpt[1];

    const isCustom = type.charAt(0) === CUSTOM;
    type = isCustom ? type.slice(1) : type;
    const isOnce = type.charAt(0) === ONCE;
    type = isOnce ? type.slice(1) : type;

    if (eventsArray && isMatchEventType(eventType, type)) {
      eventsArray.forEach(eventArray => {
        const methodName = eventArray[0];
        if (methodName) {
          let handlerCtx = this.$vm;
          if (handlerCtx.$options.generic) { // mp-weixin,mp-toutiao 抽象节点模拟 scoped slots
            handlerCtx = getContextVm(handlerCtx) || handlerCtx;
          }
          if (methodName === '$emit') {
            handlerCtx.$emit.apply(handlerCtx,
              processEventArgs(
                this.$vm,
                event,
                eventArray[1],
                eventArray[2],
                isCustom,
                methodName
              ));
            return
          }
          const handler = handlerCtx[methodName];
          if (!isFn(handler)) {
            throw new Error(` _vm.${methodName} is not a function`)
          }
          if (isOnce) {
            if (handler.once) {
              return
            }
            handler.once = true;
          }
          let params = processEventArgs(
            this.$vm,
            event,
            eventArray[1],
            eventArray[2],
            isCustom,
            methodName
          );
          params = Array.isArray(params) ? params : [];
          // 参数尾部增加原始事件对象用于复杂表达式内获取额外数据
          if (/=\s*\S+\.eventParams\s*\|\|\s*\S+\[['"]event-params['"]\]/.test(handler.toString())) {
            // eslint-disable-next-line no-sparse-arrays
            params = params.concat([, , , , , , , , , , event]);
          }
          ret.push(handler.apply(handlerCtx, params));
        }
      });
    }
  });

  if (
    eventType === 'input' &&
    ret.length === 1 &&
    typeof ret[0] !== 'undefined'
  ) {
    return ret[0]
  }
}
```

### 微信小程序自定义组件Component

mp-wx中的Component文档：https://developers.weixin.qq.com/miniprogram/dev/reference/api/Component.html

#### 构造器Component()

在uniapp-mp-wx中，组件的装载是通过实例化Component进行的。uniapp会默认装载如下8个参数：

```typescript
interface optionsList {
  options: Object | Map<any, any>,
  data: Object,
  properties: Object | Map<any, any>,
  behaviors: string | Array<any>,
  lifetimes: Object,
  pageLifetimes: Object,
  methods: Object,
  created: Function
}
```

并且在methods中注入如下两个函数：

```javascript
methods: {
      __l: handleLink,   //建立组件父子关系
      __e: handleEvent   //事件处理器
}
```

#### 劫持自定义组件构造器Component

劫持Component的构造器，在每个组件的__e中注入自定义的事件劫持器eventProxy

```javascript
// 劫持Component
const _componentProto_ = Component;
Component = function(options) {
  //options.methods内有uniapp注入的事件处理器__e及mpHook
  Object.keys(options.methods).forEach(methodName => {
    //劫持事件处理器__e
    if (methodName == "__e") {
      eventProxy(options.methods, methodName)
    }
  })
  _componentProto_.apply(this, arguments);
}
```

通过劫持事件处理器__e，我们可以实现触发事件时执行我们想要的逻辑了。

### 分析事件对象并编写事件处理器劫持函数eventProxy

微信小程序事件对象描述文档：https://developers.weixin.qq.com/miniprogram/dev/framework/view/wxml/event.html#%E4%BA%8B%E4%BB%B6%E5%AF%B9%E8%B1%A1

在上一步里我们劫持了Component，并且成功获得了事件处理器__e，那么编写针对事件处理器的劫持函数吧。

```javascript
function eventProxy(methodList, methodName) {
  const _funcProto_ = methodList[methodName];
  methodList[methodName] = function() {
    _funcProto_.apply(this, arguments);
    let prop = {};
    if (isObject(arguments[0])) {
      if (Object.keys(arguments[0]).length > 0) {
        //arguments[0]即为事件对象的属性
      }
    }
  }
}
```

uniapp-mp-wx中，事件对象通常具有如下属性：

```javascript
["type", "timeStamp", "target", "currentTarget", "mark", "detail", "touches", "changedTouches", "mut", "_userTap", "mp", "stopPropagation", "preventDefault"]
```

其中，对于数据埋点尤其有用的是如下四个属性：

* type：描述事件类型。常见种类有tap（click）、input、blur、focus等

* currentTarget：事件绑定的当前组件

  从Vue模板编译一节中可知，我们应该关注currentTarget.dataset.eventOpts这个属性，这里记载了事件被触发时的一些信息。
```typescript
interface currentTarget {
  id: string,				//当前元素的id
  dataset: Object		//当前元素上由data-开头的自定义属性组成的集合
}
```
* mark：可以使用 `mark` 来识别具体触发事件的 target 节点。此外， `mark` 还可以用于承载一些自定义数据（类似于 `dataset` ）。

  当事件触发时，事件冒泡路径上所有的 `mark` 会被合并，并返回给事件回调函数。（即使事件不是冒泡事件，也会 `mark` 。）

  如果想要得到一些详细的锚点数据，可以在代码中做一些mark标记。

```html
  <view mark:myMark="last" bindtap="bindViewTap">
    <button mark:anotherMark="leaf" bindtap="bindButtonTap">按钮</button>
  </view>
  <script>
  Page({
    bindViewTap: function(e) {
  		//Object.keys(e.mark)即为触发事件的节点经过的所有mark
      e.mark.myMark === "last" // true
      e.mark.anotherMark === "leaf" // true
    }
  })
  </script>
```

* detail：自定义事件所携带的数据，如表单组件的提交事件会携带用户的输入，媒体的错误事件会携带错误信息，详见[组件](https://developers.weixin.qq.com/miniprogram/dev/component)定义中各个事件的定义。

  点击事件的`detail` 带有的 x, y 同 pageX, pageY 代表距离文档左上角的距离。

  这里给出tap及input事件返回的detail结构：

```typescript
  interface tapDetail {
    x: number,  //距离文档X轴零点的距离，零点为文档左上角
    y: number		//距离文档Y轴零点的距离
  }
  
  interface inputDetail {
    value: string,		//用户输入的值
    cursor: number,		//触发事件时光标所在的位置
    keyCode: number		//触发事件时用户输入的keyCode
  }
```

  结合如上属性，简单地完善一下事件劫持器吧：

```javascript
  function eventProxy(methodList, methodName) {
    const _funcProto_ = methodList[methodName];
    methodList[methodName] = function() {
      _funcProto_.apply(this, arguments);
      let prop = {};
      if (isObject(arguments[0])) {
        if (Object.keys(arguments[0]).length > 0) {
          //记录触发页面信息
          const pages = getCurrentPages();
          const currentPage = pages[pages.length - 1];
          prop["$page_path"] = currentPage.route; //页面路径
          prop["$page_query"] = currentPage.options || {}; //页面携带的query参数
          const type = arguments[0]["type"];
          const current_target = arguments[0].currentTarget || {};
          const dataset = current_target.dataset || {};
          prop["$event_type"] = type;
          prop["$event_timestamp"] = Date.now();
          prop["$element_id"] = current_target.id;
          const eventDetail = arguments[0].detail;
          prop["$event_detail"] = eventDetail;
          if (!!dataset.eventOpts && type) {
            if (type == "tap") { //只记录点击事件
              const event_opts = dataset.eventOpts;
              if (Array.isArray(event_opts) && event_opts[0].length === 2) {
                let eventFunc = [];
                event_opts[0][1].forEach(event => {
                  eventFunc.push({
                    name: event[0],
                    params: event[1] || ''
                  })
                })
                prop["$event_function"] = eventFunc;
              }
            }
            postWeData(prop); //在此处上传记录的事件数据
          }
        }
      }
    };
  }
```

  

## 三、完整代码结构

```javascript
(function() {
  const isObject = function(obj) {
    if (obj === undefined || obj === null) {
      return false;
    } else {
      return toString.call(obj) == "[object Object]";
    }
  };
  // 劫持Component
  const _componentProto_ = Component;
  Component = function(options) {
    //options.methods内有uniapp注入的事件处理器__e及mpHook
    Object.keys(options.methods).forEach(methodName => {
      if (methodName == "__e") {
        //劫持事件处理器
        eventProxy(options.methods, methodName)
      }
    })
    _componentProto_.apply(this, arguments);
  }

  function eventProxy(methodList, methodName) {
    //事件处理器的劫持
  }

  const postWeData = function(data) {
    //埋点上传器
    console.log(data)
  }
})()
```

使用：在项目的main.js里引入即可

```javascript
//main.js
import './common/WeData/index.js'
```

## 四、后记

上述事件劫持器只是一个例子，实现了基本的tap事件记录。实际上笔者通过扩展配置读取的方式来完成更加便捷的埋点操作，后续只需产品给出希望收集的事件名，开发在固定的配置文件中写好代码中事件触发的函数名即可实现tap白名单记录功能。更加详细的埋点功能可以通过阅读***分析事件对象***小节来扩展，在此仅做抛砖引玉。
