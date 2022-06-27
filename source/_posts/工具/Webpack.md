---
title: Webpack
date: {{ date }}
categories:
- 工具
- Webpack
---

# Webpack

## 1. npm的基本使用

**npm初始化** 

npm init -y

**npm安装jquery**

npm i jquery -S

-S： --save（保存）

包名会被注册在package.json的dependencies里面，在生产环境下这个包的依赖依然存在。

-D：--dev（开发）

包名会被注册在package.json的devDependencies里面，仅在开发环境下存在的包。

## 2 Webpack的基本使用

### 2.1 在项目中安装和配置Webpack

1. 安装webpack相关的包 

  npm i webpack webpack-cli -D

2. 在项目的根目录中，创建 webpack.config.js 配置文件

3. 在 webpack 配置文件中，初始化如下基本配置

```javascript
module.exports = {
 // mode 用来指定构建模式
 // 'development' 不压缩与混淆，速度快
 // 'production'  压缩与混淆，速度慢
 mode: 'development'
}
```

4. 在package.json配置文件中 scripts 节点下，新增 dev 脚本

```javascript
// 该节点下的脚本可以通过 npm run 运行
"scripts": {
 "dev": "webpack"
}
```

5. 终端运行 npm run dev，启动 webpack 进行打包

### 2.2 WebPack打包配置

webpack的4.x版本中默认约定

打包的入口文件为 src > index.js

打包的输出文件为 dist > main.js

如果要修改打包的入口与出口，可以在 webpack.config.js 中新增如下配置信息

```javascript
const path = require('path')    // 导入node.js中专门操作的模块
module.exports = {
    // 打包入口文件的路径，__dirname表示当前目录绝对路径
    entry: path.join(__dirname, './src/index.js'),
    output: {
        // 输出文件的存放路径
        path: path.join(__dirname, './dist'),
        // 输出文件的名称
        filename: 'bundle.js'
    }
}
```

### 2.3 WebPack自动打包

1. 安装自动打包工具

   npm i webpack-dev-server -D

2. 修改package.json

```javascript
"scripts": {
    "dev": "webpack-dev-server"
}
```

3. 修改引入的sciprt路径为  /bundle.js (该配置文件存在于内存中)

4. 运行 npm run dev

### 2.4 WebPack加载器

**1 通过 loader 打包非 js 模块**

webPack 默认只能打包 .js 后缀名结尾的模块，其它非 .js 后缀名结尾的模块需要调用 loader 才可以正常打包。

如: less-loader sass-loader url-loader

**2 打包处理 css文件**

1. 运行 npm i style-loader css-loader -D 命令，安装处理 css 文件的 loader

2. 在 webpack.config.js 的 module > rules 数组中， 添加规则如下

```javascript
module: {
    rules: [
        // test表示匹配文件的类型，use表示要调用的loader
        { test: /\.css$/, use: ['style-loader', 'css-loader']}
    ]
}
```

