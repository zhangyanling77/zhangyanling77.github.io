# webpack tapable

## lazyload

根据bundle分析：

模块在加载时是平级的，执行时是按嵌套层级的。

模块加载过后就被缓存了，不存在重复加载的问题。

如果说import加载的是一个commonjs模块会使用到__webpack_require__.t方法。

```javascript

let webpackJsonp;
function main1(modules) {
    var installedChunks = { main1: 0 }
    function webpackJsonpCallback(data) {
        let [chunkIds, moreModules] = data;
        for (let i = 0; i < chunkIds.length; i++) {
            chunkId = chunkIds[i];
            installedChunks[chunkId] = 0;
        }
        for (let moduleId in moreModules) {
            modules[moduleId] = moreModules[moduleId];
        }
        parentJsonpFunction(data);
    }
    var jsonpArray = [];
    webpackJsonp = jsonpArray;
    var oldJsonpFunction = jsonpArray.push.bind(jsonpArray);//缓存老的push方法
    jsonpArray.push = webpackJsonpCallback;
    for (var i = 0; i < jsonpArray.length; i++)
        webpackJsonpCallback(jsonpArray[i]);
    var parentJsonpFunction = oldJsonpFunction;
    webpackJsonp.push([["c1", { "./src/c1.js": () => { } }]]);
}
main1();
function main2(modules) {
    var installedChunks = { main2: 0 }
    var parentJsonpFunction;
    function webpackJsonpCallback(data) {
        let [chunkIds, moreModules] = data;

        for (let i = 0; i < chunkIds.length; i++) {
            chunkId = chunkIds[i];
            installedChunks[chunkId] = 0;
        }
        for (let moduleId in moreModules) {
            modules[moduleId] = moreModules[moduleId];
        }
        parentJsonpFunction && parentJsonpFunction(data);
    }
    var jsonpArray = webpackJsonp; //在main2.js执行的时候，先取出main1.js初始化的，挂在window 上的webpackJsonp数组
    //jsonpArray.push=main1函数内部的webpackJsonpCallback方法
    var oldJsonpFunction = jsonpArray.push.bind(jsonpArray);
    jsonpArray.push = webpackJsonpCallback;

    //循环数组，把每个元素传给了main2自己的webpackJsonpCallback
    for (var i = 0; i < jsonpArray.length; i++)
        webpackJsonpCallback(jsonpArray[i]);
    var parentJsonpFunction = oldJsonpFunction;
    webpackJsonp.push([["c2", { "./src/c2.js": () => { } }]]);
}
setTimeout(() => {
    main2();
}, 1000);
// main2.webpackJsonpCallback=>main1.webpackJsonpCallback=>jsonArray.push
/* let arr = [];
let oldPush = arr.push;
arr.push = function (...args) {
    //写一些额外的逻辑
    oldPush(...args);
} */

```
