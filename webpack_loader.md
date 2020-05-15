# webpack loader

## babel-loader
```javascript
/**
 * loader其实就是一个函数，传入一个原内容，返回新内容
 */
let babelCore = require('@babel/core');//babel-core
function loader(source) {
    let options = {
        presets: ["@babel/preset-env", "@babel/preset-react"],
        sourceMap: true //将会生成sourceMap
    }
    console.log('source', source);
    //code转换后的代码 map sourceMap用来实现调试的 ast抽象语法树
    let { code, map, ast } = babelCore.transform(source, options);
    console.log('babelCore-source', code);
    return code;
}
module.exports = loader;
```
