title: 在ES5中模拟拓展运算符
date: 2016-03-22
tags: javascript
categories: javascript
---
拓展运算符
==========

JavaScript 在 ES6 中引入了[拓展运算符]， 可以把可遍历对象（例如数组）直接拓展为参数列表。例如数组

``` javascript
  var args = [1, 2, 3]
  console.log(Math.max(args)) // output: NaN, 因为 Math.max 的参数不是数组
  console.log(Math.max(...args)) // output: 3
```

<!--more-->

简单的模拟
==========

上面的例子在 ES5 中其实也很容易实现，只要用 [apply] 就可以了

`apply` 的用法

``` javascript
  fun.apply(thisArg, [argsArray])
```

`thisArg` 表示绑定的 `this` ，而 `argsArray` 恰巧就是用数组。

``` javascript
  console.log(Math.max.apply(null,args)) // output: 3
```

复杂的情况：构造函数
====================

如果只有上面一种简单的场景，也就没有引入拓展运算符的必要了。但有时候情况会变得复杂，例如构造函数。看下面这个例子：

``` javascript
  function Obj(){
      this.result = arguments[0] + arguments[1]
  }
  function newObj(){
      // ES6
      return new Obj(...arguments)
  }
  console.log(new Obj(3,4).result) // output: 7
  console.log(newObj(3,4).result)  // output: 7
```

由于这里涉及到了 `new` 的使用， `Obj` 被作为构造函数而不是简单的函数调用，于是 `apply` 也就不能实现同样的效果。\[[1]\] 。

通过 bind 实现
--------------

[bind] 常用于绑定 `this` ，但其实也可以用于绑定 `arguments` ，看下面的例子：

``` javascript
  var new_max = Math.max.bind(null, 4, 5, 6, 7)
  new_max(1, 2) // 无论参数是什么返回值都是 7 [max(4, 5, 6, 7)]
```

在 `bind` 返回的函数作为构造函数时， `bind` 的 `this` 会失效，但 `arguments` 仍然有效。 \[[2]\]

这种特性正好可以用来完成我们的目的

``` javascript
  function Obj(){
      this.result = arguments[0] + arguments[1]
  }
  function newObj(){
      // ES5
      return new (Obj.bind.apply([null].concat(Array.prototype.call(arguments))))
  }
  console.log(newObj(4,5).result)     // return 9
```

其中 `Array.prototype.call(arguments)` 是为了把 `arguments` 转化为真正的数组，然后通过第一个元素为 `null` 的数组构成 `bind` 需要的参数列表并 `apply` ,就能够正确的作为构造函数使用了。

构造高阶函数
------------

从上面的方法看，bind 实际上是一个高阶函数（返回函数的函数），只是它正好满足了我们的需求，我们也可以通过构造一个高阶函数来达到目的。

``` javascript
  function Obj(){
      this.result = arguments[0] + arguments[1]
  }

  function funcWithArgArray(func, argArray){
      return function(){
          func.apply(this, argArray)
      }
  }

  function newObj(){
      return new (funcWithArgArray(Obj, arguments))
  }

  console.log(newObj(3,4))  // output: 7
```

总结
====

上面的几种方法，除了在全面拥抱 ES6 前提供了一个等价的写法外，也体现了 JavaScript 强大的表现力，高阶函数的恰当使用让 JavaScript 散发着 Lisp 的光辉。 

[1] 参见 [Apply for new]

[2] 参见[Bound functions used as constructors]

  [拓展运算符]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_operator
  [apply]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/apply
  [bind]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind
  [Apply for new]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_operator#Apply_for_new
  [Bound functions used as constructors]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind#Bound_functions_used_as_constructors
