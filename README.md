# JS-Interpreter 中文文档

JS-Interpreter 是用 JavaScript 写的具有沙箱环境的 JavaScript 解析器. 它可以让你任意的, 一行一行地执行 JavaScript 代码. 它的执行过程与主要的 JavaScript 代码环境是分离开的. JS-Interpreter 的多个实例可以允许多线程并发JavaScript，而无需使用Web Workers.

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
然后, 实例化一个 interpreter, 并且把你需要转化的 JavaScript 代码放进去

``` javascript
var myCode = 'var a=1; for(var i=0;i<4;i++){a*=i;} a;';
var myInterpreter = new Interpreter(myCode);
```
可以随时添加其他JavaScript代码(经常用于交互式调用(译者注:我猜是事件之类的)预先定义的函数)
``` javascript
myInterpreter.appendCode('foo();');
```
为了去一步一步地跑这些代码, 需要循环地去调用 step 函数, 直到它返回 false
``` javascript
function nextStep() {
  if (myInterpreter.step()) {
    window.setTimeout(nextStep, 0);
  }
}
nextStep();
```
或者, 如果已知代码里面没有死循环, 则可以通过调用run函数执行一次以完成
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

