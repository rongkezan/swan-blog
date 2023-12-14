---
title: Vue3
date: {{ date }}
categories:
- vue
---

## 创建项目

包管理器：pnpm

```sh
npm install -g pnpm
```

创建项目

```sh
pnpm create vue
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/direct/5f9f6636423d46ba9a2b98a79ae842fe.png)

## 配置代码风格 Eslint

配置文件 `.eslintrc.cjs`

prettier 风格配置 https://prettier.io

推荐配置如下

```json
/* eslint-env node */
require('@rushstack/eslint-patch/modern-module-resolution')

module.exports = {
  root: true,
  extends: ['plugin:vue/vue3-essential', 'eslint:recommended', '@vue/eslint-config-prettier/skip-formatting'],
  parserOptions: {
    ecmaVersion: 'latest'
  },
  rules: {
    // 1. 禁用格式化插件prettier，并且关闭format on save
    // 2. 安装Eslint插件
    'prettier/prettier': [
      'warn',
      {
        singleQuote: true, // 单引号
        semi: false, // 无分号
        printWidth: 800, // 每行宽度至多800个字符
        trailingComma: 'none', // 不加对象|数组最后逗号
        endOfLine: 'auto' // 换行符不限制（win mac 不一致）
      }
    ],
    'vue/multi-word-component-names': [
      'warn',
      {
        ignores: ['index'] // vue组件由多个单词组成（忽略index.vue）
      }
    ],
    'vue/no-setup-props-destructure': ['off'], // 关闭props解构校验
    'no-undef': 'error' // 添加未定义变量错误
  }
}
```

在vscode的 `settings.json` 文件中增加两个配置

```json
"editor.codeActionsOnSave": {
    "source.fixAll": true
},
"editor.formatOnSave": false
```

## 配置提交前代码检查 Husky

### 配置全局校验

初始化git仓库

```sh
git init
```

初始化husky工具配置

```sh
pnpm dlx husky-init
pnpm install
```

修改 `.husky/pre-commit` 文件

```sh
# 将原来的 npm test 替换
pnpm lint
```

### 配置暂存区eslint校验（推荐）

安装依赖

```sh
pnpm i lint-staged -D
```

配置 `package.json`

```json
{
  "scripts": {
     ...
    "lint-staged": "lint-staged"
  },
  "lint-staged": {
    "*.{js,ts,vue}": [
      "eslint --fix"
    ]
  }
}
```

修改 `.husky/pre-commit` 文件

```sh
pnpm lint-staged
```

## 安装Sass

```sh
pnpm add sass -D
```

## Vue-router4

history模式：createWebHistory 地址栏不带#

hash模式：createWebHashHistory 地址栏带#\

import.meta.env.BASE_URL 默认是 / ，是基础地址，在后续访问时会自动拼上相应地址

```js
import { createRouter, createWebHistory } from 'vue-router'

const router = createRouter({
  // import.meta.env.BASE_URL 是 vite 的环境变量，在 vite.config.js 中配置
  history: createWebHistory(import.meta.env.BASE_URL),
  routes: []
})

export default router
```

 `vite.config.js` 配置 import.meta.env.BASE_URL，增加 `base` 参数，参考 https://vitejs.dev/guide/env-and-mode.html

```javascript
export default defineConfig({
  plugins: [vue()],
  base: 'app'
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url))
    }
  }
})
```





在Vue3的CompositionAPI中，获取路由对象方法是

```sh
import { useRoute, useRouter } from 'vue-router'
const router = useRouter()
const route = useRoute()

const goList = () => {
  console.log(route, router)
}
```

## element-plus
安装
```sh
pnpm install element-plus
```
配置按需自动导入
```sh
pnpm install -D unplugin-vue-components unplugin-auto-import
```
```js
// vite.config.ts
import { defineConfig } from 'vite'
import AutoImport from 'unplugin-auto-import/vite'
import Components from 'unplugin-vue-components/vite'
import { ElementPlusResolver } from 'unplugin-vue-components/resolvers'

export default defineConfig({
  // ...
  plugins: [
    // ...
    AutoImport({
      resolvers: [ElementPlusResolver()],
    }),
    Components({
      resolvers: [ElementPlusResolver()],
    }),
  ],
})
```

## Pinia
```sh
pnpm add pinia-plugin-persistedstate -D
```

```js
// main.js
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import piniaPluginPersistedstate from 'pinia-plugin-persistedstate'
import App from './App.vue'
import router from './router'
import '@/assets/main.scss'

const app = createApp(App)

app.use(createPinia().use(piniaPluginPersistedstate))
app.use(router)

app.mount('#app')
```

## Axios
```sh
pnpm add axios
```

### 图标库
```sh
pnpm i @element-plus/icons-vu
```