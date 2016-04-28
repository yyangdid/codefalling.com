title: underscore 源码阅读
date: 2016-04-24
tags: javascript, underscore
categories: javascript
---

## 背景

阅读源码是一种非常重要的学习方式，我们可以直接从高手的源码中收获其他地方难以获得的最佳实践和编码思想。因为 `underscore` 的代码结构较为简单，而且注释非常详细，所以先从 `underscore` 开始。

本文阅读的源码版本是`1.8.3`，可以直接在 [Github](https://github.com/jashkenas/underscore/blob/e4743ab712b8ab42ad4ccb48b155034d02394e4d/underscore.js) 上看到代码。

有些地方可能不按照代码原有顺序，因为部分代码依赖后面的内容，单独阅读很难理解。

<!-- more -->

## 代码阅读

### 闭包

``` js
(function () {
// ...
}).call(this);
```

整个代码块包含在 `function` 中，防止污染外部作用域。

### 变量创建

``` js
// 将`this`赋值给`root`，浏览器中`this`指向`window`，node环境`this`指向`export`
var root = this;

// 保存已经存在的 `_`
var previousUnderscore = root._;

// 在最小化但是尚未gzip压缩的版本中减少体积
var ArrayProto = Array.prototype, ObjProto = Object.prototype, FuncProto = Function.prototype;

// 创建引用对象来提升访问核心原型的访问速度
var
  push             = ArrayProto.push,
  slice            = ArrayProto.slice,
  toString         = ObjProto.toString,
  hasOwnProperty   = ObjProto.hasOwnProperty;

// 声明一下要使用的`ES5`原生方法
var
  nativeIsArray      = Array.isArray,
  nativeKeys         = Object.keys,
  nativeBind         = FuncProto.bind,
  nativeCreate       = Object.create;
  
```

其中的减小体积的方式较为有趣，因为最小化（minifiy）会将变量名替换成诸如 `a,b,ab` 等字符尽可能少的命名，但是对于 `Array.prototype` 这类不会做任何处理，使用 `ArrayProto` 来替代则能在最小化时尽可能优化下面的使用。但这种方式带来的收效有限，而且 `gzip` 很适合处理这种相同重复的字符，在开启 `gzip` 的情况下收效就更不显著了。

关于替代原型交换先放下，这个变量在后面的程序中才用到。

### 导出 underscore

```js
// Create a safe reference to the Underscore object for use below.
var _ = function(obj) {
  if (obj instanceof _) return obj;
  if (!(this instanceof _)) return new _(obj);
  this._wrapped = obj;
};

// 在 nodejs 中导出兼容老式 `require()` API 的 underscore 对象
// 如果在浏览器中则将 `_` 加入公共对象
if (typeof exports !== 'undefined') {
  if (typeof module !== 'undefined' && module.exports) {
    exports = module.exports = _;
  }
  exports._ = _;
} else {
  root._ = _;
}

// Current version.
_.VERSION = '1.8.3';
```

其中 `if (!(this instanceof _)) return new _(obj);` 是为了使构造函数被直接调用仍然起到构造函数的作用，当不是作为构造函数被调用（即不是 `new _` ）时，`this instanceof _` 返回 `false`。

TODO: 为什么要尝试访问 `exports` 而不是直接使用 `module.exports` ？

### 内部函数

#### optimzeCb
```js
// 返回一个针对当前引擎的高效的回调函数
// Internal function that returns an efficient (for current engines) version
// of the passed-in callback, to be repeatedly applied in other Underscore
// functions.
var optimizeCb = function(func, context, argCount) {
  if (context === void 0) return func;
  switch (argCount == null ? 3 : argCount) {
    case 1:
      return function(value) {
        return func.call(context, value);
      };
    case 2:
      return function(value, other) {
        return func.call(context, value, other);
      };
    case 3:
      return function(value, index, collection) {
        return func.call(context, value, index, collection);
      };
    case 4:
      return function(accumulator, value, index, collection) {
        return func.call(context, accumulator, value, index, collection);
      };
  }
  return function() {
    return func.apply(context, arguments);
  };
};
```

看起来吓人一跳，注释看着也比较摸不着头脑（特意保留了原注释）。但简化一下其实就是

```js
var optimizeCb = function(func, context) {
  return function() {
    return func.apply(context, arguments);
  };
}
```

实际上就是一个 `bind` 函数，前面的一大段 `switch` 是为了在已知参数个数的情况下让编译器能够优化，而不是直接将 `arguments` 传递过去（部分引擎无法针对其优化）。

我写了一个[简单的 brenchmark](http://paste.ubuntu.com/16023395/)，在我的 Chrome 50 中，这种『优化』的效果并不明显，当然他们都比 `native bind` 快了很多。

```
#optimizeCb: 455.607ms
#non_optimizeCb: 421.372ms
#native_bind: 1314.254ms
```

### createAssigner

``` javascript
// 返回一个赋值器函数，新的函数第一个参数为对象，接受多个参数，将后面参数的对象的属性赋值给第一个对象
// keysFunc 用于决定复制一个对象的哪些 keys
var createAssigner = function(keysFunc, undefinedOnly) {
  return function(obj) {
    var length = arguments.length;
    if (length < 2 || obj == null) return obj;
    for (var index = 1; index < length; index++) {
      var source = arguments[index],
        keys = keysFunc(source),  // 使用 keysFunc 来获取一个对象需要被拷贝的keys
        l = keys.length;
      for (var i = 0; i < l; i++) {
        var key = keys[i];
        if (!undefinedOnly || obj[key] === void 0) obj[key] = source[key];
      }
    }
    return obj;
  };
};
```

### baseCreate

```js
// line 35
// 用于代理原型交换（surrogate-prototype-swapping）的空函数
var Ctor = function(){};

// An internal function for creating a new object that inherits from another.
var baseCreate = function(prototype) {
  if (!_.isObject(prototype)) return {};
  if (nativeCreate) return nativeCreate(prototype);
  Ctor.prototype = prototype;
  var result = new Ctor;
  Ctor.prototype = null;
  return result;
};
```

`baseCreate` 其实就是 `Object.create` 的 `polyfill`。创建一个新的对象，其原型指向目标对象。

可能是为了避免每次都创建一个空函数，所以 `underscore` 直接创建了一个 `Ctor`。

### property

```js
var property = function(key) {
  return function(obj) {
    return obj == null ? void 0 : obj[key];
  };
};
```

这个内部函数的意图目前不是很清楚，看起来只是生成了指定属性的访问器。

```js
var obj = {test: 'hello'};
obj['test']; // 'hello'
property('test')(obj); // 'hello'
```

唯一的区别是当 `obj == null` 时返回 `undefined` 而不是报错，可能是为了避免写 `obj && obj['test']` 这种代码？

### isArrayLike
```js
// Helper for collection methods to determine whether a collection
// should be iterated as an array or as an object
// Related: http://people.mozilla.org/~jorendorff/es6-draft.html#sec-tolength
// Avoids a very nasty iOS 8 JIT bug on ARM-64. #2094
var MAX_ARRAY_INDEX = Math.pow(2, 53) - 1;
var getLength = property('length');
var isArrayLike = function(collection) {
  var length = getLength(collection);
  return typeof length == 'number' && length >= 0 && length <= MAX_ARRAY_INDEX;
};
```

即 `length` 为合法值的对象都被认为是 `ArrayLike` 的

## 函数

### _.each
```js
_.each = _.forEach = function(obj, iteratee, context) {
  iteratee = optimizeCb(iteratee, context); // 创建回调函数
  var i, length;
  if (isArrayLike(obj)) {
    for (i = 0, length = obj.length; i < length; i++) {
      iteratee(obj[i], i, obj);
    }
  } else {
    var keys = _.keys(obj);
    for (i = 0, length = keys.length; i < length; i++) {
      iteratee(obj[keys[i]], keys[i], obj);
    }
  }
  return obj;
};
```

一个 `each` 实现，非常清晰，`iteratee` 是回调函数，如果是 `ArrayLike` 对象则从 `0` 迭代到 `length - 1`，否则按 `keys` 迭代。

### _.map

```js
_.map = _.collect = function(obj, iteratee, context) {
  iteratee = cb(iteratee, context);
  var keys = !isArrayLike(obj) && _.keys(obj), // 不是ArrayLike对象则获取其keys
    length = (keys || obj).length, // 获取其（或者keys）的长度
    results = Array(length); // 这里可以看到返回值是一个数组
  for (var index = 0; index < length; index++) {
    var currentKey = keys ? keys[index] : index;
    results[index] = iteratee(obj[currentKey], currentKey, obj);
  }
  return results;
};
```

类似原生的 `Array.map`，但是同样作用于普通对象（仍然返回数组）。

```js
var obj = {
  a: 'test',
  b: 'test'
}
_.map(obj, function(item, key){
  return item + key
}) // ['testa', 'testb']
```

### _.reduce

``` javascript
// **Reduce** builds up a single result from a list of values, aka `inject`,
// or `foldl`.
_.reduce = _.foldl = _.inject = createReduce(1);

// The right-associative version of reduce, also known as `foldr`.
_.reduceRight = _.foldr = createReduce(-1);
```

直接看 `createReduce` 有点云里雾里，所以我们先看看它的用法，可以看到下面马上用这个函数生成了 `_.reduce` 和 `_.reduceRight`，`dir` 显然是步长。

我们来看看文档中 `reduce` 的用法：

``` javascript
function reduce(list, iteratee, memo, context) {
  var result = memo
  for(var i = 0; i < list.length; i++) {
    result = iteratee.call(context, list[i], result)
  }
  return result
}

reduce([1, 2, 3], function(memo, b){return memo + b}, 0, null)

var sum = _.reduce([1, 2, 3], function(memo, num){ return memo + num; }, 0);
=> 6
```

和我们平时常用的 `Array.prototype.reduce` 很类似。我们可以自己尝试着实现一个简单的 `reduce`。

``` javascript
function reduce(list, iteratee, memo, context) {
  var result = memo
  for(var i = 0; i < list.length; i++) {
    result = iteratee.call(context, list[i], result)
  }
  return result
}

reduce([1, 2, 3], function(memo, b){return memo + b}, 0, null)
```

现在我们再来看 `createReduce` 的代码就很清晰了，它只是为了统一 `ArrayLike` 和普通对象加了个 `iterator` 作为统一的迭代器。

``` javascript
// Create a reducing function iterating left or right.
function createReduce(dir) {
  // Optimized iterator function as using arguments.length
  // in the main function will deoptimize the, see #1991.
  // 用于统一 ArrayLike 和普通对象的迭代器
  function iterator(obj, iteratee, memo, keys, index, length) {
    for (; index >= 0 && index < length; index += dir) {
      var currentKey = keys ? keys[index] : index;
      memo = iteratee(memo, obj[currentKey], currentKey, obj);
    }
    return memo;
  }

  return function(obj, iteratee, memo, context) {
    iteratee = optimizeCb(iteratee, context, 4);
    var keys = !isArrayLike(obj) && _.keys(obj),
        length = (keys || obj).length,
        index = dir > 0 ? 0 : length - 1;
    // Determine the initial value if none is provided.
    if (arguments.length < 3) {
      memo = obj[keys ? keys[index] : index]; // 当 memo 没有提供时使用第一个元素作为初始值
      index += dir;
    }
    return iterator(obj, iteratee, memo, keys, index, length);
  };
}
```

### _.find
直接调用的 `_.findIndex` 和 `_.findKey` ，没什么好讲的。

### _.filter

> Looks through each value in the list, returning an array of all the values that pass a truth test (predicate).

``` javascript
var evens = _.filter([1, 2, 3, 4, 5, 6], function(num){ return num % 2 == 0; });
// => [2, 4, 6]
```

其实也能够遍历对象，但是最终返回值都是数组。

``` javascript
// Return all the elements that pass a truth test.
// Aliased as `select`.
_.filter = _.select = function(obj, predicate, context) {
  var results = [];
  predicate = cb(predicate, context); // 绑定 context
  _.each(obj, function(value, index, list) {
    if (predicate(value, index, list)) results.push(value); // 满足条件则push进result
  });
  return results;
};
```

## _.reject

和上面的反过来，返回所有不符合结果的值的数组

``` javascript
// Return all the elements for which a truth test fails.
_.reject = function(obj, predicate, context) {
    return _.filter(obj, _.negate(cb(predicate)), context);
};

// line 856
// Returns a negated version of the passed-in predicate.
_.negate = function(predicate) {
  return function() {
    return !predicate.apply(this, arguments); // 返回值取反
  };
};
```

这里用到了一个 `_.negate` 函数，是将一个判断函数变成与其相反的判断函数。

本文将持续更新。。
