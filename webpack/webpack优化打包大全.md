# webpack 优化打包大全

随着我们的项目项目越做越大，引入的第三方库会越来越多，打包的依赖也越来越多，每次 build 的时间越来越长，打包出来的文件会越来越大。最糟糕的是单页面应用首页白屏时间长，用户体验差。

此时优化 webpack 打包方法不可回避。下面我们来整理一下常用的 webpack 打包优化方法。

**我们的目的**

- 减小打包后的文件大小
- 首页按需引入文件，减少白屏时间
- 优化 webpack 打包时间

## 分析 webpack 打包性能瓶颈

首先我们来分析一下 webpack 打包性能瓶颈，找出问题所在，然后才能对症下药。

#### 1、webpack-bundle-analyzer 分析体积

- vue-cli3 需要安装依赖 `webpack-bundle-analyzer`

```js
npm install webpack-bundle-analyzer -D
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');
plugins:[
  new BundleAnalyzerPlugin(),
]
```

- vue-cli2 直接在命令行输入 `npm run build --report`, 构建完成后会在 8888 端口展示大小

<img src="../img/webpack2.png">

#### 2、测量构建时间

我们可以通过 `speed-measure-webpack-plugin` 测量你的 webpack 构建期间各个阶段花费的时间。

1. 步骤一：安装依赖包

```
npm install speed-measure-webpack-plugin --save-dev
```

2. 配置 vue.config.js

```js
// 分析打包时间
const SpeedMeasurePlugin = require('speed-measure-webpack-plugin')
const smp = new SpeedMeasurePlugin()
// ...
module.exports = {
  configureWebpack: smp.wrap({
    plugins: [new BundleAnalyzerPlugin()],
  }),
}
```

打包构建后会看到以下输出
<img src="../img/webpack1.png">

**找出问题所在后我们开始来总结一下优化方法。**

## 1、 按需加载

1.1 路由组件按需加载

```javascript
const router = [
  {
    path: '/index',
    component: (resolve) => require.ensure([], () => resolve(require('@/components/index'))),
  },
  {
    path: '/about',
    component: (resolve) => require.ensure([], () => resolve(require('@/components/about'))),
  },
]
```

1.2 第三方组件和插件。按需加载需引入第三方组件

```javascript
// 引入全部组件
import ElementUI from 'element-ui'
import 'element-ui/lib/theme-chalk/index.css'
Vue.use(ElementUI)

// 按需引入组件
import { Button } from 'element-ui'
Vue.component(Button.name, Button)
```

1.3 对于一些插件，如果只是在个别组件中用的到，也可以不要在 main.js 里面引入，而是在组件中按需引入

```javascript
// 在main.js引入
import Vue from vue
import Vuelidate from 'vuelidate'
Vue.use(Vuelidate)

// 按组件按需引入
import { Vuelidate } from 'vuelidate'
```

1.4 去除打包后文件的预加载 prefetch/preload

vuecli 3 默认开启 prefetch(预先加载模块)，提前获取用户未来可能会访问的内容，在首屏会把这十几个路由文件，都一口气下载了。所以我们要关闭这个功能

```js
//细节配置修改
chainWebpack: (config) => {
  // 移除 prefetch 插件
  config.plugins.delete('prefetch-index')
  // 移除 preload 插件
  config.plugins.delete('preload-index')
}
```

- preload 是告诉浏览器页面必定需要的资源，浏览器一定会加载这些资源
- prefetch 是告诉浏览器页面可能需要的资源，浏览器不一定会加载这些资源

当 prefetch 插件被禁用时，你可以通过 webpack 的内联注释手动选定要提前获取的代码区块：

```js
import(/* webpackPrefetch: true */ './someAsyncComponent.vue')
```

## 2、缩小构建目标

#### 优化 loader 配置

排除 Webpack 不需要解析的模块，即使用 loader 的时候，在尽量少的模块中去使用。

- 优化正则匹配
- 通过 `cacheDirectory` 选项开启缓存
- 通过 `include、exclude` 来减少被处理的文件。

```javascript
module: {
  rules: [
    {
      test: /\.js$/,
      loader: 'babel-loader?cacheDirectory',
      include: [resolve('src')],
    },
  ]
}
```

注意：保存和读取这些缓存文件会有一些时间开销，所以请只对性能开销较大的 loader 使用此 loader。

#### 合理使用 resolve.extensions

在导入语句没带文件后缀时，Webpack 会自动带上后缀后去尝试询问文件是否存在，查询的顺序是按照我们配置 的 resolve.extensions 顺序从前到后查找，Webpack 默认支持的后缀是 js 与 json。

#### 配置别名 alias

alias 的意思为 别名，能把原导入路径映射成一个新的导入路径，我们可以使用 alias 配置来减少查找过程。

```javascript
  resolve: {
    extensions: ['.js', '.vue', '.json'],
    alias: {
      'vue$': 'vue/dist/vue.esm.js',
      '@': resolve('src'),
    }
  },
```

#### 使用 module.noParse:

让 webpack 忽略对部分没采用模块化的文件的递归解析处理，这样做的好处是能提高构建性能。 因为如 `jQuery` 、`echart` 等库庞大又没有采用模块化标准，让 webpack 去解析这些文件耗时又没有意义。

```js
module:{
	noParse:/jquery/,//不去解析jquery中的依赖库
  ...
},
```

## 4、生产环境关闭 sourceMap

`sourceMap` 本质上是一种映射关系，打包出来的 js 文件中的代码可以映射到代码文件的具体位置,这种映射关系会帮助我们直接找到在源代码中的错误。
在生产环境，打包速度减慢，生产文件变大，所以开发环境使用 `sourceMap`，生产环境则关闭。

sourceMap 的种类

- source-map: 会生成 map 格式的文件，里面包含映射关系的代码
- inline-source-map: 不会生成 map 格式的文件，包含映射关系的代码会放在打包后生成的代码中
- inline-cheap-source-map: 一是将错误只定位到行，不定位到列。二是映射业务代码，不映射 loader 和第三方库等。会提升打包构建的速度。
- inline-cheap-module-source-map: module 会映射 loader 和第三方库
- eval: 用 eval 的方式生成映射关系代码，效率和性能最佳。但是当代码复杂时，提示信息可能不精确。

## 5、代码压缩

#### UglifyJS

`UglifyJS` 是 vue-cli 默认使用的压缩代码方式，它使用的是单线程压缩代码，打包时间较慢。

```js
plugins: [
  new UglifyJsPlugin({
    uglifyOptions: {
      compress: {
        warnings: false
      }
    },
    sourceMap: true,
    parallel: true
  })
```

#### ParallelUglifyPlugin

`ParallelUglifyPlugin` 开启多个子进程，把对多个文件压缩的工作分别给多个子进程去完成，每个子进程其实还是通过 `UglifyJS` 去压缩代码，但是变成了并行执行。

```javascript
plugins: [
  new ParallelUglifyPlugin({
    //缓存压缩后的结果，下次遇到一样的输入时直接从缓存中获取压缩后的结果并返回，
    //cacheDir 用于配置缓存存放的目录路径。
    cacheDir: '.cache/',
    sourceMap: true,
    uglifyJS: {
      output: {
        comments: false,
      },
      compress: {
        warnings: false,
      },
    },
  }),
]
```

打包速度和打包后的文件大小啊对比
| 方法 | 文件大小 | 打包速度 |
| ------------------ | :------- | :------- |
| 不用插件 | 14.6M | 32s |
| UglifyJsPlugin | 12.9M | 33s |
| ParallelUglifyPlugi | 7.98M | 17s |

#### terser-webpack-plugin

Webpack4.0 默认是使用 `terser-webpack-plugin` 这个压缩插件，在此之前是使用 `uglifyjs-webpack-plugin`，两者的区别是后者对 ES6 的压缩不是很好，同时我们可以开启 `parallel` 参数，使用多进程压缩，加快压缩。

```js
const TerserPlugin = require('terser-webpack-plugin') // 压缩js代码
const OptimizeCSSAssetsPlugin = require('optimize-css-assets-webpack-plugin') // 压缩css代码

optimization: {
  minimizer: [
    new TerserPlugin({
      parallel: 4, // 开启几个进程来处理压缩，默认是 os.cpus().length - 1
      cache: true, // 是否缓存
      sourceMap: false,
    }),
  ]
}
```

#### CSS 压缩

我们可以借助 optimize-css-assets-webpack-plugin 插件来压缩 css，其默认使用的压缩引擎是 cssnano

```js
const OptimizeCSSAssetsPlugin = require('optimize-css-assets-webpack-plugin') // 压缩css代码

optimization: {
  minimizer: [
    // 压缩css
    new OptimizeCSSAssetsPlugin({}),
  ]
}
```

## 6、提取公共代码

在用 webpack 打包的时候，对于一些不经常更新的第三方库，比如 `react，lodash，vue` 我们希望能和自己的代码分离开，webpack 社区有以下两种方案：

#### CommonsChunkPlugin 及 splitChunks

通过将公共模块拆出来，最终合成的文件能够在最开始的时候加载一次，便存到缓存中供后续使用。这个带来速度上的提升，因为浏览器会迅速将公共的代码从缓存中取出来，而不是每次访问一个新页面时，再去加载一个更大的文件。

webpack3 使用 `CommonsChunkPlugin` 的实现：

```javascript
plugins: [
  new webpack.optimize.CommonsChunkPlugin({
    name: 'vendor',
    minChunks: function (module, count) {
      console.log(module.resource, `引用次数${count}`)
      //"有正在处理文件" + "这个文件是 .js 后缀" + "这个文件是在 node_modules 中"
      return (
        module.resource &&
        /\.js$/.test(module.resource) &&
        module.resource.indexOf(path.join(__dirname, './node_modules')) === 0
      )
    },
  }),
  new webpack.optimize.CommonsChunkPlugin({
    name: 'common',
    chunks: 'initial',
    minChunks: 2,
  }),
]
```

webpack4 使用 `splitChunks` 的实现：

```javascript
module.exports = {
  optimization: {
    splitChunks: {
      cacheGroups: {
        vendor: {
          priority: 1, //添加权重
          test: /node_modules/, //把这个目录下符合下面几个条件的库抽离出来
          chunks: 'initial', //刚开始就要抽离
          minChunks: 2, //重复2次使用的时候需要抽离出来
        },
        common: {
          //公共的模块
          chunks: 'initial',
          minChunks: 2,
        },
      },
    },
  },
}
```

#### DLLPlugin

webpack.DllPlugin 就是来解决这个问题的插件，使用它可以在第一次编译打包后就生成一份不变的代码供其他模块引用，这样下一次构建的时候就可以节省开发时编译打包的时间。

1、在 build 下创建 `webpack.dll.config.js`

```js
const path = require('path')
const webpack = require('webpack')
module.exports = {
  entry: {
    vendor: [
      'vue-router',
      'vuex',
      'vue/dist/vue.common.js',
      'vue/dist/vue.js',
      'vue-loader/lib/component-normalizer.js',
      'vue',
      'axios',
      'echarts',
    ],
  },
  output: {
    path: path.resolve('./dist'),
    filename: '[name].dll.js',
    library: '[name]_library',
  },
  plugins: [
    new webpack.DllPlugin({
      path: path.resolve('./dist', '[name]-manifest.json'),
      name: '[name]_library',
    }),
    // 建议加上代码压缩插件，否则dll包会比较大。
    new webpack.optimize.UglifyJsPlugin({
      compress: {
        warnings: false,
      },
    }),
  ],
}
```

- `library` 的意思其实就是将 dll 文件以一个全局变量的形式导出出去，便于接下来引用。
- `mainfest.json` 文件是一个映射关系，它的作用就是帮助 webpack 使用我们之前打包好的 `***.dll.js` 文件，而不是重新再去 `node_modules` 中去寻找。

2、在 `webpack.prod.conf.js` 的 plugin 后面加入配置

```js
new webpack.DllReferencePlugin({
  manifest: require('../dist/vendor-manifest.json'),
})
```

3、`package.json`文件中添加快捷命令`(build:dll)`

```js
  "scripts": {
    "dev": "webpack-dev-server --inline --progress --config build/webpack.dev.conf.js",
    "start": "npm run dev",
    "lint": "eslint --ext .js,.vue src",
    "build": "node build/build.js",
    "build:dll": "webpack --config build/webpack.dll.conf.js"
  }
```

生产环境打包的时候先`npm run build:dll`命令会在打包目录下生成 `vendor-manifest.json` 文件与 vendor.dll.js 文件。然后`npm run build`生产其他文件。

4、根目录下的入口 `index.html` 加入引用

```html
<script type="text/javascript" src="./vendor.dll.js"></script>
```

## 7、CDN 优化

- 随着项目越做越大，依赖的第三方 npm 包越来越多，构建之后的文件也会越来越大。
- 再加上又是单页应用，这就会导致在网速较慢或者服务器带宽有限的情况出现长时间的白屏。

1、将 `vue、vue-router、vuex、element-ui 和 axios` 这五个库，全部改为通过 CDN 链接获取，在 `index.html` 里插入 相应链接。

```html
<head>
  <link rel="stylesheet" href="https://cdn.bootcss.com/element-ui/2.0.7/theme-chalk/index.css" />
</head>
<body>
  <div id="app"></div>
  <script src="https://cdn.bootcss.com/vue/2.6.10/vue.min.js"></script>
  <script src="https://cdn.bootcss.com/axios/0.19.0-beta.1/axios.min.js"></script>
  <script src="https://cdn.bootcss.com/vuex/3.1.0/vuex.min.js"></script>
  <script src="https://cdn.bootcss.com/vue-router/3.0.2/vue-router.min.js"></script>
  <script src="https://cdn.bootcss.com/element-ui/2.6.1/index.js"></script>
  <!-- built files will be auto injected -->
</body>
```

2、在 `webpack.config.js` 配置文件

```javascript
module.exports = {
 ···
    externals: {
      'vue': 'Vue',
      'vuex': 'Vuex',
      'vue-router': 'VueRouter',
      'element-ui': 'ELEMENT',
      'Axios':'axios'
    }
  },
```

3、卸载依赖的 npm 包

```
npm uninstall axios element-ui vue vue-router vuex
```

4、修改 `main.js` 文件里之前的引包方式

```js
// import Vue from 'vue'
// import ElementUI from 'element-ui'
// import 'element-ui/lib/theme-chalk/index.css'
// import VueRouter from 'vue-router'

import App from './App.vue'
import routes from './router'
import utils from './utils/Utils'

Vue.use(ELEMENT)
Vue.use(VueRouter)

const router = new VueRouter({
  mode: 'hash', //路由的模式
  routes,
})

new Vue({
  router,
  el: '#app',
  render: (h) => h(App),
})
```

#### html-webpack-externals-plugin

这种方法每次都需要在 index.html 模板中手动引入需要的 cdn 文件，然后还要在 webpack 里配置，有点繁琐了
`html-webpack-externals-plugin`这样的插件就应运而生了。

```js
// webpack.config.js文件
const HtmlWebpackExternalsPlugin = require('html-webpack-externals-plugin')

module.exports = {
  plugins: [
    new HtmlWebpackExternalsPlugin({
      externals: [
        {
          // 引入的模块
          module: 'jquery',
          // cdn的地址
          entry: 'https://cdn.bootcss.com/jquery/3.4.1/jquery.min.js',
          // 挂载到了window上的名称
          global: 'jQuery',
        },
        {
          module: 'vue',
          entry: 'https://cdn.bootcss.com/vue/2.6.10/vue.min.js',
          global: 'Vue',
        },
      ],
    }),
  ],
}
```

## 8、多进程解析和处理文件

由于运行在 Node.js 之上的 webpack 是单线程模型的，所以 webpack 需要处理的事情需要一件一件的做，不能多件事一起做。当 webpack 需要打包大量文件时，打包时间就会比较漫长。

以下两个方法能让 webpack 在同一时刻处理多个任务发挥多核 CPU 电脑的功能，提升构建速度。

#### thread loader

把这个 `thread loader` 放置在其他 loader 之前， 放置在这个 loader 之后的 loader 就会在一个单独的 worker 池(worker pool)中运行。

在 worker 池(worker pool)中运行的 loader 是受到限制的。例如：

- 这些 loader 不能产生新的文件。
- 这些 loader 不能使用定制的 loader API（也就是说，通过插件）。
- 这些 loader 无法获取 webpack 的选项设置。

每个 worker 都是一个单独的有 600ms 限制的 node.js 进程。同时跨进程的数据交换也会被限制。

```js
module.exports = {
  module: {
    rules: [
      {
        test: /\.js$/,
        include: path.resolve('src'),
        use: ['thread-loader', 'expensive-loader'],
      },
    ],
  },
}
```

#### HappyPack

`HappyPack` 能让 webpack 把任务分解给多个子进程去并发的执行，子进程处理完后再把结果发送给主进程。要注意的是 HappyPack 对 file-loader、url-loader 支持的不友好，所以不建议对该 loader 使用。

使用方法如下：

1、 HappyPack 插件安装

```
npm i -D happypack
```

2、 `webpack.base.conf.js` 文件对 module.rules 进行配置

```javascript
module: {
  rules: [
    {
      test: /\.js$/,
      use: ['happypack/loader?id=babel'],
      include: [resolve('src'), resolve('test')],
      exclude: path.resolve(__dirname, 'node_modules'),
    },
    {
      test: /\.vue$/,
      use: ['happypack/loader?id=vue'],
    },
  ]
}
```

3、在生产环境 `webpack.prod.conf.js` 文件进行配置

```javascript
const HappyPack = require('happypack')
// 构造出共享进程池，在进程池中包含5个子进程
const HappyPackThreadPool = HappyPack.ThreadPool({ size: 5 })
plugins: [
  new HappyPack({
    // 用唯一的标识符id，来代表当前的HappyPack是用来处理一类特定的文件
    id: 'babel',
    // 如何处理.js文件，用法和Loader配置中一样
    loaders: ['babel-loader?cacheDirectory'],
    threadPool: HappyPackThreadPool,
  }),
  new HappyPack({
    id: 'vue', // 用唯一的标识符id，来代表当前的HappyPack是用来处理一类特定的文件
    loaders: [
      {
        loader: 'vue-loader',
        options: vueLoaderConfig,
      },
    ],
    threadPool: HappyPackThreadPool,
  }),
]
```

**注意，当项目较小时，多线程打包反而会使打包速度变慢。**

## 启用 gzip 压缩

使用 Gzip 两个明显的好处，一是可以减少存储空间，二是通过网络传输文件时，可以减少传输的时间。

1、安装依赖

```js
npm i compression-webpack-plugin --save
```

2、在 vue.congig.js 中引入并修改 webpack 配置

```js
const CompressionPlugin = require('compression-webpack-plugin')
module.exports = {
  configureWebpack: (config) => {
    if (progress.env.NODE_ENV === 'production') {
      return {
        plugins: [
          new CompressionPlugin({
            test: /\.js$|\.html$|.\css/, //匹配文件名
            threshold: 10240, //对超过10k的数据压缩
            deleteOriginalAssets: false, //不删除源文件
          }),
        ],
      }
    }
  },
}
```

### 总结

1. 比较实用的方法: 按需加载，优化 loader 配置，关闭生产环境的 sourceMap，CDN 优化。
2. vue-cli 已做的优化： 代码压缩，提取公共代码，再优化空间不大。
3. 根据项目实际需要和自身开发水平选择优化方法，必须避免因为优化产生 bug。
