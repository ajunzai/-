# webpack 插件原理

在 webpack 中，专注于处理 webpack 在编译过程中的某个特定的任务的功能模块，可以称为插件。它和 `loader` 有以下区别：

1. `loader` 是一个转换器，将 A 文件进行编译成 B 文件，比如：将 `A.less` 转换为 `A.css`，单纯的文件转换过程。webpack 自身只支持 js 和 json 这两种格式的文件，对于其他文件需要通过 `loader` 将其转换为 commonJS 规范的文件后，webpack 才能解析到。

2. `plugin` 是一个扩展器，它丰富了 webpack 本身，针对是 `loader` 结束后，webpack 打包的整个过程，它并不直接操作文件，而是基于事件机制工作，会监听 webpack 打包过程中的某些节点，执行广泛的任务。

### plugin 的特征

webpack 插件有以下特征

- 是一个独立的模块。
- 模块对外暴露一个 js 函数。
- 函数的原型 `(prototype)` 上定义了一个注入 `compiler` 对象的 `apply` 方法。
- `apply` 函数中需要有通过 `compiler` 对象挂载的 webpack 事件钩子，钩子的回调中能拿到当前编译的 `compilation` 对象，如果是异步编译插件的话可以拿到回调 callback。
- 完成自定义子编译流程并处理 `complition` 对象的内部数据。
- 如果异步编译插件的话，数据处理完成后执行 callback 回调。

```js
class HelloPlugin {
  // 在构造函数中获取用户给该插件传入的配置
  constructor(options) {}
  // Webpack 会调用 HelloPlugin 实例的 apply 方法给插件实例传入 compiler 对象
  apply(compiler) {
    // 在emit阶段插入钩子函数，用于特定时机处理额外的逻辑；
    compiler.hooks.emit.tap('HelloPlugin', (compilation) => {
      // 在功能流程完成后可以调用 webpack 提供的回调函数；
    })
    // 如果事件是异步的，会带两个参数，第二个参数为回调函数，
    compiler.plugin('emit', function (compilation, callback) {
      // 处理完毕后执行 callback 以通知 Webpack
      // 如果不执行 callback，运行流程将会一直卡在这不往下执行
      callback()
    })
  }
}

module.exports = HelloPlugin
```

1. webpack 读取配置的过程中会先执行 `new HelloPlugin(options)` 初始化一个 `HelloPlugin` 获得其实例。
2. 初始化 `compiler` 对象后调用 `HelloPlugin.apply(compiler)` 给插件实例传入 compiler 对象。
3. 插件实例在获取到 `compiler` 对象后，就可以通过 `compiler.plugin` (事件名称, 回调函数) 监听到 Webpack 广播出来的事件。 并且可以通过 `compiler` 对象去操作 Webpack。

### 事件流机制

webpack 本质上是一种事件流的机制，它的工作流程就是将各个插件串联起来，而实现这一切的核心就是 `Tapable`。

Webpack 的 `Tapable` 事件流机制保证了插件的有序性，将各个插件串联起来， Webpack 在运行过程中会广播事件，插件只需要监听它所关心的事件，就能加入到这条 webapck 机制中，去改变 webapck 的运作，使得整个系统扩展性良好。

`Tapable` 也是一个小型的 library，是 Webpack 的一个核心工具。类似于 node 中的 events 库，核心原理就是一个`订阅发布模式`。作用是提供类似的插件接口。方法如下：

```js
//  广播事件
compiler.apply('event-name', params)
compilation.apply('event-name', params)

// 监听事件
compiler.plugin('event-name', function (params) {})
compilation.plugin('event-name', function (params) {})
```

我们来看下 Tapable

```js
function Tapable() {
  this._plugins = {}
}
//发布name消息
Tapable.prototype.applyPlugins = function applyPlugins(name) {
  if (!this._plugins[name]) return
  var args = Array.prototype.slice.call(arguments, 1)
  var plugins = this._plugins[name]
  for (var i = 0; i < plugins.length; i++) {
    plugins[i].apply(this, args)
  }
}
// fn订阅name消息
Tapable.prototype.plugin = function plugin(name, fn) {
  if (!this._plugins[name]) {
    this._plugins[name] = [fn]
  } else {
    this._plugins[name].push(fn)
  }
}
//给定一个插件数组，对其中的每一个插件调用插件自身的apply方法注册插件
Tapable.prototype.apply = function apply() {
  for (var i = 0; i < arguments.length; i++) {
    arguments[i].apply(this)
  }
}
```

`Tapable` 为 webpack 提供了统一的插件接口（钩子）类型定义，它是 webpack 的核心功能库。webpack 中目前有十种 hooks，在 Tapable 源码中可以看到，他们是：

```js
exports.SyncHook = require('./SyncHook')
exports.SyncBailHook = require('./SyncBailHook')
exports.SyncWaterfallHook = require('./SyncWaterfallHook')
exports.SyncLoopHook = require('./SyncLoopHook')
exports.AsyncParallelHook = require('./AsyncParallelHook')
exports.AsyncParallelBailHook = require('./AsyncParallelBailHook')
exports.AsyncSeriesHook = require('./AsyncSeriesHook')
exports.AsyncSeriesBailHook = require('./AsyncSeriesBailHook')
exports.AsyncSeriesLoopHook = require('./AsyncSeriesLoopHook')
exports.AsyncSeriesWaterfallHook = require('./AsyncSeriesWaterfallHook')
```

`Tapable` 还统一暴露了三个方法给插件，用于注入不同类型的自定义构建行为：

- tap：可以注册同步钩子和异步钩子。
- tapAsync：回调方式注册异步钩子。
- tapPromise：Promise 方式注册异步钩子。

webpack 里的几个非常重要的对象，`Compiler`, `Compilation` 和 `JavascriptParser` 都继承了 `Tapable` 类，它们身上挂着丰富的钩子。

### 编写一个插件

一个 webpack 插件由以下组成：

- 一个 JavaScript 命名函数。
- 在插件函数的 prototype 上定义一个 apply 方法。
- 指定一个绑定到 webpack 自身的事件钩子。
- 处理 webpack 内部实例的特定数据。
- 功能完成后调用 webpack 提供的回调。

下面实现一个最简单的插件

```js
class WebpackPlugin1 {
  constructor(options) {
    this.options = options
  }
  apply(compiler) {
    compiler.hooks.done.tap('MYWebpackPlugin', () => {
      console.log(this.options)
    })
  }
}

module.exports = WebpackPlugin1
```

然后在 webpack 的配置中注册使用就行，只需要在 `webpack.config.js` 里引入并实例化就可以了：

```js
const WebpackPlugin1 = require('./src/plugin/plugin1')

module.exports = {
  entry: {
    index: path.join(__dirname, '/src/main.js'),
  },
  output: {
    path: path.join(__dirname, '/dist'),
    filename: 'index.js',
  },
  plugins: [new WebpackPlugin1({ msg: 'hello world' })],
}
```

此时我们执行一下 `npm run build` 就能看到效果了

<img src="../img/plugin1.png">

### Compiler 对象 （负责编译）

`Compiler` 对象包含了当前运行 Webpack 的配置，包括 `entry`、`output`、`loaders` 等配置，这个对象在启动 Webpack 时被实例化，而且是全局唯一的。`Plugin` 可以通过该对象获取到 Webpack 的配置信息进行处理。

compiler 上暴露的一些常用的钩子：

<img src="../img/compiler.png">

下面来举个例子

```js
class WebpackPlugin2 {
  constructor(options) {
    this.options = options
  }
  apply(compiler) {
    compiler.hooks.run.tap('run', () => {
      console.log('开始编译...')
    })

    compiler.hooks.compile.tap('compile', () => {
      console.log('compile')
    })

    compiler.hooks.done.tap('compilation', () => {
      console.log('compilation')
    })
  }
}

module.exports = WebpackPlugin2
```

此时我们执行一下 `npm run build` 就能看到效果了

<img src="../img/plugin2.png">

有一些编译插件中的步骤是异步的，这样就需要额外传入一个 callback 回调函数，并且在插件运行结束时执行这个回调函数

```js
class WebpackPlugin2 {
  constructor(options) {
    this.options = options
  }
  apply(compiler) {
    compiler.hooks.beforeCompile.tapAsync('compilation', (compilation, cb) => {
      setTimeout(() => {
        console.log('编译中...')
        cb()
      }, 1000)
    })
  }
}

module.exports = WebpackPlugin2
```

### Compilation 对象

`Compilation` 对象代表了一次资源版本构建。当运行 webpack 开发环境中间件时，每当检测到一个文件变化，就会创建一个新的 `compilation`，从而生成一组新的编译资源。一个 `Compilation` 对象表现了当前的模块资源、编译生成资源、变化的文件、以及被跟踪依赖的状态信息，简单来讲就是把本次打包编译的内容存到内存里。`Compilation` 对象也提供了插件需要自定义功能的回调，以供插件做自定义处理时选择使用拓展。

简单来说，`Compilation` 的职责就是构建模块和 Chunk，并利用插件优化构建过程。

`Compiler` 代表了整个 Webpack 从启动到关闭的生命周期，而 `Compilation` 只是代表了一次新的编译，只要文件有改动，`compilation` 就会被重新创建。

`Compilation` 上暴露的一些常用的钩子：
<img src="../img/compilation.png">

`Compiler` 和 `Compilation` 的区别

- `Compiler` 代表了整个 Webpack 从启动到关闭的生命周期
- `Compilation` 只是代表了一次新的编译，只要文件有改动，`compilation` 就会被重新创建。

### 手写插件 1：文件清单

在每次 webpack 打包之后，自动产生一个一个 markdown 文件清单，记录打包之后的文件夹 dist 里所有的文件的一些信息。

思路：

1. 通过 `compiler.hooks.emit.tapAsync()` 来触发生成资源到 output 目录之前的钩子
2. 通过 `compilation.assets` 获取文件数量
3. 定义 markdown 文件的内容，将文件信息写入 markdown 文件内
4. 给 dist 文件夹里添加一个资源名称为 fileListName 的变量
5. 写入资源的内容和文件大小
6. 执行回调，让 webpack 继续执行

```js
class FileListPlugin {
  constructor(options) {
    // 获取插件配置项
    this.filename = options && options.filename ? options.filename : 'FILELIST.md'
  }

  apply(compiler) {
    // 注册 compiler 上的 emit 钩子
    compiler.hooks.emit.tapAsync('FileListPlugin', (compilation, cb) => {
      // 通过 compilation.assets 获取文件数量
      let len = Object.keys(compilation.assets).length

      // 添加统计信息
      let content = `# ${len} file${len > 1 ? 's' : ''} emitted by webpack\n\n`

      // 通过 compilation.assets 获取文件名列表
      for (let filename in compilation.assets) {
        content += `- ${filename}\n`
      }

      // 往 compilation.assets 中添加清单文件
      compilation.assets[this.filename] = {
        // 写入新文件的内容
        source: function () {
          return content
        },
        // 新文件大小（给 webapck 输出展示用）
        size: function () {
          return content.length
        },
      }

      // 执行回调，让 webpack 继续执行
      cb()
    })
  }
}

module.exports = FileListPlugin
```

### 手写插件 2：去除注释

开发一个插件能够去除打包后代码的注释，这样我们的 `bundle.js` 将更容易阅读

思路：

1. 通过 `compiler.hooks.emit.tap()` 来触发生成文件后的钩子
2. 通过 `compilation.assets` 拿到生产后的文件，然后去遍历各个文件
3. 通过 `.source()` 获取构建产物的文本，然后用正则去 replace 调注释的代码
4. 更新构建产物对象
5. 执行回调，让 webpack 继续执行

```js
class RemoveCommentPlugin {
  constructor(options) {
    this.options = options
  }
  apply(compiler) {
    // 去除注释正则
    const reg = /("([^\\\"]*(\\.)?)*")|('([^\\\']*(\\.)?)*')|(\/{2,}.*?(\r|\n))|(\/\*(\n|.)*?\*\/)|(\/\*\*\*\*\*\*\/)/g

    compiler.hooks.emit.tap('RemoveComment', (compilation) => {
      // 遍历构建产物，.assets中包含构建产物的文件名
      Object.keys(compilation.assets).forEach((item) => {
        // .source()是获取构建产物的文本
        let content = compilation.assets[item].source()
        content = content.replace(reg, function (word) {
          // 去除注释后的文本
          return /^\/{2,}/.test(word) || /^\/\*!/.test(word) || /^\/\*{3,}\//.test(word) ? '' : word
        })
        // 更新构建产物对象
        compilation.assets[item] = {
          source: () => content,
          size: () => content.length,
        }
      })
    })
  }
}

module.exports = RemoveCommentPlugin
```
