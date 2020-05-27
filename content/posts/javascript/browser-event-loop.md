---
title: 浏览器事件循环
date: 2018-05-02 19:43:13
tags: ["javascript"]
---

学习 Javascript 的过程中，事件循环是一个绕不开的话题。异步？？？，事件模型？？？任务？？？事件循环？？？这些都是什么意思？

<!--more-->

https://www.w3.org/TR/html5/webappapis.html#event-loops 手册上是这么说的: 事件循环其实是浏览器和事件, 用户交互，渲染，网络请求等等协作的机制, 时间循环有一个或者多个任务队列，这些任务包括解析HTML，处理回调函数，网络请求等等。

<div style="display: flex; justify-content: center; align-items: center;">
![黑人问号](/images/???.jpeg)
</div>

在看了[Philip Roberts在JSConf上面的session](https://www.youtube.com/watch?v=8aGhZQkoFbQ)之后:
> 我好像有点懂了


## 栈

```
function foo() {
    console.log("foo");
}

function bar() {
    foo();
}

bar()
```
[可视化](http://latentflip.com/loupe/?code=ZnVuY3Rpb24gZm9vKCkgewogICAgY29uc29sZS5sb2coImZvbyIpOwp9CgpmdW5jdGlvbiBiYXIoKSB7CiAgICBmb28oKTsKfQoKYmFyKCk%3D!!!PGJ1dHRvbj5DbGljayBtZSE8L2J1dHRvbj4%3D)
js读取文件，调用函数就压到栈里面，可以执行返回就执行并且退出栈，直到所有都运行完成，很简单。
js在运行的过程中，更确切一些，在栈中有代码需要运行的时候，用户界面是无法响应其他事情的。


## 回调队列

```
function foo() {
    console.log("foo");
}

function bar() {
    setTimeout(foo(), 1000);
}

bar()
```

[可视化](http://latentflip.com/loupe/?code=ZnVuY3Rpb24gZm9vKCkgewogICAgY29uc29sZS5sb2coImZvbyIpOwp9CgpmdW5jdGlvbiBiYXIoKSB7CiAgICBzZXRUaW1lb3V0KGZvbygpLCAxMDAwKTsKfQoKYmFyKCk%3D!!!PGJ1dHRvbj5DbGljayBtZSE8L2J1dHRvbj4%3D)
setTimeout是浏览器实现的API，他的计时器在计算到了1000ms之后会把回调函数放入事件循环的队列里面，<strong>在栈为空的情况下</strong>，浏览器会在队列拿出第一个来压到栈里面执行。这就是为什么setTimeout不是严格的1s后执行，而是至少1s后执行。

看看这个例子:
```
for (var i = 0; i < 10; i++) {
    console.log("Hello ", i);
}

for (var i = 0; i < 10; i++) {
    setTimeout(() => {
        console.log("Hello ", i);
    }, 0);
}
```
代码想要做的事情都是一样的，打印 Hello 1 - 9， 但实际效果是不同，第一段是持续运行，会阻止用户界面的响应，而第二段代码是把 console.log 放入队列中，这就给了浏览器在运行代码过程中响应用户的机会。 <s>这里还有一个梗，看出来了吗？</s>

再读一遍手册上面的定义，是不是已经理解事件循环了。
<div style="display: flex; justify-content: center; align-items: center;">
![](/images/harden-eye.jpeg)
</div>


## 当然事情并没有那么简单

来看看另外一个例子
```javascript
console.log('script start');

setTimeout(function() {
  console.log('setTimeout');
}, 0);

Promise.resolve().then(function() {
  console.log('promise1');
}).then(function() {
  console.log('promise2');
});

console.log('script end');
```

我第一次看到这段代码回答就是 
```
script start
script end
setTimeout
promise1
promise2
```
我这么说肯定这个答案是不对的，不对的原因是什么？

<div style="display: flex; justify-content: center; align-items: center;">
![](/images/not-easy.jpeg)
</div>

setTimeout 会把回调函数放入队列里面，<strong>作为Task</strong>, 而Promise也会放入队列里面，<strong>作为Microtask</strong>，Microtask比Task拥有更高优先级，在栈为空后要立即执行Microtask，而且如果队列中有Microtask，必须执行完所有的Microtask才能返回去执行Task。
除了Promise，MutationObserver也是Microtask。

## Boss战
<div style="display: flex; justify-content: center; align-items: center;">
![灭霸](/images/mieba.jpg)
</div>


```
# HTML
<div class="outer">
  <div class="inner"></div>
</div>

# JS
// Let's get hold of those elements
var outer = document.querySelector('.outer');
var inner = document.querySelector('.inner');

// Let's listen for attribute changes on the
// outer element
new MutationObserver(function() {
  console.log('mutate');
}).observe(outer, {
  attributes: true
});

// Here's a click listener…
function onClick() {
  console.log('click');

  setTimeout(function() {
    console.log('timeout');
  }, 0);

  Promise.resolve().then(function() {
    console.log('promise');
  });

  outer.setAttribute('data-random', Math.random());
}

// …which we'll attach to both elements
inner.addEventListener('click', onClick);
outer.addEventListener('click', onClick);
```

## 测试

<style>
    .outer {
        background: grey;
        height: 100px;
        width: 100px;
        display: flex;
        justify-content: center;
        align-items: center;
        cursor: pointer;
        margin-bottom: 36px;
    }

    .inner {
        background: green;
        width: 50px;
        height: 50px;
    }

    #result-panel {
        outline: none;
        resize: none;
        font-size: 1.1em;
    }
</style>

<div class="outer">
    <div class="inner"></div>
</div>

点击绿色的方块查看结果
<textarea id="result-panel" wrap="off" rows="8"></textarea>

所以你答对了吗？


<script>
var outer = document.querySelector('.outer');
var inner = document.querySelector('.inner');

function log(msg) {
    $('#result-panel').append(msg);
    $('#result-panel').append("\n");
}

new MutationObserver(function() {
  log('mutate');
}).observe(outer, {
  attributes: true
});


function onClick() {
  log('click');

  setTimeout(function() {
    log('timeout');
  }, 0);

  Promise.resolve().then(function() {
    log('promise');
  });

  outer.setAttribute('data-random', Math.random());
}

inner.addEventListener('click', onClick);
outer.addEventListener('click', onClick);
</script>





