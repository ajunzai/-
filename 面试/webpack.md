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
