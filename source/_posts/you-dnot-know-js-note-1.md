layout: post
title: "你不知道的Javascript(中卷)读书笔记"
date: 2018-6-02 14:04
comments: true
tags:
	- Javascript
---
## 1. JS的类型转换是与其他语言不一样。
- 其中有toString,toNumber,toBoolean；并且不同操作符使用不同的类型转换
- 尽量不使用 == 进行相等判断，使用恒等于 === 进行判断，可以避免以为类型转换导致意外问题。比如`2==true`的结果是`false`，原因是`true`会进行toNumber变为1。

<!--more-->

## 2. Promise与async/await：
- 设定回调不具有信任性，调用次数过多或者过少等。Promise可以解决信任性问题（单决议，不可变）
- 具有顺序的回调会导致层级嵌套。Promise的then可以解决
- 回调不符合逻辑顺序。async/await+Promise可以解决

```javascript
    // 返回一个Promise
    function foo(){
     return new Promise(function(resolve,reject){
       var xhr=new XMLHttpRequest();
       xhr.open("GET",location.href,true);
       xhr.onreadystatechange = function(){
         if(this.readyState == 4 && this.status == 200){
            // 异步请求结束就对Promise决议
            resolve(this.responseText);
        }
       }
       xhr.send();
     });
    }


    async function main(){
      // 等待Promise决议完成
      var text = await foo();
      // 输出决议的结果
      console.log(text);
    }

    // main中的代码将本来是异步的，写成了同步的写法
    // 1. Promise保证决议的可信任性
    // 2. async/await实现了同步的写法
    main();

```

## 3. 重构腾讯选服页的思考
    - 封装支持重试机制的异步请求服务
    - JS 的this绑定比较晦涩难懂，可读性较差，可赋值给一个 self的变量
    - 使用重试机制加载JS脚本，避免网络问题
    - 避免使用全局的命名空间，可以使用一个对象包含，也易于管理
    - 可以使用自执行函数避免污染全局空间 和参数传入外部依赖模块
    - 腾讯玩吧页面缓存问题，可以把基础不变的写在页面，改动多的则封装到JS脚本文件（每次都重新加载，已解决页面没有更新问题）
    - 页面错误上报是必须的 wondow.onerror 遇到错误之后就 settimeout来处理避免页面阻塞，内部还需要try/catch异常，避免自己的错误被自己捕获...

```
(function(window){
    // 通过参数注入外部依赖的模块 window ，解耦
    window.something = 1; // 显式给全局空间设定属性
    var BaseService = function(){
        var self = this;
    }
    var service = new BaseService();
})(window)
```

## 4. Promise + 生成器（runner）

```
// 这个比较复杂，但是可以深入Promise + 生成器模式
function run(gen) {
  var args = [].slice.call(arguments, 1);
  // var it = gen.apply(this, args);
  // 直接使用ES6的对象解构
  var it = gen(...args);
  return Promise.resolve().then(function handleNext(value) {
    var next = it.next(value);
    // 对next得知进行Promise处理
    return (function handleResult(next) {
      // 生成器结束
      if (next.done) {
        return next.value;
      } else if (typeof next.value == "function") {
        // js没有elseif哟
        // 我们收到返回的thunk了吗？
        return new Promise( function(resolve,reject){
            next.value( function(err,msg) {
                if (err) {
                    reject( err );
                } else {
                    resolve( msg );
                }
            });
        }).then(
          handleNext,
          function handleError(error) {
            // 如果是yield了一个拒绝的Promise 那么向生成器内部抛出异常,并且处理之后的yield/return
            return Promise.resolve(it.throw(error)).then(handleResult);
          }
        );

      } else {
        // Promise + 生成器还没有结束
        return Promise.resolve(next.value).then(
          handleNext,
          function handleError(error) {
            // 如果是yield了一个拒绝的Promise 那么向生成器内部抛出异常,并且处理之后的yield/return
            return Promise.resolve(it.throw(error)).then(handleResult);
          }
        );
      }
    })(next);
  });
}

function get(url) {
  return new Promise(function(resolve, reject) {
    var xhr = new XMLHttpRequest();
    xhr.open("GET", url, true);
    xhr.onreadystatechange = function() {
      if (this.readyState != 4) {
        return;
      }
      if (this.status == 200) {
        // 异步请求结束就对Promise决议
        resolve(this.responseText);
      } else {
        reject(event);
      }
    }
    xhr.send();
  });
}

/*
 * yield一个拒绝的Promise会向生成器 抛出错误的, 如果是return则直接返回
 */
function* main(url) {
  try {
    var text = yield get(url);
    console.log(text);
  } catch (e) {
    console.log('外部触发生成器报错:', e);
    yield Promise.reject('error finish!');
    // 这里已经没有 try，所以不会往后执行
    console.log('never run here!');
  }
  return 'get run success!';
}

// 自动调用生成器 并且返回Promise
var a = run(main, 'http://www.baidu.com');
```

## 5. thunk
```
// 直接产生 thunk
function thunkify(fn) {
    var args = [].slice.call( arguments, 1 );
    return function(cb) {
        args.push( cb );
        return fn.apply( null, args );
    };
}

// 产生一个Thunkory,然后调用Thunkory产生thunk
function thunkify(fn) {
    return function() {
        var args = [].slice.call( arguments );
        return function(cb) {
            args.push( cb );
            return fn.apply( null, args );
        };
    };
}
```

上面的跟`promisory`很像

```
// polyfill安全的guard检查
if (!Promise.wrap) {
    Promise.wrap = function(fn) {
        return function() {
            var args = [].slice.call(arguments);
            return new Promise(function(resolve, reject) {
                fn.apply(null, args.concat(function(err, v) {
                    if (err) {
                        reject(err);
                    } else {
                        resolve(v);
                    }
                }));
            });
        };
    };
}
```

对比:

```
// 对称：构造问题提问者
var fooThunkory = thunkify(foo);
var fooPromisory = promisify(foo);
// 对称：提问
var fooThunk = fooThunkory(3, 4);
var fooPromise = fooPromisory(3, 4);
// 得到答案
fooThunk(function(err, sum) {
    if (err) {
        console.error(err);
    } else {
        console.log(sum);
        // 7
    }
});
// 得到promise答案
fooPromise.then(function(sum) {
    console.log(sum);
    // 7
}, function(err) {
    console.error(err);
});
```

* 可以把 thunk 和 promise 大体上对比一下：它们的特性并不相同，所以并不能直接互换。Promise 要比裸 thunk 功能更强、更值得信任。
* 但从另外一个角度来说，它们都可以被看作是对一个值的请求，回答可能是异步的。
