### 前言

想必大多数人在开发 vue 等 SPA 项目都时候都会直接用 `vue-cli` 等脚手架开发，一是方便省去了好多配置上的功夫，二是 `vue-cli` 毕竟是久经考验较为成熟的东西，遇到问题也能在网上找到相应解决方案。

但是，如果我们要更好地理解脚手架的配置及其构建打包的机制，我们就有必要从零开始，依葫芦画瓢自己配置一个类似于 vue-cli 这样的项目了。在此，我做了以下简单配置，请各位大佬批评指正，并诚心希望能得到大佬的指点，解决文章最后关于 `Tree Shaking` 导致打包缺失 css 的问题。

本文主要包括如下配置：

* 区分环境变量及合并配置
* `webpack-dev-server` 本地服务器 
* `HtmlWebpackPlugin` 生成 html
* `CleanWebpackPlugin` 清理文件夹
* `loader` 及 `babel` 配置
* `HotModuleReplacementPlugin` 热更新
* `postcss-loader` 增加 css 前缀
* `vue SPA` 引入及解析
* `vue-router` 安装与使用
* `mini-css-extract-plugin` 分离 css
* `purifycss-webpack purify-css` 消除冗余 css
* `optimize-css-assets-webpack-plugin` 压缩 css
* `terser-webpack-plugin` 压缩 js
* `splitChunks` 提取公共代码
* `image-webpack-loader` 图片压缩
* `gZip` 加速优化

项目源码：https://github.com/Michael-lzg/webpack-vue-cli

如果对 webpack 基本配置还不了解的小伙伴，可查看以下文章  
[从零开始构建一个 webpack 项目](https://juejin.im/post/5db0fd1bf265da4d4216a9c5)  
[搭建一个 vue-cli4+webpack 移动端框架（开箱即用）](https://juejin.im/post/5eb766296fb9a0432f0ff8c7#heading-19)

废话不多说，老司机带你立刻上路。

### 搭建 webpack 项目框架

#### 构建项目结构

1. 创建 `webpack-vue-cli` 文件夹，`npm-init-y` 初始化项目

2. 安装 webpack 相关依赖

```
npm i webpack webpack-cli webpack-dev-server webpack-merge --save-dev
```

如果 `webpack` 和 `webpack-cli` 没有全局安装的话，要先全局安装

3. 建立项目文件夹

```js
├── src   // webpack配置文件
    |——main.js  // 入口文件
├── static   // 项目打包路径
├── index.html   // 模板html
├── webpack.base.js   // 打包基本配置
├── webpack.dev.js   // 本地环境配置
├── webpack.prod.js   // 生产环境配置
```

`index.html` 和 `main.js` 的代码不多说，直接进入 webpack 配置环节。

#### 区分环境

为了更好的优化打包，我们将 webpack 的配置分开开发环境和生产环境。

- webpack.base.js 公共配置文件
- webpack.dev.js 开发环境的配置文件
- webpack.prod.js 生产环境的配置文件

在 `webpack.dev.js` 和 `webpack.prod.js`，我们可以利用 `webpack-merge` 进行配置的合并。

然后，我们在 package.json 定义不同环境的打包命令

```js
"scripts": {
  "dev": "webpack-dev-server  --config webpack.dev.js --mode development",
  "build": "webpack --config webpack.prod.js"
}
```

#### 公共配置

我们先来看一下 `webpack.base.js` 的公共配置，定义好入口文件和出口文件

```js
module.exports = {
  entry: {
    index: path.join(__dirname, '/src/main.js'),
  },
  output: {
    path: path.join(__dirname, '/dist'), //打包后的文件存放的地方
    filename: 'js/[name].[hash].js', // 每次保存 hash 都变化
  },
}
```

#### webpack-dev-server

webpack 提供了一个可选的本地开发服务器，这个本地服务器基于 `node.js` 构建，所以在 `webpack.dev.js` 进行配置

```js
const merge = require('webpack-merge') // 引入webpack-merge功能模块
const common = require('./webpack.base.js') // 引入webpack.common.js

module.exports = merge(common, {
  // 将webpack.common.js合并到当前文件
  devServer: {
    contentBase: './dist', // 本地服务器所加载文件的目录
    port: '8899', // 设置端口号为8088
    inline: true, // 文件修改后实时刷新
    historyApiFallback: true, //不跳转
    hot: true, // 热更新
  },
  mode: 'development', // 设置mode
})
```

#### HtmlWebpackPlugin

`HtmlWebpackPlugin` 简化了 HTML 文件的创建，它可以根据 html 模板在打包后自动为你生产打包后的 html 文件。这对于在文件名中包含每次会随着编译而发生变化哈希的`bundle`。

```js
plugins: [
  new HtmlWebpackPlugin({
    template: path.join(__dirname, '/index.html'), // new一个这个插件的实例，并传入相关的参数
  }),
]
```

至此就搭建好一个乞丐版的 `webpack` 项目了，你可以随意编写代码，分别在开发环境和生产环境执行命令查看效果。

#### loader 配置

`loader` 可以让 `webpack` 能够去处理那些非 `javaScript` 文件（`webpack` 自身只理解 `javaScript`）。`loader` 可以将所有类型的文件转换为 webpack 能够处理的有效模块，然后你就可以利用 webpack 的打包能力，对它们进行处理。

对于 `loader` 的科普和配置，在这里不做一一说明，直接奉上代码，分别是处理样式，`js` 和文件的 `loader`。

```js
module: {
  rules: [
    {
      test: /\.css$/, // 正则匹配以.css结尾的文件
      use: ['style-loader', 'css-loader'],
    },
    {
      test: /\.less$/,
      use: ['style-loader', 'css-loader', 'less-loader'],
    },
    {
      test: /\.js$/,
      loader: 'babel-loader',
      include: [resolve('src')],
    },
    {
      test: /\.(png|jpe?g|gif|svg)(\?.*)?$/,
      loader: 'url-loader',
      options: {
        limit: 10000,
        name: utils.assetsPath('img/[name].[hash:7].[ext]'),
      },
    },
  ]
}
```

为了更方便的配置和优化 `babel-loader`，我们可以将其提取出来，在根目录下新建 `.babelrc` 文件

```
{
  "presets": ["env"]
}
```

#### CleanWebpackPlugin

在每次构建前清理/dist 文件夹，生产最新的打包文件，这时候就用到 `CleanWebpackPlugin` 插件了。

```js
plugins: [
  new HtmlWebpackPlugin({
    template: path.join(__dirname, '/index.html'), // new一个这个插件的实例，并传入相关的参数
  }),
  new CleanWebpackPlugin(), // 所要清理的文件夹名称
]
```

#### HotModuleReplacementPlugin

`HotModuleReplacementPlugin`（HMR）是一个很实用的插件，可以在我们修改代码后自动刷新预览效果，在开发环境使用。

1. `devServer` 配置项中设置 `hot: true`

2. `HotModuleReplacementPlugin` 是 webpack 模块自带的，所以引入 webpack 后，在 `plugins` 配置项中直接使用即可。

```js
plugins: [
  new webpack.HotModuleReplacementPlugin(), // 热更新插件
]
```

#### 增加 css 前缀

平时我们写 css 时，一些属性需要手动加上前缀，比如`-webkit-border-radius: 10px;`，在 webpack 中我们可以让他自动加上

1. 安装依赖

```
npm i postcss-loader autoprefixer -D
```

2. 在项目根目录下新建 `postcss.config.js` 文件

```js
module.exports = {
  plugins: [
    require('autoprefixer'), // 引用autoprefixer模块
  ],
}
```

3. 修改样式 loader

```js
rules: [
  {
    test: /\.css$/, // 正则匹配以.css结尾的文件
    use: ['style-loader', 'css-loader', 'postcss-loader'],
  },
  {
    test: /\.less$/,
    use: ['style-loader', 'css-loader', 'postcss-loader', 'less-loader'],
  },
]
```

至此，一个 webpack 项目基本搭建而成，下面介绍 vue 的引用和项目优化。

### 搭建 vue SPA 模板

#### vue SPA

1、搭建一个类似于 `vue-cli` 的脚手架，首先我们来依葫芦画瓢，在 `main.js` 写上一下代码

```js
import Vue from 'vue'
import App from './App.vue'
new Vue({
  el: '#app',
  render: (h) => h(App),
})
```

2、然后在 src 文件夹下新建 `APP.vue`

```html
<div id="app">SPA项目</div>
```

3、安装相关依赖

到这里，我们 `npm run dev` 试一下就报错了。因为我们没有安装相关依赖，下面我们下来安装一下依赖

```
npm install vue vue-loader vue-template-compiler -D
```

- vue: vue 的源码
- vue-loader：解析.vue 文件
- vue-template-compiler： 编译 vue

4、在 webpack 配置 vue-loader

```js
const VueLoaderPlugin = require('vue-loader/lib/plugin')
module.exports = {
  module: {
    rules: [
      {
        test: /\.vue$/,
        loader: 'vue-loader',
      },
    ],
  },
  plugins: [new VueLoaderPlugin()],
}
```

#### vue-router

1、安装依赖

```
npm install vue-router -D
```

2、在 src 文件夹下新建 router/index

```js
import Vue from 'vue'
import Router from 'vue-router'

Vue.use(Router)

const router = new Router({
  routes: [
    {
      path: '/',
      component: () => import('../views/Home.vue'),
    },
    {
      path: '/admin',
      component: () => import('../views/admin.vue'),
    },
  ],
})

export default router
```

3、在 main.js 引用

```js
import Vue from 'vue'
import App from './App.vue'
import router from './router'

new Vue({
  el: '#app',
  router,
  render: (h) => h(App),
})
```

就这样，一个类似于 `vue-cli` 的脚手架就搭建好了，你可以愉快地写 `.vue` 文件进行 SPA 开发。

### webpack 优化打包

#### 分离 css

虽然 webpack 的理念是把 css、js 全都打包到一个文件里，但要是我们想把 css 分离出来，这里我们用到 `mini-css-extract-plugin`。对比另一个插件 `extract-text-webpack-plugin`，它有以下优点:

- 异步加载
- 不重复编译，性能更好
- 更容易使用
- 只针对 CSS

但是`mini-css-extract-plugin` 不支持 `HMR`，所以我们只能在生产环境使用它。

1、安装依赖

```
npm install mini-css-extract-plugin -D
```

2、在`webpack.prod.js` 配置 `loader` 和 `plugin`

```js
module: {
  rules: [
    {
      test: /\.(le|c)ss$/,
      use: [
        {
          loader: MiniCssExtractPlugin.loader,
          options: {
            publicPath: '../'
          },
        },
        'css-loader',
        'postcss-loader',
        'less-loader',
      ]
    }
  ]
},
plugins: [
  new MiniCssExtractPlugin({
    filename: "css/[name].[contenthash:8].css",
    chunkFilename: 'css/[id].[contenthash:8].css'
  })
]
```

分离 css 需要将 `css loader` 中的 `style-loader` 替换为 `MiniCssExtractPlugin`

#### 消除冗余 css

有时候我们 css 写得多了或者重复了，这就造成了多余的代码，我们希望在生产环境进行去除。

1、安装依赖

```
npm i purifycss-webpack purify-css glob -D
```

2、webpack.prod.js 配置

```js
const path = require('path')
const PurifyCssWebpack = require('purifycss-webpack') // 引入PurifyCssWebpack插件
const glob = require('glob') // 引入glob模块,用于扫描全部html文件中所引用的css

module.exports = merge(common, {
  plugins: [
    new PurifyCssWebpack({
      paths: glob.sync(path.join(__dirname, 'src/*.html')),
    }),
  ],
})
```

#### 压缩 css

我们希望减小 css 打包后的体积，可以用到 `optimize-css-assets-webpack-plugin`。
1、安装依赖

```
npm install optimize-css-assets-webpack-plugin -D
```

2、webpack.prod.js 配置

```js
const OptimizeCSSAssetsPlugin = require("optimize-css-assets-webpack-plugin") // 压缩css代码

optimization: {
  minimizer: [
    // 压缩css
    new OptimizeCSSAssetsPlugin({})
  ]
```

#### 压缩 js

Webpack4.0 默认是使用 `terser-webpack-plugin` 这个压缩插件，在此之前是使用 `uglifyjs-webpack-plugin`，两者的区别是后者对 ES6 的压缩不是很好，同时我们可以开启 `parallel` 参数，使用多进程压缩，加快压缩。

1、安装依赖

```
npm install terser-webpack-plugin -D
```

2、webpack.prod.js 配置

```js
const TerserPlugin = require('terser-webpack-plugin') // 压缩js代码

optimization: {
  minimizer: [
    new TerserPlugin({
      parallel: 4, // 开启几个进程来处理压缩，默认是 os.cpus().length - 1
      cache: true, // 是否缓存
      sourceMap: false,
    }),
    // 压缩css
    new OptimizeCSSAssetsPlugin({}),
  ]
}
```

#### 提取公共代码

在用 webpack 打包的时候，对于一些不经常更新的第三方库，比如 vue 全家桶的一些东西， 我们希望能和自己的代码分离开。webpack4 使用 `splitChunks` 的方法进行配置。

```js
optimization: {
  // 分离chunks
  splitChunks: {
    chunks: 'all',
    cacheGroups: {
      vendor: {
        name: "vendor",
        test: /[\\/]node_modules[\\/]/,
        priority: 10,
        chunks: "initial" // 只打包初始时依赖的第三方
      },
    }
  }
}
```

#### 图片压缩

在项目中有些图片太大影响加载，我们用 `image-webpack-loader` 进行压缩。

1、安装依赖

```
npm install image-webpack-loader -D
```

2、配置 loader

```js
{
  test: /\.(png|jpg|svg|gif)$/,
  use: [
    {
      loader: 'url-loader',
      options: {
        esModule: false,
        limit: 1000,  // 限制只有小于1kb的图片才转为base64
        outputPath: 'images', // 设置打包后图片存放的文件夹名称
        name: '[name][hash:8].[ext]'
      }
    },
    {
      loader: 'image-webpack-loader',
      options: {
        // 压缩 jpeg 的配置
        mozjpeg: {
          progressive: true,
          quality: 65
        },
        // 使用 imagemin**-optipng 压缩 png，enable: false 为关闭
        optipng: {
          enabled: false,
        },
        // // 使用 imagemin-pngquant 压缩 png
        pngquant: {
          quality: [0.65, 0.90],
          speed: 4
        },
        // 压缩 gif 的配置
        gifsicle: {
          interlaced: false,
        },
        // 开启 webp，会把 jpg 和 png 图片压缩为 webp 格式
        webp: {
          quality: 75
        }
      }
    }
  ]
}
```

#### gZip 加速优化

所有现代浏览器都支持 `gzip` 压缩，启用 `gzip` 压缩可大幅缩减传输资源大小，从而缩短资源下载时间，减少首次白屏时间，提升用户体验。

gzip 对基于文本格式文件的压缩效果最好（如：CSS、JavaScript 和 HTML），在压缩较大文件时往往可实现高达 70-90% 的压缩率，对已经压缩过的资源（如：图片）进行 gzip 压缩处理，效果很不好。

```js
const CompressionPlugin = require('compression-webpack-plugin')
configureWebpack: (config) => {
  if (process.env.NODE_ENV === 'production') {
    config.plugins.push(
      new CompressionPlugin({
        // gzip压缩配置
        test: /\.js$|\.html$|\.css/, // 匹配文件名
        threshold: 10240, // 对超过10kb的数据进行压缩
        deleteOriginalAssets: false, // 是否删除原文件
      })
    )
  }
}
```

### tree-shaking（求助，生产环境打包生成不了样式）

按以上的方式构建项目，在开发环境一直都是顺顺利利的，然而一执行 `npm run build`，打开页面，发现样式全都缺失了。打开 `dist/css` 文件夹，发现三个 css 文件，只有 `index.css` 有部分文件（是 main.js 引入额初始化样式，但也是不全的），另外两个 css 文件则是空空如也，也就是.vue 里面的样式全都缺失了。

查看资源，初步判断为 webpack4 默认使用 `tree-shaking`，会把 **在模块的层面上做到打包后的代码只包含被引用并被执行的模块，而不被引用或不被执行的模块被删除掉，以起到减包的效果**。但是我已经按相关资源在 `package.json` 配置了 `sideEffects` 了，但是还是没用，实在苦恼！！！

```js
"sideEffects": [
    "*.less",
    "*.css",
    "*.vue"
  ]
```
