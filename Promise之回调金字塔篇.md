#Promise之回调金字塔篇

##Author
|Weibo Id | Name |
|:----------|:----------|
|[@Cristo在路上](http://www.weibo.com/2150756497) | Cristo.Cui |

学习`Node.js`已经两周了，之前一直比较迷惑`promise`的使用，现在终于有所理解。这篇文章将主要以解决回调金字塔的角度来看待promise，至于控制权转换以及LEGO等更深入的角度将在后续文章中剖析。通过这篇文章将基本理解promise的用法。
##什么是promise
promise是JavaScript实现优雅编程的一个非常不错的轻量级框架。该框架可以让你从杂乱的多重异步回调代码中解脱出来，并把精力集中到你的业务逻辑上。

promise是为解决问题而生的，先看看下面这段代码：

```JavaScript
function main(data, cb){
	fun1(data, function(err, data){
		if(!err){
      		fun2(data, function(err, data){
        		if(!err){
          			fun3(data, cb);
        		}else{
          			cb(err);
        		}
      		});
    	}else{
      		cb(err);
    	}
	});
}
```

这是一段非常恐怖的代码，里面有很多层的回调函数嵌套调用，我一看到这样的代码就会头疼，不仅丑陋而且很难维护。

`callback`是JavaScript中实现异步最简单常用的方式，但是这种方式不仅带来了上述令人头疼的回调金字塔的问题，而且牺牲了控制流，同时也不方便在外部捕获异常。

`promise`的出现就是为了解决上述问题.promise是一个标准，它描述了异步调用的返回结果，包括正确返回结果和错误处理。需要注意的是，promise只是一种编程方式的变化，无须在底层改变。其详细说明文档可参考[Promise/A+](https://promisesaplus.com/)。

CommonJS规范中提到了多种promise的实现，接下来将以使用较多的`Q`，以及我现在用的比较多的`LeanCloud`中promise的实现为例来介绍一下promise在nodejs中的实现及使用。

先来耐心看一下promise是如何实现的。

##promise的实现
Q的核心是一个promise对象的`then`方法，他接受两个回调方法，一个promise被定义之后有3种状态，pending（过渡状态），fullfilled（完成状态），rejected(错误状态)。一个promise只能是这三种状态种的一种。

* pending状态可以理解为promise还没有获得确定值，就相当于一个任务还没有完成。
* fullfilled状态可以理解为完成并返回结果。这时then(onFullfilled, onRejected)的onFullfilled方法会被调用。
* rejected状态可以理解为错误，并结束，返回错误。这时then(onFullfilled, onRejected)的onRejected方法会被调用。

下面来尝试封装promise的API：
先来看我学习的时候从别人的博客上看到的一个很好的例子，在这个例子中，先按照传统的callback写法来读取一个json文本，然后将其解析成javascript对象，最后将这个对象修改并保存。

```JavaScript
fs.readFile('example.json', function(err, data){
  if(err) {
    console.log(err):
  } else {
    try {
      var obj = JSON.parse(data);
      obj.prop = 'something new';
      fs.writeFile('example.json', JSON.stringify(obj), function(error){
        if(err) {
          console.log(error);
        } else {
          console.log('success');
        }
      });
    } catch(e) {
      console.log(e);
    }
  }
});
```

基本思路是利用Q的defer()方法，创建一个deferred对象。这个对象有两个关键方法resolve()和reject()。当resolve(value)执行之后，promise变成fullfilled状态，fullfilled的值即value。当reject(reason) 执行之后，promise变成rejected状态，reason会被传递到onRejected()方法。

具体如下：

```
var Q = require('q');
function readFile(callback){
  var deferred = Q.defer();
  fs.readFile('example.json', function(err, data){
    if(err){
      deferred.reject(err);
    } else {
      deferred.resolve(data);
    }
  });
  return deferred.promise.nodeify(callback);
}
```
需要注意的是，上面这个写法中依然使用了一个callback，这是为了使这个模板具有更多场景的兼容性，更一般的场景会是这样：

```
var fun = function (data) {
    var deferred = Q.defer();
    deferred.resolve("success, return data is: " + data);
    return deferred.promise;
}
```
##promise的使用
###基本promise的使用
有了上面的封装之后就可以来尝试愉快地使用promise了，读取、修改并保存json的例子就可以这样写：
```
var promise = readFile();
promise.then(function(data){
  var obj = JSON.parse(data);
  obj.prop = 'something new';
  // return a promise. so we can chain the then() method.
  return writeFile(JSON.stringify(obj));
  })
  .then(function(){
    console.log('success');
  }, function(err){
    // all error will fall down here.
    console.log(err);
  });
```
是不是立即简洁明了多了？也更加符合我们的编程思维了吧。

如果这个例子还不够明显，那最初那个令人头疼的回调金字塔的例子可能更明显：
```
function main(data,cb){
 fun1("helloworld")
     .then(fun2)
     .then(fun3)
     .done(function(data){
        cb(null,data);
     },function(err){
        cb(err);
     });
}
```
天壤之别了吧。还可以看到的一点是，使用传统的回调函数方式的话，我们需要在每一个回调函数里判断是否有错误需要处理，这样会存在很多冗余代码，使用promise的话，可以使用done或者fail统一在一个函数中处理错误，这点确实很赞。
###串行promise的使用
在你想要某一行数据做一系列的任务的时候，Promise链是很方便的, 每一个任务都等着前一个任务结束. 比如, 假设你想要删除你的blog上的所有comment：
```
_.each(commentList, function(comment){
  promise = promise.then(function(){
	// Return a promise that will be resolved when the delete is finished.
    return comment.destory();
  });
});
```
###并行promise的使用
如果你有几个异步方法，他们都返回promise，并且当这些方法都处理完之后，你才能进行下一步，Q提供了一个all()方法来解决这个问题。
```
Q.all([
  readFile('file1.json'),
  readFile('file2.json')
  ])
  .then(function(dataArray){
    for(var i = 0; i < dataArray.length; i++){
      console.log(dataArray[i]);
    }
  }, function(err){
    console.log(err);
  });
```
当任意一个promise变成rejected状态的时候，all的promise会立即reject而不等其他的完成。