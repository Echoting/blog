
# nodejs中的event-loop
在上一篇的内容中，我们简单的了解了一下浏览器环境的eventLoop，这里我们再来看一下Nodejs环境中的eventLoop。

[Node.js官方介绍](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
![](http://pik.internal.baidu.com/2019/07/30/54332db074aac479712f3baf4b034872.png)  
    从上图可以大致看出node中的事件循环顺序：
- 外部输入数据–>轮询阶段(poll)–>检查阶段(check)–>关闭事件回调阶段(close callback)–>定时器检测阶段(timer)–>I/O 事件回调阶段(I/O callbacks)–>闲置阶段(idle, prepare)–>轮询阶段（按照该顺序反复运行）…  


> Phases Overview
- timers: this phase executes callbacks scheduled by setTimeout() and setInterval().
- pending callbacks: executes I/O callbacks deferred to the next loop iteration.
- idle, prepare: only used internally.
- poll: retrieve new I/O events; execute I/O related callbacks (almost all with the exception of close callbacks, the ones scheduled by timers, and setImmediate()); node will block here when appropriate.
- check: setImmediate() callbacks are invoked here.
- close callbacks: some close callbacks, e.g. socket.on('close', ...).  
Between each run of the event loop, Node.js checks if it is waiting for any asynchronous I/O or timers and shuts down cleanly if there are not any.


- **timer阶段**：这个阶段执行timer的回调（setTimeout, setInterval）的回调；
- **I/O callbacks 阶段**：执行一些系统调用错误，比如网络通信的错误回调；
- **idle, prepare 阶段**：仅node内部使用；
- **poll 阶段**：获取新的I/O事件, 适当的条件下node将阻塞在这里；
- **check阶段**：执行 setImmediate() 的回调；
- **close callbacks 阶段**：一些close的回调，例如执行 socket 的 close 事件回调

### timer阶段
- timers 是事件循环的第一个阶段，Node 会去检查有无已过期的timer，如果有则把它的回调压入timer的任务队列中等待执行，事实上，Node 并不能保证timer在预设时间到了就会立即执行，因为Node对timer的过期检查不一定靠谱，它会受机器上其它运行程序影响，或者那个时间点主线程不空闲。比如下面的代码，setTimeout() 和 setImmediate() 的执行顺序是不确定的。

```
setTimeout(() => {
  console.log('timeout')
}, 0)

setImmediate(() => {
  console.log('immediate')
})
```

### pool阶段 

  poll 阶段主要有2个功能：
  - 处理 poll 队列的事件  
  - 当有已超时的 timer，执行它的回调函数  
  
even loop将同步执行poll队列里的回调，直到队列为空或执行的回调达到系统上限（上限具体多少未详），接下来even loop会去检查有无预设的setImmediate()，分两种情况：

1. 若有预设的setImmediate(), event loop将结束poll阶段进入check阶段，并执行check阶段的任务队列
2. 若没有预设的setImmediate()，event loop将阻塞在该阶段等待  

注意一个细节，没有setImmediate()会导致event loop阻塞在poll阶段，这样之前设置的timer岂不是执行不了了？所以，在poll阶段event loop会有一个检查机制，检查timer队列是否为空，如果timer队列非空，event loop就开始下一轮事件循环，即重新进入到timer阶段。

### check阶段
  setImmediate()的回调会被加入check队列中， 从event loop的阶段图可以知道，check阶段的执行顺序在poll阶段之后。

### 举个栗子
```
const fs = require('fs')

fs.readFile('test.txt', () => {
  console.log('readFile')
  setTimeout(() => {
    console.log('timeout')
  }, 0)
  setImmediate(() => {
    console.log('immediate')
  })
})
```
- 这段代码会先进入I/O callback 阶段，然后到check阶段，然后到timer阶段；
因此，输出结果为：
```
readFile
immediate
timeout
```
- event loop 的每个阶段都有一个任务队列;
- 当 event loop 到达某个阶段时，将执行该阶段的任务队列，直到队列清空或执行的回调达到系统上限后，才会转入下一个阶段;
- 当所有阶段被顺序执行一次后，称 event loop 完成了一个 tick;

### process.nextTick()
- process.nextTick() 会在各个事件阶段之间执行，一旦执行，要直到nextTick队列被清空，才会进入到下一个事件阶段，所以如果递归调用 process.nextTick()，会导致出现I/O starving（饥饿）的问题，比如下面例子的readFile已经完成，但它的回调一直无法执行：
```
const fs = require('fs')
const starttime = Date.now()
let endtime

fs.readFile('text.txt', () => {
  endtime = Date.now()
  console.log('finish reading time: ', endtime - starttime)
})

let index = 0

function handler () {
  if (index++ >= 1000) return
  console.log(`nextTick ${index}`)
  process.nextTick(handler)
  // console.log(`setImmediate ${index}`)
  // setImmediate(handler)
}

handler()

console.log('333')
```
- node 执行完所有的同步任务，接下来就会执行process.nextTick的任务队列

### 微任务
![](http://pik.internal.baidu.com/2019/08/01/4359667f6fbc7364f07541dd47513ed4.png)
- 跟浏览器不同的是，在每个阶段完成后，，microTask队列就会被执行
```
setTimeout(()=>{
    console.log('timer1')

    Promise.resolve().then(function() {
        console.log('promise1')
    })
}, 0)

setTimeout(()=>{
    console.log('timer2')

    Promise.resolve().then(function() {
        console.log('promise2')
    })
}, 0)
```
按照上面的图示，期待的输出结果为：
```
timer1
timer2
promise1
promise2
```

but，我们发现，用node 10运行结果确实是这样的，但是node 11的运行结果竟然是：
```
timer1
promise1
timer2
promise2
```
- 原因应该是node做了修改
- 现在 node11 在 timer 阶段的 setTimeout,setInterval…和在 check 阶段的 immediate 都在 node11 里面都修改为一旦执行一个阶段里的一个任务就立刻执行微任务队列。

### 参考文章
- http://lynnelv.github.io/js-event-loop-nodejs
- https://segmentfault.com/a/1190000013861128
- https://juejin.im/post/5bd72beb5188257e4a681cf6
- https://blog.fundebug.com/2019/04/02/nodejs-event-loop-has-changed/