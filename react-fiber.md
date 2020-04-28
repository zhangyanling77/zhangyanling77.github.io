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
### schedule.js

```javascript
import { TAG_ROOT, ELEMENT_TEXT, TAG_TEXT, TAG_HOST, PLACEMENT } from "./constants";
import { setProps } from './utils';
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
let workInProgressRoot = null;//RootFiber应用的根
export function scheduleRoot(rootFiber) {//{tag:TAG_ROOT,stateNode:container,props: { children: [element] }}
    workInProgressRoot = rootFiber;
    nextUnitOfWork = rootFiber;
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
    if (currentFiber.tag === TAG_ROOT) {
        updateHostRoot(currentFiber);
    } else if (currentFiber.tag === TAG_TEXT) {
        updateHostText(currentFiber);
    } else if (currentFiber.tag === TAG_HOST) {//原生DOM节点
        updateHost(currentFiber);
    }
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
function reconcileChildren(currentFiber, newChildren) {//[A1]
    let newChildIndex = 0;//新子节点的索引
    let prevSibling;//上一个新的子fiber
    //遍历我们的子虚拟DOM元素数组，为每个虚拟DOM元素创建子Fiber
    while (newChildIndex < newChildren.length) {
        let newChild = newChildren[newChildIndex];//取出虚拟DOM节点[A1]{type:'A1'}
        let tag;
        if (newChild.type == ELEMENT_TEXT) {
            tag = TAG_TEXT;//这是一个文本节点
        } else if (typeof newChild.type === 'string') {
            tag = TAG_HOST;//如果是type是字符串，那么这是一个原生DOM节点 "A1" div
        }//beginWork创建fiber 在completeUnitOfWork的时候收集effect
        let newFiber = {
            tag,//TAG_HOST
            type: newChild.type,//div
            props: newChild.props,//{id="A1" style={style}}
            stateNode: null,//div还没有创建DOM元素
            return: currentFiber,//父Fiber returnFiber
            effectTag: PLACEMENT,//副作用标识 render我们要会收集副作用 增加 删除 更新
            nextEffect: null,//effect list 也是一个单链表
            //effect list顺序和 完成顺序是一样的，但是节点只放那些出钱的人的fiber节点，不出钱绕过去
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
    let currentFiber = workInProgressRoot.firstEffect;
    while (currentFiber) {
        console.log('commitRoot', currentFiber.type, currentFiber.props.id, currentFiber.props.text);
        commitWork(currentFiber);
        currentFiber = currentFiber.nextEffect;
    }
    workInProgressRoot = null;
}
function commitWork(currentFiber) {
    if (!currentFiber) return;
    let returnFiber = currentFiber.return;
    let returnDOM = returnFiber.stateNode;
    if (currentFiber.effectTag === PLACEMENT) {
        returnDOM.appendChild(currentFiber.stateNode);
    }
    //currentFiber.effectTag = null;
}

// react 请求浏览器在空闲的时候来执行workLoop里的任务
// 如果超时了就立即执行，不管浏览器你还有没有剩余空闲时间
requestIdleCallback(workLoop, { timeout: 500 });
```
