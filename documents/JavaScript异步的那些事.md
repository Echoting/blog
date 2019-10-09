
# 浏览器中的event-loop
起因是一道笔试题：
```
    async function a1() {
        console.log('a1 start');
        await a2();
        console.log('a1 end');
    }
    
    async function a2() {
        console.log('a2');
    }

    console.log('script start');
    
    setTimeout(() => {
        console.log('setTimeout');
    }, 0);

    Promise.resolve().then(() => {
        console.log('promise1');
    });

    a1();

    let promise2 = new Promise(resolve => {
        resolve('promise2.then');
        console.log('promise2');
    });

    promise2.then(res => {
        console.log(res);
        Promise.resolve().then(() => {
            console.log('promise3');
        });
    });

    console.log('script end');
```
看到这里，我们发现了几个关键字，async、 await、 setTimeout、 promise、 promise.then，也就是说，这道题是考察的JavaScript的Event Loop；因此，接下来我们来了解一下JavaScript的事件循环；

## 事件循环
- JavaScript语言的一大特色就是单线程，也就是同一时间只能做一件事；
- html有规范称：
> To coordinate events, user interaction, scripts, rendering, networking, and so forth, user agents must use event loops as described in this section. There are two kinds of event loops: those for browsing contexts, and those for workers.

> 为了协调事件、用户交互、脚本、UI 渲染和网络处理等行为，防止主线程的不阻塞，Event Loop 的方案应用而生。Event Loop 包含两类：一类是基于 Browsing Context，一种是基于 Worker。二者的运行是独立的，也就是说，每一个 JavaScript 运行的"线程环境"都有一个独立的 Event Loop，每一个 Web Worker 也有一个独立的 Event Loop。

- 我们这里先了解一下Browsing Context下的事件循环；

## 任务队列
![](http://pik.internal.baidu.com/2019/07/18/c426e1a937f71b1364621d5101f732ae.png)
- 如上图所示，左边的栈存储的是同步任务，就是那些能立即执行、不耗时的任务，如变量和函数的初始化、事件的绑定等等那些不需要回调函数的操作都可归为这一类。
- 右边的堆用来存储声明的变量、对象。
- 下面的队列就是消息队列，一旦某个异步任务有了响应就会被推入队列中

### 异步任务: 宏任务 和 微任务
- 异步任务可分为 task（宏任务） 和 microtask（微任务） 两类，不同的API注册的异步任务会依次进入自身对应的队列中，然后等待 Event Loop 将它们依次压入执行栈中执行。
- task（宏任务）主要包含：script(整体代码)、setTimeout、setInterval、I/O、UI交互事件、postMessage、MessageChannel、setImmediate(Node.js 环境)
- microtask主要包含：Promise.then、MutaionObserver、process.nextTick(Node.js 环境);


### Promise 和 async中的立即执行
- Promise中的异步体现在then和catch中，所以写在Promise中的代码是被当做同步任务立即执行的。而在async/await中，在出现await出现之前，其中的代码也是立即执行的。那么出现了await的时候发生了什么呢？

### async 怎么处理返回值
```
async function testAsync() {
    return "hello async";
}
let result = testAsync();
console.log(result)
```
输出结果
```
Promise {<resolved>: "hello async"}
__proto__: Promise
[[PromiseStatus]]: "resolved"
[[PromiseValue]]: "hello async"
```
从输出结果可以看出async函数返回的是一个promise对象，如果在函数中return一个直接量，async 会把这个直接量通过 Promise.resolve() 封装成 Promise 对象。

如果async函数没有返回值
```
async function testAsync1() {
    console.log("hello async");
}
let result1 = testAsync1();
console.log(result1);
```
输出结果
```
Promise {<resolved>: undefined}
__proto__: Promise
[[PromiseStatus]]: "resolved"
[[PromiseValue]]: undefined
```
从输入结果可以看出，如果async函数没有返回值，那么async函数返回的是Promise.resolve(undefined)。

### await做了什么
从字面意思上看await就是等待，await 等待的是一个表达式，这个表达式的返回值可以是一个promise对象也可以是其他值。
> 很多人以为await会一直等待之后的表达式执行完之后才会继续执行后面的代码，实际上await是一个让出线程的标志。await后面的函数会先执行一遍，然后就会跳出整个async函数来执行后面js栈的代码。等本轮事件循环执行完了之后又会跳回到async函数中等待await
后面表达式的返回值，如果返回值为非promise则继续执行async函数后面的代码，否则将返回的promise放入promise队列（Promise的Job Queue）

因为async await 本身就是 promise + generator 的语法糖。所以await后面的代码是microtask(微任务)，所以
```
async function async1() {
	console.log('async1 start');
	await async2();
	console.log('async1 end');
}
```
等价于
```
async function async1() {
	console.log('async1 start');
	Promise.resolve(async2()).then(() => {
        console.log('async1 end');
    })
}
```

### 回到上面的题目

```
async function a1() {
    console.log('a1 start');  // 4. 加入宏任务1
    await a2();               
    console.log('a1 end');    // 6. 加入微任务1
}

async function a2() {
    console.log('a2');        // 5. 加入宏任务1
}

console.log('script start');  // 1. 加入宏任务1

setTimeout(() => {
    console.log('setTimeout');
}, 0);                        // 2. 加入宏任务2

Promise.resolve().then(() => {
    console.log('promise1');
});                           // 3. promise.then 加入微任务1

a1();  // 执行a1

let promise2 = new Promise(resolve => {
    resolve('promise2.then');
    console.log('promise2');  //  7. 加入宏任务1
});

promise2.then(res => {
    console.log(res);
    Promise.resolve().then(() => {
        console.log('promise3');  // 9. 加入微任务1
    });
});  // 8. 加入微任务1

console.log('script end');  // 10. 加入宏任务1
```

- 首先，事件循环从宏任务(macrotask)队列开始，这个时候，宏任务队列中，只有一个script(整体代码)任务；当遇到任务源(task source)时，则会先分发任务到对应的任务队列中去。所以，上面例子的执行如下所示：

```
宏任务1：  
console.log('script start')   // 1
console.log('a1 start')       // 4
console.log('a2')             // 5
console.log('promise2')       // 7
console.log('script end')     // 10

微任务1：
console.log('promise1')    // 3
console.log('a1 end')      // 6
console.log(res) // promise2.then    // 8
conosle.log('promise3')     // 9

宏任务2：  
console.log('setTimeout')   // 2

```

- 然后 宏任务1 -> 微任务1 -> 宏任务2
输出结果为：
```
script start
a1 start
a2
promise2
script end
promise1
a1 end
promise2.then
promise3
setTimeout
```

#### 变式一
```
async function async1() {
    console.log('async1 start');
    await async2();
    //更改如下：
    setTimeout(function() {
        console.log('setTimeout1')
    },0)
}
async function async2() {
    //更改如下：
	setTimeout(function() {
		console.log('setTimeout2')
	},0)
}
console.log('script start');

setTimeout(function() {
    console.log('setTimeout3');
}, 0)
async1();

new Promise(function(resolve) {
    console.log('promise1');
    resolve();
}).then(function() {
    console.log('promise2');
});
console.log('script end');
```

#### 变式二
```
async function async1() {
    console.log('async1 start');
    await async2();
    console.log('async1 end');
}
async function async2() {
    //async2做出如下更改：
    new Promise(function(resolve) {
    console.log('promise1');
    resolve();
}).then(function() {
    console.log('promise2');
    });
}
console.log('script start');

setTimeout(function() {
    console.log('setTimeout');
}, 0)
async1();

new Promise(function(resolve) {
    console.log('promise3');
    resolve();
}).then(function() {
    console.log('promise4');
});

console.log('script end');
```
可以试着做一下这两个变式，如果能不费吹灰之力的搞定，说明你已经了解了事件循环，反之，还需要仔细的看看网上的文档去理解；

### 参考文章
- [async/await 执行顺序详解](https://segmentfault.com/a/1190000011296839)
- [从一道题浅说 JavaScript 的事件循环](https://github.com/dwqs/blog/issues/61)
- [浏览器事件循环机制（event loop）](https://juejin.im/post/5afbc62151882542af04112d)

