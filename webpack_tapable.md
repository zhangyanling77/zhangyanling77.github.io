# webpack tapable
是webpack运转的核心，整个架构都是基于这个来实现的

## tapable分类

### 1、按同步异步分类

`Async：`

异步并行

- AsyncParallelHook

- AsyncParallelBailHook

异步串行

- AsyncSeriesHook

- AsyncSeriesBailHook

- AsyncSeriesWaterfallHook

`Sync: `

- SyncHook

- SyncBailHook

- SyncWaterfallHook

- SyncLoopHook

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
