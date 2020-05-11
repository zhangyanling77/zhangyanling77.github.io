# webpack tapable

## 基础知识

根据bundle分析：

模块在加载时是平级的，执行时是按嵌套层级的。

模块加载过后就被缓存了，不存在重复加载的问题。

如果说import加载的是一个commonjs模块会使用到__webpack_require__.t方法。
