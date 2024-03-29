# 从面试题入手，浅析JavaScript单线程执行机制

## 一道面试题

在下面的代码中，数字 1-4 会以什么顺序输出？为什么会这样输出？

```
(function() {
    console.log(1); 
    setTimeout(function(){console.log(2)}, 1000); 
    setTimeout(function(){console.log(3)}, 0); 
    console.log(4);
})();
```

这道面试题的答案是`1 4 3 2`，相信很多人都能做对。

<br>

## 异步任务执行机制

JavaScript语言的一大特点就是**单线程**。

所有任务可以分成两种，一种是**同步任务**（synchronous），另一种是**异步任务**（asynchronous）。

- **同步任务**指的是，在主线程上排队执行的任务，只有前一个任务执行完毕，才能执行后一个任务；

- **异步任务**指的是，不进入主线程、而进入**任务队列**（task queue）的任务，只有主线程通知"任务队列"，某个异步任务可以执行了，该任务才会进入主线程执行。


具体来说，异步执行的运行机制如下。（同步执行也是如此，因为它可以被视为没有异步任务的异步执行。）

1. 所有同步任务都在主线程上执行，形成一个**执行栈**（execution context stack）。

2. 主线程之外，还存在一个**任务队列**（task queue）。只要异步任务有了运行结果，就在**任务队列**之中放置一个事件。

3. 一旦**执行栈**中的所有同步任务执行完毕，系统就会读取**任务队列**，看看里面有哪些事件, 哪些对应的异步任务，于是结束等待状态，进入执行栈，开始执行。

4. 主线程不断重复上面的第三步。

<br>

## 事件及回调函数

**任务队列**中的事件，除了IO设备的事件以外，还包括一些用户产生的事件（比如鼠标点击、页面滚动等等）。只要**指定过回调函数**，这些事件发生时就会进入**任务队列**，等待主线程读取。

所谓回调函数（callback），就是那些会被主线程挂起来的代码。异步任务必须指定回调函数，当主线程**开始执行异步任务，就是执行对应的回调函数**。

任务队列是一个**先进先出**的数据结构，排在前面的事件，优先被主线程读取。主线程的读取过程基本上是自动的，只要执行栈一清空，任务队列上第一位的事件就自动进入主线程。但是，由于存在**定时器**功能，主线程**首先要检查一下执行时间，某些事件只有到了规定的时间，才能返回主线程**。

<br>

## 定时器

`setInterval()`、 `setTimeout()`只是将事件插入了**任务队列**，必须等到当前代码（执行栈）执行完，主线程才会去执行它指定的回调函数。要是当前代码耗时很长，有可能要等很久，所以并没有办法保证，回调函数一定会在定时器指定的时间执行。

<br>

## 简单总结

一段js代码（里面可能包含一些`setTimeout`、鼠标点击、`ajax`等事件），从上到下开始执行，遇到`setTimeout`、鼠标点击等事件，异步执行它们，此时并不会影响代码主体继续往下执行(当线程中没有执行任何同步代码的前提下才会执行异步代码)；一旦异步事件执行完，回调函数返回，将它们按次序加到任务队列中。

当执行栈清空后，主线程开始按照次序读取任务队列中的回调函数，同时检查其是否满足规定的执行时间（如定时器），满足了就开始执行。

要注意的是，如果主体代码没有执行完的话，是永远也不会触发任务队列中的`callback`的。


<br>


## 再看一道面试题

下面的代码输出什么？

```
console.log('script start');

setTimeout(function() {
  console.log('setTimeout');
}, 0);

Promise.resolve().then(function() {
  console.log('promise1');
}).then(function() {
  console.log('promise2');
});

new Promise(function (resolve) {
  console.log("new");
});

console.log('script end');
```

答案是:
```
script start
new
script end
promise1
promise2
setTimeout
```

这道题就与时俱进，加入了ES6的`Promise`，难度上升一个级别。

<br>

## 宏任务和微任务

前面提到，js中任务分为同步任务和异步任务，这是按执行方式划分的。

如果根据性质把所有任务放到执行环境中来划分，其实可以分为**宏任务**(task)和**微任务**(microtask)。

**宏任务**：`script（全局任务）`、`setTimeout`、 `setInterval`、`setImmediate`、`I/O`、`UI rendering`

**微任务**：`process.nextTick`、`Promise`、`Object.observe`、`MutationObserver`


任务队列也可分为**宏任务队列**和**微任务队列**。

<br>

## 执行顺序

1. `script`全局任务先进入**函数调用栈**；

2. 在执行期间遇到任何其他**宏任务**（如`setTimeout`），就把其他任务放进宏任务队列中；

3. 等到函数调用栈的所有内容出栈后，然后执行**微任务队列**；

4. 接着从**宏任务队列中**取出任务进入函数调用栈执行；

5. 完毕后再执行微任务队列；

6. 不断重复`4~5`这个过程，直到宏任务队列清空。

上面这道面试题，`Promise.resolve().then()`自然是属于微任务，所以紧接着`script`任务出栈后执行。

需要注意的是，虽然`new Promise()`遇到了`Promise`，但由于用`new`声明，立即执行，所以在`script`任务中就会输出。


<br>

## 最终总结

由于`JavaScript`语言本身的**单线程**运行机制，为了提高执行效率，出现了**异步**任务。

当**全局任务**（宏任务）在函数调用栈中执行时，遇到其他的**宏任务**，会放进宏任务队列，遇到**微任务**，就放进微任务队列，这些任务队列中的都属于**异步任务**，会等待当前执行栈中的宏任务结束后再执行。

微任务紧接着当前宏任务结束后执行，然后宏任务队列中的任务再依次进栈，每一次宏任务执行完都会执行微任务队列中的任务。

这也可以解释一些框架中的数据监视为什么能够如此及时（`observer`）。



***

参考资料:

- [JavaScript 运行机制详解：再谈Event Loop - 阮一峰的网络日志](http://www.ruanyifeng.com/blog/2014/10/event-loop.html)

- [从setTimeout谈JavaScript运行机制 - 韩子迟](http://www.cnblogs.com/zichi/p/4604053.html)

- [Javascript定时器学习笔记 - 木的树](http://www.cnblogs.com/dojo-lzz/p/4606448.html)

- [Tasks, microtasks, queues and schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)


