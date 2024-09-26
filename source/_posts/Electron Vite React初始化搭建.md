---
title: Electron Vite React初始化搭建
toc: true
date: 2023-05-22 00:00:00
tags: [Electron, Vite, React, Node.js]
categories:
  - [前端技术, 工程搭建]
---
### 用Vite初始化工程
```powershell
yarn create vite font-spider-shell --template react-ts
cd font-spider-shell
yarn install
```

### 安装electron及HMR相关包
```powershell
yarn add -D electron
yarn add -D @types/node
yarn add -D concurrently wait-on
yarn add -D cross-env electron-builder
```

### electron相关入口点及配置
新建electron文件夹  
  
main.js
```javascript
// 控制应用生命周期和创建原生浏览器窗口的模组
const { app, BrowserWindow } = require('electron')
const path = require('path')
const NODE_ENV = process.env.NODE_ENV

function createWindow() {
  // 创建浏览器窗口
  const mainWindow = new BrowserWindow({
    width: 1280,
    height: 720,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js')
    }
  })

  mainWindow.loadURL(
    NODE_ENV === 'development'
      ? 'http://localhost:11451'
      : `file://${path.join(__dirname, '../dist/index.html')}`
  );

  // 打开开发工具
  if (NODE_ENV === "development") {
    mainWindow.webContents.openDevTools()
  }
}

// 这段程序将会在 Electron 结束初始化
// 和创建浏览器窗口的时候调用
// 部分 API 在 ready 事件触发后才能使用。
app.whenReady().then(() => {
  createWindow()

  app.on('activate', function () {
    // 通常在 macOS 上，当点击 dock 中的应用程序图标时，如果没有其他
    // 打开的窗口，那么程序会重新创建一个窗口。
    if (BrowserWindow.getAllWindows().length === 0) createWindow()
  })
})

// 除了 macOS 外，当所有窗口都被关闭的时候退出程序。 因此，通常对程序和它们在
// 任务栏上的图标来说，应当保持活跃状态，直到用户使用 Cmd + Q 退出。
app.on('window-all-closed', function () {
  if (process.platform !== 'darwin') app.quit()
})
```

preload.js
```javascript
window.addEventListener('DOMContentLoaded', () => {
  const replaceText = (selector, text) => {
    const element = document.getElementById(selector)
    if (element) element.innerText = text
  }

  for (const dependency of ['chrome', 'node', 'electron']) {
    replaceText(`${dependency}-version`, process.versions[dependency])
  }
})
```

### 修改vite.config.ts
```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  base: './',
  plugins: [react()],
  server: {
    port: 11451
  }
})
```

### 修改package.json
添加属性`main`、`build`，修改`scripts`
```json
{
  "name": "font-spider-shell",
  "private": true,
  "version": "0.0.0",
  "main": "electron/main.js",
  "build": {
    "appId": "com.your-website.your-app",
    "productName": "ElectronApp",
    "copyright": "Copyright © 2021 <your-name>",
    "mac": {
      "category": "public.app-category.utilities"
    },
    "nsis": {
      "oneClick": false,
      "allowToChangeInstallationDirectory": true
    },
    "files": [
      "dist/**/*",
      "electron/**/*"
    ],
    "directories": {
      "buildResources": "assets",
      "output": "dist_electron"
    }
  },
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "electron": "wait-on tcp:11451 && cross-env NODE_ENV=development electron .",
    "electron:serve": "concurrently -k \"yarn dev\" \"yarn electron\"",
    "electron:build": "vite build && electron-builder"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "@types/node": "^18.15.10",
    "@types/react": "^18.0.28",
    "@types/react-dom": "^18.0.11",
    "@vitejs/plugin-react": "^3.1.0",
    "concurrently": "^7.6.0",
    "cross-env": "^7.0.3",
    "electron": "^23.2.0",
    "electron-builder": "^23.6.0",
    "typescript": "^4.9.3",
    "vite": "^4.2.0",
    "wait-on": "^7.0.1"
  }
}
```
