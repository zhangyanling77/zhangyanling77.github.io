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
