面试季临近，`Event Loop` 这个概念也开始热了，博客上到处都在写，面试到处都在问，于是我也借此机会查阅了一些相关资料弥补自己的知识盲区，把自己学习完之后对于浏览器的 `Event Loop` 写一篇个人总结，有理解不对之处欢迎大佬指正~
> 这篇暂不做 `Node` 环境下的 `Event Loop` 的讨论


## JavaScript 里的栈和队列
在说 `Event Loop` 之前，我们要先理解栈（`stack`）和队列（`queue`）的概念。

栈和队列，两者都是线性结构，但是栈遵循的是后进先出(`last in first off，LIFO`)，开口封底。而队列遵循的是先进先出 (`fisrt in first out，FIFO`)，两头通透。

![](https://user-gold-cdn.xitu.io/2019/3/7/16956fe474a55bbf?w=643&h=243&f=png&s=17830)

`Event Loop`得以顺利执行，它所依赖的容器环境，就和这两个概念有关。

我们知道，在 `js` 代码执行过程中，会生成一个当前环境的“<font color="red">执行上下文</font>（ <span style="background-color:#333;color:white;padding:4px;border-radius: 4px">执行环境 / 作用域</span>）”，用于存放当前环境中的变量，这个上下文环境被生成以后，就会被推入`js`的执行栈。一旦执行完成，那么这个执行上下文就会被执行栈弹出，里面相关的变量会被销毁，在下一轮垃圾收集到来的时候，环境里的变量占据的内存就能得以释放。

这个执行栈，也可以理解为`JavaScript`的单一线程，所有代码都跑在这个里面，以同步的方式依次执行，或者阻塞，这就是同步场景。

那么异步场景呢？显然就需要一个独立于“执行栈”之外的容器，专门管理这些异步的状态，于是在“主线程”、“执行栈”之外，有了一个 `Task` 的队列结构，专门用于管理异步逻辑。所有异步操作的回调，都会暂时被塞入这个队列。`Event Loop` 处在两者之间，扮演一个大管家的角色，它会以一个固定的时间间隔不断轮询，当它发现主线程空闲，就会去到 `Task` 队列里拿一个异步回调，把它塞入执行栈中执行，一段时间后，主线程执行完成，弹出上下文环境，再次空闲，`Event Loop` 又会执行同样的操作。。。依次循环，于是构成了一套完整的事件循环运行机制。

![](https://user-gold-cdn.xitu.io/2019/3/7/169571c6f705e235?w=640&h=489&f=png&s=114129)
> 上图是笔者在 Google 上找的，比较简洁地描绘了整个过程，只不过其中多了 `heap` （堆）的概念，堆和栈，简单来说，堆是留给开发者分配的内存空间，而栈是原生编译器要使用的内存空间，二者独立。

## microtask 和 macrotask 
如果只想应付普通点的面试，上面一节的内容就足够了，但是想要答出下面的这条面试题，就必须再次深入 `Event Loop` ，了解任务队列的深层原理：`microtask`（微任务）和 `macrotask`（宏任务）。

### 面试题
```
// 请给出下面这段代码执行后，log 的打印顺序
console.log('script start')

async function async1() {
  await async2()
  console.log('async1 end')
}
async function async2() {
  console.log('async2 end')
}
async1()

setTimeout(function() {
  console.log('setTimeout')
}, 0)

new Promise(resolve => {
  console.log('Promise')
  resolve()
})
  .then(function() {
    console.log('promise1')
  })
  .then(function() {
    console.log('promise2')
  })

console.log('script end')

// log 打印顺序：script start -> async2 end -> Promise -> script end -> promise1 -> promise2 -> async1 end -> setTimeout
```

如果只有一个单一的 `Task` 队列，就不存在上面的顺序问题了。但事实情况是，浏览器会根据任务性质的不同，将不同的任务源塞入不同的队列中，任务源可以分为<font color="red">微任务</font>（`microtask`） 和<font color="red">宏任务</font>（`macrotask`），介于浏览器对两种不同任务源队列中回调函数的读取机制，造成了上述代码中的执行顺序问题。

![](https://user-gold-cdn.xitu.io/2019/3/9/16962288aa68a196?w=3161&h=1274&f=png&s=105169)
> 上图摘自《掘金小册：前端面试之道》

### 过程解析
让我们首先来分析一下上述代码的执行流程：

1. `JavaScript` 解析引擎在脚本开头碰到了 `console.log` 于是打印 `script strt`
2. 解析引擎解析至 `async1()` ，`async1` 执行环境被推入执行栈，解析引擎进入 `async1` 内部
3. 引擎发现 `async1` 内部调用了 `async2`，于是继续进入 `async 2`，并将 `async 2` 执行环境推入执行栈
4. 引擎碰到 `console.log`，于是打印 `async2  end`
5. `async2` 函数执行完成，返回了一个 `Promise.resolve(undefined)`，<font color="red">此时，该回调被推入 <strong>microtask</strong> </font>，`async1` 函数中的执行权被让出，等待主线程空闲
6. 引擎解析至 `setTimeout`，<font color="red">等待 0ms 后将其回调推入 <strong>macrotask</strong></font>，执行权继续让出
7. 引擎指针继续下移，直到碰到了 `Promise`，解析进入注入函数的内部，碰到 `console.log`，于是打印 `Promise`，再往下，碰到了 `resolve`，<font color="red">此时，该回调被推入 <strong>microtask</strong> </font>，执行权被让出
8. 引擎继续往下，碰到 `console.log`，打印完 `script end`
9. 至此，主线程空闲，<font color="red"><strong>Event Loop</strong> 事件循环启动，开始从 <strong>microtask</strong> 里拿出 promise 回调，放入主线程执行</font>，首先拿出最早注入的 `async2` 的 `Promise.resolve(undefined)`执行，<font color="red">此时 <strong>await</strong> 操作符解析该表达式，得到结果 <strong>undefined</strong>，并将 <strong>async1 [Promise] 函数</strong> 标志为 <strong>resolve</strong> 状态，将 <strong>await</strong> 后面的代码作为回调，继续推入 <strong>microtask</strong>，等待执行，执行权被让出</font>
10. 此时主线程没有可执行的代码，再次空闲，<font color="red"><strong>Event Loop</strong> 启动，去 <strong>microtask</strong> 中拿到之前的 `new Promise` 回调，放入主线程执行，打印结果 `promise1` 和 `promise2`</font>
11. 主线程空闲，`Event Loop` 去 `microtask` 里拿 `aysnc1` 的回调，打印出 `async1 end`
12. 最后，主线程空闲，`microtask` 队列空，`Event Loop` 去 `macrotask` 里拿到 `setTimeout` 的回调，放入主线程，打印最后的 `setTimeout`


### 常见的微任务和宏任务
<font color="red"><strong>微任务</strong></font>包括 `process.nextTick` ，`promise` ，`MutationObserver`，其中 `process.nextTick` 为 Node 独有。

<font color="red"><strong>宏任务</strong></font>包括 `script` ， `setTimeout` ，`setInterval` ，`setImmediate` ，`I/O` ，`UI rendering`。

## 关于 `async` 和 `await` 
上述的面试题里，大部分逻辑解释下来都很好懂，除了一处，就是 `await` 后的 `console.log(async1 end)` 和 `new Promise` `resolve` 后的回调，到底哪个先执行？由于浏览器底层的解析引擎实现不同，对于不同的浏览器其结果可能不一样（<font color="red">最新版的 chrome 浏览器对于 await 的处理变快了，async1 会先于 promise 1 打印</font>）。

但是相比于这个执行顺序，上述题目衍生出的一个更重要的问题，是对于 `async/await` 的理解。

对于 `async/await` 的更详细解释，大家可以参照这篇 [理解 JavaScript 的 async/await](https://segmentfault.com/a/1190000007535316)，懒得看的童鞋可以看下面的结论：

1. 一个函数，只要被 `async` 关键字包装过，就会返回一个 `promise`，如果该函数有返回值，那么这个返回值就会作为 `then` 处理的 `response` ，如果没有返回值，那么 `then` 就处理 `undefined`
2. `await` 表达式，只能用在被 `async` 包装过的函数里，否则会报错
3. `await` 表达式后接的函数返回值，类型可以为 `promise`，或者其他任何的值，`await` 后的代码在当前执行环境下，会被阻塞至拿到该函数调用后的结果，等拿到结果后，会将 `await` 后面的代码继续包装成新的 `promise`，并将之前拿到的结果作为 `response` 传入其中，同时让出线程控权
4. `async/await` 本质上是 `Generator` 的语法糖

## 宏任务与微任务，哪个先执行？

关于这个问题，众说纷纭，很多大佬都说是宏任务先于微任务执行，但是代码的运行结果却显示是微任务先执行。
先看看大佬们的解释：

> 这里很多人会有个误区，认为微任务快于宏任务，其实是错误的。因为宏任务中包括了 `script` ，浏览器会先执行一个宏任务，接下来有异步代码的话才会先执行微任务。
> <span style="float: right"><font color="blue">《掘金小册：前端面试之道》</font></span>

也就是说，`Event Loop` 抓取回调的逻辑是先执行宏任务，再执行微任务，再执行宏任务。。。以此循环，本质上来说，当前执行栈里的代码都属于宏任务，于是等待执行栈清空，宏任务执行完成，浏览器回去 `microtask` 里抓取微任务来执行，除非 `microtask` 里没有，才会去 `macrotask` 抓取任务执行。


