# webpack tapable
是webpack运转的核心，整个架构都是基于这个来实现的

## tapable分类

### 1、按同步异步分类

`Sync: `

#### SyncHook

  - 不关心返回值，按顺序执行
  
  - 调用call时传入的参数可以传给回调函数
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
#### SyncBailHook

  - 关心返回值，顺序执行
  
  - 调用call时传入的参数可以传给回调函数
  
  - 当回调函数返回非undefined的值时停止调用后续回调

#### SyncWaterfallHook

  - 如果上一个回调的返回值不为undefined，则作为下一个回调函数的第一个参数
  
  - 回调函数接受的参数来自于上一个回调的结果
  
  - 调用call传入的第一个参数，会被上一个参数的非undefined结果替换
  
  - 当回调函数返回非undefined不会停止回调栈的调用

#### SyncLoopHook

  特点时不停的执行循环事件函数，直到结果等于undefined。注意，每次都是从头开始循环的。

`Async：`

> 凡是带有Async关键字的钩子都支持三种注册方式：
> tap、tapAsync、tapPromise 监听;
> tap、tapAsync 这两种用 hook.callAsync() 触发;
> tapPromise 用hook.promise()触发

#### AsyncParallelHook
  异步并行

  - tapAsync这种情况，回调比参数列表传入的参数多了一个callback参数。等异步执行时间最长的那个返回就结束。

  - tapPromise这种情况，每一步返回的是个promise。因为是promise，所以可以then。

#### AsyncParallelBailHook
  异步并行 失败钩子
  
  - 带保险的异步并行执行钩子
  
  - 有一个任务返回值不为undefined就直接结束
  
  当触发失败就结束任务。reject('err')或者callback('有值')即可触发.

#### AsyncSeriesHook
  异步串行
  
  执行时间是总和。执行看起来和同步差不多，但是支持异步代码而已。

#### AsyncSeriesBailHook
  异步串行 失败钩子
  
  执行时间是总和，触发失败就立即结束。

#### AsyncSeriesWaterfallHook
  异步串行瀑布 (异步、串行、上一个的返回值作为下一个的输入)
  
  如何返回值：

  - tapAsync的情况：凡是异步callback，第一个参数表示的是有没有错误，如果为null表示没有错，否则有错。第二个参数才是正确的返回值。
  
  - tapPromise的情况：resolve传出成功的值。如果resolve你没有传值，那么还是初始传入的值，如果resolve里传了值，则为你传入的值。

### 2、按值返回分类

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


### 3、使用：类似于event类

```javascript
let hook = new XXXHook()

hook.tap() / hook.tapAsync() / hook.tapPromise()  监听事件

hook.call() / hook.callAsyc() / hook.promise()  触发事件
```

## tapable的实现
核心：注册监听函数，触发执行

### 1、基类Hook
```javascript
/**
* args: 传入的参数列表，如['name', 'age']
* taps: 用来存放钩子回调函数的数组
* _x: 用来拼函数的
*/
class Hook {
    constructor(args) {
        if (!Array.isArray(args)) args = [];
        this.args = args;// Hook类的实例 this.args = args=['name','age']
        this.taps = [];// 这是存放钩子回调函数的数组
        this._x = undefined;//后面会用到，是用来拼函数的
    }
    tap(options, fn) {
        if (typeof options === 'string') options = { name: options };
        options.fn = fn;//options = {name:'1',fn:function1};
        this._insert(options);
    }
    tapAsync(options, fn) {
        if (typeof options === 'string') options = { name: options };
        options.fn = fn;//options = {name:'1',fn:function1};
        this._insert(options);
    }
    _insert(options) {
        this.taps[this.taps.length] = options;//= this.taps.push(options);
    }
    call(...args) {// args=['zhufeng',10]
        //第一个动态编译出来一个函数
        let callMethod = this._createCall();//(function anonymous(name, age) {})
        return callMethod.apply(this, args);;
    }
    callAsync(...args) {// args=['zhufeng',10]
        //第一个动态编译出来一个函数
        let callMethod = this._createCall();//(function anonymous(name, age) {})
        return callMethod.apply(this, args);;
    }
    _createCall() {
        return this.compile({
            taps: this.taps,//监听函数 [{name:'1',fn:function1},{name:'2',fn:function2}]
            args: this.args//['name','age']
        });
    }
}
module.exports = Hook;
```
### 2、HookCodeFactory 工厂函数
```javascript
class HookCodeFactory {
    setup(hookInstance, options) {
        this.options = options;//{args,taps}args=[name,age]
        //[ {name:'1',fn:function1}, {name:'2',fn:function2}]
        hookInstance._x = options.taps.map(t => t.fn);//fn的数组 [function1,function2]
    }
    args({ before, after } = {}) {
        let allArgs = this.options.args || [];//[name,age]
        if (before) allArgs = [before, ...allArgs];
        if (after) allArgs = [...allArgs, after];
        return allArgs.join(',');//name,age,_callback
    }
    header() {
        return `var _x = this._x;\n`;
    }
    content() {
        let code = ``;
        for (let i = 0; i < this.options.taps.length; i++) {
            code += `
                    var _fn${i} = _x[${i}];
                    _fn${i}(${this.args()});
                    `
        }
        return code;
    }
    create() {
        return new Function(
            this.args(),
            this.header() + this.content()
        );
    }
}
module.exports = HookCodeFactory;
```
### 3、SyncHook 
```javascript
let HookCodeFactory = require('./HookCodeFactory');
let Hook = require('./Hook');

const factory = new HookCodeFactory();
class SyncHook extends Hook {
    constructor(args) {
        super(args);
    }
    compile(options) {//{taps,args}
        factory.setup(this, options);//做一些工厂准备工作
        return factory.create(options);//真正创建函数的 (function anonymous(name, age) {}
    }
}
module.exports = SyncHook;
```
