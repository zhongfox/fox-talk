---
layout: post
tags : [nodejs, 单元测试, mocha, assert, chai, sinon]
title: Node.js 单元测试
subtitle: "Mocha, Assert, Chai, Sinon"

---

## 1. Mocha

### 1.1 异步代码测试

* 使用`done`函数

  ```javascript
  it('', function(done) {
    ......
  })
  ```

  mocha是如何知道一个case是否是异步操作呢, 其实是根据it的第二个参数(是一个function)的形参是不是声明了done参数来判断的:

  ```javascript
  function Runnable (title, fn) {
    ......
    this.async = fn && fn.length;
  ```

  只要it的回调函数形参列表大于0, mocha就会等待done的调用.

  done 接收一个可选的error参数, 如果此参数不为空, 表示测试失败

* return 一个promise

  如果promise resolved, 测试成功

  如果promise rejected, 测试失败

  如果promise pending, 测试一直会等待, 直到超时, 测试失败

* 不能同时使用done又使用promise:

  ```javascript
  it('should complete this test', function (done) {
    return new Promise(function (resolve) {
      assert.ok(true);
      resolve();
      done();
    })
  });
  ```

  将抛出异常

* done 接收一个可选的error参数, 如果此参数不为空, 表示测试失败, 所以一下第一个成功, 第二个失败:

  ```javascript
  it('should complete this test', function (done) {
    return new Promise(function (resolve) {
      resolve();
    }).then(done);
  });

  it('should complete this test', function (done) {
    return new Promise(function (resolve) {
      resolve('ok');
    }).then(done);
  });
  ```

  如果要使用`then(done)` 形式, 应该确保resolve参数为空.


### 1.2 HOOKS

* before(func)
* after(func)
* beforeEach(func)
* afterEach(func)

---

## 2. Assert

assert是node.js提供的标准模块, 主要用于断言, 常用的API有:

* `assert(value[, message])`

  期望`value == true`, 否则抛出异常, message是异常消息

  别名`assert.ok()`

* `assert.equal(actual, expected, [message])`

  内部使用`==`进行比较

  相反断言: `assert.notEqual`, 使用`!=`

  严格断言: `assert.strictEqual`, 采用`===`

  相反严格断言: `assert.notStrictEqual`, 使用`!==`

* `assert.deepEqual(actual, expected, [message])`

  常用于数组和对象的比较, 对象比较只比较各自的own properties

  比较采用`==`

  相反断言: `assert.notDeepEqual`

  严格断言: `assert.deepStrictEqual`

  相反严格断言: `assert.notDeepStrictEqual`

* `assert.throws(func[, error][, message])`

  期望执行func产生执行类型错误

  error可以是错误类型, 正则表达式, 或者判断的函数

* `assert.ifError(value)`

  value 期望为逻辑假, 常用于回调中的错误判断.

  否则将value抛出为异常

---

## 3. Chai

### 3.1 风格

提供三种断言风格, 参考[Assertion Styles](http://chaijs.com/guide/styles/)

#### assert

`let assert = require('chai').assert`

* TDD
* 类似node.js原生的`assert-dot`语法
* 最后一个参数是可选的错误消息

#### expect

```javascript
let expect = require('chai').expect
expect(target).chainableCall...
```

* BDD
* 链式语法构建断言
* expect函数的最后一个参数是可选的错误消息

#### should

```javascript
require('chai').should()
target.should.chainableCall...
```

* BDD
* 链式语法构建断言
* 扩展了js 对象, 每个对象增加了should属性

#### expect 和 should 区别

```javascript
// chai.expect 不需要执行, chai.expect(目标对象)作为所有断言的开始
let expect = chai.expect;

// chai.should 需要执行, 对Object.prototype增加should属性, 作为js对象的断言开始
let should = chai.should();
```

因为null, undefined 不是Object, 因此它们不能直接链式使用should, should提供了一下自启动的断言作为替代方案:

* should.exist
* should.not.exist
* should.equal
* should.not.equal
* should.Throw
* should.not.Throw

另外should不兼容IE.

### 3.2 配置项

* config.includeStack
* config.showDiff
* config.truncateThreshold

### 3.3 Chai API

#### 语义链

仅为增强可读性, 无实际的断言功能:

* to
* be
* been
* is
* that
* which
* and
* has
* have
* with
* at
* of
* same

源代码:

```javascript
[ 'to', 'be', 'been'
, 'is', 'and', 'has', 'have'
, 'with', 'that', 'which', 'at'
, 'of', 'same' ].forEach(function (chain) {
  Assertion.addProperty(chain, function () {
    return this;
  });
});
```

#### Flag

flag 只为断言设置标识(Assertion的属性`__flags`)

在源码中是使用`Assertion.addProperty`, 如 `not`:

```javascript
Assertion.addProperty('not', function () {
  flag(this, 'negate', true);
});
```

* `.not`

  设置取反标识: `__flags.negate`

* `.deep`

  设置deep标识: `__flags.deep`

  影响的后续断言:

  `.deep.equal()` 递归深度比较

  `deep.property()` 验证层级属性:

  > expect({ foo: { bar: { baz: 'quux' } } })
  >   .to.have.deep.property('foo.bar.baz', 'quux');

* `.any` `.all`

  设置deep标识: `__flags.any`和`__flags.all` 同时设置, 互异

  影响的后续断言: keys

  `expect(foo).to.have.any.keys('bar', 'baz')`

#### 断言

断言源码使用Assertion.addChainableMethod, 如:

```javascript
Assertion.addChainableMethod('an', an);
Assertion.addChainableMethod('a', an);
```

* `.a` `.an` `.a()` `.an()`

  有2个作用

  1) 类型断言: `.a()` `.an()`, 类型采用[type-detect](https://www.npmjs.com/package/type-detect)

  2) 语义链 `.a` `.an`: `expect(foo).to.be.an.instanceof(Foo)`

* 相等判断

  `.equal(value)` 采用`===` 进行比较

  `.eql(value)`

    递归深度比较

    比较时采用的`===`, 不允许类型转换

* `.include(value)`

  可以用于数组, 字符串, 对象key的判断:

  ```javascript
  expect([1,2,3]).to.include(2);
  expect('foobar').to.contain('foo');
  expect({ foo: 'bar', hello: 'universe' }).to.include.keys('foo')
  ```

* 值判断:

  `.true` 判断测试目标值为true

  `.false` 判断测试目标值为false

  `.null` `.undefined` `.NaN` 以此类推

  `.exist` 非undefined, 非null

  `.empty` length 为0, 或者对象的keys length为0

* 数值比较

  `.above(value)` 大于

  `.least(value)` 小于等于

  `.below(value)` 小于

  `.most(value)` 大于等于

  `.within(start, finish)` 数值范围比较

* `.instanceof(constructor)` 使用instanceof进行判断

* `.property(name, [value])`

  判断有该属性, 可选的value用于判断属性值

  `.deep.property` 允许数值或者层级对象判断属性存在性:

  `expect(deepObj).to.have.deep.property('green.tea', 'matcha')`

  `expect(deepObj).to.have.deep.property('teas[2].tea', 'konacha')`

* `.match(regexp)` 正则判断

* `.string(string)` 字符串包含另一个字符串

---

## 4. Sinon

### 4.1 spy/stub/mock

* spy

  用于监视一个函数的调用情况, 相当于给该函数增加一层wrapper, 被监视的函数在wrapper下层, 最终会被调用.

* stub

  用于替代一个函数的调用, 往往用于目标函数调用场景复杂的情况, stub是目标函数的替身, 不会真正被调用.

  stub 是一种spy, stub 支持所有的spy API, 另外支持一套可以改变stub行为的操作.

  主要使用场景:

  1) 控制指定方法执行指定的路径用于指定的场景测试, 比如强制要求某方法抛出异常

  2) 避免指定方法真正调用, 因为在单元测试中这个真正操作的现实条件并不具备, 比如和服务器交互的网络操作.

* mock

  用于监视一个object的活动,如目标object的方法调用等, object的mock收到的数据或是调用并没有真正执行, 一切针对mock的调用都是假的。所以mock可以用来测试具有side effect的函数.


**比较**

|      | 目标对象 | 操作是否真正执行 | 作用                                            |
|------|----------|------------------|-------------------------------------------------|
| spy  | 函数     | 执行             | 监视, 记录函数调用, 并提供类似断言的执行判断API |
| stub | 函数     | 不执行           | 模拟函数调用, 替换函数的行为                    |
| mock | 对象     | 不执行           | 模拟对象上的操作                                |

### 4.2 Spy

#### 创建与卸载

有2种spy, 一种spy是基于匿名函数, 常常用于判定回调函数是否执行:

```javascript
var callback = sinon.spy(); //作为回调函数使用
......
assertTrue(callback.called);
```

另一种是基于已有函数的spy: `sinon.spy(object, "method")`

* 创建spy:

  * `var spy = sinon.spy()`
  * `var spy = sinon.spy(myFunc)`
  * `var spy = sinon.spy(object, "method")` 其中 `spy === object.method`, 二者都可用于`restore()`

* spy重置状态: `spy.reset()`

* 卸载spy:

  `spy.restore()` 只能用于基于已有函数的spy

#### spy API

Spy API 主要用于判断, 类似断言的作用, API的特点是使用语法过去时

* 判断不同spy的执行先后顺序

  `spy.calledBefore(anotherSpy) -> bool`

  `spy.calledAfter(anotherSpy) -> bool`

* 判断spy调用时的this:

  `spy.calledOn(obj) -> bool` 至少用obj调用过该spy一次

  `spy.alwaysCalledOn(obj) -> bool`

  允许使用matcher: `spyCall.calledOn(sinon.match(fn))`

* 判断spy调用时的参数:

  `spy.calledWith(arg1, arg2, ...) -> bool` 允许多余参数, 至少验证通过一次

  `spy.alwaysCalledWith(arg1, arg2, ...) -> bool` 允许多余参数, 每次调用都验证

  `spy.calledWithExactly(arg1, arg2, ...)` 不允许多余参数, 至少验证通过一次

  `spy.alwaysCalledWithExactly(arg1, arg2, ...)` 不允许多余参数, 每次调用都验证

* 判断spy调用时的参数, 支持matcher:

  `spy.calledWithMatch(arg1, arg2, ...)` 等价于`spy.calledWith(sinon.match(arg1), sinon.match(arg2), ...)`

  `spy.alwaysCalledWithMatch(arg1, arg2, ...)` 等价于 `spy.alwaysCalledWith(sinon.match(arg1), sinon.match(arg2), ...)`

* 判断spy调用抛出异常

  `spy.threw() -> bool`

  `spy.threw("TypeError")`

* 判断spy的返回值

  `spy.returned(obj)` `spy.alwaysReturned(obj)`

spy 的常用属性:

* `spy.thisValues` 数组, 每次调用时的this

* `spy.returnValues` 数组, 每次的返回值

* `spy.args`

  一个二维数组, 一维是调用, 二维是每次调用的参数

  `assertEquals(message, spy.args[0][0])`

  也可以用`getCall` 获得指定调用的参数: `spy.getCall(0).args[0])`

* `spy.callCount` 调用次数

* `spy.called` `spy.notCalled`

* `spy.calledOnce` `spy.calledTwice` `spy.calledThrice`

* `spy.firstCall` `spy.secondCall` `spy.lastCall`

### 4.3 Stub

#### 创建与卸载

* 创建

  `var stub = sinon.stub()`

  `var stub = sinon.stub(object, "method")` 要求该方法必须存在

  `var stub = sinon.stub(object, "method", func)` 用指定的方法代替已存在的方法(执行新的行为的一种办法)


* 卸载

  `object.method.restore()`

  `stub.restore()`

#### Stub API

Stub 可以定制条件行为, 一个stub可以有多个条件行为组合.

指定条件:

* `stub.withArgs(arg1[, arg2, ...])` 在指定的参数下执行指定的结果

* `stub.onCall(n)` 在第几次调用时执行指定结果

  `onFirstCall, onSecondCall, onThirdCall`

* 条件可以组合: `.withArgs(42).onFirstCall().returns(1)`

指定行为:

* `stub.returns(obj)`

* `stub.returnsArg(index)` 返回之前withArgs的第几个参数

* `stub.returnsThis()` 返回this, 用于流式调用

* `stub.callsArg(index)` 将之前withArgs的第几个参数进行调用

* `stub.callsFake(fakeFunction)`

* 抛出异常

  `stub.throws()`

  `stub.throws("TypeError")`

  `stub.throws(obj)`

重置行为:

* `stub.resetBehavior()`

自定义行为API:

* `sinon.addBehavior(name, fn)`

  name 是行为API的方法名称, fn的函数参数是`(fake, userargs...)` fake 代表stub对象, userargs 是用户参数

### 4.4 Mock

TODO

### 4.5 Sandbox

主要用途是快速restore所有的spy/stub/mock

创建: `sandbox = sinon.sandbox.create()`

创建的sandbox可以`stub/spy/mock`

`sandbox.restore()` 可以一次性restore该sandbox创建的sinon组件

### 4.6 Matchers

Matchers 主要用于参数和返回值的验证(判断), 如`spy.calledOn, spy.calledWith, spy.returned` 以及 `sinon.assert` `spy.withArgs`

`sinon.assert.calledWith(spy, sinon.match({ author: "cjno" }))` 对象部分属性匹配

`sinon.assert.calledWith(spy, sinon.match.has("pages", 42))` 貌似同上

`callback.withArgs(sinon.match.string).returns(true)` 匹配参数类型

`sinon.match.instanceOf(type)` 匹配类型

#### 组合matcher:

```javascript
var stringOrNumber = sinon.match.string.or(sinon.match.number);
var bookWithPages = sinon.match.instanceOf(Book).and(sinon.match.has("pages"));
```

组合matcher语法返回一个新的matcher

#### 自定义matcher

```javascript
var newmatcher = sinon.match(function (value) {
  return !!value;
}, "match失败时的错误消息");
```

---

### 参考资料

* [Mocha](https://mochajs.org/)
* [Assert](https://nodejs.org/api/assert.html)
* [Chai](http://chaijs.com/api/)
* [Sinon](http://sinonjs.org/)
