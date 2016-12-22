### Q.js【译文】

* [Getting Start](#getting-start)
* [Resources](#resources)
* [Tutorial](#tutorial)
  * [Propagation](#propagation)
  * [Chaining](#chaining)
  * [Combination](#combination)
  * [Sequences](#sequences)
  * [Handling Errors](#handling-errors)
  * [Progress Notification](#progress-notification)
  * [The End](#the-end)
  * 开始
  * 中间
  * Over the Wire
  * 适配Node
  * 跟踪长堆栈
*  测试
* 关于作者



> 这是从Git分支而来的Q版本`1.0`。本文档适用于最新的版本1.0和版本0.9发行版，这些版本都是稳定的，在0.9.7版本之后就没有0.9版本的进一步发行了，所以这相当于就是1.0版本。
> `q@~1.0`的所有进一步版本都将被兼容到。2.0发行版变化显著，并且其向后不兼容的更改，此刻正在实验着。 <br>

如果一个`function`没有返回一个值，或者没有抛出一个异常阻塞，它就可以返回一个promise。Promise是一个对象，表示返回值或者是函数最终提供的的抛出的异常。
Promise也可以作为一个[远程对象](https://github.com/kriskowal/q-connection)的代理来克服延迟问题。  <br>

在第一次实现的时候，Promise是为了拯救[“Pyramid of Doom”](http://calculist.org/blog/2011/12/14/why-coroutines-wont-work-on-the-web/)： the situation where code marches to the right faster
than it marches forward.  <br>
```javascript
step1( function( value1 ){
  step2( function( value2 ){
    step3( function( value3 ){
      step4( function( value4 ){ 
        // Do something with value4
    })
  })
})
```
有了promise库，你就可以压平金字塔了。
```javascript
Q.fcall( promisedStep1 )
.then( promisedStep2 )
.then( promisedStep3 )
.then( promisedStep4 )
.then( function( value4 ){
  // Do something with value4
})
.catch( function( error ){
  // Handle any error from all above steps
})
.done();
```
运用promise，你也能得到通过隐式传播的错误，就像`try`,`catch`,`finally`。在`promisedStep1`产生的错误就会一路传向`catch`函数，在`catch`里面就可以捕获到这个错误，并且可以对它做出相应的处理。
（这里的'promisedStpeN'只是`stepN`的另一个形式，它会返回一个promise。） <br>
回调方法称为“控制反转”，一个函数接收一个回调而不是返回一个值，此函数的意思是：“别找我，我会找你的了”。 <br>

Promise不会[反转](http://www.slideshare.net/domenicdenicola/callbacks-promises-and-coroutines-oh-my-the-evolution-of-asynchronicity-in-javascript)倒转，
清晰地分离从控制流参数组输入的参数。
这简化了用法和创建了API，侧重体现在variadic、rest和spread arguments。  

#### Getting Start
Q可以用以下方式加载：
* `<script>`标签（创建一个Q的全局变量）：简化和压缩后 ~2.5KB 
* Node.js或CommonJS模块，可用[npm](https://www.npmjs.com/)里[q](https://www.npmjs.com/package/q)的包
* AMD模板
* 使用[component](https://github.com/componentjs/component)如同`microjs/q`
* 使用[bower](https://bower.io/)如同`q#1.0.1`
* 使用[NuGet](http://www.nuget.org/)如同[Q](https://www.nuget.org/packages/q)

Q能够和 jQuery, Dojo, When.js, WinJS,等等混合使用。



#### Resources
[wiki](https://github.com/kriskowal/q/wiki)包含了很多有用的资源，包括：
* 渐进式的[Q API reference](https://github.com/kriskowal/q/wiki/API-Reference)
* 正在完善的[examples gallery](https://github.com/kriskowal/q/wiki/Examples-Gallery)，给我们展示为什么Q能够让所有事情更美好。从XHR到数据库，access to accessing the Flickr API，Q为你而生
* 有很多库再开发和采用Q promise，从文件系统/数据库访问、RPC到模板。一些很受欢迎的列表，可以点击这里查看[Libraries](https://github.com/kriskowal/q/wiki/Libraries)
* 如果你想要介绍promise概念的一些资料，那么以下的教程不大适合你，查看我们的收藏夹[presentations, blog posts, and podcasts](https://github.com/kriskowal/q/wiki/General-Promise-Resources)
* [Coming from jQuery`$.Deferred`](https://github.com/kriskowal/q/wiki/Coming-from-jQuery)的指南
我们非常欢迎你加入我们的[网上论坛](https://groups.google.com/forum/#!forum/q-continuum)


#### Tutorial
Promise拥有`then`方法：可以用来获取程序返回的值——fulfillment，或者是捕获程序抛出的异常——rejection。   
```javascript
promiseMeSomething()
.then(function (value) {
}, function (reason) {
});
```
如果`promiseMeSomething`函数返回一个promise对象是程序的返回值，那么第一个函数（the fulfillment handler）就会访问这个值。但是如果`promiseMeSomething`函数得到的是一个程序抛出的异常，那么第二个函数（the rejection handler）就会对这个异常进行相应处理。 <br>

需要注意的是：promise的解析是针对异步机制的。也就是说，成功或者失败处理程序将会在下一个事件循环中被调用（即Node中的`process.nextTick`）。这给我们在跟踪代码流的时候一个很好地保障。换句话说，`then`将总是在任何处理程序执行之前返回。   <br>
在本教程中，我们将从如何使用、运用promise进行开发开始。我们将会讲解如何去创建它们，创建一个类似'promiseMeSomething'会返回一个promise对象的函数。

##### Propagation
`then`方法返回一个promise对象，在下面的这个例子中，我们设为'outputPromise' 
```javascript
var outputPromise = getInputPromise()
.then(function (input) {
}, function (reason) {
});
```
`outputPromise`变量将在程序结束前作为一个promise对象，返回一个值。此时函数只能返回一个值或者抛出一个异常。只有一个处理程序将会被调用，它将负责处理'outputPromise'。   
* 如果处理程序返回的是一个值，'outputPromise'的状态就变成“成功(fulfilled)”
* 如果程序抛出一个异常，'outputPromise'的状态就变成“失败(rejected)”
* 如果在处理程序中返回了一个promise对象，'outputPromise'就会"变成"那个promise。能够成为一个新的promise对于管理延迟、组合结果，或者是恢复错误都是很有帮助的。 <br>

如果`getInputPromise()`promise结果失败，并且你省略了对于rejection的处理程序，那么error就会返回到`outputPromise`：
```javascript
var outputPromise = getInputPromise()
.then(function (value) {
});
```
 如果输入的promise得到一个成功的结果，但是你省略了对于fulfillment 的处理程序，那么得到的值就会返回到'outputPromise'：
 ```javascript
 var outputPromise = getInputPromise()
 .then(null,function(error){
 });
 ```
 
 当你只关注于对程序抛出的异常的处理的时候，Q promise提供了一个`fail`方法，可以取代`then`。
 ```javascript
 var outputPromise = getInputPromise()
 .fail(function(error){
 });
 ```
如果你只为现代引擎编写JavaScript或者使用CoffeeScript，你就可以使用`catch`来替代`fail`。   <br>
 
Promise 还有一个`fin`方法，类似于`finally`。当promise通过`getInputPromise()`在返回一个值或者抛出一个异常之前返回的时候，final处理程序就会在无参的情况下被调用。函数返回的值或是抛出的异常通过`getInputPromise()`直接传递到`outputPromise`，除非最终处理程序失败，也有可能会延迟，如果final程序返回一个promise。    <br>
```javascript
var outputPromise = getInputPromise()
.fin(function(){
 // close files, database connections, stop servers, conclude tests
})
```
* 如果程序返回的是一个值，那么这个值就会被忽略
* 如果程序抛出了一个异常，那么这个异常就会被传递到`outputPromise`
* 如果程序返回一个promise对象，`outputPromise `就会被延迟。最终返回的值或是抛出的异常具有和立即返回值或是抛出异常相同的作用：返回的值就会被忽略，抛出的异常就会被传递。   <br>

如果你只为现代引擎编写JavaScript或者使用CoffeeScript，你就可以使用`finally`来替代`fin`。   <br>

##### Chaining
Promise的链有两种形式，可以通过内部或者外部处理程序，下面两个例子是等价的。
```javascript
return getUsername()
.then(function(username){
 return getUser(username)
 .then(function(user){
  // if we get here without an error,the value returned here
  // or the exception thrown here,resolves the promise returned by the first line  
 })
});
```
```javascript
return getUsername()
.then(function(username){
 return getUser(username);
})
.then(function(user){
  // if we get here without an error,the value returned here
  // or the exception thrown here,resolves the promise returned by the first line  
});
```
可以看出唯一的区别就是嵌套的不同。如果你需要在闭包中捕获多个输入值，嵌套处理程序是非常有利于开发的。  <br>
```javascript
function authenticate(){
 return getUsername()
 .then(function(username){
  return getUser(username);
 })
 // chained because we will not need the user name in the next event
 .then(function(user){
  return getPassword()
   // nested because we need both user and password next
    .then(function(password){
     if(user.passwordHash !== hash(password)){
      throw new Error ("Can't authenticate");
     }
    });
 });
}
```
##### Combination
你可以将一个promise数组整合成一个整体的promise。成功时候的数组可以用`all`。
```javascript
return Q.all({
 eventualAdd(2,2),
 eventualAdd(10,20)
});
```
如果你有一个数组的promise，你可以使用`spread`替代'then'。`spread`方法会把成功处理程序返回的参数给“分开”。失败处理程序会访问第一个失败的信号。无论哪一个接收promise失败的函数都会首先由失败处理程序处理。   
```javascript
function eventualAdd(a,b){
 return Q.spread([a,b],function(a,b){
  return a + b;
 })
}
```
不过`spread`最开始是`all`的一种，所以你可以跳过它的链。
```javascript
return getUsername()
 .then(function (username){
  return [username,getUser(username)];
 })
 .spread(function(username,user){
 });
```
`all`方法返回一个数组的promise，当Promise的状态是成功的时候，在原始的promise中，这些数组会按照这些promise的顺序包含成功状态返回的值。如果其中一个给定的promise的状态为失败，那么返回的那个promise就会立即被拒绝，不必等待剩余的批次。如果你想要等到所有的promise确定其状态为成功或者失败，那么你可以使用`allSettled`。
```javascript
Q.allSettled(promise)
 .then(function(results){
  results.forEach(function(result){
   if( result.state === 'fulfilled' ){
    var value = result.value;
   }else{
     var reson = result.reason;
   }
  })
 })
```
`any`方法接受一个promise数组，并返回一个promise的状态是成功（第一个给定的promise的状态为成功）或是失败（所有给定的promise的状态都为失败）。
```javascript
Q.any(promise)
.then(function (first){
 // Any of the promise was fulfilled
},function(error){
 // All of the promise were rejected
})
```

##### Sequences
如果你有一些promise-producing函数需要顺序执行，你可以像一下例子那样：
```javascript
return foo(initialVal).then(bar).then(baz).then(qux); 
```
然而，如果你想运行一个动态构造的序列函数，你可以这样做：
```javascript
var funcs = [foo,bar,baz,qux];
var result = Q(initialVal);
funcs.forEach(function(f){
 result = result.then(f);
});
return result;
```
通过使用`reduce`可以使此稍微更紧凑一点：
```javascript
return funcs.reduce(function (soFar,f){
 return soFar.then(f);
},Q(initialVal));
```
你可以使用更紧凑的写法：
```javascript
return funcs.reduce(Q.when,Q(initialVal));
```

#####　Handling Errors

Promise有一个有时不直观的方面的，如果你在成功状态时候抛出一个异常，此异常将不会被失败状态的处理程序捕获。
```javascript
return foo()
.then(function (value){
 throw new Error("Can't bar.");
},function(error){
 // We only get here if "foo" fails
});
```
为什么会这样呢？考虑一下promise之间的平行关系和`try`/`catch`。我们尝试去执行`foo()`：错误处理程序类似于`foo()`里的`catch`，而成功处理程序表示在`try`/`catch`之后的代码块。此时，代码需要自身的`try`/`catch`代码块。   <br>

对于promise而言，这意味着链接在你的失败处理程序里：
```javascript
return foo()
.then(function (value) {
    throw new Error("Can't bar.");
})
.fail(function (error) {
    // We get here with either foo's error or bar's error
});
```

##### Progress Notification
让promise时刻反馈他们的进度是有可能的，例如：上传文件所需时间较长时。不是所有的promise对象都有进度反馈，但是对于那些需要知道其进度的，你可以传递第三个参数给`then`来对进度情况作操作：
```javascript
return uploadFile()
.then(function(){
 // Success uploading the file
}.function(err){
 // There was an error,and we get the reason for error
},function(progress){
 // We get notified of the upload's progress as it is executed 
});
```
就像`fail`，Q提供了进度回调的缩写`progress`：
```javascript
return uploadFile().progress(function(progress){
 // We get notified of the upload's progress
});
```

##### The End
当一条promise链快结束的时候，此时你要么返回最后一个promise，要么结束promise链。因为失败处理程序可以捕获异常，这是一个不好的模式，因为异常的去处不能被观察。   <br>
所以，要么返回它：
```javascript
return foo()
 .then(function(){
  return "bar";
 });
```
要么结束它：
```javascript
foo()
.then(function(){
 return "bar";
})
.done();
```
结束一个promise链能够确保：如果一个错误没有在结束之前被失败处理程序捕获，它就会被重新载入并且报告。  <br>
 我们正在探索如果去使得一个未处理的错误可见而无任何明确的处理。
 
 ##### The Beginning
 
 上面的所有假设，你已经从某处得到一个promise，这是一种很常见的情况。每隔一段时间，你需要从头开始创建一个promise。
 
 ###### Using `Q.fcall`
 
 promise可以来自于一个使用`Q.fcall`的值，下面的例子返回一个IO promise：
 ```javascript
 return Q.fcall(function(){
  return 10;
 });
 ```
 
 你也可以使用`fcall`创建一个带有异常的promise：
 ```javascript
 return Q.fcall(function(){
  throw new Error("Can't do it");
 });
 ```
 
 顾名思义，`fcall`能称为一个函数，也能是一个promise方法。可以使用`eventualAdd`方法去添加两个数字：
 ```javascript
 return Q.fcall(eventualAdd,2,2);
 ```
 
 ###### Using Deferreds
 
 