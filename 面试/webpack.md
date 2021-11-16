### webpack plugin实现
暴露apply方法
```javascript
function VariablePlugin(options) {
  let varsMap = {};

  try {
    // 读取传入的map字段，约定传入json
    const map = options.map || '';
    varsMap = JSON.parse(map);
  } catch (error) {}

  // 格式化为多个赋值语句的脚本
  const varsStr = Object.keys(varsMap).reduce((accumulator, key) => {
    return accumulator += `${key}='${varsMap[key]}';`
  }, '');

  this.script = varsStr;
}

VariablePlugin.prototype.apply = function(compiler) {
  const self = this;
  compiler.plugin('compilation', function(compilation) {
    compilation.plugin('html-webpack-plugin-before-html-processing', function(htmlPluginData, callback) {
      // 注入全局变量
      htmlPluginData.html = htmlPluginData.html.replace('<head>', `<head><script>${self.script}</script>`);
      callback(null, htmlPluginData);
    });
  });
};

module.exports = VariablePlugin;
```


### loader编写
```javascript
// 导出一个函数，source为webpack传递给loader的文件源内容
module.exports = function(source) {
    const content = doSomeThing2JsString(source);
    
    // 如果 loader 配置了 options 对象，那么this.query将指向 options
    const options = this.query;
    
    // 可以用作解析其他模块路径的上下文
    console.log('this.context');
    
    /*
     * this.callback 参数：
     * error：Error | null，当 loader 出错时向外抛出一个 error
     * content：String | Buffer，经过 loader 编译后需要导出的内容
     * sourceMap：为方便调试生成的编译后内容的 source map
     * ast：本次编译生成的 AST 静态语法树，之后执行的 loader 可以直接使用这个 AST，进而省去重复生成 AST 的过程
     */
    this.callback(null, content); // 异步
    return content; // 同步
}
```
### 热更新
#### 流程
1. 当模块变化是，webpack监听到文件变化对文件重新打包，编译生成唯一的hash值，这个hash值用来作为下一次热更新的标识
2. 根据变化的内容生成两个补丁： manifest和chunk.js模块
3. socket服务再`HMR Runtime`和`HMR Server`之间建立websock链接，文件变化服务端会想浏览器推送一条消息，消息就包含改动后生成的hash值。
4. 会发送一个ajax去服务端获取变化内容的manifest文件里面包含重新build生成的hash，和变化的模块。
5. 浏览器根据manifest文件获取模块变化内容，从而触发render流程，实现局部模块更新

#### 关于webpack热模块更新的总结如下：
1. 通过webpack-dev-server创建两个服务器：提供静态资源的服务（express）和Socket服务
2. express server 负责直接提供静态资源的服务（打包后的资源直接被浏览器请求和解析）
3. socket server 是一个 websocket 的长连接，双方可以通信
4. 当 socket server 监听到对应的模块发生变化时，会生成两个文件.json（manifest文件）和.js文件（update chunk）
5. 通过长连接，socket server 可以直接将这两个文件主动发送给客户端（浏览器）
6. 浏览器拿到两个新的文件后，通过HMR runtime机制，加载这两个文件，并且针对修改的模块进行更新


### webpack 优化
- 构建速度优化
  优化 loader 配置 ： 通过include 和exclude指定优化
  合理使用 resolve.extensions： 解析文件自动添加拓展名
  优化 resolve.modules： 解析第三方包
  优化 resolve.alias ： 添加查询路径别名
  使用 DLLPlugin 插件： 引入动态链，打包一个经常使用库再引入
  使用 cache-loader： 缓存磁盘
  terser 启动多线程
  合理使用 sourceMap
- webpack优化前端的手段有：
  JS代码压缩
  CSS代码压缩
  Html文件代码压缩
  文件大小压缩
  图片压缩
  Tree Shaking
  代码分离
  内联 chunk
