---
title: HbuilderX格式化scss、less文件
toc: true
date: 2022-08-09 00:00:00
tags: [HbuilderX]
categories:
  - [前端技术, uniApp]
---
HbuilderX自带了jsbeautify插件以进行代码格式化。然而当我们尝试新建外部CSS文件以分割过长组件代码时会发现，以scss或less为后缀的样式表文件无法正确地被格式化。解决该问题的方法有两种：

* 一、在DCloud插件市场下载[Prettier插件](https://ext.dcloud.net.cn/plugin?id=2025)

* 二、手动修改HbuilderX相关配置，使用自带的jsbeautify进行格式化

  由于HbuilderX可能会存在更新覆盖相关配置的可能性，所以这里编写node脚本进行批量修改。版本更新后若失效，重新执行即可。以Mac开发环境为例，Windows用户可自行修改路径设置。

  使用方式：

  ```shell
  node ~/enableCssFormater.js
  ```

  enableCssFormater.js代码：

  ```js
  //Mac用户请使用这两个变量，jsbeautifyrc注意将用户名替换为本机用户名
  const formatterPackage = "/Applications/HBuilderX.app/Contents/HBuilderX/plugins/format/package.json";
  const jsbeautifyrc = "/Users/用户名/Library/Application\ Support/HBuilder\ X/extensions/format/jsbeautifyrc.js";
  //Windows用户请使用这两个变量，也请注意修改相关路径
  const formatterPackage = "HbuilderX安装路径\plugins\format\package.json";
  const jsbeautifyrc = "用户目录下AppData\Roaming\HBuilder X\extensions\format\jsbeautifyrc.js";
  
  const fs = require('fs');
  try {
    console.log("Start writing HbuilderX plugins...")
    const package = fs.readFileSync(formatterPackage, "utf-8");
    const packageConfig = JSON.parse(package);
    let fileTypes = packageConfig.contributes.formator.filetypes;
    ["less", "scss"].forEach(type => {
      if (!fileTypes.includes(type)) {
        fileTypes.push(type)
      }
    })
    fs.writeFileSync(formatterPackage, JSON.stringify(packageConfig), "utf-8");
    console.log("Write HbuilderX plugins succeed.")
  } catch (err) {
    console.log(`Error writing HbuilderX Plugins: ${err}`);
  }
  try {
    console.log("Start writing JSBeautify...")
    const rc = require(jsbeautifyrc)
    let fileTypes = Object.keys(rc.parsers);
    [".less", ".scss"].forEach(type => {
      if (!fileTypes.includes(type)) {
        rc.parsers[type] = "css"
      }
    })
    fs.writeFileSync(jsbeautifyrc, "module.exports = " + JSON.stringify(rc), "utf-8");
    console.log("Write JSBeautify succeed.")
  } catch (err) {
    console.log(`Error writing JSBeautify: ${err}`);
  }
  ```

