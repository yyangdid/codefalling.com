title: underscore 源码阅读
date: 2016-04-24
tags: javascript, underscore
categories: javascript
---

## 背景

阅读源码是一种非常重要的学习方式，我们可以直接从高手的源码中收获其他地方难以获得的最佳实践和编码思想。因为 `underscore` 的代码结构较为简单，而且注释非常详细，所以先从 `underscore` 开始。

本文阅读的源码版本是`1.8.3`，可以直接在 [Github](https://github.com/jashkenas/underscore/blob/e4743ab712b8ab42ad4ccb48b155034d02394e4d/underscore.js) 上看到代码。可以尝试从 `Github` 上 clone 代码，`npm install`，按照自己的想法进行一些改动，然后通过 `npm run test` 检验修改后的代码是否仍能通过测试。

有些地方可能不按照代码原有顺序，因为部分代码依赖后面的内容，单独阅读很难理解。

<!-- more -->

## 工具

一个非常实用的 blame 辅助插件：[Github Blame Tool - Chrome Web Store](https://chrome.google.com/webstore/detail/github-blame-tool/kipdndanedkendebejagldikdfogakig)

因为 Github 本身的 blame 只能看到上一次的无法向上追溯，这个插件显得极为有用。

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
var sum = _.reduce([1, 2, 3], function(memo, num){ return memo + num; }, 0);
// => 6
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

## _.every/_.some

``` javascript
// Determine whether all of the elements match a truth test.
// Aliased as `all`.
_.every = _.all = function(obj, predicate, context) {
    predicate = cb(predicate, context);
    var keys = !isArrayLike(obj) && _.keys(obj),
        length = (keys || obj).length;
    for (var index = 0; index < length; index++) {
        var currentKey = keuys ? keys[index] : index;
        if (!predicate(obj[currentKey], currentKey, obj)) return false;
    }
    return true;
}; 
```

上面是 `_.every` 的代码，非常简单，遍历每一个元素，一旦有元素不满足条件就返回 `false`，全部遍历完返回 `true`， `_.some` 也是类似的道理。

其实可以使用 `_.each` 改写代码，我使用下面的代码替代源代码的实现，仍然能够通过测试。

``` javascript
_.every = _.all = function(obj, predicate, context) {
  predicate = cb(predicate, context);
  var result = true;
  _.each(obj, function(item, index){
    if (!predicate(item, index, obj)) result = false;
  });
  return result;
};
```

这里之所以没有采取这种实现，是处于性能考虑，因为一旦有一项确认不满足条件，即可直接返回 `false`，而 `_.each` 无法做到这一点。

## invoke

> Calls the method named by **methodName** on each value in the list. Any extra arguments passed to invoke will be forwarded on to the method invocation.

``` javascript
_.invoke([[5, 1, 7], [3, 2, 1]], 'sort');
=> [[1, 5, 7], [1, 2, 3]]
```

乍一看好像和 `_.map` 没啥区别，但文档中特地强调是 `methodName`，也就是说会调用每个元素对应的**方法**。例如说：

``` javascript
_.invoke([[5, 1, 7], [3, 2, 1]], 'toString');
=> ["5,1,7", "3,2,1"]
```

源代码：

``` javascript
// Invoke a method (with arguments) on every item in a collection.
_.invoke = function(obj, method) {
  var args = slice.call(arguments, 2); // 获得要传递的 `arguments`（前两个忽略）
  var isFunc = _.isFunction(method);
  return _.map(obj, function(value) {
    var func = isFunc ? method : value[method]; // 取出对应的方法
    return func == null ? func : func.apply(value, args);
  });
};
```

可以注意到代码中针对 `method` 是函数的情况作了单独处理，当其为函数时直接调用它，可以通过 `this` 访问对应的元素。

``` javascript
_.invoke([1,2,3],function(){return this*2})
=> [2, 4, 6]
```

### _.max/_.min

``` javascript
// Return the maximum element (or element-based computation).
_.max = function(obj, iteratee, context) {
  var result = -Infinity, lastComputed = -Infinity, // 初始值为无限小
      value, computed;
  if (iteratee == null && obj != null) { // 没有给定回调则直接取出元素
    obj = isArrayLike(obj) ? obj : _.values(obj);
    for (var i = 0, length = obj.length; i < length; i++) {
      value = obj[i];
      if (value > result) {
        result = value;
      }
    }
  } else {
    iteratee = cb(iteratee, context);
    _.each(obj, function(value, index, list) {
      computed = iteratee(value, index, list); // 给定回调的情况下使用回调函数对每个取出的元素进行处理
      if (computed > lastComputed || computed === -Infinity && result === -Infinity) {
        result = value;
        lastComputed = computed;
      }
    });
  }
  return result;
};
```
奇怪的是，这里前部分的遍历同样可以用 `_.each` 实现，这里却选择了前半部分使用 `for` 循环，后半部分使用 `_.each`。

我尝试用下面的代码取代原代码，并且顺利的通过了测试：

``` javascript
_.max = function(obj, iteratee, context) {
  var result = -Infinity, lastComputed = -Infinity,
      computed;
  // 判断 iteratee 更复杂了些，是因为后面的版本有更新
  iteratee = !(iteratee == null || (typeof iteratee == 'number' && typeof obj[0] != 'object') && obj != null) && cb(iteratee, context);
  _.each(obj, function(v, index, list) {
    computed = iteratee ? iteratee(v, index, list) : v;
    if (computed > lastComputed || computed === -Infinity && result === -Infinity) {
      result = v;
      lastComputed = computed;
    }
  });
  return result;
};
```

后来在 [#1448 Optimize _.max and _.min for performance](https://github.com/jashkenas/underscore/pull/1448) 看到应该是出于性能考虑。

### _.shuffle

洗牌，返回一个数组的随机洗牌（打乱顺序）拷贝。使用的是 [Fisher–Yates_shuffle](https://en.wikipedia.org/wiki/Fisher%E2%80%93Yates_shuffle) 算法。

听起来很高端，其实非常简单，就是遍历整个数组，每次生成一个随机的 `index`，然后交换当前值和这个 `index` 对应的值。

``` javascript
_.shuffle = function(obj) {
  var set = isArrayLike(obj) ? obj : _.values(obj);
  var length = set.length;
  var shuffled = Array(length);
  for (var index = 0, rand; index < length; index++) {
    rand = _.random(0, index);
    if (rand !== index) shuffled[index] = shuffled[rand];
    shuffled[rand] = set[index];
  }
  return shuffled;
};
```

这里非常有意思的一个问题是： **如何测试洗牌程序？** 因为洗牌的结果是随机的，所以看起来似乎没有什么好的办法进行测试。我们可以看到 `underscore` 里选择测试了一些和随机无关（可测试）的特性，例如不改变元素的内容，以及几乎不可能出现不改变顺序的情况。

``` javascript
QUnit.test('shuffle', function(assert) {
  assert.deepEqual(_.shuffle([1]), [1], 'behaves correctly on size 1 arrays');
  var numbers = _.range(20);
  var shuffled = _.shuffle(numbers);
  assert.notDeepEqual(numbers, shuffled, 'does change the order'); // Chance of false negative: 1 in ~2.4*10^18 （不改变顺序的可能性非常小）
  assert.notStrictEqual(numbers, shuffled, 'original object is unmodified');
  assert.deepEqual(numbers, _.sortBy(shuffled), 'contains the same members before and after shuffle');

  shuffled = _.shuffle({a: 1, b: 2, c: 3, d: 4});
  assert.equal(shuffled.length, 4);
  assert.deepEqual(shuffled.sort(), [1, 2, 3, 4], 'works on objects'); // 重新排序后和原来一致
});
```
随后我看到一篇关于测试洗牌程序的文章：[如何测试洗牌程序 | 酷 壳 - CoolShell.cn](http://coolshell.cn/articles/8593.html)

但是文章里介绍的方法对于单元测试显然是太慢了，不适合在单元测试中采用。

### _.sample

返回一个集合中的 `n` 个随机值。

``` javascript
// Sample **n** random values from a collection.
// If **n** is not specified, returns a single random element.
// The internal `guard` argument allows it to work with `map`.
_.sample = function(obj, n, guard) {
  if (n == null || guard) {
    if (!isArrayLike(obj)) obj = _.values(obj);
    return obj[_.random(obj.length - 1)];
  }
  return _.shuffle(obj).slice(0, Math.max(0, n));
};
```

其实本来是个挺简单的函数，但是这里的 `guard` 有点令人疑惑。`guard` 未在文档中描述，似乎只有内部使用，注释中则强调是使其 **work with `map`**。举个例子：

``` javascript
_.map([[1, 2], [3, 4]], _.sample);
```

其中 `_.map` 传递的参数为

``` javascript
_.map([[1, 2], [3, 4]], function(val, key, obj) {
    // val = [1, 2], [3, 4]
    // key = 0, 1
    // obj = [[1, 2], [3, 4]]
});
```

而事实上 `key` 却被 `_.sample` 当成了 `n`，这里正好还都是数字，无法通过类型来排除。于是 `underscore` 采取的做法是检测到第三个参数的存在时（被 `map` 调用了）即无视 `n`，这样来保证和 `_.map` 能够一起工作。

### _.sortBy

``` javascript
// Sort the object's values by a criterion produced by an iteratee.
_.sortBy = function(obj, iteratee, context) {
  iteratee = cb(iteratee, context);
  return _.pluck(_.map(obj, function(value, index, list) {
    return {
      value: value,
      index: index,
      criteria: iteratee(value, index, list)
    };
  }).sort(function(left, right) {
    var a = left.criteria;
    var b = right.criteria;
    if (a !== b) {
      if (a > b || a === void 0) return 1;
      if (a < b || b === void 0) return -1;
    }
    return left.index - right.index;
  }), 'value');
};
```

其中对 `_.pluck` 和 `_.map` 的组合运用比较有意思，先用 `_.map` 将原数组或者对象 wrap 一层，在完成排序后再通过 `_.pluck` 取出各个元素的 `value` 属性并返回。

### _.groupBy/_.indexBy/_.countBy

``` javascript
// An internal function used for aggregate "group by" operations.
var group = function(behavior) {
  return function(obj, iteratee, context) {
    var result = {};
    iteratee = cb(iteratee, context);
    // 遍历并将 result 的引用和遍历的元素传递给迭代函数
    _.each(obj, function(value, index) {
      var key = iteratee(value, index, obj);
      behavior(result, value, key);
    });
    return result;
  };
};

// _.groupBy([1.3, 2.1, 2.4], function(num){ return Math.floor(num); });
// => {1: [1.3], 2: [2.1, 2.4]}
_.groupBy = group(function(result, value, key) {
  
  // 根据返回值决定分到哪个key下面，若不存在此key则创建
  if (_.has(result, key)) result[key].push(value); else result[key] = [value];
});

// var stooges = [{name: 'moe', age: 40}, {name: 'larry', age: 50}, {name: 'curly', age: 60}];
// _.indexBy(stooges, 'age');
// => {
//   "40": {name: 'moe', age: 40},
//   "50": {name: 'larry', age: 50},
//   "60": {name: 'curly', age: 60}
// }
// 根据特定的 key 进行分类
_.indexBy = group(function(result, value, key) {
  result[key] = value;
});

// _.countBy([1, 2, 3, 4, 5], function(num) {
//   return num % 2 == 0 ? 'even': 'odd';
// });
// => {odd: 3, even: 2}
_.countBy = group(function(result, value, key) {
  if (_.has(result, key)) result[key]++; else result[key] = 1;
});
```

这里创建了一个内部函数 `group` 用于创造各种分组函数，和 `map` 的主要区别是这里的 `result` 会传递给迭代函数，将结果交给迭代函数处理。

### _.toArray

``` javascript
// Safely create a real, live array from anything iterable.
_.toArray = function(obj) {
  if (!obj) return [];
  if (_.isArray(obj)) return slice.call(obj);
  if (isArrayLike(obj)) return _.map(obj, _.identity);
  return _.values(obj);
};
```

将可迭代对象转化为 `Array`，一般来说我们都是直接用 `Array.prototype.slice.call(obj)` 即可，但是针对 `isArrayLike` 使用了 `map`。

然而我将其替换为 `if (_.isArray(obj) || isArrayLike(obj)) return slice.call(obj);` 仍然成功通过了测试。

通过 blame 看到 [toArray: only call Array.prototype.slice on actual arrays. · jashkenas/underscore@26a3055](https://github.com/jashkenas/underscore/commit/26a30551f912f8180e6c2381d0eae4b24259fb70)，又是万恶的 IE 导致的。

当 IE8 下对 NodeList（ArrayLike）使用 `Array.prototype.slice.call` 时会报错：`JScript object expected`。

本文将持续更新。。
