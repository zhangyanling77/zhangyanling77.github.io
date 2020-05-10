# webpack bundle 打包文件分析

## 基础知识

### toStringTag.js
```javascript
/* let toString = Object.prototype.toString;
console.log(toString.call('foo'));
console.log(toString.call([1, 2, 3]));
console.log(toString.call(4));
console.log(toString.call(true));
console.log(toString.call(undefined));
console.log(toString.call(null)); */
let myExports = {};
Object.defineProperty(myExports, Symbol.toStringTag, { value: 'Module' });
//console.log(toString.call(myExports));//[object Object]
console.log(toString.call(myExports));
function toString() {
    return `[object ${this[Symbol.toStringTag] || 'Object'}]`;
}
```

### create.js
```javascript

/**
 * Object.create(null)
 * 创建一个纯净的对象
 */
/* let obj = {};// new Object() obj.__proto__ = Object.prototype {}
console.log(Object.getPrototypeOf(obj).toString); */

function create(proto) {
    function Temp() { }
    Temp.prototype = proto;
    return new Temp();// new Temp()  .__proto__ = proto
}
//let ns = Object.create(null);
let ns = create(null);//ns.__proto__=null
console.log(Object.getPrototypeOf(ns));//说明原型链是空的

```

### getter.js
```javascript
/**
 *  getter
 */
let obj = {};
let ageValue;
Object.defineProperty(obj, 'age', {
    //value: 10,
    //writable: true,//是否可以修改 是否可以修改值 obj.age = 20;
    get() {
        return ageValue;
    },
    set(newValue) {
        ageValue = newValue;
    },

    enumerable: true, //是否可以枚举 for in 
    configurable: true//是否可以配置  是否可以删除此属性  delete obj.age
});
obj.age = 20;
console.log(obj.age);

//Invalid property descriptor.
//Cannot both specify accessors and a value or writable attribute,
```

### myBundle.js   __webpack_require__实现
```javascript
(function (modules) {
    var installedModules = {};//模块缓存
    function __webpack_require__(moduleId) {
        if (installedModules[moduleId]) {
            return installedModules[moduleId].exports;
        }
        //创建一个新的模块并赋值给module ,并且放置到模块的缓存(installedModules)中
        var module = installedModules[moduleId] = {
            i: moduleId,
            l: false,
            exports: {}
        }
        modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);
        module.l = true;//把此模块设置为已加载
        return module.exports;
    }
    //把模块的定义对象缓存在__webpack_require__.m属性上
    __webpack_require__.m = modules;
    //把已经回去过的模块组缓存放在__webpack_require__.c属性上，方便以后获取
    __webpack_require__.c = installedModules;
    //给一个对象增加一个属性 d=defineProperty
    __webpack_require__.d = function (exports, name, getter) {
        //o = hasOwnProperty
        if (!__webpack_require__.o(exports, name)) {
            Object.defineProperty(exports, name, { enumerable: true, get: getter });
        }
    }
    __webpack_require__.o = function (object, property) {
        return Object.prototype.hasOwnProperty.call(object, property);
    }
    //表示这是一个es模块 exports.__esModule=true 表示这是一个es6模块
    __webpack_require__.r = function (exports) {
        if (typeof Symbol !== 'undefined' && Symbol.toStringTag) {
            Object.defineProperty(exports, Symbol.toStringTag, { value: 'Module' });
        }
        Object.defineProperty(exports, '__esModule', { value: true });
    }
    // 1111  0001
    //作用是创建一个模拟的命名对象
    //t的核心作用是把一个任意的模块common.js es module都包装成es module的形式
    __webpack_require__.t = function (value, mode) {
        //如果与1为true,说明第一位是1，那么表示value是模块ID，需要直接通过require加载,获取它的module.exports对象
        if (mode & 1) {//加载这个模块ID，把value重新赋值为导出对象
            value = __webpack_require__(value);//从模块ID变成了模块的导出对象了
        }//mode =0b0001 { name: 'zhufeng', age: 10, __esModule: true }
        // 1000 说明可以直接返回 1001  ,如果是&1 &8 ,行为类似于__webpack_require__
        if (mode & 8) {
            return value;
        }
        //0100 如果与4为true的话，并且value有值，并且value.__esModule 说明已经是包装过的模块了
        if (mode & 4 && typeof value === 'object' && value.__esModule) {
            return value;
        }
        var ns = Object.create(null);//创建一个新对象，并且给它添加一个default属性，属性的值就是value
        Object.defineProperty(ns, 'default', { enumerable: true, value });
        //&2等于表示要把value所有属性拷贝到命令空间上ns
        if (mode & 2 && typeof value !== 'string') {
            for (let key in value) {
                __webpack_require__.d(ns, key, function (key) { return value[key] }.bind(null, key));
            }
        }
        return ns;//此方法在后面讲懒加载的时候会用的到
    }
    //publicPath 
    __webpack_require__.p = "";
    //.s缓存模块ID
    return __webpack_require__(__webpack_require__.s = "./src/index.js");

})({
    "./src/index.js":
        (function (module, exports, __webpack_require__) {
            //mode = {shouldRequire:true,directReturn:false,nowWrapper:true,copyProperties:true};            1                 1                1               1
            let title = __webpack_require__.t("./src/title.js", 0b0001);
            console.log(title);//{}
        }),
    "./src/title.js":
        (function (module, exports) {
            module.exports = { name: 'zhufeng', age: 10, __esModule: true };
        })
})
//modules key 是模块的相对路径  ./src/index.js
// 值是一个value 是一个函数，格式有点像node中的common.js
```

### myMain.js  懒加载
```javascript
(function (modules) {
    /**
     * 1.把通过jsonp获取回来的新的模块定义 ./src/c.js 合并到modules对象上去，以方便下面的模块加载
     * 2.在 installedChunks里面进行标识，标识此c代码块成功
     * 2.让promise变成成功态
     */
    function webpackJsonpCallback(data) {
        var chunkIds = data[0];//第一个元素是代码块的数组
        var moreModules = data[1];//新的模块定义
        var moduleId, chunkId, i = 0, resolves = [];
        for (i = 0; i < chunkIds.length; i++) {
            chunkId = chunkIds[i];
            //把promise的resolve方法先都添加到resolves数组中去
            resolves.push(installedChunks[chunkId][0]);
            installedChunks[chunkId] = 0;//installedChunks{c:0}
        }
        for (moduleId in moreModules) {//模块定义的合并modules={index.js,c.js}
            modules[moduleId] = moreModules[moduleId];
        }
        //遍历resolves数组，挨个执行里面的resolve方法
        while (resolves.length) {
            resolves.shift()();//调用resolve方法可以让promise成功
        }
    }
    var installedModules = {};//模块缓存
    var installedChunks = {
        "main": 0
    }
    function __webpack_require__(moduleId) {
        if (installedModules[moduleId]) {
            return installedModules[moduleId].exports;
        }
        //创建一个新的模块并赋值给module ,并且放置到模块的缓存(installedModules)中
        var module = installedModules[moduleId] = {
            i: moduleId,
            l: false,
            exports: {}
        }
        debugger;
        modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);
        module.l = true;//把此模块设置为已加载
        return module.exports;
    }
    //把模块的定义对象缓存在__webpack_require__.m属性上
    __webpack_require__.m = modules;
    //把已经回去过的模块组缓存放在__webpack_require__.c属性上，方便以后获取
    __webpack_require__.c = installedModules;
    //给一个对象增加一个属性 d=defineProperty
    __webpack_require__.d = function (exports, name, getter) {
        //o = hasOwnProperty
        if (!__webpack_require__.o(exports, name)) {
            Object.defineProperty(exports, name, { enumerable: true, get: getter });
        }
    }
    __webpack_require__.o = function (object, property) {
        return Object.prototype.hasOwnProperty.call(object, property);
    }
    //表示这是一个es模块 exports.__esModule=true 表示这是一个es6模块
    __webpack_require__.r = function (exports) {
        if (typeof Symbol !== 'undefined' && Symbol.toStringTag) {
            Object.defineProperty(exports, Symbol.toStringTag, { value: 'Module' });
        }
        Object.defineProperty(exports, '__esModule', { value: true });
    }
    function jsonpScriptSrc(chunkId) {
        return __webpack_require__.p + "" + chunkId + ".js";//c:js
    }
    //懒加载代码块chunkId
    __webpack_require__.e = function requireEnsure(chunkId) {
        let promises = [];
        let installedChunkData = installedChunks[chunkId];
        if (installedChunkData !== 0) {//说明还尚未加载成功,需要加载
            if (installedChunkData) {//如果installedChunkData不为0，但是有值说明正在加载中
                promises.push(installedChunkData[2]);//直接 把promise放进来
            } else {
                var promise = new Promise(function (resolve, reject) {
                    installedChunkData = installedChunks[chunkId] = [resolve, reject];
                });
                promises.push(promise);
                //installedChunkData[0]就是promise的resolve方法
                //installedChunkData.push(promise);
                installedChunkData[2] = promise;//installedChunkData=[resolve, reject,promise]
                let script = document.createElement('script');
                script.src = jsonpScriptSrc(chunkId);//script.src = c.js
                document.head.appendChild(script);
            }
        }
        return Promise.all(promises);//如果已经有数据的话,此Promise会立刻成功
    }
    // 1111  0001
    //作用是创建一个模拟的命名对象
    //t的核心作用是把一个任意的模块common.js es module都包装成es module的形式
    __webpack_require__.t = function (value, mode) {
        //如果与1为true,说明第一位是1，那么表示value是模块ID，需要直接通过require加载,获取它的module.exports对象
        if (mode & 1) {//加载这个模块ID，把value重新赋值为导出对象
            value = __webpack_require__(value);//从模块ID变成了模块的导出对象了
        }//mode =0b0001 { name: 'zhufeng', age: 10, __esModule: true }
        // 1000 说明可以直接返回 1001  ,如果是&1 &8 ,行为类似于__webpack_require__
        if (mode & 8) {
            return value;
        }
        //0100 如果与4为true的话，并且value有值，并且value.__esModule 说明已经是包装过的模块了
        if (mode & 4 && typeof value === 'object' && value.__esModule) {
            return value;
        }
        var ns = Object.create(null);//创建一个新对象，并且给它添加一个default属性，属性的值就是value
        Object.defineProperty(ns, 'default', { enumerable: true, value });
        //&2等于表示要把value所有属性拷贝到命令空间上ns
        if (mode & 2 && typeof value !== 'string') {
            for (let key in value) {
                __webpack_require__.d(ns, key, function (key) { return value[key] }.bind(null, key));
            }
        }
        return ns;//此方法在后面讲懒加载的时候会用的到
    }
    //publicPath 
    __webpack_require__.p = "";
    //先把jsonArray=window["webpackJsonp"]=[]
    var jsonArray = window["webpackJsonp"] = window["webpackJsonp"] || [];
    //重写了jsonArray.push方法，等于了webpackJsonpCallback
    jsonArray.push = webpackJsonpCallback;
    //.s缓存模块ID
    return __webpack_require__(__webpack_require__.s = "./src/index.js");

})({
    "./src/index.js":
        (function (module, exports, __webpack_require__) {
            let loadCButton = document.getElementById('loadC');
            loadCButton.addEventListener('click', () => {
                debugger
                __webpack_require__.e("c").then(() => {
                    return __webpack_require__("./src/c.js");
                }).then(c => {
                    console.log(c);
                });
            })
        })
})
```
