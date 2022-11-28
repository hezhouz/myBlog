# 一个关于promise的问题
```text
promise 的起源 就是来解决地狱回调问题
```

### 提问下面几种 promise 有什么区别
```js
  doSomething().then(function () {
    return doSomethingElse();
  });

  doSomething().then(function () {
    doSomethingElse();
  });

  doSomething().then(doSomethingElse());

  doSomething().then(doSomethingElse);
  // 
```

#### 错误一
```text
 最开始接触promise 都是这么写的
```
```js
  reqData(data).then(res => {
      const { data, code } = res
      if (code == 200) {
       data.forEach(item => {
        reqInfoData(data.id).then(data => {
          ....
        })
        .catch(err => {
          ....
          })
       })
      }
  })
  .catch(err => {

  })
```
- 确实可以这么写， 但是是不是感觉又回到了地狱回调当中。

- 如果换成下面这种看起来是不是好多了。

```js
  reqData(data).then((...) => {
    return reqInfoData(...)
  })
  .then((...) => {
     return reqXXXData(...)
  })
  .then((...) => {
     return reqXXXData(...)
  })
  .then((...) => {
     return reqXXXData(...)
  })
```
- 每一个函数只会在前一个 promise 被调用并且完成回调后调用，并且这个函数会被前一个 promise 的输出调用， 

#### 错误二

- 用了 promise 后怎么用 forEach ?
- 当然是用 promise.all 啦!
```js
  Promise.all(this.arr.map(item => reqGetData(...))).then(res => {
    ...
  })

// Promise.all() 以一个 promise 数组为输出， 并且返回一个新的 promise。 他是异步版的for循环， 
// 并且他会将执行结果组成数返回到下一个函数， 此外如果有其中有一个失败，直接会终止直接返回错误
```

### 错误三

- 忘记使用  .catch()

这是一个常见的错误， 单纯的坚信自己写的程序不会出错，永远不会出现异常，然而很不幸的是 任何被抛出的错误都没吃掉，并且你无法在console.log 中察觉到，这类问题dbug 起来狠头疼。


### 错误四 
-  使用副作用 调用而非返回
```js
 somePromeis().then(res => {
    someOtherPromies()
 }).then(() => {
  ....
 })
```
```text 
  我们知道 每一个 promise 都会提供一个 then 函数。 当我们在 then() 函数内部 我们可以做那些事情？

  1. return 另一个 promise 
  2. return 一个同步的值（或者 undefined）
  3. throw 一个同步异常
```
##### 1. 返回另一个 promise
```js
// 我们在错误一种提到的
// 注意： 我们必须 return 下一个 promise 不然下一个 。then 会收到 undefined 
  reqData(data).then((...) => {
    return reqInfoData(...)
  }).then((...) => {
    return reqXXXXData(...)
  })
```

##### 2. 返回一个同步的值

```js
  reqData('ids').then(res => {
    if(inMemoryCache) {
      return  inMemoryCache[res,id]
    }

    return getUserById(res.id)
  }).then(res => {
    // 如果是缓存中的id res 都可以拿到！！
  })
```
- 是不是狠强！ 第二个函数不需要关心 res 是从 同步方法中获取的还是 异步方法中获取的， 并且第一个函数可以自由的返回一个同步或者异步的值

#####  3. throw 一个同步异常
- 还可以在第一个函数中直接抛出异常， 直接让我们的 catch 接收到并执行

```js
  reqData('ids').then(res => {
    if(islogin()) {
      throw new Error('用户已经登出')
    }
    if(inMemoryCache) {
      return  inMemoryCache[res,id]
    }
    return getUserById(res.id)
  }).then(res => {
    // 如果是缓存中的id res 都可以拿到！！
  })
```



### 高级错误

##### 进阶错误一， 不知道 Promise.resolve()

- resolve 正常写法
```js
 new Promise((resolve, reject) => {
    resolve(someFunction)
 }).then()
```
- 更加简洁的写法
```js
Promise.resolve(someFunction).then(/** */)
```

- 他在用来捕获同步异常也是极其好用的
```js
function someApi() {
  return Promise.resovle().then(() => {
    doSomething() // 做一些可以丢掉的东西 or 数据
    return 'foo'
  }).then(/** */)
}
```

##### 进阶错误二， catch() 与 then() 并非完全相等

- catch() 只是一个语法糖， 因此下面两段代码是等价的

```js
somePromise().catch((err) => {})


somePromise().then(null, (err) => {})
```

- 而下面两个并不是等价的， 考虑如果第一个函数抛出异常会发生什么事情

```js
  somePromise().then(function () {
    return someOtherPromise();
  }).catch(function (err) {
    // handle error
  });

  somePromise().then(function () {
    return someOtherPromise();
  }, function (err) {
    // handle error
  });
```

##### 进阶错误三
- 当我们希望执行一个个的执行一个 promises 序列，即类似 Promise.all() 但是并非并行的执行所有 promises。

```js
function executeSequentially(promises) {
  let result = Promise.resolve();
  promises.forEach(function (promise) {
    result = result.then(promise);
  });
  return result;
}
```

- 不幸的是，这份代码不会按照你的期望去执行，你传入 executeSequentially() 的 promises 依然会并行执行。

- 其根源在于你所希望的，实际上根本不是去执行一个 promises 序列。依照 promises 规范，一旦一个 promise 被创建，它就被执行了。因此你实际上需要的是一个 promise factories 数组。

```js
  function executeSequentially(promiseFactories) {
    let result = Promise.resolve();
    promiseFactories.forEach(function (promiseFactory) {
      result = result.then(promiseFactory);
    });
    return result;
  }
```

```js
// 他仅仅是一个可以返回promise的函数
function myPromiseFactory() {
  return somethingThatCreatesAPromise();
}
```


这样就可以了？？ 这是因为 promise工厂函数在被执行之前并不会创建promise 他就像 then 函数一样，而实际上，他们就是完全一样的东西！！


##### 进阶错误四， 如果我希望获得两个 promise 的结果怎么办

- 有的时候， 一个promise 会依赖另一个， 但是如果我们希望同时获得这两个 promise 的输出。

```js
  getUserByName('nolan').then(function (user) {
    return getUserAccountById(user.id);
  }).then(function (userAccount) {
    // dangit, I need the "user" object too!   该死的，我也需要 "用户 "对象!
  });
```

- 为了成为一个优秀的开发者， 我们可能会将 user 对象存在一个更高的作用与变量中：
```js
  var user;
  getUserByName('nolan').then(function (result) {
    user = result;
    return getUserAccountById(user.id);
  }).then(function (userAccount) {
    // okay, I have both the "user" and the "userAccount"  好的，我有 "user "和 "userAccount"。
  });
```

- 或者这样

```js
  getUserByName('nolan').then(function (user) {
    return getUserAccountById(user.id).then(function (userAccount) {
      // okay, I have both the "user" and the "userAccount"  好的，我有 "user "和 "userAccount"。
    }); 
  });
```

-  这样暂时是没有问题的， 我们将函数抽离到一个命名函数中

```js
  function onGetUserAndUserAccount(user, userAccount) {
    return doSomething(user, userAccount);
  }

  function onGetUser(user) {
    return getUserAccountById(user.id).then(function (userAccount) {
      return onGetUserAndUserAccount(user, userAccount);
    });
  }

  getUserByName('nolan')
    .then(onGetUser)
    .then(function () {
    // 在这一点上，doSomething()已经完成，我们又回到了缩进0。
    // at this point, doSomething() is done, and we are back to indentation 0
  });
```

- 由于你的 promise 代码开始变得复杂， 你发现自己将越来越多的函数抽离到命名函数中，我发现这样做，你的代码会越来越漂亮，就像这样 成为大佬！！！   

```js
  putYourRightFooIn()
    .then(putYourRightFootOut)
    .then(putYourRightFootIn)  
    .then(shakeItAllAbout);
```

##### 进阶的错误五，   promise 穿透

- 最后看下开头说的 promise 谜题， 可能用不到，但是很震惊🤯

下面这段代码会打印什么？  
```js
  Promise.resolve('foo').then(Promise.resolve('bar')).then(function (result) {
    console.log(result);  
  });
```


 - foo！！！ 添加任意数量的then() 都会输出 foo 

 ```js
  //  简单的说， 你可以直接传递一个 pormise 到 then（） 函数中， 但是他并不会按照你期望的去执行。 then（） 是期望获取一个函数 因此你希望做的可能是：
    Promise.resolve('foo').then(function () {
      return Promise.resolve('bar');
    }).then(function (result) {
      console.log(result);
    });
  // 这样他就会如我们所想的打印出 bar。
 ``` 


 1. ***因此记住：永远都是往 then() 中传递函数！***




### 关于开头问题的解题思路, 请到相关参考文章中了解 

 ### 参考文章

 [中文](http://fex.baidu.com/blog/2015/07/we-have-a-problem-with-promises/)
 [英文](http://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html) 