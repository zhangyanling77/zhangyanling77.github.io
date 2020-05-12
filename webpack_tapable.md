# webpack tapable

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

`Bail:  关心返回值（当函数的返回值不为null，熔断操作）`

- SyncBailHook

- AsyncParallelBailHook

- AsyncSeriesBailHook

`Basic:  不关心返回值`

- SyncHook

- AsyncParallelHook

- AsyncSeriesHook

`Loop:  循环执行`

- SyncLoopHook

`Waterfall:  上一步是下一步的输入`

- SyncWaterfallHook

- AsyncSeriesWaterfallHook


