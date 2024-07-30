---
title: 记录 vue ssr 无法完整地注入页面所依赖的资源问题
date: 2021-06-21 18:36:00
tags:
---

**版本**

2.6.x

**可以复现的地址**

https://github.com/tcstory/vue-ssr-issue

**复现的步骤**

1. npm run build
2. npm run start
3. 分别在禁用和启动 js 的情况下, 访问 http://localhost:3000/page1, 你将会发现禁用 js 的情况下, html 文件中, 缺少某些 css 样式



## 这会导致什么问题呢?

其实, 是否禁用 js 来浏览页面不是问题的关键, 我们考虑下面的场景

1. 用户访问页面
2. 页面加载完毕后, 浏览器执行了 js, 然后注入 css
3. css 加载完毕后, 页面更新

主要在于, 在这个过程中, 用户能观察到页面在"抖动", 原因是一开始加载页面的时候, 缺少了某些 css 样式, 直到 js 执行后, 这个 css 样式才被加载回来.



## 个人分析的产生这个 bug 的原因

下面的分析, 会涉及到 webpack 相关的概念.

vue 在服务端渲染的过程中, 是通过 vue-loader 在组件内部生成一串 hash, 通过这一串的 hash, 在渲染过程中, vue 就能知道当前渲染的组件, 所属于哪一个 chunk, 然后, 再把这个 chunk 相关到的 files  给注入到 html 中

这个过程通常情况下, 是没有任何问题的, 除非遇到了 webpack 的 splitChunks.


我们先假设有 page1 和 page2 两个页面, 以及他们共同依赖的 btn1.vue 文件

1. page1.vue 依赖 btn1.vue
2. page2.vue 依赖 btn1.vue
3. btn1.vue 依赖了 btn1.css

通过构建, 你能拿到下面几个文件

1. page1.js
2. page2.js
3. page1~page2.js(这个文件包含了组件的 js 和 css 代码)

btn1.vue 组件被打包到了 page1~page2.js 中, 由于这个 js 文件包含了组件所需的 js 和 css 代码, 所以, 资源的注入是正常的


但是, 如果你多了一个 page3 页面, 并且, 这个页面也依赖了 btn1.css, 这个时候, 经过构建, 你将会拿到下面这些文件

1. page1.js
2. page2.js
3. page3.js
4. page1~page2.js (这个文件包含了 btn1.vue 的 js 代码)
5. page1~page2 ~page3.js (这个文件包含了 btn.css 的代码)

webpack 在打包过程当中, 首先生成了 page1~ page2 chunk, 然后, 它发现, 这个 chunk 和 page3 chunk 都共同依赖了 btn1.css 这个模块, 所以, webpack 就继续拆包, 最后就生成了 page1 ~page2 ~page3 chunk.

在 ssr 渲染过程中, vue 只能知道 btn1 被打包进入了  page1~page2.js, 但是却不知道 css 被抽离到别的文件中了. 导致 btn.css 不会被注入到 html 中.

其实这些被抽离出去的 chunk, 通过他的 sibling 也能拿到相关的信息, 只是即使有了这些信息也不好判断哪些 sibling 应该被注入到 html 中


issue: https://github.com/vuejs/vue/issues/12143

