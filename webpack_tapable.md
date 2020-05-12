# webpack tapable
是webpack运转的核心，整个架构都是基于这个来实现的

## tapable分类

### 1、按同步异步分类

`Sync: `

- SyncHook

  1. 不关心返回值，按顺序执行
  
  2. 调用call时传入的参数可以传给回调函数
```javascript
  // 例子：

  const { SyncHook } = require('tapable');

  let hook = new SyncHook(['name', 'age'])

  hook.tap('1', (name, age) => {

  // do domething

  });

  hook.tap('2', (name, age) => {

  // do domething

  });

  hook.call('zhangsan', 26)  // 触发事件或者说执行事件

  // 实现事件的订阅和发布。
  /**
  * tap：注册事件
  * 第一个参数就是个事件名称。没有什么太大的用途，就似乎给开发人员看的
  * 第二个参数是个钩子函数，参数是new Hook时传入的参数
  */
```
- SyncBailHook

  1. 关心返回值，顺序执行
  
  2. 调用call时传入的参数可以传给回调函数
  
  3. 当回调函数返回非undefined的值时停止调用后续回调

- SyncWaterfallHook

  1. 如果上一个回调的返回值不为undefined，则作为下一个回调函数的第一个参数
  
  2. 回调函数接受的参数来自于上一个回调的结果
  
  3. 调用call传入的第一个参数，会被上一个参数的非undefined结果替换
  
  4. 当回调函数返回非undefined不会停止回调栈的调用

- SyncLoopHook

  特点时不停的执行循环事件函数，直到结果等于undefined。注意，每次都是从头开始循环的。

`Async：`

异步并行 同步的钩子指支持 `tap` 注册方式

- AsyncParallelHook

- AsyncParallelBailHook

异步串行

- AsyncSeriesHook

- AsyncSeriesBailHook

- AsyncSeriesWaterfallHook

2、按值返回分类

`Bail:  关心返回值（当遇到第一个结果 result !== undefined 则返回，不再继续执行。熔断操作）`

- SyncBailHook

- AsyncParallelBailHook

- AsyncSeriesBailHook

`Basic:  不关心返回值，一个个执行`

- SyncHook

- AsyncParallelHook

- AsyncSeriesHook

`Loop:  不停的循环执行事件函数，直到所有的结果 result===undefined`

- SyncLoopHook

`Waterfall:  瀑布。上一步的结果是下一步的输入`

- SyncWaterfallHook

- AsyncSeriesWaterfallHook


### 2、使用：类似于event类

let hook = new XXXHook()

hook.tap()  监听事件

hook.call()  触发事件
