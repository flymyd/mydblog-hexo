---
title: 用WebStorm开发uniapp微信小程序
toc: true
date: 2023-05-16 10:42:38
tags: [uniApp, 微信小程序, WebStorm, Vite]
categories:
  - [前端技术, uniApp移植Linux记录]
---
## 研究动机

公司从22年底开始全面将uniapp项目切换至`Vue3 + Vite + TypeScript`
技术栈，但HbuilderX直到本文编写时也依然存在很多问题，比如语法检查服务崩溃、无法跳转代码引用等，开发体验非常差，经常出现一个组件打开全是红线或没法进行代码补全的情况，Vue3和Vite带来的开发速度优势基本在这方面全还回去了。

基于如上痛点，我在一个工作相对清闲的下午对HbuilderX的启动机制进行了研究并写了一份运行在命令行里的uniapp编译调试脚本，可以使用它来甩掉HbuilderX来进行dev阶段的开发工作。该文章以微信小程序举例，H5也大同小异。

*懒得看哔哔叨叨，只想捞代码来用？请直接阅读`脚本代码`和`使用流程`两节*

## 调试 && 分析

首先使用微软的[DebugView](https://learn.microsoft.com/en-us/sysinternals/downloads/debugview)
尝试查看调试信息，没想到HbuilderX真的有很多Debug级别的输出，乐。

```shell
[16764] 2023-05-16 09:50:20.583 [INFO:] node "D:/HBuilderX/plugins/node/node.exe"
[16764] 2023-05-16 09:50:20.583 [INFO:] args ("--max-old-space-size=2048", "--no-warnings", "D:/HBuilderX/plugins/uniapp-cli-vite/node_modules/@dcloudio/vite-plugin-uni/bin/uni.js")
```

观察日志可以发现如下几点：

* HbuilderX使用了自己自带的Node.js可执行文件，位于`/plugins/node/node.exe`
* 启动脚本位于`/plugins/uniapp-cli-vite/node_modules/@dcloudio/vite-plugin-uni/bin/uni.js`

那么先去看看`/plugins/uniapp-cli-vite/node_modules/@dcloudio/vite-plugin-uni/package.json`，可以看到`uni.js`
是作为这个包的`bin`存在的。那么想必在启动过程中还要附加很多用于编译的环境变量等。`uni.js`
干货不太多，主要代码就是一些dev环境相关的逻辑，重点在最底部的`require('../dist/cli/index.js')`。顺藤摸瓜看到若干cli定义：

```javascript
const cac_1 = require("cac");
const uni_cli_shared_1 = require("@dcloudio/uni-cli-shared");
const utils_1 = require("./utils");
const action_1 = require("./action");
(0, uni_cli_shared_1.fixBinaryPath)();
const cli = (0, cac_1.cac)('uni');
cli
  .option('-c, --config <file>', `[string] use specified config file`)
  .option('-p, --platform [platform]', '[string] ' + utils_1.PLATFORMS.join(' | '), {
    default: 'h5',
  })
```

很显然`platform`也就是执行平台的枚举值在`./utils.js`
下，其实根据经验来说压根没什么再去看枚举值的必要，微信小程序在uniapp里就叫做`mp-weixin`，但保险起见还是检查一番：

```javascript
exports.PLATFORMS = [
  'app',
  'h5',
  'mp-alipay',
  'mp-baidu',
  'mp-qq',
  'mp-lark',
  'mp-toutiao',
  'mp-weixin',
  'quickapp-webview',
  'quickapp-webview-huawei',
  'quickapp-webview-union',
];
```

没啥好说的了，`uni -p mp-weixin`就是启动微信小程序调试的命令。那么怎么调用呢？观察文件路径可知`vite-plugin-uni`
是`uniapp-cli-vite`的一个插件（或者说npm依赖），那么直接去看`uniapp-cli-vite`的`package.json`
，添加一行`"dev:weixin": "uni -p mp-weixin"`即可。

```json
{
  "scripts": {
    "dev:app": "uni -p app",
    "dev:weixin": "uni -p mp-weixin",
    "build:app": "uni build -p app",
    "dev:h5": "uni",
    "build:h5": "uni build",
    "dev:h5:ssr": "uni --ssr",
    "build:h5:ssr": "uni build --ssr"
  }
}
```

接下来是针对运行环境的一些收尾工作。为了保证执行效果和使用HbuilderX启动时的一致性，先用HbuilderX在`main.js`
中加一行`console.log(process.env)`并启动，检查打印出的环境变量信息。经过整理后保留一些较为有用的环境变量并在脚本中写入。
除此之外Node.js和npm也使用HbuilderX自带的。

这里吐槽一下微信开发者工具，在Windows下的默认安装路径竟然带中文，真无语...Windows下批处理还要因为它多写一行切换代码页。

## 脚本代码

Windows批处理脚本：ws.bat

```shell
chcp 65001
@REM ==============根据实际修改如下两个环境变量==============
set HBUILDER_DIR=D:\HBuilderX
set WEIXIN_DEV_DIR="C:\Program Files (x86)\Tencent\微信web开发者工具"
@REM ==============注意变量尾部不要带斜杠==============

set UNI_INPUT_DIR=%cd%
set UNI_OUTPUT_DIR=%UNI_INPUT_DIR%\unpackage\dist\dev\mp-weixin
set VITE_ROOT_DIR=%UNI_INPUT_DIR%

set UNI_CLI_CONTEXT=%HBUILDER_DIR%\plugins\uniapp-cli-vite
set UNI_HBUILDERX_PLUGINS=%HBUILDER_DIR%\plugins
set UNI_NPM_DIR=%HBUILDER_DIR%\plugins\npm
set UNI_NODE_DIR=%HBUILDER_DIR%\plugins\node

set UNI_COMPILER_VERSION_TYPE=r
set UNI_NODE_ENV=development
set UNI_UTS_PLATFORM=mp-weixin
set UNI_HBUILDERX_LANGID=zh_CN
set UNI_PLATFORM=mp-weixin
set VITE_USER_NODE_ENV=development
set NODE_ENV=development
set NODE_SKIP_PLATFORM_CHECK=1

set NODE="%UNI_NODE_DIR%/node.exe"

cd /d %UNI_CLI_CONTEXT%
cls

start "" /B node --max-old-space-size=2048 --no-warnings "%UNI_NPM_DIR%\node_modules\npm\bin\npm-cli.js" run dev:weixin
%WEIXIN_DEV_DIR%/cli.bat -o %UNI_OUTPUT_DIR%
```

Mac用户：ws.sh（未经验证，可能需要微调）

```shell
#!/bin/bash
# ==============根据实际修改如下两个环境变量==============
export HBUILDER_DIR=/Applications/HBuilderX
export WEIXIN_DEV_DIR="/Applications/wechatwebdevtools.app"
# ==============注意变量尾部不要带斜杠==============

export UNI_INPUT_DIR=$(pwd)
export UNI_OUTPUT_DIR=$UNI_INPUT_DIR/unpackage/dist/dev/mp-weixin
export VITE_ROOT_DIR=$UNI_INPUT_DIR

export UNI_CLI_CONTEXT=$HBUILDER_DIR/plugins/uniapp-cli-vite
export UNI_HBUILDERX_PLUGINS=$HBUILDER_DIR/plugins
export UNI_NPM_DIR=$HBUILDER_DIR/plugins/npm
export UNI_NODE_DIR=$HBUILDER_DIR/plugins/node

export UNI_COMPILER_VERSION_TYPE=r
export UNI_NODE_ENV=development
export UNI_UTS_PLATFORM=mp-weixin
export UNI_HBUILDERX_LANGID=zh_CN
export UNI_PLATFORM=mp-weixin
export VITE_USER_NODE_ENV=development
export NODE_ENV=development
export NODE_SKIP_PLATFORM_CHECK=1

export NODE="$UNI_NODE_DIR/node"

cd "$UNI_CLI_CONTEXT" || exit
clear

open -a "$WEIXIN_DEV_DIR" --args -o "$UNI_OUTPUT_DIR"

"$NODE" --max-old-space-size=2048 --no-warnings "$UNI_NPM_DIR/node_modules/npm/bin/npm-cli.js" run dev:weixin
```

## 使用流程

### 一、配置脚本

* 打开项目根目录下的`ws.bat`，修改`HBUILDER_DIR`和`WEIXIN_DEV_DIR`两个路径，**路径的末尾不要带斜杠。**
* 编辑HbuilderX安装目录下的`plugins\uniapp-cli-vite\package.json`文件，在`scripts`
  节点下增加一行:`"dev:weixin": "uni -p mp-weixin",`

### 二、配置Webstorm

* 设置-插件：安装WeChat mini program support
* 设置-语言和框架-WeChat Mini Program：小程序支持 => 启用

### 三、使用

用WebStorm打开该工程，新建一个内置终端，执行`./ws.bat`即可。

### 四、补充

* Vue相关函数缺少提示是因为Webstorm检测`package.json`下没有Vue3的依赖导致的，装一个就好。但不建议上传修改后的`package.json`和`package-lock.json`。
* 注意启动路径一定要在项目根目录下，因为脚本中关于编译及输出产物的项目路径变量`UNI_INPUT_DIR`被设置成了`%cd%`，即当前工作目录。如果害怕因粗心导致错误，可将`UNI_INPUT_DIR`设置为项目的绝对路径，如`set UNI_INPUT_DIR=C:\Users\XXX\Documents\xxx-wx-v2`。

