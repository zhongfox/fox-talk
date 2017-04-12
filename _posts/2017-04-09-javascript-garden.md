---
layout: post
tags : [javascript, 调试, nodejs]
title: Javascript 秘密花园
subtitle: "意外的收获"

---

很多年前学习过[Javascript 秘密花园](https://bonsaiden.github.io/JavaScript-Garden/)当时被文中的「奇技淫巧」所折服. 最近在编写[Koa800](https://github.com/zhongfox/koa800)过程中实践了一下BDD, 顺便研究了一下nodejs中测试相关的类库的源码, 有一些意外的收获, 比如[Mocha](https://mochajs.org/)、[chai](http://chaijs.com/api/)、[sinon](http://sinonjs.org/)这些类库, 除了要实现相应功能, 还需要提供很好的可读性, 链式调用, DSL等体验, 因此它们的实现使用了一些平时编程不常用的javascript技巧, 这是我意外发现的一处「Javascript 秘密花园」.

---

### 通过属性读取来改变对象状态

chaijs 提供的BDD 链式调用API 具有很好的可读性:

> expect(foo).to.be.true;

> expect('foo').to.not.equal('bar');

> expect([[1, 2], 3]).to.deep.equal([[1, 2], 3]);

`expect(tartge)` 返回一个断言判断对象, Assertion 实例.

chai 提供的 API 大致可以分为3类:

* 语义链: 仅是为了提供链式调用的可读性, 如上面的`to` `be`等, 即使把这些去掉也不影响结果
* 标识设置器: 用于设置Assertion的 flag, 如`not` 设置状态`negate`, `deep`设置状态
* 断言: 如上面的equal

让我感兴趣的是标识设置器, 如上, `not` `deep` 等是Assertion实例的属性读取, 而不是方法, 但是读取这些属性却可以改变对象的状态, 通常读取属性是没法执行操作的.

如果用ruby来实现, 事情很简单, 因为ruby中的方法调用可以省略`()`, 因此在ruby中`not` `deep`这些可以设计为Assertion的实例方法; 但是在Javascript中方法调用是不能省略`()`的, 所以chai的实现是另有奥秘.

查阅源码, chai使用了[Object.defineProperty](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty), 对Assertion上的标识属性如`not` `deep`等设置了getter, 在getter中可以进行属性设置.

大致的源码:

```javascript
Assertion.addProperty('not', function () {
  flag(this, 'negate', true);
});

......

Assertion.addProperty = function (name, fn) {
  util.addProperty(this.prototype, name, fn);
};

......

// addProperty 定义如下
// lib/chai/utils/addProperty.js
function (ctx, name, getter) {
  Object.defineProperty(ctx, name,
    { get: function addProperty() {
        var old_ssfi = flag(this, 'ssfi');
        if (old_ssfi && config.includeStack === false)
          flag(this, 'ssfi', addProperty);

        var result = getter.call(this);
        return result === undefined ? this : result;
      }
    , configurable: true
  });
};

```

---

### 追踪属性读取

mocha 在处理异步调用时, 有一个规则: 要么调用done告诉mocha异步程序结束了, 要么return 一个pormise

> Alternately, instead of using the done() callback, you may return a Promise

> In Mocha v3.0.0 and newer, returning a Promise and calling done() will result in an exception

如果选择利用done作为异步调用结束的通知机制, mocha是如何知道当前case需要等待异步调用结束呢?

测试了一下, 猜测标识来源于it的第二个参数的形参列表:

```javascript
it('test title', function(done) {
  // mocha 认为这是一个同步调用, 不会等待已调用 (不考虑返回promise的情况)
});


it('test title', function(done) {
  // mocha 认为这个case包含异步调用, 将等待done的调用 (不考虑返回promise的情况)
});
```

以上是我当时的猜测, 需要看下源码验证一下.

javascript 可以通过function的属性`length`获取形参的个数, 所以源码中应该有读取`length === 1` 或者`length > 0` 的情况, 尝试grep验证一下, 无奈这个的搜索目标太不显眼, 一时无法定位.

结合之前学习的`Object.defineProperty`, 可以给函数对象设置getter, 在getter中用`console.trace`打印调用栈:

```javascript
let testcase = function(done) {
  expect(1).to.be.equal(1)
}

Object.defineProperty(testcase , 'length', {
  get: function() {
    console.trace()
    return 1
  }
});

it('try to location source', testcase)
```

输出:

```
at Function.Object.defineProperty.get (....../test/s.test.js:40:13)
at Test.Runnable [as constructor] (/usr/local/lib/node_modules/mocha/lib/runnable.js:52:24)
at new Test (/usr/local/lib/node_modules/mocha/lib/test.js:28:12)
```

找到源码出处: `mocha/lib/runnable.js:52:24`, 如下:

```javascript
function Runnable (title, fn) {
  ......
  this.async = fn && fn.length;
```

推测得证.

稍加包装, 可以得到一个对象属性读取追踪函数:

```javascript
function traceAccessor(target, attr) {
  let oldAttr = target[attr];

  Object.defineProperty(target, attr, {
    get: function() {
      console.trace();
      return oldAttr;
    }
  });
}
```

该函数对于对象上方法调用的查找同样有效, 属性方法调用也需要先读取.

---

### Hack Node.js 包管理

#### 利用require.cache

1. 清除缓存, 以便require时可以重新加载:

   `require.cache[a_module_path]`

2. 手动构造缓存, 用于替换已经缓存的模块:

  ```
  require.cache[a_module_path] = {
    id: a_module_path,
    filename: a_module_path,
    loaded: true,
    exports: a_module_exports
  };
  ```


#### 解决node.js中的循环依赖

```javascript
// a.js
var a = {};
var b = require('./b');

a.hello = function () {
  console.log('a hello!');
  b.hello();
};

a.bye = function() {
  console.log('a bye!');
}

module.exports = a;
```

```javascript
// b.js
var a = require('./a');
var b = {};

b.hello = function () {
  console.log('b hello!');
  console.log('b bye!');
  a.bye();
};

module.exports = b;
```

```javascript
// main.js
var a = require('./a');

a.hello();
```

执行出错:
```
% node main.js
a hello!
b hello!
b bye!
  a.bye();
    ^
TypeError: a.bye is not a function
```


错误复盘:

* `mian.js` 执行 `require a.js`
  * `a.js` 开始解释执行, 在`a.js` exports之前, `require.cache[a.js]`将指向一个代表a模块的临时对象.
  * `a.js` 中`require('./b')`, `b.js` 开始解释执行
    * `b.js` 中`require('./a')`, 这一步将直接获取到第二步中的空对象, 这个空对象将被`b.js`中的a变量保留
    * `b.js` 解释执行完毕, `require.cache[a.js]` 将指向`b.js`的exports对象
  * `a.js`完成执行, `require.cache[a.js]`将指向新的对象: `a.js`的exports对象
* `mian.js` 执行`a.hello()`

问题的关键是: 一个模块在开始执行到exports之前, `require.cache` 会有一个临时对象, 当该模块完成exports时, `require.cache`会重新指向exports的结果. 在这个过程中, 如果有其他模块require了这个未完成exports的模块, 将得到临时对象, 并且得不到更新.

修改: 将模块在陷入循环引用之前, 提前exports:

```javascript
// a.js
var a = {};
module.exports = a;

var b = require('./b');

a.hello = function () {
  console.log('a hello!');
  b.hello();
};

a.bye = function() {
  console.log('a bye!');
}
```


#### 改变执行模块的寻包路径

当模块执行`require`时, 寻包路径是根据`module.paths`依次遍历, 大致的规则是根据当前模块的路径, 向上查找`node_modules`中的模块.

有些不常见的需求, 希望改变某些指定目标模板的寻找路径, 可以通过修改`Module._resolveFilename`来实现, 注意该方法是一个内部方法, 修改需要慎重:

```javascript
let moduleNames = ['the-module-names-you-want-change-find-path'];
let Module = require('module');
let realResolve = Module._resolveFilename;
Module._resolveFilename = function fakeResolve(request, parent) {
  if (moduleNames.indexOf(request) >= 0) {
    return realResolve(path.join('the/path/you/want/be', request), parent);
  }
  return realResolve(request, parent);
};
```

#### proxyquire

<https://github.com/thlorenz/proxyquire>

proxyquire 也用于实现替换require依赖, 不同在于: proxyquire并不改变目标模块的寻包路径, 而是创建一个新的模块, 用户可以使用新的模块替代目标模块


```javascript
targetModulePath = 'target/module.path'
newModule = proxyquire(targetModulePath, {
  'module1': fakeModule1,
  'module2': fakeModule2,
});
```

上面的代码实现了以下几点:

* `require(targetModulePath)` 不会受到任何影响
* 在后续代码中, 可以使用newModule代替`require(targetModulePath)`, newModule执行了targetModulePath的代码, 只是targetModulePath代码中的2个指定模块被替换了.


---

### bind

bind 和 apply, call 类似, 都可以绑定this, 对比一下:

* apply 、 call 、bind 三者的第一个参数都是用来改变函数的this对象的指向的
* apply 、 call 、bind 三者都可以利用后续参数传参
* bind 是返回对应函数, 便于稍后调用(bind 创建了新的函数, 称为绑定函数)；apply 、call 则是立即调用

应用一: 利用bind复制模板:


```
let env = {} // 一些模板中需要的变量
let sourceContent = yield fs.readFile(sourcePath, 'utf8');
let targetContent = (new Function('return `' + sourceContent + '`')).bind(env)();
yield fs.writeFile(targetPath, targetContent);
```

模板中可以使用模板变量, 获得env中的变量.

----

### 参考资料

* [Circular dependencies in node.js](https://coderwall.com/p/myzvmg/circular-dependencies-in-node-js)
