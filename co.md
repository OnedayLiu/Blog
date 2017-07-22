这段时间在看Koa的源码，发现Koa的处理异步回调函数是基于co框架上面的，于是想进一步深究这个co到底做了什么黑魔法

## co的github地址：
https://github.com/tj/co
从文档的API可以看出，co支持promise，thunks，array，object，generator这几大类型

## co的本质
co一旦开始执行就会像永动机一样不断执行下去，直到遇到return才会停止，现在问题来了，为什么会co可以不断执行下去？如何处理异步回调函数？

1. 为什么co可以不断执行下去？
   这主要是利用generator的特性，co接受一个generator作为入参，co会不断执行generator里面的yield语句，直到遇到return才停止

2. 如何处理异步回调函数？
   这主要是利用Promise的特性，yield后面的语句本质上是Promise对象的，咦，不对啊，yield后面不是可以支持Promise,thunks,array,object,generator吗，怎么现在说是Promise对象的呢？别急，下面会解释这个问题

**co返回Promise对象，之后可以用then进行接收；co是利用generator和Promise特性进行处理，要想真正理解co的原理，就要先好好理解generator和Promise了~**



## 细读源码
代码不多，仅仅两百多行，下面来探究其中的奥妙，我把代码连同注释也一起复制过来，中文注释是我在读的过程中加上的笔记，方便日后翻看

```javascript

/**
 * slice() reference.
 */

var slice = Array.prototype.slice;

/**
 * Expose `co`.
 */

module.exports = co['default'] = co.co = co;

/**
 * Wrap the given generator `fn` into a
 * function that returns a promise.
 * This is a separate function so that
 * every `co()` call doesn't create a new,
 * unnecessary closure.
 *
 * @param {GeneratorFunction} fn
 * @return {Function}
 * @api public
 */

/**
 * co.wrap仅仅是对co的一层封装，方便把代码包裹起来，多处复用，最终调用的还是co函数
 */
co.wrap = function (fn) {
  createPromise.__generatorFunction__ = fn;
  return createPromise;
  function createPromise() {
    return co.call(this, fn.apply(this, arguments));
  }
};

/**
 * Execute the generator function or a generator
 * and return a promise.
 *
 * @param {Function} fn
 * @return {Promise}
 * @api public
 */

// co实际上是返回一个Promise对象，并等待这个对象的处理结果
// 即resolve和reject
function co(gen) {
  var ctx = this;
  var args = slice.call(arguments, 1)

  // we wrap everything in a promise to avoid promise chaining,
  // which leads to memory leak errors.
  // see https://github.com/tj/co/issues/180
  return new Promise(function(resolve, reject) {
    // 开始调用generator，生成对应的实例，等待yield的指令执行具体方法
    if (typeof gen === 'function') gen = gen.apply(ctx, args);
    if (!gen || typeof gen.next !== 'function') return resolve(gen);

    // generatior开始执行，直到遇到return才会停止
    onFulfilled();

    /**
     * @param {Mixed} res
     * @return {Promise}
     * @api private
     */
    
    function onFulfilled(res) {
      var ret;
      try {
        // res是上一个yield指令执行的实际结果，这里把res结果赋值给上一个yield语句，
        // 作为上一个yield语句的执行结果
        // 这里又开始执行下一条yield语句，并把结果{"done":"true/false", "value": "res"}
        // 传给next函数进行结果分析 
        ret = gen.next(res);
      } catch (e) {
        return reject(e);
      }
      // 开始分析yield结果
      next(ret);
    }

    /**
     * @param {Error} err
     * @return {Promise}
     * @api private
     */

    function onRejected(err) {
      var ret;
      try {
        ret = gen.throw(err);
      } catch (e) {
        return reject(e);
      }
      next(ret);
    }

    /**
     * Get the next value in the generator,
     * return a promise.
     *
     * @param {Object} ret
     * @return {Promise}
     * @api private
     */

    // 这个方法主要对yield语句的结果{"done":"true/false", "value": "res"}进行分析
    function next(ret) {
      // 如果done是true，则表示generator运行结束,并将ret.value作为co
      // （即Promise对象的处理结果，注意用then或catch接收此处理结果）
      if (ret.done) return resolve(ret.value);
      // 如果done是false，则表示generator还没运行结束，并将ret.value
      // 转换为Promise并生成promise对象，将处理结果抛给then处理
      // 即处理结果是onFulfilled和onRejected的入参，继续执行这两个方法
      // 达到一种永动机的状态
      var value = toPromise.call(ctx, ret.value);
      if (value && isPromise(value)) return value.then(onFulfilled, onRejected);
      return onRejected(new TypeError('You may only yield a function, promise, generator, array, or object, '
        + 'but the following object was passed: "' + String(ret.value) + '"'));
    }
  });
}
```

**即在co里面的代码，如果是yield语句则会等待该语句执行完毕才会继续下一条yield语句，如果是return语句则不管return后面的是什么类型的代码，无论是普通function也好，promise也好，array也好都会作为结果原封不动作为结果返回**