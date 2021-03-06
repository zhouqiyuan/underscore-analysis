
## underscore源码解析(2)--有关迭代的两个内部函数(Baseline setup)

有两个重要的内部函数对 underscore 实现迭代方法十分重要，underscore 定义的所有有关迭代的方法基本都用到了这两个内部函数。我们通过 `_.each()` 和 `_.map()` 两个迭代方法来学习一下这两个内部函数。

### 1. 用 optimizeCb 优化迭代函数（回调函数）
```JavaScript
/**
 * each方法将ES5的forEach换为了函数式表达，既能迭代数组又能迭代对象
 * @param obj 待迭代集合
 * @param iteratee 迭代过程中每个被迭代元素的回调函数
 * @param context 上下文
 * @example
 * // 数组迭代
 * _.each([1, 2, 3], alert);
 * // 对象迭代
 * _.each({one: 1, two: 2, three: 3}, alert);
 */
_.each = _.forEach = function(obj, iteratee, context) {

  // 优化回调函数
  iteratee = optimizeCb(iteratee, context);

  var i, length;

  // 区分数组和对象的迭代过程
  if (isArrayLike(obj)) {
    for (i = 0, length = obj.length; i < length; i++) {

      // 数组的迭代回调函数传入三个参数（迭代值， 迭代索引， 迭代对象）
      iteratee(obj[i], i, obj);
    }
  } else {
    var keys = _.keys(obj);
    for (i = 0, length = keys.length; i < length; i++) {

      // 对象的迭代回调传入三个参数（迭代值， 迭代属性， 迭代对象）
      iteratee(obj[keys[i]], keys[i], obj);
    }
  }

  // 返回对象自身， 以便进行链式调用
  return obj;
};
```
上面的代码定义了 each 方法，函数体内首先是对回调函数进行优化，这里用到了
underscore 的一个内部函数 `optimizeCb`，我们来看一下这个内部函数是怎么工作的

```JavaScript
/**
 * 优化回调（函数中传入的回调）
 * @param func 待优化的回调函数
 * @param context 执行上下文
 * @param argCount 回调函数中参数的个数
 *
 */

var optimizeCb = function(func, context, argCount) {

  // 该函数的主要作用就是为回调函数绑定上下文，所以如果没有指定上下文，
  // 则直接将其返回
  if (context === void 0) return func;

  // 如果有上下文，那么根据参数的不同分别进行优化，没有传入则默认为3个
  switch (argCount == null ? 3 : argCount) {

    // 回调参数为1，即迭代过程中我们只需要值
    case 1: return function(value) {
      return func.call(context, value);
    };

    // 2个参数的情况不需要

    // 3个参数分别为（值，索引，被迭代的集合对象）
    case 3: return function(value, index, collection) {
      return func.call(context, value, index, collection);
    };

    // 4 个参数（累加器，值，索引， 被迭代的集合对象）
    case 4: return function(accumulator, value, index, collection) {
      return func.call(context, accumulator, value, index, collection);
    };
  }


  return function() {
    return func.apply(context, arguments);
  };

};

```
其中 switch 那部分并没有指定回调中参数的个数，而只是判断参数的个数，确保优化的时候不会弄乱参数的个数。
说白了，optimizeCb 函数唯一的作用就是在不扰乱回调参数个数的情况下为回调指定上下文。

### 2. 用 cb 判断类型并优化
```JavaScript
/**
 * map(collect)函数将ES5的数组的map方法换为了函数是表达
 * @param obj 对象或数组
 * @param iteratee 传入的回调函数
 * @param context 传入的上下文
 */
_.map = _.collect = function(obj, iteratee, context) {
  iteratee = cb(iteratee, context);

  // 判断传入的集合是数组（或类数组）还是对象
  // 获取集合的长度，并根据这个长度定长初始化一个数组
  var keys = !isArrayLike(obj) && _.keys(obj),
      length = (keys || obj).length,
      results = Array(length);

  // 在数组中存入将每个集合元素传入回调函数后的返回值
  for (var index = 0; index < length; index++) {
    var currentKey = keys ? keys[index] : index;
    results[index] = iteratee(obj[currentKey], currentKey, obj);
  }

  return results;
};
```
上面的代码定义了 map 方法，这个方法的第二个参数可能是函数，对象，字符创等，所以，函数体内一开始要对第二个参数类型进行判断，再选择如何对其进行优化：
```JavaScript
/**
 * 类型判断后优化
 * @param value
 * @param context 执行上下文
 * @param argCount 如果 value 是函数，则代表函数中参数的个数
 */
var cb = function(value, context, argCount) {
  // 如果 value 为空，既 map 中没有传入第二个参数
  // 返回一个能返回自身值的函数
  if (value == null) return _.identity;

  // 如果 value 为函数，调用 optimizeCb 对其进行优化
  if (isFunction(value)) return optimizeCb(value, context, argCount);

  // 如果 value 为对象，返回一个是否匹配属性的函数
  if (isObject(value)) return _.matcher(value);

  // 其他情况（字符串），调用 property，返回一个可以获取对象属性的函数
  return _.property(value);
};
```
总的来说，cb 函数的返回值是一个函数，根据 value 值的类型的不同，返回的函数的功能也不同，对于 cb 函数来说，主要完成下面的功能：
1. 如果 value 为空，那么返回一个能返回自身值的函数（这里返回的是_.identity）
2. 如果 value 为函数，那么调用 optimizeCb 函数，为 value 绑定上下文并将其返回
3. 如果 value 为对象，返回一个是否匹配属性的函数
4. 如果 value 为字符串，那么返回一个可以获取 _对象_ 属性值的函数

### 3. 涉及的依赖
#### 3.1 `_.identity`
```JavaScript
_.identity = function(value) {
  return value;
};
var results = _.map([1,2,3]); // => results：[1,2,3]
```

#### 3.2 `_.property`
这个函数返回一个函数，返回的函数可以得到传入其中的集合（obj）的相应属性值（key）
```JavaScript
var property = function(key) {
  return function(obj) {
    return obj == null ? void 0 : obj[key];
  };
};
var results = _.map([{name:'cxp'},{name:'comma'}],'name');
// => results: ['cxp', 'comma'];
```
#### 3.3 `isArrayLike`
**这个函数可以判断传入的集合是不是一个数组或类数组**，是的话返回 true，不是返回 false
```JavaScript
// 数组所能拥有的最大元素数目
var MAX_ARRAY_INDEX = Math.pow(2, 53) - 1;
var isArrayLike = function(obj) {

  // 获得 obj 集合的 length 属性值
  // 对象没有 length 属性，返回 undefined
  var getLength = property("length"),
      length = getLength(obj);

  // 如果得到的 length 属性值是一个数字，且符合数组长度要求，
  // 那么 obj 是数组或类数组
  return typeof length === "number" && length >= 0 &&
      length <= MAX_ARRAY_INDEX;
}
```
#### 3.4 `isFunction`
```JavaScript
_.isFunction = function(obj) {
  return typeof obj === "function" || false;
}
```

#### 3.5 `isObject`
```JavaScript
_.isObject = function(obj) {
  var type = typeof obj;

  // 函数被认为是对象，而 undefined，null, NaN 等不被认为是对象
  return type === "function" || type === "object" && !!obj;
}
```

#### 3.6 `matcher`
matcher 函数将在之后的文章中介绍，因为它依赖了其他几个 underscore 中定义的关于对象的方法。
