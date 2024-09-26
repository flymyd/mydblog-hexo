---
title: 2、uniApp-Jenkins集成打包工具链的搭建（优化生产）
toc: true
date: 2023-12-04 00:00:00
tags: [Linux, uniApp, 剖析精研]
categories:
- [前端技术, uniApp移植Linux记录]
---
## 零、写在前面

HBuilderX是DCloud旗下的IDE产品，目前只提供了Windows和Mac版本使用。本项目组在开发阶段经常需要向测试环境提交热更新包，使用Jenkins进行CD是非常有必要的一步。尽管HBuilderX提供了CLI，但Jenkins服务通常都是搭建在Linux环境下的。当前的Uniapp wgt打包服务是使用了Windows Server + HBuilderX CLI的解决方案来进行打包，再用Jenkins远程调用接口。这套方案的弊病有如下几点：

* Jenkins侧仅负责少量参数的传递，如项目名、Git repo地址、分支名等，大部分流程不受控制，流水线的构建阶段显示不透明。
* 核心由一个shell script和一个python脚本实现，代码逻辑存在一定重复，维护难度也比较高。
* 从Git更新代码的流程耗费较长时间，因为每次执行流水线都要删除掉本地的repo并重新拉取。这对于带宽只有个位数的公网测试服务器来说是致命的，每次构建花费在此步的时间就有2分钟以上。
* 后端平台侧获取的Token没有缓存，即使打包提速了也会受到验证码获取1分钟节流的限制。
* Windows服务器上运行的HBuilderX经常出现登录态失效或启动打包任务失败的情况，测试在Jenkins侧只能得到简单的任务失败提示。而且这个提示是出现在Git拉取代码之后的，意味着每次失败前都要干等2分钟。

为了避免服务器资源的浪费，节省不必要的维护开支，我决定在等待测试的gap期研究一下这套流程的优（chong）化（gou）。

## 一、初次优化-Windows下脱离HBuilderX主程序

### 打包指令提取

首先从Windows端下手，先打开HBuilderX并登录账号，正常打一个wgt包。在打包前使用`DebugView`来查看HBuilderX执行任务时的输出，这是一款用于捕获Windows桌面系统程序中由TRACE和OutputDebugString输出的信息的工具。抓取到的有效日志如下：

```sh
[16764] 2023-05-16 09:50:20.583 [INFO:] node "D:/HBuilderX/plugins/node/node.exe"
[16764] 2023-05-16 09:50:20.583 [INFO:] args ("--max-old-space-size=2048", "--no-warnings", "D:/HBuilderX/plugins/uniapp-cli-vite/node_modules/@dcloudio/vite-plugin-uni/bin/uni.js")
```

盘一下`uni.js`

```javascript
require('../dist/cli/index.js')
```

可以看到备注很完善的函数`cli`，这证明Uniapp打包使用的工具就是由`node`构建的，原则上来说可以不受系统平台的限制，移植到Linux上使用是没有问题的。至于HBuilderX软件本体就是Qt写的一个壳子，只要剖析整个打包流程就可以脱离HBuilderX本体了。

```javascript
cli
    .command('build')
    .option('--outDir <dir>', `[string] output directory (default: dist)`)
    .option('--assetsInlineLimit <number>', `[number] static asset base64 inline threshold in bytes (default: 4096)`)
    .option('--sourcemap', `[boolean] output source maps for build (default: false)`)
    .option('--manifest', `[boolean] emit build manifest json`)
    .option('--ssrManifest', `[boolean] emit ssr manifest json`)
    .option('--emptyOutDir', `[boolean] force empty outDir when it's outside of root`, {
    default: true,
})
    .option('-w, --watch', `[boolean] rebuilds when modules have changed on disk`)
    .action(action_1.runBuild);
```

根据参数表整理一下调用命令：

```sh
node --max-old-space-size=2048 --no-warnings "D:/HBuilderX/plugins/uniapp-cli-vite/node_modules/@dcloudio/vite-plugin-uni/bin/uni.js" build --platform app --outDir "D:/test/crp-app-dist"
```

直接执行上述命令是不行的，因为uniapp的node_modules下根本就没有编译该工程所需的各种依赖，并且缺少很多环境变量。具体逻辑在`uniapp-cli-vite/node_modules/@dcloudio/vite-plugin-uni`的`/dist/cli/build.js`和`/dist/index.js`中都有所表现。

显而易见的是，我们的工程目录下（非cli模式）根本就没有`vue`、`pinia`之类的包，那么上面的命令是如何确定工作目录（工程目录）和其它依赖所在地的呢？先说其它依赖的问题，分析`plugins`目录可以看到HBuilderX自带了一套`node`和`npm`，那么去看一下`npm`的脚本（在Windows版本叫做`npm.cmd`）：

```sh
#!/bin/sh
(set -o igncr) 2>/dev/null && set -o igncr; # cygwin encoding fix
basepath=$(cd `dirname $0`; pwd)
plugin_dir=$(dirname $basepath)

which "node" >/dev/null 2>&1
if ! [ $? -eq 0 ]; then
  node_Path=$plugin_dir/node
  new_path=$PATH:$node_Path
  export PATH=$new_path 
fi
$basepath/node_modules/npm/bin/npm-cli.js $@
```

### 环境变量修补

HBuilderX在打包阶段会使用自己的`node`和`npm`，我们要做的就是修补***环境变量***使得这些工具都能找到正确的工作目录。

查看打包前的环境变量也很简单，既然调用栈是`uniapp-cli-vite/node_modules/@dcloudio/vite-plugin-uni/bin/uni.js`，那么直接在`uni.js`的尾部追加`console.log(process.env)`后再次在HBuilderX里执行打包，记录控制台输出并去除无关的环境变量即可。

以下是一个使用`node.js`成功在`Windows`下脱离`HbuilderX`主程序调用打包的例子：

  ```javascript
  /**
   * @author myd
   */
  import {exec} from "child_process";
  import * as util from "util";
  import path from "path";
  import os from "os";
  
  const execAsync = util.promisify(exec);
  export const build = (repoName: string) => {
    const systemTempFolderPath = os.tmpdir();
    return new Promise(async (resolve, reject) => {
      try {
        const HBUILDER_DIR = "D:\\HBuilderX";
        const UNI_INPUT_DIR = path.join(systemTempFolderPath, repoName);
        const VITE_ROOT_DIR = UNI_INPUT_DIR;
        const UNI_HBUILDERX_PLUGINS = path.join(HBUILDER_DIR, 'plugins');
        const UNI_CLI_CONTEXT = path.join(UNI_HBUILDERX_PLUGINS, 'uniapp-cli-vite');
        const UNI_NPM_DIR = path.join(UNI_HBUILDERX_PLUGINS, 'npm');
        const UNI_NODE_DIR = path.join(UNI_HBUILDERX_PLUGINS, 'node');
        const NODE_ENV: any = 'production';
        const NODE = path.join(UNI_NODE_DIR, 'node');
        const UNI_CLI = path.join(UNI_CLI_CONTEXT, 'node_modules', '@dcloudio', 'vite-plugin-uni', 'bin', 'uni.js');
        const PATH_ADDONS = process.env.PATH + `;${UNI_INPUT_DIR}/node_modules/.bin;`;
        const childEnv = {
          ...process.env,
          PATH: PATH_ADDONS,
          HBUILDER_DIR,
          UNI_INPUT_DIR,
          VITE_ROOT_DIR,
          UNI_CLI_CONTEXT,
          UNI_HBUILDERX_PLUGINS,
          UNI_NPM_DIR,
          UNI_NODE_DIR,
          NODE_ENV,
          NODE
        };
        process.chdir(UNI_CLI_CONTEXT);
        const buildCommand = `"${NODE}" --max-old-space-size=2048 --no-warnings "${UNI_CLI}" build --platform app --outDir ${path.join(systemTempFolderPath, repoName + '-dist')}`
        const {stdout, stderr} = await execAsync(buildCommand, {env: {...childEnv}});
        console.error('stderr:', stderr);
        resolve(1)
      } catch (error) {
        console.error('Error during build:', error);
        reject(0)
      }
    })
  }
  ```

产物是一个文件夹，把这个文件夹以`zip`格式压缩后，将后缀重命名为`wgt`即可。

做到这一步后，我就用`Next.js`写了一个简单的GUI，并配合一系列辅助逻辑完成了beta版的新构建平台，部署到先前的Windows服务器上提供给同事进行测试。这样做的目的主要是为了验证以上工作的正确性和稳定性，还可以从反馈意见中思考一下我对于该流程的重构设想是否正确，还有哪些点没有考虑到。

## 二、第二次优化-移植到Linux上并集成回Jenkins

### 思考

首先明确一点，这件事本身就是一个内部需求的解决方案延伸，闭门造车是不可取的。当做出一个阶段性的工作后，立刻部署demo并持续收集同事的反馈意见，及时调整，这样才能避免后续的更多问题，因为最终用户不止是我本人，还有所有参与项目的其他同事。软件的易用性和稳定性同样重要，必须在正式部署前反复地进行预先测试才能推行使用。

在测试了一天后，同事认为打包速度大大提高，但简陋的web控制台不能同时执行多个打包任务，打包期间不能刷新或离开页面也是硬伤。当时我还没有想好如何把这套代码搬到Linux上跑，因为这涉及HBuilderX的平台差异问题。将这套代码的构建核心抽出来集成回Jenkins是最佳选择，但我当时是有一些偷懒的想法的，因为这套轮子也同样提供了工程选择、Git同步的处理逻辑等，已经解决了上面的大部分痛点。但轮子毕竟是轮子，最终我还是决定放弃推行自己的平台给大家使用的想法，而是以这个平台作为测试工具，在此之上研究将HBuilderX的打包器迁移到Linux上的方法。

`uniapp-cli-vite`承担了大部分打包功能，按理说它作为一个`node`包本来是不挑系统的。但我陷入了思维上的误区，一定要迁移`node`和`npm`再给`plugins`搬家。实际上只要`node`环境隔离得当，只迁移`npm`即可。而对于`npm`来说，Linux和Windows的`npm`结构差异较大，但几乎可以直接使用macOS的包。所以迁移工作我选择在macOS平台下研究。

### 编写脚本&部署

明确一下思路，只要在macOS下写一个*接受工程目录的路径、打包产物的路径和HBuilderX plugins的路径，输出打包产物*的脚本并测试成功，那么就算成功了80%了。把上面的`node`函数用GPT转成sh脚本，自己再微调一下：

```sh
#!/bin/bash

# HBuilder目录修改此处
HBUILDER_DIR=/root/HBuilderX
NODE_ENV=production
repoDir=$1
# 导出目录修改此处
distExportDir=$2
# Nodejs修改此处
NODE=/root/HBuilderX/plugins/node/node

UNI_INPUT_DIR="$repoDir"
VITE_ROOT_DIR="$UNI_INPUT_DIR"
UNI_HBUILDERX_PLUGINS="$HBUILDER_DIR/plugins"
UNI_CLI_CONTEXT="$UNI_HBUILDERX_PLUGINS/uniapp-cli-vite"
UNI_NPM_DIR="$UNI_HBUILDERX_PLUGINS/npm"
UNI_NODE_DIR="$UNI_HBUILDERX_PLUGINS/node"
UNI_CLI="$UNI_CLI_CONTEXT/node_modules/@dcloudio/vite-plugin-uni/bin/uni.js"

export HBUILDER_DIR
export UNI_INPUT_DIR
export VITE_ROOT_DIR
export UNI_CLI_CONTEXT
export UNI_HBUILDERX_PLUGINS
export UNI_NPM_DIR
export UNI_NODE_DIR
export NODE_ENV
export NODE
export PATH="$PATH:$UNI_INPUT_DIR/node_modules/.bin"

cd "$UNI_CLI_CONTEXT"
buildCommand="$NODE --max-old-space-size=2048 --no-warnings $UNI_CLI build --platform app --outDir $distExportDir/${repoDir}-dist"
eval $buildCommand
exitCode=$?
if [ $exitCode -eq 0 ]; then
  echo "Build successful"
  exit 1
else
  echo "Error during build"
  exit 0
fi
```

效果很好，先把HBuilderX的主目录打包为tar并传到Linux服务器上展开：

```sh
tar -cf ~/HbuilderX-3.9.5-darwin.tar /Applications/HBuilderX.app/Contents/HBuilderX
scp -P 22 -r ~/HbuilderX-3.9.5-darwin.tar root@192.168.1.252:/root/
tar -xf ./HbuilderX-3.9.5-darwin.tar
```

在macOS机器上看一下`HBuilderX`使用的Node版本：

```sh
myd@myddeMac-Pro ~ % /Applications/HBuilderX.app/Contents/HBuilderX/plugins/node/node -v
v16.17.0
```

为Linux服务器下载对应系统和架构的Node二进制包并覆盖HBuilderX所使用的node，这里以`Linux-amd64-16.17.0`为例：

```sh
wget https://nodejs.org/download/release/v16.17.0/node-v16.17.0-linux-x64.tar.gz
tar -xzvf ./node-v16.17.0-linux-x64.tar.gz
cp ./node-v16.17.0-linux-x64/bin/node ./HBuilderX/plugins/node
chmod +x ~/HBuilderX/plugins/node/node
```

更多版本可以在`https://nodejs.org/download/release/`查看。

公司服务器的`Ubuntu 18.04`缺少node 18的依赖`glibc-2.28`，因此需要进一步对系统环境进行修补。编译`glibc-2.28`：

```sh
sudo apt-get install g++ make gcc bison
apt install -y gawk
cd ~
wget -c https://ftp.gnu.org/gnu/glibc/glibc-2.28.tar.gz
tar -zxf glibc-2.28.tar.gz
cd glibc-2.28
mkdir glibc-build
cd glibc-build
../configure --prefix=/opt/glibc-2.28
make -j 6
make install
cd ~
rm -rf ./glibc-2.28 ./glibc-2.28.tar.gz
apt install -y patchelf
# 直装的node使用如下命令
patchelf --set-interpreter /opt/glibc-2.28/lib/ld-linux-x86-64.so.2 --set-rpath /opt/glibc-2.28/lib/:/lib/x86_64-linux-gnu/:/usr/lib/x86_64-linux-gnu/ /usr/local/bin/node
# nvm使用如下命令
patchelf --set-interpreter /opt/glibc-2.28/lib/ld-linux-x86-64.so.2 --set-rpath /opt/glibc-2.28/lib/:/lib/x86_64-linux-gnu/:/usr/lib/x86_64-linux-gnu/ /root/.nvm/versions/node/v18.18.2/bin/node
# 记得修补HBuilderX的node
patchelf --set-interpreter /opt/glibc-2.28/lib/ld-linux-x86-64.so.2 --set-rpath /opt/glibc-2.28/lib/:/lib/x86_64-linux-gnu/:/usr/lib/x86_64-linux-gnu/ ~/HBuilderX/plugins/node/node
```

接下来修改`uniapp-cli-vite`的`package.json`

```sh
vi /root/HBuilderX/plugins/uniapp-cli-vite/package.json
```

删除`devDependencies`节点下的`@esbuild/darwin-arm64`、`@esbuild/darwin-x64`和`fsevents`，并安装对应目标平台的`esbuild`：

```sh
npm i -D -f @esbuild/linux-x64@0.17.19
npm i -f
```

领导非常支持这件事，抽出时间将上面的脚本集成回了Jenkins。我也将之前对于几个痛点的思考和解决方案提供了出来，至此整个wgt打包流程得到了巨大的优化，无论是速度还是稳定性。

## 三、第三次优化-封装Docker进行环境隔离

### 思考

在第二次优化的过程中，因为出了修补`glibc`这档子事，我意识到环境隔离非常重要。`Ubuntu 18.04`作为LTS版本，至今仍在广泛使用。这种不算特别老的系统尚且出现环境导致的兼容性问题，假如我要在一个使用`musl`库的发行版上部署，比如`Alpine Linux`或者`Gentoo Linux`的时候又该怎么办呢？

显然，解决这个问题的最好办法就是封装Docker镜像。

### 精简HBuilderX包

封装Docker镜像自然是产物体积越小越好。HBuilderX macOS版在安装若干打包所需依赖后，主目录膨胀到2个多G，这里的大多数文件都是用不上的。在macOS下复制一份HBuilderX主程序目录，开搞：

* 只有plugins文件夹需要保留，HBuilderX主程序及其相关的文件完全用不上。删之。
* plugins文件夹下的大部分包也用不上，比如为HBuilderX提供代码补全、语法检查之类的IDE所需包，或者项目中根本没有使用到的UTS相关包，统统删之。
* 每删几个包，就执行一遍之前的sh构建脚本，将HBuilderX目录指向当前魔改的目录下，进行可用性测试。
* 删到最后，只留下`about`、`complie-dart-sass`、`node`、`npm`、`uniapp-cli-vite`与`node_modules`这几个目录。
* 鲁迅说过，`node_modules`的体积比珠穆朗玛峰还要大，结构比马里亚纳海沟还要深。提取出`node_modules`目录下的`package.json`和`package-lock.json`，复制到`plugins`目录下，再将`node_modules`删之。
* 根据上文的步骤，修改一下`uniapp-cli-vite`的`package.json`。

精简以后压缩一下，原来2个多G的plugins只剩7.2M。将压缩包命名为`core-3.9.5.zip`备用。

### Docker封装前的思考和一些选择

既然要做Docker镜像，那么Docker镜像的系统就最好选个轻量点的。`Alpine Linux`只有50多M，采用`APK`包管理器，是一个非常理想的选择。然而必须注意的是，`Alpine Linux`的libc实现使用的是`musl`而非常用的`glibc`。但我们既然是为了封装HBuilderX的专属镜像，那nodejs版本理应和原环境一样，也就是`v16.17.0`以保障兼容性，但是`APK`包管理器只能下载latest版本的`nodejs`，`nodejs`又依赖于`glibc`。从源码编译不太现实，所以这里使用多阶段构建的方法，先通过`nodejs`官方提供的`node-alpine`来获取可用的`nodejs`二进制文件，再进行后续的操作。

至于封装好的镜像如何使用，我的解决方案是启动一个小型的`HTTP API`服务。为了减少不必要的依赖，这个服务也使用`node`来写。服务器应该向镜像挂载一个项目文件夹，通过API调用进行打包。Git代码同步之类的操作全部交给Jenkins侧执行，Docker容器只负责最核心的打包部分——如无必要，勿增实体。

### 编写Dockerfile

以下是开发阶段的Dockerfile：

```dockerfile
ARG NODE_VERSION=16.17.1
ARG ALPINE_VERSION=3.18
FROM node:${NODE_VERSION}-alpine AS node
ARG ALPINE_VERSION
FROM alpine:${ALPINE_VERSION}
ENV API_SERVER_URL=https://gitclone.com/github.com/hbuilderx-vanilla/api-server.git

# Set China APK Manager Mirrors
RUN echo "https://mirrors.aliyun.com/alpine/v3.18/main/" > /etc/apk/repositories \
    && echo "https://mirrors.aliyun.com/alpine/v3.18/community/" >> /etc/apk/repositories
RUN apk update && apk add --no-cache bash unzip wget git

# Install node-16.17.1
COPY --from=node /usr/lib /usr/lib
COPY --from=node /usr/local/lib /usr/local/lib
COPY --from=node /usr/local/include /usr/local/include
COPY --from=node /usr/local/bin /usr/local/bin

# Set China NPM Mirrors
RUN npm install -g yarn --force \ 
    && npm config set registry https://registry.npmmirror.com

# Inject HBuilderX 3.9.5 core
COPY core-3.9.5.zip /opt/
RUN unzip /opt/core-3.9.5.zip -d /opt/ \
    && rm /opt/core-3.9.5.zip && mkdir /projects

# Install HBuilderX core dependencies
COPY core-install.sh /root/
RUN chmod +x /root/core-install.sh
# Need manual run it if minimal version
# RUN /root/core-install.sh

# Install and start api server
WORKDIR /root
RUN git clone ${API_SERVER_URL} && \
    cd api-server && \
    npm i
EXPOSE 3000
CMD [ "node","/root/api-server/index.js" ]
```

我把`HTTP API`服务单独做了一个Git仓库出来，这样方便后续扩充功能和进行bugfix。在开发环境下，一些反向代理和镜像源的设置是必不可少的。为了减少Docker镜像体积，我选择在镜像运行后再让用户手动安装HBuilderX的巨型npm依赖。

### 容器的部署和初始化

容器启动示例：

```sh
docker run -d --restart=always -v /<user_name>/<projects_folder>:/projects -p 13300:3000 --name hbuilder-vanilla flymyd114/hbuilderx-vanilla:latest
```

* `/<user_name>/<projects_folder>`是本机的待打包工程父目录，你的所有工程均应处于该目录下，如`/Users/myd/projects`下有`hello-uniapp`文件夹。
* `13300`为建议的API端口映射点。

容器首次启动后，执行如下命令以初始化依赖：

```sh
docker exec -it <docker_id> /bin/sh
chmod +x /root/core-install.sh && /root/core-install.sh
exit
```

访问`http://127.0.0.1:13300`以检查API服务是否正确启动。

打包示例：

```sh
curl --location 'http://localhost:13300/build?project=crp-app'
```

产物将会在`/projects/crp-app/wgt-dist`中生成。

### 发布到Github上并编写workflows

可以看到，上面的容器已经被发布到了DockerHub。同样的，我也将仓库开源并编写了`workflows`以自动构建新版本的Docker镜像。以下是`docker-build.yml`：

```yaml
name: Build and Push Docker image

on:
  push:
    branches:
      - main
    tags:
      - 'v*'
    paths:
      - 'Dockerfile'
  workflow_dispatch:

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
    - name: Check out the repo
      uses: actions/checkout@v2

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v1

    - name: Log in to Docker Hub
      uses: docker/login-action@v1
      with:
        username: ${{ secrets.DOCKER_HUB_USERNAME }}
        password: ${{ secrets.DOCKER_HUB_ACCESS_TOKEN }}

    - name: Extract metadata (tags, labels) for Docker
      id: meta
      uses: docker/metadata-action@v3
      with:
        images: flymyd114/hbuilderx-vanilla
        tags: |
          type=semver,pattern={{version}}

    - name: Build and push Docker image
      uses: docker/build-push-action@v2
      with:
        context: .
        file: ./Dockerfile
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
```

当main分支上的commit被打上形如`v3.9.5`的Tag时即可触发`workflows`。当然，也可以手动触发这个构建。

注意，在Github上发布的`Dockerfile`应该去除镜像源和反代的部分，否则构建速度反而会变慢：

```dockerfile
ARG NODE_VERSION=16.17.1
ARG ALPINE_VERSION=3.18
FROM node:${NODE_VERSION}-alpine AS node
FROM alpine:${ALPINE_VERSION}
ENV API_SERVER_URL=https://github.com/hbuilderx-vanilla/api-server.git

RUN apk update && \
    apk add --no-cache bash unzip wget git && \
    rm -rf /var/cache/apk/*

COPY --from=node /usr/lib /usr/lib
COPY --from=node /usr/local/lib /usr/local/lib
COPY --from=node /usr/local/include /usr/local/include
COPY --from=node /usr/local/bin /usr/local/bin
RUN npm install -g yarn --force

COPY core-3.9.8.zip /opt/
RUN unzip /opt/core-3.9.8.zip -d /opt/ && \
    rm /opt/core-3.9.8.zip && \
    mkdir /projects

COPY core-install.sh /root/
RUN chmod +x /root/core-install.sh

# Need manual run it if minimal version
# RUN /root/core-install.sh

WORKDIR /root
RUN git clone ${API_SERVER_URL} && \
    cd api-server && \
    npm i
EXPOSE 3000
CMD [ "node","/root/api-server/index.js" ]
```

### 后续的版本升级

公司的项目目前统一使用HBuilderX 3.9.5的基座。但随着未来的升级，我们也需要及时更新打包核心以兼容高版本基座。天下没有不散的筵席，这篇文章既是我对这项工作的一个总结归纳，也是为后来人提供一个维护的文档和应对版本改变的思路。对于`plugins`文件夹下的依赖来说，我们只需要在一台电脑上将HBuilderX升级到想要的版本，然后提取里面的`package.json`进行依赖版本的替换就好。通常来说，要更新的`package.json`涉及`about`、`uniapp-cli-vite`和根目录。更新后重新将`plugins`放到`core`文件夹下打包，同步修改`Dockerfile`内的zip名即可。

以3.9.5升级至3.9.8举例，修改`core/plugins`内的如下文件：

```sh
about/package.json
uniapp-cli-vite/package.json
package.json
```

然后压缩`core`文件夹，重命名为`core-3.9.8.zip`。修改`Dockerfile`：

```dockerfile
COPY core-3.9.8.zip /opt/
RUN unzip /opt/core-3.9.8.zip -d /opt/ && \
    rm /opt/core-3.9.8.zip && \
    mkdir /projects
```

### Github项目地址

Docker项目：`https://github.com/hbuilderx-vanilla/docker`

API项目：`https://github.com/hbuilderx-vanilla/api-server`

## 联系我

邮箱：`flymyd@foxmail.com`

或在上方项目处提issue，`@flymyd`
