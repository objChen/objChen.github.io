---
title: JS Promise&Generator&Co
date: 2018-06-05 01:41:57
tags: JS
---

## Generator 和 co

首先我们简单了解下 `generator`：

```
// 定义一个 generators
function* foo(){
    yield console.log("bar");
    yield console.log("baz");
}

var g = foo();
g.next(); // prints "bar"
g.next(); // prints "baz"
```

简单来说，`generator` 实现了状态暂停/函数暂停 —— 通过 `yield` 关键字暂停函数，并返回当前函数的状态。

`co` 实现了 `generator` 的 **自动执行**，我们使用 `co` 和 `Promise` 修改上面的代码：

```
var co = require('co');

function* foo() {
    yield Promise.resolve(console.log("bar"));
    yield Promise.resolve(console.log("baz"));
}

var co = require('co');
co(foo);
```

有人可能要说 "我自己写个循环执行 next 不也可以么？ 为什么一个循环还要依赖一个库？"

`co` 有个使用条件：`generator` 函数的 `yield` 命令后面，只能是 [Thunk](https://tasaid.com/link?url=http%3A%2F%2Fwww.ruanyifeng.com%2Fblog%2F2015%2F05%2Fthunk.html) 函数或 `Promise` 对象。

正是这个条件，让 `co` 强悍无比。

## Callback

我们一步一步来看异步，首先使用 `回调函数/Callback` 的方式封装一个常见的 `ajax` 异步任务：

```
function ajax(q, callback) {
    var xhr = new XMLHttpRequest();
    xhr.onreadystatechange = function () {
        if (xhr.readyState == 4 && xhr.status == 200) {
            callback(xhr.responseText);
        }
    }
    xhr.open("GET", "query?q=" + q);
}
```

我们使用 `回调函数` 的方式连续发 2 条请求：

```
ajax('foo', function (foo) {
    console.log(foo);
    ajax('bar', function (bar) {
        console.log(bar);
    });
});
```

这是 js 中最典型的异步处理方案。

## Promise

再使用 Promise 封装异步 ajax，让回调函数扁平化：

```
function ajax(q, callback) {
    // 使用 Promise 封装
    return new Promise(function (resolve) {
        var xhr = new XMLHttpRequest();
        xhr.onreadystatechange = function () {
            if (xhr.readyState == 4 && xhr.status == 200) {
                resolve(xhr.responseText);
            }
        }
        xhr.open("GET", "query?q=" + q);
    });
}
```

然后修改请求代码，扁平化异步代码：

```
ajax('foo')
    .then(function (foo) {
        console.log(foo);
        return ajax('bar')
    })
    .then(function (bar) {
        console.log(bar);
    });
```

## co

最后，让我们见一下 `co` 的强悍之处吧。我们使用 `co.js` 来修改请求代码：

```
var co = require('co');

co(function* () {
    var foo = yield ajax('foo');
    console.log(foo);

    var bar = yield ajax('bar');
    console.log(bar);
});
```

最终我们的异步任务，在代码中同步化了。

对于异步代码来说，回调函数是最基础的方案，带来的弊端也显而易见。`Promise` 让代码扁平化，而 `co` 让代码同步化。