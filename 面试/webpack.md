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
