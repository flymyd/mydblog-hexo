---
title: 1、在Ubuntu下生成uniApp Wgt包（初版研究）
toc: true
date: 2023-05-22 00:00:00
tags: [Linux, uniApp]
categories:
  - [前端技术, uniApp移植Linux记录]
---
剖析HBuilderX结构，完成打包环境的移植。

* 使用Debugview抓取HbuilderX打wgt包的调试信息，可得：

   ```sh
  [16764] 2023-05-16 09:50:20.583 [INFO:] node "D:/HBuilderX/plugins/node/node.exe"
  [16764] 2023-05-16 09:50:20.583 [INFO:] args ("--max-old-space-size=2048", "--no-warnings", "D:/HBuilderX/plugins/uniapp-cli-vite/node_modules/@dcloudio/vite-plugin-uni/bin/uni.js")
  ```

* 看一下这个`uni.js`：

  ```javascript
  require('../dist/cli/index.js')
  ```

* 跟着看一下`/dist/cli/index.js`，很明显是uniapp所使用的内部cli：

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

* 根据参数表整理一下调用命令：

  ```sh
  node --max-old-space-size=2048 --no-warnings "D:/HBuilderX/plugins/uniapp-cli-vite/node_modules/@dcloudio/vite-plugin-uni/bin/uni.js" build --platform app --outDir "D:/test/crp-app-dist"
  ```

* 直接执行上述命令是不行的，因为uniapp的node_modules下根本就没有编译该工程所需的各种依赖，并且缺少很多环境变量。具体逻辑在`/dist/cli/build.js`和`/dist/index.js`中都有所表现。

* 首先，需要使用`HbuilderX/plugins/npm`下的npm而非shell环境变量下的npm以便使用`HbuilderX/plugins`下的各种`node_modules`。

* 其次，需要在执行前对环境变量做修补。以下是一个使用`node.js`成功在`Windows`下脱离`HbuilderX`主程序调用打包的例子：

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

* 调试成功后，将HbuilderX的主目录打包为tar并传到Linux服务器上展开：

  ```sh
  tar -cf ~/HbuilderX-3.9.5-darwin.tar /Applications/HBuilderX.app/Contents/HBuilderX
  scp -P 22 -r ~/HbuilderX-3.9.5-darwin.tar root@192.168.1.252:/root/
  tar -xf ./HbuilderX-3.9.5-darwin.tar
  ```

  注意，为了移植到Linux服务器上，一定要提取**macOS版本的HbuilderX**，因为Windows和Unix-Like的npm结构不同，即使npm-cli的实现是跨平台的。

* `HBuilderX 3.9.5`使用的是`Node 16.17.0`。如果不确定Node版本，提取前在宿主机上查询`HBuilderX`使用的Node版本：

  ```sh
  myd@myddeMac-Pro ~ % /Applications/HBuilderX.app/Contents/HBuilderX/plugins/node/node -v
  v16.17.0
  ```

* 为Linux服务器下载对应系统和架构的Node二进制包并覆盖HBuilderX所使用的node，这里以`Linux-amd64-16.17.0`为例：

  ```sh
  wget https://nodejs.org/download/release/v16.17.0/node-v16.17.0-linux-x64.tar.gz
  tar -xzvf ./node-v16.17.0-linux-x64.tar.gz
  cp ./node-v16.17.0-linux-x64/bin/node ./HBuilderX/plugins/node
  chmod +x ~/HBuilderX/plugins/node/node
  ```

  更多版本可以在`https://nodejs.org/download/release/`查看。

* 部署uniapp-build项目。该项目的最低node版本为v18。

  ```sh
  git clone https://[保密涂抹]
  cd ./uniapp-build
  npm i
  npm run build
  npm run start
  ```
  
* 比较难绷的是，公司服务器的`Ubuntu 18.04`缺少node 18的依赖`glibc-2.28`，因此需要进一步对系统环境进行修补。编译`glibc-2.28`：

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

* 接下来修改`uniapp-cli-vite`的`package.json`

   ```sh
   vi /root/HBuilderX/plugins/uniapp-cli-vite/package.json
   ```

   删除`devDependencies`节点下的`@esbuild/darwin-arm64`、`@esbuild/darwin-x64`和`fsevents`并安装对应目标平台的`esbuild`

   ```sh
   npm i -D -f @esbuild/linux-x64@0.17.19
   npm i -f
   ```

* 访问`http://server.ip:3000`查看效果。注意初次拉代码时需要配置全局git用户并登录一次。

   ```sh
   git config --global credential.helper store
   ```

