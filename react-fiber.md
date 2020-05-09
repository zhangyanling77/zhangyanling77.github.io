# React Fiber架构的实现

## 1、requestAnimationFrame

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>

<body>
    <div id="progress-bar" style="background: lightblue;width:0px;height:20px"></div>
    <button id="btn">开始</button>
    <script>
        // rAf 的用法 页面上绘制一个进度条 值0%=>100%
        let btn = document.getElementById('btn');
        let oDiv = document.getElementById('progress-bar');
        let start;
        function progress() {
            oDiv.style.width = oDiv.offsetWidth + 1 + 'px';
            oDiv.innerHTML = (oDiv.offsetWidth) + '%';// 修改文本为百分比
            if (oDiv.offsetWidth < 100) {
                let current = Date.now();
                // 假如说浏览器本身的任务执行是5ms
                console.log(current - start);// 打印的是开始准备执行的时候到真正执行的时间的时间差
                start = current;
                requestAnimationFrame(progress);
            }
        }
        btn.addEventListener('click', () => {
            oDiv.style.width = 0;// 先把宽度清除 rAf 后面会用到
            let current = Date.now();// 先获取到当前的时间 current是毫秒数
            start = current;
            requestAnimationFrame(progress);
        });
    </script>
</body>

</html>
```

## 2、requestIdleCallback

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Idle</title>
</head>

<body>
    <script>
        // 这是一个全局属性
        // 我作为用户，告诉浏览器，我现在执行callback函数，但是它的优先级比较低。告诉 浏览器，可以空闲的时候执行callback
        // 但是如何这个到了超时时间了，就必须马上执行
        // window.requestIdleCallback(callback, { timeout: 1000 });

        function sleep(delay) { //d=50//
            // 在JS里实现睡眠的功能 t=当前时间
            for (var start = Date.now(); Date.now() - start <= delay;) { }
        }
        let allStart = Date.now();
        // fiber是把整个任务分成很多个小任务，每次执行一个任务
        // 执行完成后会看看有没剩余时间，如果有继续下一个任务，如果没有放弃执行，交给浏览器进行调度
        const works = [
            () => {
                console.log('第1个任务开始');
                while (true) { }
                sleep(20);// 一帧16.6 因为此任务的执行时间已经超过了16.6毫秒，所需要把控制 权交给浏览器
                console.log('第1个任务结束 ');
            },
            () => {
                console.log('第2个任务开始');
                sleep(20);
                console.log('第2个任务结束 ');
            },
            () => {
                console.log('第3个任务开始');
                sleep(20);
                console.log('第3个任务结束 ');
                console.log(Date.now() - allStart);
            }
        ]
        // 告诉 浏览器在1000毫秒后，即使你没有空闲时间也得帮我执行，因为我等不及你
        requestIdleCallback(workLoop, { timeout: 1000 });
        // deadline是一个对象 有两个属性
        // timeRemaining()可以返回此帧还剩下多少时间供用户使用
        // didTimeout 此callback任务是否超时
        function workLoop(deadline) {
            console.log(`本帧的剩余时间为${parseInt(deadline.timeRemaining())}`);
            // 如果此帧的剩余时间超过0,或者此已经超时了
            while ((deadline.timeRemaining() > 1 || deadline.didTimeout) && works.length > 0) {
                performUnitOfWork();
            }// 如果说没有剩余时间了，就需要放弃执行任务控制权，执行权交还给浏览器

            if (works.length > 0) {//说明还有未完成的任务
                window.requestIdleCallback(workLoop, { timeout: 1000 });
            }
        }
        function performUnitOfWork() {
            // shift取出数组中的第1个元素
            works.shift()();
        }

    </script>
</body>

</html>
```

## 3、Fiber 实现

### constants.js
```javascript
//表示这是一个文本元素
export const ELEMENT_TEXT = Symbol.for('ELEMENT_TEXT');
//React应用需要一个根Fiber
export const TAG_ROOT = Symbol.for('TAG_ROOT');
//原生的节点 span div p  函数组件 类组件
export const TAG_HOST = Symbol.for('TAG_HOST');
//这是文本节点
export const TAG_TEXT = Symbol.for('TAG_TEXT');
//这是类组件
export const TAG_CLASS = Symbol.for('TAG_CLASS');
//函数组件
export const TAG_FUNCTION_COMPONENT = Symbol.for('TAG_FUNCTION_COMPONENT');
//插入节点
export const PLACEMENT = Symbol.for('PLACEMENT');
//更新节点
export const UPDATE = Symbol.for('UPDATE');
//删除节点
export const DELETION = Symbol.for('DELETION');



//文本节点的话 fiber {tag:TAG_TEXT,type:ELEMENT_TEXT}
```

### utils.js
```javascript

export function setProps(dom, oldProps, newProps) {
    for (let key in oldProps) {
        if (key !== 'children') {
            if (newProps.hasOwnProperty(key)) {
                setProp(dom, key, newProps[key]);// 新老都有，则更新
            } else {
                dom.removeAttribute(key);//老props里有此属性，新 props没有，则删除
            }
        }
    }
    for (let key in newProps) {
        if (key !== 'children') {
            if (!oldProps.hasOwnProperty(key)) {//老的没有，新的有，就添加此属性
                setProp(dom, key, newProps[key]);
            }
        }
    }
}
function setProp(dom, key, value) {
    if (/^on/.test(key)) {//onClick
        dom[key.toLowerCase()] = value;//没有用合成事件
    } else if (key === 'style') {
        if (value) {
            for (let styleName in value) {
                dom.style[styleName] = value[styleName];
            }
        }
    } else {
        dom.setAttribute(key, value);
    }
}
```

### react.js

```javascript
import { ELEMENT_TEXT } from './constants';
import { Update } from './UpdateQueue';
import { scheduleRoot,useReducer ,useState} from './scheduler';
/**
 * 创建元素(虚拟DOM)的广法
 * @param {} type  元素的类型div span p
 * @param {*} config  配置对象 属性 key ref
 * @param  {...any} children  放着所有的儿子，这里会做成一个数组
 */
function createElement(type, config, ...children) {
    delete config.__self;
    delete config.__source;//表示这个元素是在哪行哪列哪个文件生成的
    return {
        type,
        props: {
            ...config,//做了一个兼容处理，如果是React元素的话返回自己，如果是文本类型，如果是一个字符串的话，返回元素对象
            children: children.map(child => {
                //如果这个child是一个React.createElement返回的React元素，如果是字符串的话，才会转成文本节点
                return typeof child === 'object' ? child : {
                    type: ELEMENT_TEXT,
                    props: { text: child, children: [] }
                }
            })
        }
    }
}
class Component {
    constructor(props) {
        this.props = props;
        //this.updateQueue = new UpdateQueue();
    }
    setState(payload) {//可能是对象，也可能是一个函数
        let update = new Update(payload);
        //updateQueue其实是放在此类组件对应的fiber节点的 internalFiber
        this.internalFiber.updateQueue.enqueueUpdate(update);
        //this.updateQueue.enqueueUpdate(update);
        debugger
        scheduleRoot();//从根节点开始调度
    }
}
Component.prototype.isReactComponent = {};//类组件
const React = {
    createElement,
    Component,
    useReducer,
    useState
}
export default React;
```
### react-dom.js
```javascript
import { TAG_ROOT } from './constants';
import { scheduleRoot } from './scheduler';
/**
 * render是要把一个元素渲染到一个容器内部
 */
function render(element, container) {//container=root DOM节点
    let rootFiber = {
        tag: TAG_ROOT,//每个fiber会有一个tag标识 此元素的类型
        stateNode: container,//一般情况下如果这个元素是一个原生节点的话，stateNode指向真实DOM元素
        //props.children是一个数组，里面放的是React元素 虚拟DOM 后面会根据每个React元素创建 对应的Fiber
        props: { children: [element] }//这个fiber的属性对象children属性，里面放的是要渲染的元素
    }
    scheduleRoot(rootFiber);
}
const ReactDOM = {
    render
}
export default ReactDOM;
/**
 * reconciler
 * schedule
 */
```

### schedule.js

```javascript
import { TAG_ROOT, ELEMENT_TEXT, TAG_TEXT, TAG_HOST, PLACEMENT, DELETION, UPDATE, TAG_CLASS, TAG_FUNCTION_COMPONENT } from "./constants";
import { setProps } from './utils';
import { UpdateQueue, Update } from "./UpdateQueue";
/**
 * 从根节点开始渲染和调度 两个阶段 
 * 
 * diff阶段 对比新旧的虚拟DOM，进行增量 更新或创建. render阶段
 * 这个阶段可以比较花时间，可以我们对任务进行拆分，拆分的维度虚拟DOM。此阶段可以暂停
 * render阶段成果是effect list知道哪些节点更新哪些节点删除了，哪些节点增加了
 * render阶段有两个任务1.根据虚拟DOM生成fiber树 2.收集effectlist
 * commit阶段，进行DOM更新创建阶段，此阶段不能暂停，要一气呵成
 */
let nextUnitOfWork = null;//下一个工作单元
let workInProgressRoot = null;//正在渲染的根ROOT fiber
let currentRoot = null;//渲染成功之后当前根ROOTFiber
let deletions = [];//删除的节点我们并不放在effect list里，所以需要单独记录并执行
let workInProgressFiber = null;//正在工作中的fiber
let hookIndex = 0;//hooks索引 
export function scheduleRoot(rootFiber) {//{tag:TAG_ROOT,stateNode:container,props: { children: [element] }}
    if (currentRoot && currentRoot.alternate) {//第二次之后的更新
        workInProgressRoot = currentRoot.alternate;//第一次渲染出来的那个fiber tree
        workInProgressRoot.alternate = currentRoot;//让这个树的替身指向的当前的currentRoot
        if (rootFiber) workInProgressRoot.props = rootFiber.props;//让它的props更新成新的props
    } else if (currentRoot) {//说明至少已经渲染过一次了 第一次更新
        if (rootFiber) {
            rootFiber.alternate = currentRoot;
            workInProgressRoot = rootFiber;
        } else {
            workInProgressRoot = {
                ...currentRoot,
                alternate: currentRoot
            }
        }
    } else {//如果说是第一次渲染
        workInProgressRoot = rootFiber;
    }
    workInProgressRoot.firstEffect = workInProgressRoot.lastEffect = workInProgressRoot.nextEffect = null;
    nextUnitOfWork = workInProgressRoot;
}
function performUnitOfWork(currentFiber) {
    beginWork(currentFiber);//开
    if (currentFiber.child) {
        return currentFiber.child;
    }

    while (currentFiber) {
        completeUnitOfWork(currentFiber);//没有儿子让自己完成
        if (currentFiber.sibling) {//看有没有弟弟
            return currentFiber.sibling;//有弟弟返回弟弟
        }
        currentFiber = currentFiber.return;//找父亲然后让父亲完成
    }
}
//在完成的时候要收集有副作用的fiber，然后组成effect list
//每个fiber有两个属性 firstEffect指向第一个有副作用的子fiber lastEffect 指儿 最后一个有副作用子Fiber
//中间的用nextEffect做成一个单链表 firstEffect=大儿子.nextEffect二儿子.nextEffect三儿子 lastEffect
function completeUnitOfWork(currentFiber) {//第一个完成的A1(TEXT)
    let returnFiber = currentFiber.return;//A1
    if (returnFiber) {
        ////这一段是把自己儿子的effect 链挂到父亲身上
        if (!returnFiber.firstEffect) {
            returnFiber.firstEffect = currentFiber.firstEffect;
        }
        if (currentFiber.lastEffect) {
            if (returnFiber.lastEffect) {
                returnFiber.lastEffect.nextEffect = currentFiber.firstEffect;
            }
            returnFiber.lastEffect = currentFiber.lastEffect;
        }
        //把自己挂到父亲 身上
        const effectTag = currentFiber.effectTag;
        if (effectTag) {// 自己有副作用 A1 first last=A1(Text)
            if (returnFiber.lastEffect) {
                returnFiber.lastEffect.nextEffect = currentFiber;
            } else {
                returnFiber.firstEffect = currentFiber;
            }
            returnFiber.lastEffect = currentFiber;
        }
    }
}
/**
 * beginWork开始收下线的钱
 * completeUnitOfWork把下线的钱收完了
 * 1.创建真实DOM元素
 * 2.创建子fiber  
 */
function beginWork(currentFiber) {
    if (currentFiber.tag === TAG_ROOT) {//根fiber
        updateHostRoot(currentFiber);
    } else if (currentFiber.tag === TAG_TEXT) {//文本fiber
        updateHostText(currentFiber);
    } else if (currentFiber.tag === TAG_HOST) {//原生DOM节点 stateNode dom
        updateHost(currentFiber);
    } else if (currentFiber.tag === TAG_CLASS) {//类组件
        updateClassComponent(currentFiber);
    } else if (currentFiber.tag === TAG_FUNCTION_COMPONENT) {//类组件
        updateFunctionComponent(currentFiber);
    }
}
function updateFunctionComponent(currentFiber) {
    workInProgressFiber = currentFiber;
    hookIndex = 0;
    workInProgressFiber.hooks = [];
    const newChildren = [currentFiber.type(currentFiber.props)];
    reconcileChildren(currentFiber, newChildren);
}
function updateClassComponent(currentFiber) {
    if (!currentFiber.stateNode) {//类组件 stateNode 组件的实例
        // new ClassCounter(); 类组件实例   fiber双向指向 
        currentFiber.stateNode = new currentFiber.type(currentFiber.props);
        currentFiber.stateNode.internalFiber = currentFiber;
        currentFiber.updateQueue = new UpdateQueue();
    }
    //给组件的实例的state 赋值
    currentFiber.stateNode.state = currentFiber.updateQueue.forceUpdate(currentFiber.stateNode.state);
    let newElement = currentFiber.stateNode.render();
    const newChildren = [newElement];
    reconcileChildren(currentFiber, newChildren);
}
function updateHost(currentFiber) {
    if (!currentFiber.stateNode) {//如果此fiber没有创建DOM节点
        currentFiber.stateNode = createDOM(currentFiber);
    }
    const newChildren = currentFiber.props.children;
    reconcileChildren(currentFiber, newChildren);
}
function createDOM(currentFiber) {
    if (currentFiber.tag === TAG_TEXT) {
        return document.createTextNode(currentFiber.props.text);
    } else if (currentFiber.tag === TAG_HOST) {// span div
        let stateNode = document.createElement(currentFiber.type);//div
        updateDOM(stateNode, {}, currentFiber.props);
        return stateNode;
    }
}
function updateDOM(stateNode, oldProps, newProps) {
    if (stateNode && stateNode.setAttribute)
        setProps(stateNode, oldProps, newProps);
}
function updateHostText(currentFiber) {
    if (!currentFiber.stateNode) {//如果此fiber没有创建DOM节点
        currentFiber.stateNode = createDOM(currentFiber);
    }
}
function updateHostRoot(currentFiber) {
    //先处理自己 如果是一个原生节点，创建真实DOM 2.创建子fiber 
    let newChildren = currentFiber.props.children;//[element=<div id="A1"]
    reconcileChildren(currentFiber, newChildren);
}
//newChildren是一个虚拟DOM的数组 把虚拟DOM转成Fiber节点
function reconcileChildren(currentFiber, newChildren) {//[A1]
    let newChildIndex = 0;//新子节点的索引
    //如果说currentFiber有alternate并且alternate有child属性
    let oldFiber = currentFiber.alternate && currentFiber.alternate.child;
    if (oldFiber) oldFiber.firstEffect = oldFiber.lastEffect = oldFiber.nextEffect = null;
    let prevSibling;//上一个新的子fiber
    //遍历我们的子虚拟DOM元素数组，为每个虚拟DOM元素创建子Fiber
    while (newChildIndex < newChildren.length || oldFiber) {
        let newChild = newChildren[newChildIndex];//取出虚拟DOM节点[A1]{type:'A1'}
        let newFiber;//新的Fiber
        const sameType = oldFiber && newChild && oldFiber.type === newChild.type;
        let tag;
        if (newChild && typeof newChild.type === 'function' && newChild.type.prototype.isReactComponent) {
            tag = TAG_CLASS;//
        } else if (newChild && typeof newChild.type === 'function') {
            tag = TAG_FUNCTION_COMPONENT;//这是一个文本节点
        } else if (newChild && newChild.type == ELEMENT_TEXT) {
            tag = TAG_TEXT;//这是一个文本节点
        } else if (newChild && typeof newChild.type === 'string') {
            tag = TAG_HOST;//如果是type是字符串，那么这是一个原生DOM节点 "A1" div
        }//beginWork创建fiber 在completeUnitOfWork的时候收集effect
        if (sameType) {//说明老fiber和新虚拟DOM类型一样，可以复用老的DOM节点，更新即可
            if (oldFiber.alternate) {//说明至少已经更新一次了
                newFiber = oldFiber.alternate;//如果有上上次的fiber,就拿 过来作为这一次的fiber
                newFiber.props = newChild.props;
                newFiber.alternate = oldFiber;
                newFiber.effectTag = UPDATE;
                newFiber.updateQueue = oldFiber.updateQueue || new UpdateQueue();
                newFiber.nextEffect = null;
            } else {
                newFiber = {
                    tag: oldFiber.tag,//TAG_HOST
                    type: oldFiber.type,//div
                    props: newChild.props,//{id="A1" style={style}} 一定要用新的元素的props
                    stateNode: oldFiber.stateNode,//div还没有创建DOM元素
                    return: currentFiber,//父Fiber returnFiber
                    updateQueue: oldFiber.updateQueue || new UpdateQueue(),
                    alternate: oldFiber,//让新的fiber的alternate指向老的fiber节点
                    effectTag: UPDATE,//副作用标识 render我们要会收集副作用 增加 删除 更新
                    nextEffect: null,//effect list 也是一个单链表
                }
            }
        } else {
            if (newChild) {//看看新的虚拟DOM是不是为null
                newFiber = {
                    tag,//TAG_HOST
                    type: newChild.type,//div
                    props: newChild.props,//{id="A1" style={style}}
                    stateNode: null,//div还没有创建DOM元素
                    return: currentFiber,//父Fiber returnFiber
                    effectTag: PLACEMENT,//副作用标识 render我们要会收集副作用 增加 删除 更新
                    updateQueue: new UpdateQueue(),
                    nextEffect: null,//effect list 也是一个单链表
                    //effect list顺序和 完成顺序是一样的，但是节点只放那些出钱的人的fiber节点，不出钱绕过去
                }
            }
            if (oldFiber) {
                oldFiber.effectTag = DELETION;
                deletions.push(oldFiber);
            }

        }
        if (oldFiber) {
            oldFiber = oldFiber.sibling;//oldFiber指针往后移动一次
        }
        //最小的儿子是没有弟弟的
        if (newFiber) {
            if (newChildIndex == 0) {//如果当前索引为0，说明这是太子
                currentFiber.child = newFiber;
            } else {
                prevSibling.sibling = newFiber;//让太子的sibling弟弟指向二皇子
            }
            prevSibling = newFiber;
        }
        newChildIndex++;
    }

}
//循环执行工作 nextUnitWork
function workLoop(deadline) {
    let shouldYield = false;//是否要让出时间片或者说控制权
    while (nextUnitOfWork && !shouldYield) {
        nextUnitOfWork = performUnitOfWork(nextUnitOfWork);//执行完一个任务后
        shouldYield = deadline.timeRemaining() < 1;//没有时间的话就要让出控制权
    }
    if (!nextUnitOfWork && workInProgressRoot) {//如果时间片到期后还有任务没有完成，就需要请求浏览器再次调度
        console.log('render阶段结束');
        commitRoot();
    }
    //不管有没有任务，都请求再次调度 每一帧都要执行一次workLoop
    requestIdleCallback(workLoop, { timeout: 500 });
}
function commitRoot() {
    deletions.forEach(commitWork);//执行effect list之前先把该删除的元素删除
    let currentFiber = workInProgressRoot.firstEffect;
    while (currentFiber) {
        commitWork(currentFiber);
        currentFiber = currentFiber.nextEffect;
    }
    deletions.length = 0;//提交之后要清空deletion数组
    currentRoot = workInProgressRoot;//把当前渲染成功的根fiber 赋给currentRoot
    workInProgressRoot = null;
}
function commitWork(currentFiber) {
    if (!currentFiber) return;
    let returnFiber = currentFiber.return;
    while (returnFiber.tag !== TAG_HOST &&
        returnFiber.tag !== TAG_ROOT &&
        returnFiber.tag !== TAG_TEXT) {
        returnFiber = returnFiber.return;
    }
    let domReturn = returnFiber.stateNode;
    if (currentFiber.effectTag === PLACEMENT) {//新增加节点
        let nextFiber = currentFiber;
        /*  if (nextFiber.tag === TAG_CLASS) {
                    return;
                } */
        // 如果要挂载的节点不是DOM节点，比如说是类组件Fiber,一直找第一个儿子，直到找到一个真实DOM节点为止
        while (nextFiber.tag !== TAG_HOST && nextFiber.tag !== TAG_TEXT) {
            nextFiber = currentFiber.child;
        }
        domReturn.appendChild(nextFiber.stateNode);
    } else if (currentFiber.effectTag === DELETION) {//删除节点
        return commitDeletion(currentFiber, domReturn);
    } else if (currentFiber.effectTag === UPDATE) {
        if (currentFiber.type === ELEMENT_TEXT) {
            if (currentFiber.alternate.props.text != currentFiber.props.text)
                currentFiber.stateNode.textContent = currentFiber.props.text;
        } else {
            updateDOM(currentFiber.stateNode,
                currentFiber.alternate.props, currentFiber.props);
        }
    }
    currentFiber.effectTag = null;
}
function commitDeletion(currentFiber, domReturn) {
    if (currentFiber.tag == TAG_HOST || currentFiber.tag == TAG_TEXT) {
        domReturn.removeChild(currentFiber.stateNode);
    } else {
        commitDeletion(currentFiber.child, domReturn)
    }
}
/**
    workInProgressFiber = currentFiber;
    hookIndex = 0;
    workInProgressFiber.hooks = [];
 */
export function useReducer(reducer, initialValue) {
    let newHook = workInProgressFiber.alternate && workInProgressFiber.alternate.hooks
        && workInProgressFiber.alternate.hooks[hookIndex];
    if (newHook) {
        newHook.state = newHook.updateQueue.forceUpdate(newHook.state);
    } else {
        newHook = {
            state: initialValue,
            updateQueue: new UpdateQueue()// 空的更新队列
        }
    }
    const dispatch = action => {//{type:'ADD'}
        let payload = reducer ? reducer(newHook.state, action) : action;
        newHook.updateQueue.enqueueUpdate(
            new Update(payload)
        );
        scheduleRoot();
    }
    workInProgressFiber.hooks[hookIndex++] = newHook;
    return [newHook.state, dispatch];
}
export function useState(initialValue) {
    return useReducer(null, initialValue);
}
//react告诉 浏览器，我现在有任务请你在闲的时候，
//有一个优先级的概念。expirationTime 
requestIdleCallback(workLoop, { timeout: 500 });
```

### updateQueue.js

```javascript
export class Update {
    constructor(payload) {
        this.payload = payload;
    }
}
//数据结构是一个单链表
export class UpdateQueue {
    constructor() {
        this.firstUpdate = null;
        this.lastUpdate = null;
    }
    enqueueUpdate(update) {
        if (this.lastUpdate === null) {
            this.firstUpdate = this.lastUpdate = update;
        } else {
            this.lastUpdate.nextUpdate = update;
            this.lastUpdate = update;
        }
    }
    forceUpdate(state) {
        let currentUpdate = this.firstUpdate;
        while (currentUpdate) {
            let nextState = typeof currentUpdate.payload === 'function' ? currentUpdate.payload(state) : currentUpdate.payload;
            state = { ...state, ...nextState };
            currentUpdate = currentUpdate.nextUpdate;
        }
        this.firstUpdate = this.lastUpdate = null;
        return state;
    }
}
```
