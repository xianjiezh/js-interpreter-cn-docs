# JS-Interpreter 中文文档

翻译自 [JS-Interpreter Documentation](https://neil.fraser.name/software/JS-Interpreter/docs.html)

JS-Interpreter 是用 JavaScript 写的具有沙箱环境的 JavaScript 解析器. 它可以让你任意的, 一行一行地执行 JavaScript 代码. 它的执行过程与主要的 JavaScript 代码环境是分离开的. JS-Interpreter 的多个实例可以允许多线程并发JavaScript, 而无需使用Web Workers.

在这可以你可以尝试一下它 : [JS-Interpreter demo](https://neil.fraser.name/software/JS-Interpreter/index.html)

获取它的源代码 [源代码](https://github.com/NeilFraser/JS-Interpreter)

## 使用方法

- 引入这两个 JavaScript 源代码
``` HTML
<script src="acorn.js"></script>
<script src="interpreter.js"></script>
```
- 当然, 你也可以选择它们的压缩包(70kb)
``` HTML
<script src="acorn_interpreter.js"></script>
```
然后, 实例化一个 interpreter, 并且把你需要还原的 JavaScript 代码放进去

``` javascript
var myCode = 'var a=1; for(var i=0;i<4;i++){a*=i;} a;';
var myInterpreter = new Interpreter(myCode);
```
可以随时添加其他 JavaScript 代码(经常用于交互式调用(译者注:我猜是事件之类的)预先定义的函数)
``` javascript
myInterpreter.appendCode('foo();');
```
为了去一步一步地跑这些代码, 需要重复地去调用 step 函数, 直到它返回 false
``` javascript
function nextStep() {
  if (myInterpreter.step()) {
    window.setTimeout(nextStep, 0);
  }
}
nextStep();
```
或者, 如果已知代码里面没有死循环, 则可以直接调用 run 函数执行一次完成全部的 step
```javascript
myInterpreter.run();
```

在代码遇到异步API调用的情况下(见下文), 如果阻塞了并且需要在以后重新执行, `run`将返回 true

## 外部的 API
与 `eval` 类似, 最后一个语句的结果可以在 myInterpreter.value 中找到
```JavaScript
var myInterpreter = new Interpreter('6 * 7');
myInterpreter.run();
alert(myInterpreter.value);
```
另外, API 的调用可以在创建的时候被添加到 interpreter, 下面是添加了 `alert()` 和 变量 `url`
```  JavaScript  
var initFunc = function(interpreter, scope) {
  interpreter.setProperty(scope, 'url', String(location));

  var wrapper = function(text) {
    return alert(text);
  };
  interpreter.setProperty(scope, 'alert',
      interpreter.createNativeFunction(wrapper));
};
var myInterpreter = new Interpreter(myCode, initFunc);
```
[JSON demo](https://neil.fraser.name/software/JS-Interpreter/demos/json.html) 是在浏览器和 interpreter 之间转换 JSON 的一个例子. 有关更复杂的示例, 请参阅 `initGlobalScope` 函数, 该函数为 Math, Array, Function 和其他全局变量创建API. 

异步的 API 函数也可以被包起来, 使它们看起来与解释器同步. 例如, 可以在 `initFunc` 中定义返回 XMLHttpRequest 内容的getXhr(url)函数, 如下所示:
```javascript
var wrapper = function(href, callback) {
  var req = new XMLHttpRequest();
  req.open('GET', href, true);
  req.onreadystatechange = function() {
    if (req.readyState == 4 && req.status == 200) {
      callback(req.responseText);
    }
  };
  req.send(null);
};
interpreter.setProperty(scope, 'getXhr',
    interpreter.createAsyncFunction(wrapper));
```
上面的代码片段使用了 `createAsyncFunction`, 正如之前使用的 `createNativeFunction` 一样. 区别是被包裹(wrapped)的异步函数的返回值被忽略了. 取而代之的是, 当 wrapper 函数被调用的时候, 会有一个回调函数会被传递进去. 当 wrapper 函数要有返回值的时候, 它会调用这个回调函数, 这样你就有了它的返回值了. 从 JS-Interpreter 内部运行的代码的角度来看, 进行了函数调用并立即返回结果. 

这里是一个能跑的例子 [async demo](https://neil.fraser.name/software/JS-Interpreter/demos/async.html)

## 序列化
JS-Interpreter的一个独特功能是它能够暂停执行, 序列化(把变量从内存中变成可存储或传输的过程)当前状态, 然后在稍后的时间点恢复执行. 保留了循环, 变量, 闭包和所有其他状态.

使用这个功能包括连续执行在服务器重新启动后继续执行的程序, 加载已计算到某个点的堆栈映像, 分支执行或回滚到一个已经被存储了的状态.

一个缺点是序列化格式不是可读的, 并且也不能保证 JS-Interpreter 的未来版本能够解析旧版本的序列化. 另一个缺点是序列化格式相当大; 在标准的 polyfill 下, 它的开销为 60kb. 

这里是一个能跑的例子 [serialization demo](https://neil.fraser.name/software/JS-Interpreter/demos/serialize.html)

## 多线程
JavaScript 是单线程的语言, 但是 JS-Interpreter 允许同时运行多个进程. 创建两个或更多的独立进程, 它们彼此分开运行是很简单的: 只需创建两个或多个 Interpreter 实例, 每个实例都有自己的代码, 并且可以调用每个 interpreter 的 step 函数. 它们可以通过提供的任何外部 API 间接地相互通信. 

稍微复杂的情况是两个或多个线程想要共享相同全局作用域. 要实现这一点, 1)创建一个JS-Interpreter, 2)创建一个单独的堆栈列表, 3)将每个堆栈的根节点的`.scope`属性分配给 interpreter 的 `.global`属性, 4)然后将所需的堆栈分配给调用 step 之前interpreter的 `stateStack` 属性.  

这里有一个例子: [tread demo](https://neil.fraser.name/software/JS-Interpreter/demos/thread.html)

## 限制
interpreter 实现的 JavaScript 版本与在浏览器中执行的版本略有不同
- API
没有暴露 DOM 的 API, 这也是沙箱的重点. 如果您需要这些, 请编写自己的接口

- ES6 
最近添加的 JavaScript (例如 let 或 Set)未实现. 如果您需要超过 ES5, 请为这个项目贡献代码.

- toString & valueOf
将对象转换为原语时，不会调用用户创建的函数

- 性能
interpreter 并不是十分快, 它大概比原生 JS 慢 200 倍

## 依赖
唯一的依赖就是 [Acorn](http://marijnhaverbeke.nl/acorn/), 这是 Marijn Haverbeke 写的 JavaScript 的还原器. 它已经被包含进了 JS-Interpreter 

## 兼容性
需要浏览器支持使用 `Object.create(null)`在 Acorn 和 JS-Interpreter 中创建 Object. 这最低的浏览器要求:
- Chrome 5
- Firefox 4.0
- IE 9
- Opera 11.6
- Safari 5

## 免责声明
这个并不是 Google 官方的产品