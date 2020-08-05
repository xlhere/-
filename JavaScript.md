# JavaScript总结

[toc]

## 1.异步编程

### 1.1.实现Promise.limit()，实现对promise数组的限制运行

```javascript
class LimitPromise {
  constructor (max) {
    // 异步任务“并发”上限
    this._max = max
    // 当前正在执行的任务数量
    this._count = 0
    // 等待执行的任务队列
    this._taskQueue = []
  }

  /**
   * 调用器，将异步任务函数和它的参数传入
   * @param caller 异步任务函数，它必须是async函数或者返回Promise的函数
   * @param args 异步任务函数的参数列表
   * @returns {Promise<unknown>} 返回一个新的Promise
   */
  call (caller, ...args) {
    return new Promise((resolve, reject) => {
      const task = this._createTask(caller, args, resolve, reject)
      if (this._count >= this._max) {
        // console.log('count >= max, push a task to queue')
        this._taskQueue.push(task)
      } else {
        task()
      }
    })
  }

  /**
   * 创建一个任务
   * @param caller 实际执行的函数
   * @param args 执行函数的参数
   * @param resolve
   * @param reject
   * @returns {Function} 返回一个任务函数
   */
  _createTask (caller, args, resolve, reject) {
    return () => {
      // 实际上是在这里调用了异步任务，并将异步任务的返回（resolve和reject）抛给了上层
      caller(...args)
        .then(resolve)
        .catch(reject)
        .finally(() => {
          // 任务队列的消费区，利用Promise的finally方法，在异步任务结束后，取出下一个任务执行
          this._count--
          if (this._taskQueue.length) {
            // console.log('a task run over, pop a task to run')
            let task = this._taskQueue.shift()
            task()
          } else {
            // console.log('task count = ', count)
          }
        })
      this._count++
      // console.log('task run , task count = ', count)
    }
  }
}
```

```javascript
const LimitPromise = require('limit-promise')
const request = require('./request')
// 请求上限
const MAX = 10
// 核心控制器
const limitP = new LimitPromise(MAX)

// 利用核心控制器包装request中的函数
function get (url, params) {
  return limitP.call(request.get, url, params)
}
function post (url, params) {
  return limitP.call(request.post, url, params)
}
// 导出
module.exports = {get, post}
```

[demo](https://github.com/onechunlin/JS_Demo/tree/master/Promise.limit的原生实现)

### 1.2.异步编程的实现方式

    js 中的异步机制可以分为以下几种
    
    第一种最常见的是使用回调函数的方式，使用回调函数的方式有一个缺点是，多个回调函数嵌套的时候会造成回调函数地狱，上下两层的回调函数间的代码耦合度太高，不利于代码的可维护。
    
    第二种是 Promise 的方式，使用 Promise 的方式可以将嵌套的回调函数作为链式调用。但是使用这种方法，有时会造成多个 then 的链式调用，可能会造成代码的语义不够明确。
    
    第三种是使用 generator 的方式，它可以在函数的执行过程中，将函数的执行权转移出去，在函数外部我们还可以将执行权转移回来。当我们遇到异步函数执行的时候，将函数执行权转移出去，当异步函数执行完毕的时候我们再将执行权给转移回来。因此我们在 generator 内部对于异步操作的方式，可以以同步的顺序来书写。使用这种方式我们需要考虑的问题是何时将函数的控制权转移回来，因此我们需要有一个自动执行 generator 的机制，比如说 co 模块等方式来实现 generator 的自动执行。
    
    第四种是使用 async 函数的形式，async 函数是 generator 和 promise 实现的一个自动执行的语法糖，它内部自带执行器，当函数内部执行到一个 await 语句的时候，如果语句返回一个 promise 对象，那么函数将会等待 promise 对象的状态变为 resolve 后再继续向下执行。因此我们可以将异步逻辑，转化为同步的顺序来书写，并且这个函数可以自动执行。
### 1.3为什么使用 setTimeout 实现 setInterval？怎么模拟

```javascript
思路是使用递归函数，不断地去执行 setTimeout 从而达到 setInterval 的效果

function mySetInterval(fn, timeout) {
  // 控制器，控制定时器是否继续执行
  var timer = {
    flag: true
  };
  // 设置递归函数，模拟定时器执行。
  function interval() {
    if (timer.flag) {
      fn();
      setTimeout(interval, timeout);
    }
  }
  // 启动定时器
  setTimeout(interval, timeout);
  // 返回控制器
  return timer;
}
```
    setInterval 的作用是每隔一段指定时间执行一个函数，但是这个执行不是真的到了时间立即执行，它真正的作用是每隔一段时间将事件加入事件队列中去，只有当当前的执行栈为空的时候，才能去从事件队列中取出事件执行。所以可能会出现这样的情况，就是当前执行栈执行的时间很长，导致事件队列里边积累多个定时器加入的事件，当执行栈结束的时候，这些事件会依次执行，因此就不能到间隔一段时间执行的效果。
    
    针对 setInterval 的这个缺点，我们可以使用 setTimeout 递归调用来模拟 setInterval，这样我们就确保了只有一个事件结束了，我们才会触发下一个定时器事件，这样解决了 setInterval 的问题。
 [《用 setTimeout 实现 setInterval》](https://www.jianshu.com/p/32479bdfd851)
 [《setInterval 有什么缺点？》](https://zhuanlan.zhihu.com/p/51995737)

### 1.4.手写一个 Promise

```javascript
class MyPromise {
  constructor(excutor) {
    // 状态
    this._state = "pending";
    // 成功回调
    this._callbacks_resolved = [];
    // 失败回调
    this._callbacks_rejected = [];
    // 执行
    excutor(
      (data) => {
        // 避免重复执行
        if (this._state !== "pending") {
          return;
        }
        // 保存结果
        this._data = data;
        // 状态变更
        this._state = "resolved";
        // 处理
        this._handle();
      },
      (err) => {
        if (this._state !== "pending") {
          return;
        }
        // 错误信息
        this._err = err;
        // 状态变更
        this._state = "rejected";
        this._handle();
      }
    );
  }
  then(onfulfilled, onrejected) {
    return new MyPromise((resolve, reject) => {
      this._callbacks_resolved.push((data) => {
        // 没有处理，直接往后传
        if (!onfulfilled) {
          return resolve(data);
        }
        try {
          let r = onfulfilled(data);
          // 有then函数
          if (r && typeof r.then === "function") {
            return r.then(resolve, reject);
          }
          resolve(r);
        } catch (e) {
          // 有错误直接向后传
          reject(e);
        }
      });
      this._callbacks_rejected.push((err) => {
        if (!onrejected) {
          return reject(err);
        }
        try {
          let r = onrejected(err);
          if (r && typeof r.then === "function") {
            return r.then(resolve, reject);
          }
          resolve(r);
        } catch (e) {
          reject(e);
        }
      });
      this._handle();
    });
  }

  catch(onrejected) {
    return this.then(undefined, onrejected);
  }

  _handle() {
    if (this._state === "pending") {
      return;
    }
    // 为什么要延时，then的东西是不能马上执行的，可以看看micro-task和macro-task，这里只能模拟
    setTimeout(() => {
      if (this._state === "resolved") {
        this._callbacks_resolved.forEach((cb) => cb(this._data));
      } else {
        this._callbacks_rejected.forEach((cb) => cb(this._err));
      }
      this._callbacks_resolved = [];
      this._callbacks_rejected = [];
    });
  }
}

module.exports = MyPromise;
```

[动手写一个promise](http://www.fly63.com/article/detial/8711)

### 1.5.尾部js的加载也是阻塞式的，有什么办法解决吗

```javascript
js的延时加载
	defer的使用 
  defer是一个常用的优化方案，它表示脚本会被延迟到文档完全被解析和显示之后再执行，加载后续文档元素的过程将和js脚本的加载并行进行（异步），这样页面加载会更快
	<script src="main.js" defer></script>
	async的使用
	async是html5的属性，async 属性规定一旦脚本可用，则会异步执行。async不保证按照脚本出现的先后顺序执行，因此，确保两者之前互不依赖非常重要，指定async属性的目的是不让页面等待两个脚本的下载和执行，从而异步加载页面其他内容，异步脚本一定会在页面的load事件前执行，但可能会在DOMContentLoaded事件触发之前或之后执行。支持异步脚本的浏览器有Firefox 3.6、Safari 5 和Chrome。
<script async src="main.js"></script>

js的动态加载
	由于文档对象模型（DOM）允许您使用 JavaScript 动态创建 HTML 的几乎全部文档内容。  当然也就应许我们动态创建<script>元素
  function loadScript(url, callback){
   var script = document.createElement ("script")
   script.type = "text/javascript";
   if (script.readyState){ //IE
    script.onreadystatechange = function(){
     if (script.readyState == "loaded" || script.readyState == "complete"){
      script.onreadystatechange = null;
      callback();
     }
    };
   }else{ //Others
    script.onload = function(){
     callback();
    };
   }
   script.src = url;
   document.getElementsByTagName("head")[0].appendChild(script);
  }
	loadScript("main.js", function(){
  	console.log("File is loaded!");
  });

js动态注入
	创建一个XMLHttpRequest对象，然后下载 JavaScript 文件，接着用一个动态 <script> 元素将 JavaScript 代码注入页面
  var xhr = new XMLHttpRequest();
  xhr.open("get", "main.js", true);
  xhr.onreadystatechange = function(){
   if (xhr.readyState == 4){
    if (xhr.status >= 200 && xhr.status < 300 || xhr.status == 304){
     var script = document.createElement ("script");
     script.type = "text/javascript";
     script.text = xhr.responseText;
     document.body.appendChild(script);
    }
   }
  };
  xhr.send(null);
	向服务器发送一个GET 请求 ,去获取main.js文件，当状态成功，我们把获取的js文件动态加载到页面。采取了它的好处是：它下载后不会自动执行，这使得您可以推迟执行，直到一切都准备好了。另一个优点是，同样的代码在所有现代浏览器中都不会引发异常。需要注意的是，加载的js文件不能跨域获取，所以一般网站通常不采用 XHR 脚本注入技术
```

[js无阻塞加载方式](http://www.fly63.com/article/detial/44)

### 1.6.Promise.all如果有一个reject了，会怎样？如何忽略会报reject的异步(意思就是一系列异步任务如何让它一直执行到最后，不管会不会reject)

[Promise.all中对于reject的处理方法](https://blog.csdn.net/q3254421/article/details/84347433)

### 1.7.Generator函数有哪些接口？Generator函数是异步的，它跟Promise有什么区别

### 1.8.ajax原理，手写实现

```javascript
function createxmlHttpRequest() {
  if (window.XMLHttpRequest) {
    return new XMLHttpRequest();
  } else {
    return new ActiveXObject('Microsoft.XMLHTTP');
  }
}
//元数据格式 data:{"user":"abc","pwd":"123"};
//目标数据格式 convertData(data):"user=abc&pwd=123"
function convertData(data) {
  if (typeof data === 'object') {
    var convertResult = "";
    for (var i in data) {
      convertResult += i + "=" + data[i] + "&";
    }
    // 去掉最后一个&
    convertResult.substring(0, convertResult.length - 1);
    return convertResult;
  } else {
    return data;
  }
}
 
// 原生ajax
function ajax() {
  var ajaxData = {
    type: arguments[0].type || 'GET',
    url: arguments[0].url || '',
    async: arguments[0].async || "true", //是否使用异步
    data: arguments[0].data || null, //发送数据
    dataType: arguments[0].dataType || 'text',
    contentType: arguments[0].contentType || 'application/x-www-form-urlencoded',
    beforeSend: arguments[0].beforeSend || function () {},
    success: arguments[0].success || function () {},
    error: arguments[0].error || function () {},
  }
  ajaxData.beforeSend();
 
  var xhr = createxmlHttpRequest();
  xhr.open(ajaxData.type, ajaxData.url, ajaxData.async);
  xhr.setRequestHeader('Content-Type', ajaxData.contentType);
  xhr.send(convertData(ajaxData.data));
  xhr.onReadyStateChange = function () {
    if (xhr.readyState == 4) {
      if (xhr.status == 200) {
        ajaxData.success(xhr.response);
      } else {
        ajax.error()
      }
    }
  }
}
 
//使用案例
ajax({
  type: "POST",
  url: '/graphql',
  dataType: 'json',
  data: {
    "val1": "abc",
    "val2": 123,
    "val3": "def"
  },
  beforeSend: function () {
 
  },
  success: function (msg) {
    console.log(msg);
  },
  error: function (msg) {
    console.log(msg);
  }
})
```

[了解ajax工作原理及手写ajax](https://blog.csdn.net/chenzeze0707/article/details/84393295)

### 1.9.fetch原理，与ajax区别，优点

[ajax和fetch、axios的优缺点以及比较](https://cloud.tencent.com/developer/article/1359550)

## 2.框架

### 2.1.你能写一下Vue的双向绑定吗？

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    <label for="name">姓名：</label>
    <input id="name" type="text" oninput="changeValue(this)" onfocus="init(this)" v-model="data.name">
    <label for="age">年龄</label>
    <input id="age" type="text" oninput="changeValue(this)" onfocus="init(this)" v-model="data.age">
    <h3>内容展示</h3>
    <p id="text"></p>
    <script>
        let data = {
            name: "李四",
            age: 12
        };
        data = new Proxy(data,{
            set(target,property,value){
                target[property] = value;
                let input = document.querySelector(`input[v-model='data.${property}']`)
                input.value = value;
                text.innerHTML = value;
            }
        })
        function changeValue(ele) {
            let modelDate = ele.getAttribute("v-model")
            property = modelDate.split(".")[1]
            data[property] = ele.value;
        }
        function init(ele){
            let modelData = ele.getAttribute("v-model")
            property = modelData.split(".")[1]
            ele.value = data[property]
        }
    </script>
</body>
</html>
```

[proxy实现双向绑定](https://github.com/onechunlin/JS_Demo/tree/master/Vue双向绑定的实现)

[defineProperty实现双向绑定](https://segmentfault.com/a/1190000014274840)

[[Vue2.x 与Vue3.x 双向数据绑定区别](https://segmentfault.com/a/1190000019101006)](https://segmentfault.com/a/1190000019101006)

### 2.2.Vue 2.x和Vue 3.x的区别

```
数据劫持方式、脚手架命令、UI界面等等
```

[数据劫持](https://www.toutiao.com/a6791364801932034574/?timestamp=1581475651&app=news_article&group_id=6791364801932034574&req_id=202002121047310100260601501E337E8D)

[创建一个项目的脚手架命令](https://cli.vuejs.org/zh/guide/creating-a-project.html#vue-create)

### 2.3. 什么是 MVVM？比之 MVC 有什么区别？什么又是 MVP ？

    MVC、MVP 和 MVVM 是三种常见的软件架构设计模式，主要通过分离关注点的方式来组织代码结构，优化我们的开发效率。
    
    比如说我们实验室在以前项目开发的时候，使用单页应用时，往往一个路由页面对应了一个脚本文件，所有的页面逻辑都在一个脚本文件里。页面的渲染、数据的获取，对用户事件的响应所有的应用逻辑都混合在一起，这样在开发简单项目时，可能看不出什么问题，当时一旦项目变得复杂，那么整个文件就会变得冗长，混乱，这样对我们的项目开发和后期的项目维护是非常不利的。
    
    MVC 通过分离 Model、View 和 Controller 的方式来组织代码结构。其中 View 负责页面的显示逻辑，Model 负责存储页面的业务数据，以及对相应数据的操作。并且 View 和 Model 应用了观察者模式，当 Model 层发生改变的时候它会通知有关 View 层更新页面。Controller 层是 View 层和 Model 层的纽带，它主要负责用户与应用的响应操作，当用户与页面产生交互的时候，Controller 中的事件触发器就开始工作了，通过调用 Model 层，来完成对 Model 的修改，然后 Model 层再去通知 View 层更新。
    
    MVP 模式与 MVC 唯一不同的在于 Presenter 和 Controller。在 MVC 模式中我们使用观察者模式，来实现当 Model 层数据发生变化的时候，通知 View 层的更新。这样 View 层和 Model 层耦合在一起，当项目逻辑变得复杂的时候，可能会造成代码的混乱，并且可能会对代码的复用性造成一些问题。MVP 的模式通过使用 Presenter 来实现对 View 层和 Model 层的解耦。MVC 中的Controller 只知道 Model 的接口，因此它没有办法控制 View 层的更新，MVP 模式中，View 层的接口暴露给了 Presenter 因此我们可以在 Presenter 中将 Model 的变化和 View 的变化绑定在一起，以此来实现 View 和 Model 的同步更新。这样就实现了对 View 和 Model 的解耦，Presenter 还包含了其他的响应逻辑。
    
    MVVM 模式中的 VM，指的是 ViewModel，它和 MVP 的思想其实是相同的，不过它通过双向的数据绑定，将 View 和 Model 的同步更新给自动化了。当 Model 发生变化的时候，ViewModel 就会自动更新；ViewModel 变化了，View 也会更新。这样就将 Presenter 中的工作给自动化了。我了解过一点双向数据绑定的原理，比如 vue 是通过使用数据劫持和发布订阅者模式来实现的这一功能。
   [《浅析前端开发中的 MVC/MVP/MVVM 模式》](https://juejin.im/post/593021272f301e0058273468)
   [《MVC，MVP 和 MVVM 的图示》](http://www.ruanyifeng.com/blog/2015/02/mvcmvp_mvvm.html)
   [《MVVM》](https://juejin.im/book/5bdc715fe51d454e755f75ef/section/5bdc72e6e51d45054f664dbf)
   [《一篇文章了解架构模式：MVC/MVP/MVVM》](https://segmentfault.com/a/1190000015310674)

   [界面之下：还原真实的MV*模式 ](https://github.com/livoras/blog/issues/11)

### 2.4. 什么是 Virtual DOM？为什么 Virtual DOM 比原生 DOM 快？

   ```
    我对 Virtual DOM 的理解是，

    首先对我们将要插入到文档中的 DOM 树结构进行分析，使用 js 对象将其表示出来，比如一个元素对象，包含 TagName、props 和 Children 这些属性。然后我们将这个 js 对象树给保存下来，最后再将 DOM 片段插入到文档中。

    当页面的状态发生改变，我们需要对页面的 DOM 的结构进行调整的时候，我们首先根据变更的状态，重新构建起一棵对象树，然后将这棵新的对象树和旧的对象树进行比较，记录下两棵树的的差异。

    最后将记录的有差异的地方应用到真正的 DOM 树中去，这样视图就更新了。

    我认为 Virtual DOM 这种方法对于我们需要有大量的 DOM 操作的时候，能够很好的提高我们的操作效率，通过在操作前确定需要做的最小修改，尽可能的减少 DOM 操作带来的重流和重绘的影响。其实 Virtual DOM 并不一定比我们真实的操作 DOM 要快，这种方法的目的是为了提高我们开发时的可维护性，在任意的情况下，都能保证一个尽量小的性能消耗去进行操作。
   ```

   详细资料可以参考：
   [《Virtual DOM》](https://juejin.im/book/5bdc715fe51d454e755f75ef/section/5bdc72e6e51d45054f664dbf)
   [《理解 Virtual DOM》](https://github.com/y8n/blog/issues/5)
   [《深度剖析：如何实现一个 Virtual DOM 算法》](https://github.com/livoras/blog/issues/13)
   [《网上都说操作真实 DOM 慢，但测试结果却比 React 更快，为什么？》](https://www.zhihu.com/question/31809713/answer/53544875)

### 2.5.如何比较两个 DOM 树的差异？

   ```
    两个树的完全 diff 算法的时间复杂度为 O(n^3) ，但是在前端中，我们很少会跨层级的移动元素，所以我们只需要比较同一层级的元素进行比较，这样就可以将算法的时间复杂度降低为 O(n)。

    算法首先会对新旧两棵树进行一个深度优先的遍历，这样每个节点都会有一个序号。在深度遍历的时候，每遍历到一个节点，我们就将这个节点和新的树中的节点进行比较，如果有差异，则将这个差异记录到一个对象中。

    在对列表元素进行对比的时候，由于 TagName 是重复的，所以我们不能使用这个来对比。我们需要给每一个子节点加上一个 key，列表对比的时候使用 key 来进行比较，这样我们才能够复用老的 DOM 树上的节点。
   ```

### 2.6. vue 双向数据绑定原理？

   ```
    vue 通过使用双向数据绑定，来实现了 View 和 Model 的同步更新。vue 的双向数据绑定主要是通过使用数据劫持和发布订阅者模式来实现的。

    首先我们通过 Object.defineProperty() 方法来对 Model 数据各个属性添加访问器属性，以此来实现数据的劫持，因此当 Model 中的数据发生变化的时候，我们可以通过配置的 setter 和 getter 方法来实现对 View 层数据更新的通知。

    数据在 html 模板中一共有两种绑定情况，一种是使用 v-model 来对 value 值进行绑定，一种是作为文本绑定，在对模板引擎进行解析的过程中。

    如果遇到元素节点，并且属性值包含 v-model 的话，我们就从 Model 中去获取 v-model 所对应的属性的值，并赋值给元素的value 值。然后给这个元素设置一个监听事件，当 View 中元素的数据发生变化的时候触发该事件，通知 Model 中的对应的属性的值进行更新。

    如果遇到了绑定的文本节点，我们使用 Model 中对应的属性的值来替换这个文本。对于文本节点的更新，我们使用了发布订阅者模式，属性作为一个主题，我们为这个节点设置一个订阅者对象，将这个订阅者对象加入这个属性主题的订阅者列表中。当 Model 层数据发生改变的时候，Model 作为发布者向主题发出通知，主题收到通知再向它的所有订阅者推送，订阅者收到通知后更改自己的数据。

   ```

   详细资料可以参考：
   [《Vue.js 双向绑定的实现原理》](http://www.cnblogs.com/kidney/p/6052935.html?utm_source=gold_browser_extension)

### 2.7. Object.defineProperty 介绍？

   ```
    Object.defineProperty 函数一共有三个参数，第一个参数是需要定义属性的对象，第二个参数是需要定义的属性，第三个是该属性描述符。

    一个属性的描述符有四个属性，分别是 value 属性的值，writable 属性是否可写，enumerable 属性是否可枚举，configurable 属性是否可配置修改。
   ```

   详细资料可以参考：
   [《Object.defineProperty()》](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)

### 2.8.使用 Object.defineProperty() 来进行数据劫持有什么缺点？

   ```
    有一些对属性的操作，使用这种方法无法拦截，比如说通过下标方式修改数组数据或者给对象新增属性，vue 内部通过重写函数解决了这个问题。在 Vue3.0 中已经不使用这种方式了，而是通过使用 Proxy 对对象进行代理，从而实现数据劫持。使用 Proxy 的好处是它可以完美的监听到任何方式的数据改变，唯一的缺点是兼容性的问题，因为这是 ES6 的语法。
   ```

### 2.9.Vue 的生命周期是什么？

   ```
    Vue 的生命周期指的是组件从创建到销毁的一系列的过程，被称为 Vue 的生命周期。通过提供的 Vue 在生命周期各个阶段的钩子函数，我们可以很好的在 Vue 的各个生命阶段实现一些操作。
   ```

### 2.10.Vue 的各个生命阶段是什么？

   ```
    Vue 一共有8个生命阶段，分别是创建前、创建后、加载前、加载后、更新前、更新后、销毁前和销毁后，每个阶段对应了一个生命周期的钩子函数。

    （1）beforeCreate 钩子函数，在实例初始化之后，在数据监听和事件配置之前触发。因此在这个事件中我们是获取不到 data 数据的。

    （2）created 钩子函数，在实例创建完成后触发，此时可以访问 data、methods 等属性。但这个时候组件还没有被挂载到页面中去，所以这个时候访问不到 $el 属性。一般我们可以在这个函数中进行一些页面初始化的工作，比如通过 ajax 请求数据来对页面进行初始化。

    （3）beforeMount 钩子函数，在组件被挂载到页面之前触发。在 beforeMount 之前，会找到对应的 template，并编译成 render 函数。

    （4）mounted 钩子函数，在组件挂载到页面之后触发。此时可以通过 DOM API 获取到页面中的 DOM 元素。

    （5）beforeUpdate 钩子函数，在响应式数据更新时触发，发生在虚拟 DOM 重新渲染和打补丁之前，这个时候我们可以对可能会被移除的元素做一些操作，比如移除事件监听器。

    （6）updated 钩子函数，虚拟 DOM 重新渲染和打补丁之后调用。

    （7）beforeDestroy 钩子函数，在实例销毁之前调用。一般在这一步我们可以销毁定时器、解绑全局事件等。

    （8）destroyed 钩子函数，在实例销毁之后调用，调用后，Vue 实例中的所有东西都会解除绑定，所有的事件监听器会被移除，所有的子实例也会被销毁。

    当我们使用 keep-alive 的时候，还有两个钩子函数，分别是 activated 和 deactivated 。用 keep-alive 包裹的组件在切换时不会进行销毁，而是缓存到内存中并执行 deactivated 钩子函数，命中缓存渲染后会执行 actived 钩子函数。
   ```

   详细资料可以参考：
   [《vue 生命周期深入》](https://juejin.im/entry/5aee8fbb518825671952308c)
   [《Vue 实例》](https://cn.vuejs.org/v2/guide/instance.html)

### 2.11. Vue 组件间的参数传递方式？

   ```
    （1）父子组件间通信

        第一种方法是子组件通过 props 属性来接受父组件的数据，然后父组件在子组件上注册监听事件，子组件通过 emit 触发事件来向父组件发送数据。

        第二种是通过 ref 属性给子组件设置一个名字。父组件通过 $refs 组件名来获得子组件，子组件通过 $parent 获得父组件，这样也可以实现通信。

        第三种是使用 provider/inject，在父组件中通过 provider 提供变量，在子组件中通过 inject 来将变量注入到组件中。不论子组件有多深，只要调用了 inject 那么就可以注入 provider 中的数据。

    （2）兄弟组件间通信
        
        第一种是使用 eventBus 的方法，它的本质是通过创建一个空的 Vue 实例来作为消息传递的对象，通信的组件引入这个实例，通信的组件通过在这个实例上监听和触发事件，来实现消息的传递。

        第二种是通过 $parent.$refs 来获取到兄弟组件，也可以进行通信。

    （3）任意组件之间

        使用 eventBus ，其实就是创建一个事件中心，相当于中转站，可以用它来传递事件和接收事件。


    如果业务逻辑复杂，很多组件之间需要同时处理一些公共的数据，这个时候采用上面这一些方法可能不利于项目的维护。这个时候可以使用 vuex ，vuex 的思想就是将这一些公共的数据抽离出来，将它作为一个全局的变量来管理，然后其他组件就可以对这个公共数据进行读写操作，这样达到了解耦的目的。
   ```

   详细资料可以参考：
   [《VUE 组件之间数据传递全集》](https://juejin.im/entry/5ba215ac5188255c6d0d8345)

### 2.12.computed 和 watch 的差异？

   ```
    （1）computed 是计算一个新的属性，并将该属性挂载到 Vue 实例上，而 watch 是监听已经存在且已挂载到 Vue 实例上的数据，所以用 watch 同样可以监听 computed 计算属性的变化。

    （2）computed 本质是一个惰性求值的观察者，具有缓存性，只有当依赖变化后，第一次访问 computed 属性，才会计算新的值。而 watch 则是当数据发生变化便会调用执行函数。

    （3）从使用场景上说，computed 适用一个数据被多个数据影响，而 watch 适用一个数据影响多个数据。
   ```

   详细资料可以参考：
   [《做面试的不倒翁：浅谈 Vue 中 computed 实现原理》](https://juejin.im/post/5b98c4da6fb9a05d353c5fd7)
   [《深入理解 Vue 的 watch 实现原理及其实现方式》](https://juejin.im/post/5af908ea5188254265399009)

[计算属性和侦听器](https://cn.vuejs.org/v2/guide/computed.html)

### 2.13. vue-router 中的导航钩子函数

   ```
    （1）全局的钩子函数 beforeEach 和 afterEach

        beforeEach 有三个参数，to 代表要进入的路由对象，from 代表离开的路由对象。next 是一个必须要执行的函数，如果不传参数，那就执行下一个钩子函数，如果传入 false，则终止跳转，如果传入一个路径，则导航到对应的路由，如果传入 error ，则导航终止，error 传入错误的监听函数。

    （2）单个路由独享的钩子函数 beforeEnter，它是在路由配置上直接进行定义的。

    （3）组件内的导航钩子主要有这三种：beforeRouteEnter、beforeRouteUpdate、beforeRouteLeave。它们是直接在路由组件内部直接进行定义的。
   ```

   详细资料可以参考：
   [《导航守卫》](https://router.vuejs.org/zh/guide/advanced/navigation-guards.html#%E5%85%A8%E5%B1%80%E5%89%8D%E7%BD%AE%E5%AE%88%E5%8D%AB)

### 2.14.\$route  和  \$router 的区别？

   ```
    $route 是“路由信息对象”，包括 path，params，hash，query，fullPath，matched，name 等路由信息参数。而 $router 是“路由实例”对象包括了路由的跳转方法，钩子函数等。
   ```

### 2.15. vue 常用的修饰符？

   ```
    .stop: 阻止单击事件冒泡
    .prevent: 提交事件不再重载页面
    .capture: 即内部元素触发的事件先在此处理，然后才交由内部元素进行处理
    .self: 只当事件是从事件绑定的元素本身触发时才触发回调
    .once: 只能用一次，绑定了事件以后只能触发一次，第二次就不会触发
    .passive: 当我们在监听元素滚动事件的时候，会一直触发onscroll事件，在pc端是没啥问题的，但是在移动端，会让							我们的网页变卡，因此我们使用这个修饰符的时候，相当于给onscroll事件整了一个.lazy修饰符
    .native: 可以理解为该修饰符的作用就是把一个vue组件转化为一个普通的HTML标签，
						 注意：使用.native修饰符来操作普通HTML标签是会令事件失效的
   ```

### 2.16. vue中 key 值的作用？

   ```
    vue 中 key 值的作用可以分为两种情况来考虑。

    第一种情况是 v-if 中使用 key。由于 Vue 会尽可能高效地渲染元素，通常会复用已有元素而不是从头开始渲染。因此当我们使用 v-if 来实现元素切换的时候，如果切换前后含有相同类型的元素，那么这个元素就会被复用。如果是相同的 input 元素，那么切换前后用户的输入不会被清除掉，这样是不符合需求的。因此我们可以通过使用 key 来唯一的标识一个元素，这个情况下，使用 key 的元素不会被复用。这个时候 key 的作用是用来标识一个独立的元素。

    第二种情况是 v-for 中使用 key。用 v-for 更新已渲染过的元素列表时，它默认使用“就地复用”的策略。如果数据项的顺序发生了改变，Vue 不会移动 DOM 元素来匹配数据项的顺序，而是简单复用此处的每个元素。因此通过为每个列表项提供一个 key 值，来以便 Vue 跟踪元素的身份，从而高效的实现复用。这个时候 key 的作用是为了高效的更新渲染虚拟 DOM。
   ```

   详细资料可以参考：
   [《Vue 面试中，经常会被问到的面试题 Vue 知识点整理》](https://segmentfault.com/a/1190000016344599)
   [《Vue2.0 v-for 中 :key 到底有什么用？》](https://www.zhihu.com/question/61064119)
   [《vue 中 key 的作用》](https://www.cnblogs.com/RainyBear/p/8563101.html)

### 2.17. keep-alive 组件有什么作用？

   ```
    如果你需要在组件切换的时候，保存一些组件的状态防止多次渲染，就可以使用 keep-alive 组件包裹需要保存的组件。
   ```

### 2.18.vue 中 mixin 和 mixins 区别？

   ```
     mixin 用于全局混入，会影响到每个组件实例。

     mixins 应该是我们最常使用的扩展组件的方式了。如果多个组件中有相同的业务逻辑，就可以将这些逻辑剥离出来，通过 mixins 混入代码，比如上拉下拉加载数据这种逻辑等等。另外需要注意的是 mixins 混入的钩子函数会先于组件内的钩子函数执行，并且在遇到同名选项的时候也会有选择性的进行合并，
   ```

   详细资料可以参考：
   [《前端面试之道》](https://juejin.im/book/5bdc715fe51d454e755f75ef/section/5bdc731b51882516c56ced6f)
   [《混入》](https://cn.vuejs.org/v2/guide/mixins.html)
     

## 3.cookie

### 3.1.简单谈一下cookie

```
    我的理解是 cookie 是服务器提供的一种用于维护会话状态信息的数据，通过服务器发送到浏览器，浏览器保存在本地，当下一次有同源的请求时，将保存的 cookie 值添加到请求头部，发送给服务端。这可以用来实现记录用户登录状态等功能。cookie 一般可以存储 4k 大小的数据，并且只能够被同源的网页所共享访问。

    服务器端可以使用 Set-Cookie 的响应头部来配置 cookie 信息。一条 cookie 包括了5个属性值 expires、domain、path、secure、HttpOnly。其中 expires 指定了 cookie 失效的时间，domain 是域名、path是路径，domain 和 path 一起限制了 cookie 能够被哪些 url 访问。secure 规定了 cookie 只能在确保安全的情况下传输，HttpOnly 规定了这个 cookie 只能被服务器访问，不能使用 js 脚本访问。

    在发生 xhr 的跨域请求的时候，即使是同源下的 cookie，也不会被自动添加到请求头部，除非显示地规定。
```

[《HTTP cookies》 ](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Cookies)
[《聊一聊 cookie》 ](https://segmentfault.com/a/1190000004556040)

### 3.2.客户端和服务器怎么维持登录态

HTTP协议与状态保持

```
HTTP协议本身是无状态的，这与HTTP协议本来的目的是相符的，客户端只需要简单的向服务器请求下载某些文件，无论是客户端还是服务器都没有必要纪录彼此过去的行为，每一次请求之间都是独立的，好比一个顾客和一个自动售货机或者一个普通的（非会员制）大卖场之间的关系一样。
```

举例

```
让我们用几个例子来描述一下cookie和session机制之间的区别与联系。笔者曾经常去的一家咖啡店有喝5杯咖啡免费赠一杯咖啡的优惠，然而一次性消费5杯咖啡的机会微乎其微，这时就需要某种方式来纪录某位顾客的消费数量。想象一下其实也无外乎下面的几种方案：
1、该店的店员很厉害，能记住每位顾客的消费数量，只要顾客一走进咖啡店，店员就知道该怎么对待了。这种做法就是协议本身支持状态。
2、发给顾客一张卡片，上面记录着消费的数量，一般还有个有效期限。每次消费时，如果顾客出示这张卡片，则此次消费就会与以前或以后的消费相联系起来。这种做法就是在客户端保持状态。
3、发给顾客一张会员卡，除了卡号之外什么信息也不纪录，每次消费时，如果顾客出示该卡片，则店员在店里的纪录本上找到这个卡号对应的纪录添加一些消费信息。这种做法就是在服务器端保持状态。
```

```
由于HTTP协议是无状态的，而出于种种考虑也不希望使之成为有状态的，因此，后面两种方案就成为现实的选择。具体来说cookie机制采用的是在客户端保持状态的方案
session机制采用的是在服务器端保持状态的方案。
同时我们也看到，由于采用服务器端保持状态的方案在客户端也需要保存一个标识，所以session机制可能需要借助于cookie机制来达到保存标识的目的
token,登录成功获取token，以后每个请求都要带上token
```

  3.3.cookie是每次请求都会发送给服务器吗

```
不会，因为服务器设置cookie时会指定其domain和path，只有请求相应的域时才会带上cookie
```

## 4.同源策略&跨域

### 4.1.同源策略

    我对浏览器的同源政策的理解是，一个域下的 js 脚本在未经允许的情况下，不能够访问另一个域的内容。这里的同源指的是两个域的协议、域名、端口号必须相同，否则则不属于同一个域。
    
    同源政策主要限制了三个方面
    
    第一个是当前域下的 js 脚本不能够访问其他域下的 cookie、localStorage 和 indexDB。
    
    第二个是当前域下的 js 脚本不能够操作访问其他域下的 DOM。
    
    第三个是当前域下 ajax 无法发送跨域请求。
    
    同源政策的目的主要是为了保证用户的信息安全，它只是对 js 脚本的一种限制，并不是对浏览器的限制，对于一般的 img、或者script 脚本请求都不会有跨域的限制，这是因为这些操作都不会通过响应结果来进行可能出现安全问题的操作。
[《浏览器同源政策及其规避方法》](http://www.ruanyifeng.com/blog/2016/04/same-origin-policy.html)

### 4.2.跨域的方法

 相关知识点：

   ```
    （1） 通过 jsonp 跨域
    （2） document.domain + iframe 跨域
    （3） location.hash + iframe
    （4） window.name + iframe 跨域
    （5） postMessage 跨域
    （6） 跨域资源共享（CORS）
    （7） nginx 代理跨域
    （8） nodejs 中间件代理跨域
    （9） WebSocket 协议跨域
   ```

   回答：

   ```
    解决跨域的方法我们可以根据我们想要实现的目的来划分。

    首先我们如果只是想要实现主域名下的不同子域名的跨域操作，我们可以使用设置 document.domain 来解决。

    （1）将 document.domain 设置为主域名，来实现相同子域名的跨域操作，这个时候主域名下的 cookie 就能够被子域名所访问。同时如果文档中含有主域名相同，子域名不同的 iframe 的话，我们也可以对这个 iframe 进行操作。

    如果是想要解决不同跨域窗口间的通信问题，比如说一个页面想要和页面的中的不同源的 iframe 进行通信的问题，我们可以使用 location.hash 或者 window.name 或者 postMessage 来解决。

    （2）使用 location.hash 的方法，我们可以在主页面动态的修改 iframe 窗口的 hash 值，然后在 iframe 窗口里实现监听函数来实现这样一个单向的通信。因为在 iframe 是没有办法访问到不同源的父级窗口的，所以我们不能直接修改父级窗口的 hash 值来实现通信，我们可以在 iframe 中再加入一个 iframe ，这个 iframe 的内容是和父级页面同源的，所以我们可以 window.parent.parent 来修改最顶级页面的 src，以此来实现双向通信。

    （3）使用 window.name 的方法，主要是基于同一个窗口中设置了 window.name 后不同源的页面也可以访问，所以不同源的子页面可以首先在 window.name 中写入数据，然后跳转到一个和父级同源的页面。这个时候级页面就可以访问同源的子页面中 window.name 中的数据了，这种方式的好处是可以传输的数据量大。

    （4）使用 postMessage 来解决的方法，这是一个 h5 中新增的一个 api。通过它我们可以实现多窗口间的信息传递，通过获取到指定窗口的引用，然后调用 postMessage 来发送信息，在窗口中我们通过对 message 信息的监听来接收信息，以此来实现不同源间的信息交换。
    如果是像解决 ajax 无法提交跨域请求的问题，我们可以使用 jsonp、cors、websocket 协议、服务器代理来解决问题。
        
    （5）使用 jsonp 来实现跨域请求，它的主要原理是通过动态构建 script  标签来实现跨域请求，因为浏览器对 script 标签的引入没有跨域的访问限制 。通过在请求的 url 后指定一个回调函数，然后服务器在返回数据的时候，构建一个 json 数据的包装，这个包装就是回调函数，然后返回给前端，前端接收到数据后，因为请求的是脚本文件，所以会直接执行，这样我们先前定义好的回调函数就可以被调用，从而实现了跨域请求的处理。这种方式只能用于 get 请求。

    （6）使用 CORS 的方式，CORS 是一个 W3C 标准，全称是"跨域资源共享"。CORS 需要浏览器和服务器同时支持。目前，所有浏览器都支持该功能，因此我们只需要在服务器端配置就行。浏览器将 CORS 请求分成两类：简单请求和非简单请求。对于简单请求，浏览器直接发出 CORS 请求。具体来说，就是会在头信息之中，增加一个 Origin 字段。Origin 字段用来说明本次请求来自哪个源。服务器根据这个值，决定是否同意这次请求。对于如果 Origin 指定的源，不在许可范围内，服务器会返回一个正常的 HTTP 回应。浏览器发现，这个回应的头信息没有包含 Access-Control-Allow-Origin 字段，就知道出错了，从而抛出一个错误，ajax 不会收到响应信息。如果成功的话会包含一些以 Access-Control- 开头的字段。
非简单请求，浏览器会先发出一次预检请求，来判断该域名是否在服务器的白名单中，如果收到肯定回复后才会发起请求。

    （7）使用 websocket 协议，这个协议没有同源限制。

    （8）使用服务器来代理跨域的访问请求，就是有跨域的请求操作时发送请求给后端，让后端代为请求，然后最后将获取的结果发返回。
   ```

   详细资料可以参考：
   [《前端常见跨域解决方案（全）》](https://segmentfault.com/a/1190000011145364)
   [《浏览器同源政策及其规避方法》](http://www.ruanyifeng.com/blog/2016/04/same-origin-policy.html)
   [《跨域，你需要知道的全在这里》](https://juejin.im/entry/59feae9df265da43094488f6)
   [《为什么 form 表单提交没有跨域问题，但 ajax 提交有跨域问题？》](https://www.zhihu.com/question/31592553)

### 4.3.你说的跨域资源共享(CORS)是怎么用的呢

[跨域资源共享 CORS 详解](http://www.ruanyifeng.com/blog/2016/04/cors.html)

## 5.作用域(链)&闭包

### 5.1.作用域

```
在JS中，变量的作用域有两种：全局作用域和函数作用域。ES6中有了块级作用域的概念，使用let、const定义变量即可。
1）全局作用域:
	1.没有用var声明的变量（除去函数的参数）都具有全局作用域，成为全局变量；
	2.window的所有属性都具有全局作用域；
	3.最外层函数体外声明的变量也具有全局作用域
  可以被任何函数访问
  容易引起冲突
  
2）函数作用域：在函数作用域中定义的变量，在函数外部是无法访问的

3）ES6中的块级作用域：
	任何一个对花括号（{}）中的语句都属于一个块，在这之中定义的所有变量在代码块外都是不可见的，称之为块级作用域。
	let所声明的变量，只在使用let所在的代码块内有效，在代码块外调用let声明的变量则会报错
	说明：
	(1)不存在变量提升，使用let命令声明的变量，只能在声明后使用，语法上称为“暂时性死区”
	(2)使用const声明的变量不能再次赋值。
```

### 5.2.变量提升&函数提升

```
1)使用var声明的变量存在变量提升，即可在声明变量之前使用变量。使用let、const声明的变量不能在声明变量之前使用变量。
2)函数提升优先于变量提升，函数提升会把整个函数挪到作用域顶部，变量提升会把变量声明挪到作用域顶部。
```

### 5.3.作用域链

```
要得到一个变量的值，若当前作用域没有定义，就到父级作用域寻找。如果父级作用域中也没找到，就再向上一层去寻找，直到找到全局作用域。这种一层一层的关系，就是作用域链
```

### 5.4闭包

```
是指有权访问另一个函数作用域中的变量的函数
1）优点：
 可以避免全局变量的污染
2）缺点
	参数和变量不会被垃圾回收机制回收，闭包会常驻内存，增大内存使用率，使用不当容易造成内存泄漏
```

### 5.5.垃圾回收机制

    v8 的垃圾回收机制基于分代回收机制，这个机制又基于世代假说，这个假说有两个特点，一是新生的对象容易早死，另一个是不死的对象会活得更久。基于这个假说，v8 引擎将内存分为了新生代和老生代。
    
    新创建的对象或者只经历过一次的垃圾回收的对象被称为新生代。经历过多次垃圾回收的对象被称为老生代。
    
    新生代被分为 From 和 To 两个空间，To 一般是闲置的。当 From 空间满了的时候会执行 Scavenge 算法进行垃圾回收。当我们执行垃圾回收算法的时候应用逻辑将会停止，等垃圾回收结束后再继续执行。这个算法分为三步：
    
    （1）首先检查 From 空间的存活对象，如果对象存活则判断对象是否满足晋升到老生代的条件，如果满足条件则晋升到老生代。如果不满足条件则移动 To 空间。
    
    （2）如果对象不存活，则释放对象的空间。
    
    （3）最后将 From 空间和 To 空间角色进行交换。
    
    新生代对象晋升到老生代有两个条件：
    
    （1）第一个是判断是对象否已经经过一次 Scavenge 回收。若经历过，则将对象从 From 空间复制到老生代中；若没有经历，则复制到 To 空间。
    
    （2）第二个是 To 空间的内存使用占比是否超过限制。当对象从 From 空间复制到 To 空间时，若 To 空间使用超过 25%，则对象直接晋升到老生代中。设置 25% 的原因主要是因为算法结束后，两个空间结束后会交换位置，如果 To 空间的内存太小，会影响后续的内存分配。
    
    老生代采用了标记清除法和标记压缩法。标记清除法首先会对内存中存活的对象进行标记，标记结束后清除掉那些没有标记的对象。由于标记清除后会造成很多的内存碎片，不便于后面的内存分配。所以了解决内存碎片的问题引入了标记压缩法。
    
    由于在进行垃圾回收的时候会暂停应用的逻辑，对于新生代方法由于内存小，每次停顿的时间不会太长，但对于老生代来说每次垃圾回收的时间长，停顿会造成很大的影响。 为了解决这个问题 V8 引入了增量标记的方法，将一次停顿进行的过程分为了多步，每次执行完一小步就让运行逻辑执行一会，就这样交替运行。
   [《深入理解 V8 的垃圾回收原理》](https://www.jianshu.com/p/b8ed21e8a4fb)
   [《JavaScript 中的垃圾回收》](https://zhuanlan.zhihu.com/p/23992332)

## 6.事件机制

### 6.1.写一个通用的事件侦听器函数

```javascript
    const EventUtils = {
      // 视能力分别使用dom0||dom2||IE方式 来绑定事件
      // 添加事件
      addEvent: function (element, type, handler) {
        if (element.addEventListener) {
          element.addEventListener(type, handler, false);
        } else if (element.attachEvent) {
          element.attachEvent("on" + type, handler);
        } else {
          element["on" + type] = handler;
        }

      },
      // 移除事件
      removeEvent: function (element, type, handler) {
        if (element.removeEventListener) {
          element.removeEventListener(type, handler, false);
        } else if (element.detachEvent) {
          element.detachEvent("on" + type, handler);
        } else {
          element["on" + type] = null;
        }
      },
      // 获取事件目标
      getTarget: function (event) {
        return event.target || event.srcElement;
      },

      // 获取 event 对象的引用，取到事件的所有信息，确保随时能使用 event
      getEvent: function (event) {
        return event || window.event;
      },

      // 阻止事件（主要是事件冒泡，因为 IE 不支持事件捕获）
      stopPropagation: function (event) {
        if (event.stopPropagation) {
          event.stopPropagation();
        } else {
          event.cancelBubble = true;
        }
      },

      // 取消事件的默认行为
      preventDefault: function (event) {
        if (event.preventDefault) {
          event.preventDefault();
        } else {
          event.returnValue = false;
        }
      }
    }
```

   [《JS 事件模型》](https://segmentfault.com/a/1190000006934031#articleHeader6)

### 6.2. 事件是什么？IE 与火狐的事件机制有什么区别？ 如何阻止冒泡

   ```
    （1）事件是用户操作网页时发生的交互动作，比如 click/move， 事件除了用户触发的动作外，还可以是文档加载，窗口滚动和大小调整。事件被封装成一个 event 对象，包含了该事件发生时的所有相关信息（ event 的属性）以及可以对事件进行的操作（ event 的方法）。
    
    （2）事件处理机制：IE 支持事件冒泡、Firefox 同时支持两种事件模型，也就是：事件冒泡和事件捕获。

    （3）event.stopPropagation() 或者 ie 下的方法 event.cancelBubble = true;
   ```

   详细资料可以参考：
   [《Javascript 事件模型系列（一）事件及事件的三种模型》](https://www.cnblogs.com/lvdabao/p/3265870.html)
   [《Javascript 事件模型：事件捕获和事件冒泡》](https://blog.csdn.net/wuseyukui/article/details/13771493)

### 6.3. 三种事件模型是什么？

   ```
    事件是用户操作网页时发生的交互动作或者网页本身的一些操作，现代浏览器一共有三种事件模型。

    第一种事件模型是最早的 DOM0 级模型，这种模型不会传播，所以没有事件流的概念，但是现在有的浏览器支持以冒泡的方式实现，它可以在网页中直接定义监听函数，也可以通过 js 属性来指定监听函数。这种方式是所有浏览器都兼容的。

    第二种事件模型是 IE 事件模型，在该事件模型中，一次事件共有两个过程，事件处理阶段，和事件冒泡阶段。事件处理阶段会首先执行目标元素绑定的监听事件。然后是事件冒泡阶段，冒泡指的是事件从目标元素冒泡到 document，依次检查经过的节点是否绑定了事件监听函数，如果有则执行。这种模型通过 attachEvent 来添加监听函数，可以添加多个监听函数，会按顺序依次执行。

    第三种是 DOM2 级事件模型，在该事件模型中，一次事件共有三个过程，第一个过程是事件捕获阶段。捕获指的是事件从 document 一直向下传播到目标元素，依次检查经过的节点是否绑定了事件监听函数，如果有则执行。后面两个阶段和 IE 事件模型的两个阶段相同。这种事件模型，事件绑定的函数是 addEventListener，其中第三个参数可以指定事件是否在捕获阶段执行。
   ```

   详细资料可以参考：
   [《一个 DOM 元素绑定多个事件时，先执行冒泡还是捕获》](https://blog.csdn.net/u013217071/article/details/77613706)

 ### 6.4.事件委托是什么

   ```
    事件委托本质上是利用了浏览器事件冒泡的机制。因为事件在冒泡过程中会上传到父节点，并且父节点可以通过事件对象获取到目标节点，因此可以把子节点的监听函数定义在父节点上，由父节点的监听函数统一处理多个子元素的事件，这种方式称为事件代理。

    使用事件代理我们可以不必要为每一个子元素都绑定一个监听事件，这样减少了内存上的消耗。并且使用事件代理我们还可以实现事件的动态绑定，比如说新增了一个子节点，我们并不需要单独地为它添加一个监听事件，它所发生的事件会交给父元素中的监听函数来处理。
   ```

   详细资料可以参考：
[《JavaScript 事件委托详解》](https://zhuanlan.zhihu.com/p/26536815)

## 7.ES6语法新特性

```
let、const命令
变量的解构赋值
模板字符串
rest参数、箭头函数
扩展运算符(...):将一个数组转为用逗号分隔的参数序列,还可以替代函数的apply方法，赋值数组，合并数组等
Symbol
set和map数据结构
Proxy
Reflect
Promise对象
Class
```

[ES6 入门教程](https://es6.ruanyifeng.com/?search=instanceof&x=0&y=0)

## 8.性能

### 8.1防抖debounce

函数防抖：当持续触发事件时，一定时间段内没有再触发事件，事件处理函数才会执行一次，如果设定的时间到来之前，又一次触发了事件，就重新开始延时。如下图，持续触发scroll事件时，并不执行handle函数，当1000毫秒内没有触发scroll事件时，才会延时触发scroll事件

![](D:\tec\interview\img\debounce.jpg)

```javascript
function debounce(fn, wait) {
  var timer = null;
  return function () {
      var context = this
      var args = arguments
      if (timer) {
          clearTimeout(timer);
          timer = null;
      }
      timer = setTimeout(function () {
          fn.apply(context, args)
      }, wait)
  }
}

var fn = function () {
  console.log('boom')
}

setInterval(debounce(fn,500),1000) // 第一次在1500ms后触发，之后每1000ms触发一次

setInterval(debounce(fn,2000),1000) // 不会触发一次（我把函数防抖看出技能读条，如果读条没完成就用技能，便会失败而且重新读条）
```

### 8.2.节流throttle

函数节流：当持续触发事件时，保证一定时间段内只调用一次事件处理函数。节流通俗解释就比如我们水龙头放水，阀门一打开，水哗哗的往下流，秉着勤俭节约的优良传统美德，我们要把水龙头关小点，最好是如我们心意按照一定规律在某个时间间隔内一滴一滴的往下滴。如下图，持续触发scroll事件时，并不立即执行handle函数，每隔1000毫秒才会执行一次handle函数

<img src="./img/throttle.jpg" style="zoom:80%;" />

```javascript
function throttle(fn, delay) {
  var preTime = Date.now();
  return function () {
    var context = this,
      args = arguments,
      nowTime = Date.now();
    // 如果两次时间间隔超过了指定时间，则执行函数。
    if (nowTime - preTime >= delay) {
      fn.apply(context, args);
      preTime = Date.now();
    }
  }
}
let fn = ()=>{
  console.log('boom')
}

setInterval(throttle(fn,1000),10)
```

[《轻松理解 JS 函数节流和函数防抖》](https://juejin.im/post/5a35ed25f265da431d3cc1b1)
[《JavaScript 事件节流和事件防抖》](https://juejin.im/post/5aa60b0e518825556b6c6d1a)
[《JS 的防抖与节流》](https://juejin.im/entry/5b1d2d54f265da6e2545bfa4)

### 8.3.Performance API

   ```
    Performance API 用于精确度量、控制、增强浏览器的性能表现。这个 API 为测量网站性能，提供以前没有办法做到的精度。
    
    使用 getTime 来计算脚本耗时的缺点，首先，getTime方法（以及 Date 对象的其他方法）都只能精确到毫秒级别（一秒的千分之一），想要得到更小的时间差别就无能为力了。其次，这种写法只能获取代码运行过程中的时间进度，无法知道一些后台事件的时间进度，比如浏览器用了多少时间从服务器加载网页。

    为了解决这两个不足之处，ECMAScript 5引入“高精度时间戳”这个 API，部署在 performance 对象上。它的精度可以达到1毫秒的千分之一（1秒的百万分之一）。

    navigationStart：当前浏览器窗口的前一个网页关闭，发生 unload 事件时的 Unix 毫秒时间戳。如果没有前一个网页，则等于fetchStart 属性。

    loadEventEnd：返回当前网页 load 事件的回调函数运行结束时的 Unix 毫秒时间戳。如果该事件还没有发生，返回 0。

    根据上面这些属性，可以计算出网页加载各个阶段的耗时。比如，网页加载整个过程的耗时的计算方法如下：

    var t = performance.timing; 
    var pageLoadTime = t.loadEventEnd - t.navigationStart;
   ```

   [《Performance API》](http://javascript.ruanyifeng.com/bom/performance.html)

### 8.4.图片的懒加载和预加载

   ```
    懒加载也叫延迟加载，指的是在长网页中延迟加载图片的时机，当用户需要访问时，再去加载，这样可以提高网站的首屏加载速度，提升用户的体验，并且可以减少服务器的压力。它适用于图片很多，页面很长的电商网站的场景。懒加载的实现原理是，将页面上的图片的 src 属性设置为空字符串，将图片的真实路径保存在一个自定义属性中，当页面滚动的时候，进行判断，如果图片进入页面可视区域内，则从自定义属性中取出真实路径赋值给图片的 src 属性，以此来实现图片的延迟加载。

    预加载指的是将所需的资源提前请求加载到本地，这样后面在需要用到时就直接从缓存取资源。通过预加载能够减少用户的等待时间，提高用户的体验。我了解的预加载的最常用的方式是使用 js 中的 image 对象，通过为 image 对象来设置 scr 属性，来实现图片的预加载。

    这两种方式都是提高网页性能的方式，两者主要区别是一个是提前加载，一个是迟缓甚至不加载。懒加载对服务器前端有一定的缓解压力作用，预加载则会增加服务器前端压力。
   ```

   详细资料可以参考：
   [《懒加载和预加载》](https://juejin.im/post/5b0c3b53f265da09253cbed0)
   [《网页图片加载优化方案》](https://juejin.im/entry/5a73f38cf265da4e99575be3)
   [《基于用户行为的图片等资源预加载》](https://www.zhangxinxu.com/wordpress/2016/06/image-preload-based-on-user-behavior/)

### 8.5.如何进行网站性能优化

[雅虎 Best Practices for Speeding Up Your Web Site](https://developer.yahoo.com/performance/rules.html)

- content 方面

  1. 减少 HTTP 请求：合并文件、CSS 精灵、inline Image
  2. 减少 DNS 查询：DNS 查询完成之前浏览器不能从这个主机下载任何任何文件。方法：DNS 缓存、将资源分布到恰当数量的主机名，平衡并行下载和 DNS 查询
  3. 避免重定向：多余的中间访问
  4. 使 Ajax 可缓存
  5. 非必须组件延迟加载
  6. 未来所需组件预加载
  7. 减少 DOM 元素数量
  8. 将资源放到不同的域下：浏览器同时从一个域下载资源的数目有限，增加域可以提高并行下载量
  9. 减少 iframe 数量
  10. 不要 404

- Server 方面
  1. 使用 CDN
  2. 添加 Expires 或者 Cache-Control 响应头
  3. 对组件使用 Gzip 压缩
  4. 配置 ETag
  5. Flush Buffer Early
  6. Ajax 使用 GET 进行请求
  7. 避免空 src 的 img 标签
- Cookie 方面
  1. 减小 cookie 大小
  2. 引入资源的域名不要包含 cookie
- css 方面
  1. 将样式表放到页面顶部
  2. 不使用 CSS 表达式
  3. 使用<link>不使用@import
  4. 不使用 IE 的 Filter
- Javascript 方面
  1. 将脚本放到页面底部
  2. 将 javascript 和 css 从外部引入
  3. 压缩 javascript 和 css
  4. 删除不需要的脚本
  5. 减少 DOM 访问
  6. 合理设计事件监听器
- 图片方面
  1. 优化图片：根据实际颜色需要选择色深、压缩
  2. 优化 css 精灵
  3. 不要在 HTML 中拉伸图片
  4. 保证 favicon.ico 小并且可缓存
- 移动方面
  1. 保证组件小于 25k
  2. Pack Components into a Multipart Document

## 9.模块

### 9.1.模块化开发怎么做

    我对模块的理解是，一个模块是实现一个特定功能的一组方法。在最开始的时候，js 只实现一些简单的功能，所以并没有模块的概念，但随着程序越来越复杂，代码的模块化开发变得越来越重要。
    
    由于函数具有独立作用域的特点，最原始的写法是使用函数来作为模块，几个函数作为一个模块，但是这种方式容易造成全局变量的污染，并且模块间没有联系。
    
    后面提出了对象写法，通过将函数作为一个对象的方法来实现，这样解决了直接使用函数作为模块的一些缺点，但是这种办法会暴露所有的模块成员，外部代码可以修改内部属性的值。
    
    现在最常用的是立即执行函数的写法，通过利用闭包来实现模块私有作用域的建立，同时不会对全局作用域造成污染。
   [《浅谈模块化开发》](https://juejin.im/post/5ab378c46fb9a028ce7b824f)
   [《Javascript 模块化编程（一）：模块的写法》](http://www.ruanyifeng.com/blog/2012/10/javascript_module.html)
   [《前端模块化：CommonJS，AMD，CMD，ES6》](https://juejin.im/post/5aaa37c8f265da23945f365c)
   [《Module 的语法》](http://es6.ruanyifeng.com/#docs/module)

### 9.2.js.的几种模块规范

    js 中现在比较成熟的有四种模块加载方案。
    
    第一种是 CommonJS 方案，它通过 require 来引入模块，通过 module.exports 定义模块的输出接口。这种模块加载方案是服务器端的解决方案，它是以同步的方式来引入模块的，因为在服务端文件都存储在本地磁盘，所以读取非常快，所以以同步的方式加载没有问题。但如果是在浏览器端，由于模块的加载是使用网络请求，因此使用异步加载的方式更加合适。
    
    第二种是 AMD 方案，这种方案采用异步加载的方式来加载模块，模块的加载不影响后面语句的执行，所有依赖这个模块的语句都定义在一个回调函数里，等到加载完成后再执行回调函数。require.js 实现了 AMD 规范。
    
    第三种是 CMD 方案，这种方案和 AMD 方案都是为了解决异步模块加载的问题，sea.js 实现了 CMD 规范。它和 require.js的区别在于模块定义时对依赖的处理不同和对依赖模块的执行时机的处理不同。参考9.3
    
    第四种方案是 ES6 提出的方案，使用 import 和 export 的形式来导入导出模块。这种方案和上面三种方案都不同。参考 9.4。
### 9.3.AMD 和 CMD 规范的区别

    它们之间的主要区别有两个方面。
    
    （1）第一个方面是在模块定义时对依赖的处理不同。AMD 推崇依赖前置，在定义模块的时候就要声明其依赖的模块。而 CMD 推崇就近依赖，只有在用到某个模块的时候再去 require。
    
    （2）第二个方面是对依赖模块的执行时机处理不同。首先 AMD 和 CMD 对于模块的加载方式都是异步加载，不过它们的区别在于模块的执行时机，AMD 在依赖模块加载完成后就直接执行依赖模块，依赖模块的执行顺序和我们书写的顺序不一定一致。而 CMD在依赖模块加载完成后并不执行，只是下载而已，等到所有的依赖模块都加载好后，进入回调函数逻辑，遇到 require 语句的时候才执行对应的模块，这样模块的执行顺序就和我们书写的顺序保持一致了。
    
    // CMD
    define(function(require, exports, module) {
     var a = require('./a')
     a.doSomething()
     // 此处略去 100 行
     var b = require('./b') // 依赖可以就近书写
     b.doSomething()
     // ...
    })
    
    // AMD 默认推荐
    define(['./a', './b'], function(a, b) { // 依赖必须一开始就写好
     a.doSomething()
     // 此处略去 100 行
     b.doSomething()
     // ...
    })
[《前端模块化，AMD 与 CMD 的区别》](https://juejin.im/post/5a422b036fb9a045211ef789)

### 9.4.ES6 模块与 CommonJS 模块、AMD、CMD 的差异

    （1）CommonJS 模块输出的是一个值的拷贝，ES6 模块输出的是值的引用。CommonJS 模块输出的是值的拷贝，也就是说，一旦输出一个值，模块内部的变化就影响不到这个值。ES6 模块的运行机制与 CommonJS 不一样。JS 引擎对脚本静态分析的时候，遇到模块加载命令 import，就会生成一个只读引用。等到脚本真正执行时，再根据这个只读引用，到被加载的那个模块里面去取值。
    
    （2）CommonJS 模块是运行时加载，ES6 模块是编译时输出接口。CommonJS 模块就是对象，即在输入时是先加载整个模块，生成一个对象，然后再从这个对象上面读取方法，这种加载称为“运行时加载”。而 ES6 模块不是对象，它的对外接口只是一种静态定义，在代码静态解析阶段就会生成。
### 9.5.requireJS 的核心原理是什么？（如何动态加载的？如何避免多次加载的？如何 缓存的？）


    require.js 的核心原理是通过动态创建 script 脚本来异步引入模块，然后对每个脚本的 load 事件进行监听，如果每个脚本都加载完成了，再调用回调函数
[《requireJS 的用法和原理分析》](https://github.com/HRFE/blog/issues/10)
[《requireJS 的核心原理是什么？》](https://zhuanlan.zhihu.com/p/55039478)
[《从 RequireJs 源码剖析脚本加载原理》](https://www.cnblogs.com/dong-xu/p/7160919.html)
[《requireJS 原理分析》](https://www.jianshu.com/p/5a39535909e4)

### 9.6.JS 模块加载器的轮子怎么造，也就是如何实现一个模块加载器

 [《JS 模块加载器加载原理是怎么样的？》](https://www.zhihu.com/question/21157540)

### 9.7谈谈你对 webpack 的看法

   ```
    我当时使用 webpack 的一个最主要原因是为了简化页面依赖的管理，并且通过将其打包为一个文件来降低页面加载时请求的资源数。

    我认为 webpack 的主要原理是，它将所有的资源都看成是一个模块，并且把页面逻辑当作一个整体，通过一个给定的入口文件，webpack 从这个文件开始，找到所有的依赖文件，将各个依赖文件模块通过 loader 和 plugins 处理后，然后打包在一起，最后输出一个浏览器可识别的 JS 文件。

    Webpack 具有四个核心的概念，分别是 Entry（入口）、Output（输出）、loader 和 Plugins（插件）。

    Entry 是 webpack 的入口起点，它指示 webpack 应该从哪个模块开始着手，来作为其构建内部依赖图的开始。

    Output 属性告诉 webpack 在哪里输出它所创建的打包文件，也可指定打包文件的名称，默认位置为 ./dist。

    loader 可以理解为 webpack 的编译器，它使得 webpack 可以处理一些非 JavaScript 文件。在对 loader 进行配置的时候，test 属性，标志有哪些后缀的文件应该被处理，是一个正则表达式。use 属性，指定 test 类型的文件应该使用哪个 loader 进行预处理。常用的 loader 有 css-loader、style-loader 等。

    插件可以用于执行范围更广的任务，包括打包、优化、压缩、搭建服务器等等，要使用一个插件，一般是先使用 npm 包管理器进行安装，然后在配置文件中引入，最后将其实例化后传递给 plugins 数组属性。

    使用 webpack 的确能够提供我们对于项目的管理，但是它的缺点就是调试和配置起来太麻烦了。但现在 webpack4.0 的免配置一定程度上解决了这个问题。但是我感觉就是对我来说，就是一个黑盒，很多时候出现了问题，没有办法很好的定位。
   ```

   详细资料可以参考：
   [《不聊 webpack 配置，来说说它的原理》](https://juejin.im/post/5b38d27451882574d87aa5d5#heading-0)
   [《前端工程化——构建工具选型：grunt、gulp、webpack》](https://juejin.im/entry/5b5724d05188251aa01647fd)
   [《浅入浅出 webpack》](https://juejin.im/post/5afa9cd0f265da0b981b9af9#heading-0)
   [《前端构建工具发展及其比较》](https://juejin.im/entry/5ae5c8c9f265da0b9f400d8e)

### 9.8.require 模块引入的查找方式

    当 Node 遇到 require(X) 时，按下面的顺序处理。
    
    （1）如果 X 是内置模块（比如 require('http')） 
    　　a. 返回该模块。 
    　　b. 不再继续执行。
    
    （2）如果 X 以 "./" 或者 "/" 或者 "../" 开头 
    　　a. 根据 X 所在的父模块，确定 X 的绝对路径。 
    　　b. 将 X 当成文件，依次查找下面文件，只要其中有一个存在，就返回该文件，不再继续执行。
        X
        X.js
        X.json
        X.node
    
    　　c. 将 X 当成目录，依次查找下面文件，只要其中有一个存在，就返回该文件，不再继续执行。
        X/package.json（main字段）
        X/index.js
        X/index.json
        X/index.node
    
    （3）如果 X 不带路径 
    　　a. 根据 X 所在的父模块，确定 X 可能的安装目录。 
    　　b. 依次在每个目录中，将 X 当成文件名或目录名加载。
    
    （4）抛出 "not found"
  [《require() 源码解读》](http://www.ruanyifeng.com/blog/2015/05/require.html)

## 10.浏览器

### 10.1.简单介绍一下 V8 引擎的垃圾回收机制

    v8 的垃圾回收机制基于分代回收机制，这个机制又基于世代假说，这个假说有两个特点，一是新生的对象容易早死，另一个是不死的对象会活得更久。基于这个假说，v8 引擎将内存分为了新生代和老生代。
    
    新创建的对象或者只经历过一次的垃圾回收的对象被称为新生代。经历过多次垃圾回收的对象被称为老生代。
    
    新生代被分为 From 和 To 两个空间，To 一般是闲置的。当 From 空间满了的时候会执行 Scavenge 算法进行垃圾回收。当我们执行垃圾回收算法的时候应用逻辑将会停止，等垃圾回收结束后再继续执行。这个算法分为三步：
    
    （1）首先检查 From 空间的存活对象，如果对象存活则判断对象是否满足晋升到老生代的条件，如果满足条件则晋升到老生代。如果不满足条件则移动 To 空间。
    
    （2）如果对象不存活，则释放对象的空间。
    
    （3）最后将 From 空间和 To 空间角色进行交换。
    
    新生代对象晋升到老生代有两个条件：
    
    （1）第一个是判断是对象否已经经过一次 Scavenge 回收。若经历过，则将对象从 From 空间复制到老生代中；若没有经历，则复制到 To 空间。
    
    （2）第二个是 To 空间的内存使用占比是否超过限制。当对象从 From 空间复制到 To 空间时，若 To 空间使用超过 25%，则对象直接晋升到老生代中。设置 25% 的原因主要是因为算法结束后，两个空间结束后会交换位置，如果 To 空间的内存太小，会影响后续的内存分配。
    
    老生代采用了标记清除法和标记压缩法。标记清除法首先会对内存中存活的对象进行标记，标记结束后清除掉那些没有标记的对象。由于标记清除后会造成很多的内存碎片，不便于后面的内存分配。所以了解决内存碎片的问题引入了标记压缩法。
    
    由于在进行垃圾回收的时候会暂停应用的逻辑，对于新生代方法由于内存小，每次停顿的时间不会太长，但对于老生代来说每次垃圾回收的时间长，停顿会造成很大的影响。 为了解决这个问题 V8 引入了增量标记的方法，将一次停顿进行的过程分为了多步，每次执行完一小步就让运行逻辑执行一会，就这样交替运行。
[《深入理解 V8 的垃圾回收原理》](https://www.jianshu.com/p/b8ed21e8a4fb)
[《JavaScript 中的垃圾回收》](https://zhuanlan.zhihu.com/p/23992332)

### 10.2.哪些操作会造成内存泄漏

```
（1）意外的全局变量
（2）被遗忘的计时器或回调函数
（3）脱离 DOM 的引用
（4）闭包
```

    第一种情况是我们由于使用未声明的变量，而意外的创建了一个全局变量，而使这个变量一直留在内存中无法被回收。
    
    第二种情况是我们设置了 setInterval 定时器，而忘记取消它，如果循环函数有对外部变量的引用的话，那么这个变量会被一直留在内存中，而无法被回收。
    
    第三种情况是我们获取一个 DOM 元素的引用，而后面这个元素被删除，由于我们一直保留了对这个元素的引用，所以它也无法被回收。
    
    第四种情况是不合理的使用闭包，从而导致某些变量一直被留在内存当中。
 [《JavaScript 内存泄漏教程》](http://www.ruanyifeng.com/blog/2017/04/memory-leak.html)
[《4类 JavaScript 内存泄漏及如何避免》](https://jinlong.github.io/2016/05/01/4-Types-of-Memory-Leaks-in-JavaScript-and-How-to-Get-Rid-Of-Them/)
[《杜绝 js 中四种内存泄漏类型的发生》](https://juejin.im/entry/5a64366c6fb9a01c9332c706)
[《javascript 典型内存泄漏及 chrome 的排查方法》](https://segmentfault.com/a/1190000008901861)

### 10.3.实现一个页面操作不会整页刷新的网站，并且能在浏览器前进、后退时正确响应。给出你的技术实现方案

    通过使用 pushState + ajax 实现浏览器无刷新前进后退，当一次 ajax 调用成功后我们将一条 state 记录加入到 history 对象中。一条 state 记录包含了 url、title 和 content 属性，在 popstate 事件中可以获取到这个 state 对象，我们可以使用 content 来传递数据。最后我们通过对 window.onpopstate 事件监听来响应浏览器的前进后退操作。
    
    使用 pushState 来实现有两个问题，一个是打开首页时没有记录，我们可以使用 replaceState 来将首页的记录替换，另一个问题是当一个页面刷新的时候，仍然会向服务器端请求数据，因此如果请求的 url 需要后端的配合将其重定向到一个页面。
   [《pushState + ajax 实现浏览器无刷新前进后退》](http://blog.chenxu.me/post/detail?id=ed4f0732-897f-48e4-9d4f-821e82f17fad)
   [《Manipulating the browser history》](https://developer.mozilla.org/zh-CN/docs/Web/API/History_API)

### 10.4.如何判断当前脚本运行在浏览器还是 node 环境中

    this === window ? 'browser' : 'node';
    
    通过判断 Global 对象是否为 window，如果不为 window，当前脚本没有运行在浏览器中。
### 10.5.把 script 标签放在页面的最底部的 body 封闭之前和封闭之后有什么区别？浏览器会如何解析它们？

[《为什么把 script 标签放在 body 结束标签之后 html 结束标签之前？》](https://www.zhihu.com/question/20027966)
[《从 Chrome 源码看浏览器如何加载资源》](https://zhuanlan.zhihu.com/p/30558018)

### 10.6什么是“前端路由”？什么时候适合使用“前端路由”？“前端路由”有哪些优点和缺点

    （1）什么是前端路由？
    
        前端路由就是把不同路由对应不同的内容或页面的任务交给前端来做，之前是通过服务端根据 url 的不同返回不同的页面实现的。
    
    （2）什么时候使用前端路由？
    
        在单页面应用，大部分页面结构不变，只改变部分内容的使用
    
    （3）前端路由有什么优点和缺点？
    
        优点
        用户体验好，不需要每次都从服务器全部获取，快速展现给用户
    
        缺点
        单页面无法记住之前滚动的位置，无法在前进，后退的时候记住滚动的位置
    
    前端路由一共有两种实现方式，一种是通过 hash 的方式，一种是通过使用 pushState 的方式。
   [《什么是“前端路由”》](https://segmentfault.com/q/1010000005336260)
   [《浅谈前端路由》 ](https://github.com/kaola-fed/blog/issues/137)
   [《前端路由是什么东西？》](https://www.zhihu.com/question/53064386)

### 10.7.js的执行过程和浏览器渲染过程

渲染过程

```
1.HTML代码转化成DOM
2.CSS代码转化成CSSOM（CSS Object Model）
3.结合DOM和CSSOM，生成一棵渲染树（包含每个节点的视觉信息）
4.生成布局（layout），即将所有渲染树的所有节点进行平面合成
5.将布局绘制（paint）在屏幕上
```

js执行过程

<img src="./img/js运行机制.png" style="zoom:40%;" />

<img src="./img/宏任务微任务.png" style="zoom:33%;" />

```js
同步和异步任务分别进入不同的执行"场所"，同步的进入主线程，异步的进入Event Table并注册函数。
当指定的事情完成时，Event Table会将这个函数移入Event Queue。
主线程内的任务执行完毕为空，会去Event Queue读取对应的函数，进入主线程执行。
上述过程会不断重复，也就是常说的Event Loop(事件循环)。

我们不禁要问了，那怎么知道主线程执行栈为空啊？js引擎存在monitoring process进程，会持续不断的检查主线程执行栈是否为空，一旦为空，就会去Event Queue那里检查是否有等待被调用的函数
let data = [];
$.ajax({
    url:www.javascript.com,
    data:data,
    success:() => {
        console.log('发送成功!');
    }
})
console.log('代码执行结束');
1.ajax进入Event Table，注册回调函数success。
2.执行console.log('代码执行结束')。
3.ajax事件完成，回调函数success进入Event Queue。
4.主线程从Event Queue读取回调函数success并执行。

setTimeout(() => {
    task()
},3000)

sleep(10000000)
1.task()进入Event Table并注册,计时开始。
2.执行sleep函数，很慢，非常慢，计时仍在继续。
3.3秒到了，计时事件timeout完成，task()进入Event Queue，但是sleep也太慢了吧，还没执行完，只好等着。
4.sleep终于执行完了，task()终于从Event Queue进入了主线程执行

macro-task(宏任务)：包括整体代码script，setTimeout，setInterval
micro-task(微任务)：Promise，process.nextTick

setTimeout(function() {
    console.log('setTimeout');
})
new Promise(function(resolve) {
    console.log('promise');
}).then(function() {
    console.log('then');
})
console.log('console');
1.这段代码作为宏任务，进入主线程。
2.先遇到setTimeout，那么将其回调函数注册后分发到宏任务Event Queue。(注册过程与上同，下文不再描述)
3.接下来遇到了Promise，new Promise立即执行，then函数分发到微任务Event Queue。
4.遇到console.log()，立即执行。
5.好啦，整体代码script作为第一个宏任务执行结束，看看有哪些微任务？我们发现了then在微任务Event Queue里面，执行。
6.ok，第一轮事件循环结束了，我们开始第二轮循环，当然要从宏任务Event Queue开始。我们发现了宏任务Event Queue中setTimeout对应的回调函数，立即执行。
7.结束


javascript是一门单线程语言
Event Loop是javascript的执行机制
```



## 11.安全

### 11.1.什么是 XSS 攻击？如何防范 XSS 攻击

     XSS 攻击指的是跨站脚本攻击，是一种代码注入攻击。攻击者通过在网站注入恶意脚本，使之在用户的浏览器上运行，从而盗取用户的信息如 cookie 等。
    
     XSS 的本质是因为网站没有对恶意代码进行过滤，与正常的代码混合在一起了，浏览器没有办法分辨哪些脚本是可信的，从而导致了恶意代码的执行。
    
     XSS 一般分为存储型、反射型和 DOM 型。
    
     存储型指的是恶意代码提交到了网站的数据库中，当用户请求数据的时候，服务器将其拼接为 HTML 后返回给了用户，从而导致了恶意代码的执行。
    
     反射型指的是攻击者构建了特殊的 URL，当服务器接收到请求后，从 URL 中获取数据，拼接到 HTML 后返回，从而导致了恶意代码的执行。
    
     DOM 型指的是攻击者构建了特殊的 URL，用户打开网站后，js 脚本从 URL 中获取数据，从而导致了恶意代码的执行。
    
     XSS 攻击的预防可以从两个方面入手，一个是恶意代码提交的时候，一个是浏览器执行恶意代码的时候。
    
     对于第一个方面，如果我们对存入数据库的数据都进行的转义处理，但是一个数据可能在多个地方使用，有的地方可能不需要转义，由于我们没有办法判断数据最后的使用场景，所以直接在输入端进行恶意代码的处理，其实是不太可靠的。
    
     因此我们可以从浏览器的执行来进行预防，一种是使用纯前端的方式，不用服务器端拼接后返回。另一种是对需要插入到 HTML 中的代码做好充分的转义。对于 DOM 型的攻击，主要是前端脚本的不可靠而造成的，我们对于数据获取渲染和字符串拼接的时候应该对可能出现的恶意代码情况进行判断。
    
     还有一些方式，比如使用 CSP ，CSP 的本质是建立一个白名单，告诉浏览器哪些外部资源可以加载和执行，从而防止恶意代码的注入攻击。
    
     还可以对一些敏感信息进行保护，比如 cookie 使用 http-only ，使得脚本无法获取。也可以使用验证码，避免脚本伪装成用户执行一些操作。
### 11.2. 什么是 CSP？

   ```
  CSP 指的是内容安全策略，它的本质是建立一个白名单，告诉浏览器哪些外部资源可以加载和执行。我们只需要配置规则，如何拦截由浏览器自己来实现。
  通常有两种方式来开启 CSP，一种是设置 HTTP 首部中的 Content-Security-Policy，一种是设置 meta 标签的方式 <meta http-equiv="Content-Security-Policy">
   ```

   详细资料可以参考：
   [《内容安全策略（CSP）》](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CSP) 
   [《前端面试之道》](https://juejin.im/book/5bdc715fe51d454e755f75ef/section/5bdc721851882516c33430a2)

### 11.3. 什么是 CSRF 攻击？如何防范 CSRF 攻击？

   ```
     CSRF 攻击指的是跨站请求伪造攻击，攻击者诱导用户进入一个第三方网站，然后该网站向被攻击网站发送跨站请求。如果用户在被攻击网站中保存了登录状态，那么攻击者就可以利用这个登录状态，绕过后台的用户验证，冒充用户向服务器执行一些操作。

     CSRF 攻击的本质是利用了 cookie 会在同源请求中携带发送给服务器的特点，以此来实现用户的冒充。

     一般的 CSRF 攻击类型有三种：
    
     第一种是 GET 类型的 CSRF 攻击，比如在网站中的一个 img 标签里构建一个请求，当用户打开这个网站的时候就会自动发起提交。

     第二种是 POST 类型的 CSRF 攻击，比如说构建一个表单，然后隐藏它，当用户进入页面时，自动提交这个表单。

     第三种是链接类型的 CSRF 攻击，比如说在 a 标签的 href 属性里构建一个请求，然后诱导用户去点击。

     CSRF 可以用下面几种方法来防护：

     第一种是同源检测的方法，服务器根据 http 请求头中 origin 或者 referer 信息来判断请求是否为允许访问的站点，从而对请求进行过滤。当 origin 或者 referer 信息都不存在的时候，直接阻止。这种方式的缺点是有些情况下 referer 可以被伪造。还有就是我们这种方法同时把搜索引擎的链接也给屏蔽了，所以一般网站会允许搜索引擎的页面请求，但是相应的页面请求这种
     请求方式也可能被攻击者给利用。
    
     第二种方法是使用 CSRF Token 来进行验证，服务器向用户返回一个随机数 Token ，当网站再次发起请求时，在请求参数中加入服务器端返回的 token ，然后服务器对这个 token 进行验证。这种方法解决了使用 cookie 单一验证方式时，可能会被冒用的问题，但是这种方法存在一个缺点就是，我们需要给网站中的所有请求都添加上这个 token，操作比较繁琐。还有一个问题是一般不会只有一台网站服务器，如果我们的请求经过负载平衡转移到了其他的服务器，但是这个服务器的 session 中没有保留这个 token 的话，就没有办法验证了。这种情况我们可以通过改变 token 的构建方式来解决。

     第三种方式使用双重 Cookie 验证的办法，服务器在用户访问网站页面时，向请求域名注入一个Cookie，内容为随机字符串，然后当用户再次向服务器发送请求的时候，从 cookie 中取出这个字符串，添加到 URL 参数中，然后服务器通过对 cookie 中的数据和参数中的数据进行比较，来进行验证。使用这种方式是利用了攻击者只能利用 cookie，但是不能访问获取 cookie 的特点。并且这种方法比 CSRF Token 的方法更加方便，并且不涉及到分布式访问的问题。这种方法的缺点是如果网站存在 XSS 漏洞的 ，那么这种方式会失效。同时这种方式不能做到子域名的隔离。

     第四种方式是使用在设置 cookie 属性的时候设置 Samesite ，限制 cookie 不能作为被第三方使用，从而可以避免被攻击者利用。Samesite 一共有两种模式，一种是严格模式，在严格模式下 cookie 在任何情况下都不可能作为第三方 Cookie 使用，
     在宽松模式下，cookie 可以被请求是 GET 请求，且会发生页面跳转的请求所使用。
 
   ```

   详细资料可以参考：
   [《前端安全系列之二：如何防止 CSRF 攻击？》](https://juejin.im/post/5bc009996fb9a05d0a055192)
   [《[ HTTP 趣谈] origin, referer 和 host 区别》](https://www.jianshu.com/p/1f9c71850299)

### 11.4. 什么是 Samesite Cookie 属性？

   ```
    Samesite Cookie 表示同站 cookie，避免 cookie 被第三方所利用。

    将 Samesite 设为 strict ，这种称为严格模式，表示这个 cookie 在任何情况下都不可能作为第三方 cookie。

    将 Samesite 设为 Lax ，这种模式称为宽松模式，如果这个请求是个 GET 请求，并且这个请求改变了当前页面或者打开了新的页面，那么这个 cookie 可以作为第三方 cookie，其余情况下都不能作为第三方 cookie。

    使用这种方法的缺点是，因为它不支持子域，所以子域没有办法与主域共享登录信息，每次转入子域的网站，都回重新登录。还有一个问题就是它的兼容性不够好。
   ```

### 11.5. 什么是点击劫持？如何防范点击劫持？

   ```
    点击劫持是一种视觉欺骗的攻击手段，攻击者将需要攻击的网站通过 iframe 嵌套的方式嵌入自己的网页中，并将 iframe 设置为透明，在页面中透出一个按钮诱导用户点击。

    我们可以在 http 相应头中设置 X-FRAME-OPTIONS 来防御用 iframe 嵌套的点击劫持攻击。通过不同的值，可以规定页面在特定的一些情况才能作为 iframe 来使用。
   ```

   详细资料可以参考：
   [《web 安全之--点击劫持攻击与防御技术简介》](https://www.jianshu.com/p/251704d8ff18)

### 11.6.SQL 注入攻击？

   ```
    SQL 注入攻击指的是攻击者在 HTTP 请求中注入恶意的 SQL 代码，服务器使用参数构建数据库 SQL 命令时，恶意 SQL 被一起构造，破坏原有 SQL 结构，并在数据库中执行，达到编写程序时意料之外结果的攻击行为。
   ```

   详细资料可以参考：
 [《Web 安全漏洞之 SQL 注入》](https://juejin.im/post/5bd5b820e51d456f72531fa8)
 [《如何防范常见的 Web 攻击》](http://blog.720ui.com/2016/security_web/#SQL%E6%B3%A8%E5%85%A5%E6%94%BB%E5%87%BB)

## 12.模式

### 12.1.单例模式是什么？

   ```
    单例模式保证了全局只有一个实例来被访问。比如说常用的如弹框组件的实现和全局状态的实现。
   ```

### 12.2.策略模式是什么？

   ```
    策略模式主要是用来将方法的实现和方法的调用分离开，外部通过不同的参数可以调用不同的策略。我主要在 MVP 模式解耦的时候用来将视图层的方法定义和方法调用分离。
   ```

### 12.3.代理模式是什么？

   ```
    代理模式是为一个对象提供一个代用品或占位符，以便控制对它的访问。比如说常见的事件代理。
   ```

### 12.4.中介者模式是什么？

   ```
    中介者模式指的是，多个对象通过一个中介者进行交流，而不是直接进行交流，这样能够将通信的各个对象解耦。
   ```

### 12.5.适配器模式是什么？

   ```
    适配器用来解决两个接口不兼容的情况，不需要改变已有的接口，通过包装一层的方式实现两个接口的正常协作。假如我们需要一种新的接口返回方式，但是老的接口由于在太多地方已经使用了，不能随意更改，这个时候就可以使用适配器模式。比如我们需要一种自定义的时间返回格式，但是我们又不能对 js 时间格式化的接口进行修改，这个时候就可以使用适配器模式。
   ```

   更多关于设计模式的资料可以参考：
   [《前端面试之道》](https://juejin.im/book/5bdc715fe51d454e755f75ef/section/5bdc74186fb9a049ab0d0b6b)
   [《JavaScript 设计模式》](https://juejin.im/post/59df4f74f265da430f311909#heading-3)
   [《JavaScript 中常见设计模式整理》](https://juejin.im/post/5afe6430518825428630bc4d)

### 12.6. 观察者模式和发布订阅模式有什么不同？

   ```
    发布订阅模式其实属于广义上的观察者模式

    在观察者模式中，观察者需要直接订阅目标事件。在目标发出内容改变的事件后，直接接收事件并作出响应。
    
    而在发布订阅模式中，发布者和订阅者之间多了一个调度中心。调度中心一方面从发布者接收事件，另一方面向订阅者发布事件，订阅者需要在调度中心中订阅事件。通过调度中心实现了发布者和订阅者关系的解耦。使用发布订阅者模式更利于我们代码的可维护性。
   ```

   详细资料可以参考：
   [《观察者模式和发布订阅模式有什么不同？》](https://www.zhihu.com/question/23486749)

### 12.7.手写一个jsonp

```javascript
function jsonp(url, params, callback) {

  // 判断是否含有参数
  let queryString = url.indexOf("?") === "-1" ? "?" : "&";

  // 添加参数
  for (var k in params) {
    if (params.hasOwnProperty(k)) {
      queryString += k + "=" + params[k] + "&";
    }
  }

  // 处理回调函数名
  let random = Math.random().toString().replace(".", ""),
    callbackName = "myJsonp" + random;

  // 添加回调函数
  queryString += "callback=" + callbackName;

  // 构建请求
  let scriptNode = document.createElement("script");
  scriptNode.src = url + queryString;

  window[callbackName] = function () {

    // 调用回调函数
    callback(...arguments);

    // 删除这个引入的脚本
    document.getElementsByTagName("head")[0].removeChild(scriptNode);
  }

  // 发起请求
  document.getElementsByTagName("head")[0].appendChild(scriptNode);
}
```
[《原生 jsonp 具体实现》](https://www.cnblogs.com/zzc5464/p/jsonp.html)
[《jsonp 的原理与实现》](https://segmentfault.com/a/1190000007665361#articleHeader1)

### 12.8.手写一个观察者模式

```javascript
var events = (function () {

  var topics = {};

  return {

    // 注册监听函数
    subscribe: function (topic, handler) {
      if (!topics.hasOwnProperty(topic)) {
        topics[topic] = [];
      }
      topics[topic].push(handler);
    },

    // 发布事件，触发观察者回调事件
    publish: function (topic, info) {
      if (topics.hasOwnProperty(topic)) {
        topics[topic].forEach(function (handler) {
          handler(info);
        })
      }
    },

    // 移除主题的一个观察者的回调事件
    remove: function (topic, handler) {

      if (!topics.hasOwnProperty(topic)) return;

      var handlerIndex = -1;
      topics[topic].forEach(function (item, index) {
        if (item === handler) {
          handlerIndex = index;
        }
      })

      if (handlerIndex >= 0) {
        topics[topic].splice(handlerIndex, 1);
      }
    },

    // 移除主题的所有观察者的回调事件
    removeAll: function (topic) {
      if (topics.hasOwnProperty(topic)) {
        topics[topic] = [];
      }
    }
  }

})();
```
## 13.网络

### 13.1.一次http请求的过程



## 14.对象

### 14.1.javascript 创建对象的几种方式

```
    我们一般使用字面量的形式直接创建对象，但是这种创建方式对于创建大量相似对象的时候，会产生大量的重复代码。但 js和一般的面向对象的语言不同，在 ES6 之前它没有类的概念。但是我们可以使用函数来进行模拟，从而产生出可复用的对象创建方式，我了解到的方式有这么几种：

    （1）第一种是工厂模式，工厂模式的主要工作原理是用函数来封装创建对象的细节，从而通过调用函数来达到复用的目的。但是它有一个很大的问题就是创建出来的对象无法和某个类型联系起来，它只是简单的封装了复用代码，而没有建立起对象和类型间的关系。

    （2）第二种是构造函数模式。js 中每一个函数都可以作为构造函数，只要一个函数是通过 new 来调用的，那么我们就可以把它称为构造函数。执行构造函数首先会创建一个对象，然后将对象的原型指向构造函数的 prototype 属性，然后将执
行上下文中的 this 指向这个对象，最后再执行整个函数，如果返回值不是对象，则返回新建的对象。因为 this 的值指向了新建的对象，因此我们可以使用 this 给对象赋值。构造函数模式相对于工厂模式的优点是，所创建的对象和构造函数建立起了联系，因此我们可以通过原型来识别对象的类型。但是构造函数存在一个缺点就是，造成了不必要的函数对象的创建，因为在 js 中函数也是一个对象，因此如果对象属性中如果包含函数的话，那么每次我们都会新建一个函数对象，浪费了不必要的内存空间，因为函数是所有的实例都可以通用的。

    （3）第三种模式是原型模式，因为每一个函数都有一个 prototype 属性，这个属性是一个对象，它包含了通过构造函数创建的所有实例都能共享的属性和方法。因此我们可以使用原型对象来添加公用属性和方法，从而实现代码的复用。这种方
式相对于构造函数模式来说，解决了函数对象的复用问题。但是这种模式也存在一些问题，一个是没有办法通过传入参数来初始化值，另一个是如果存在一个引用类型如 Array 这样的值，那么所有的实例将共享一个对象，一个实例对引用类型值的改变会影响所有的实例。

    （4）第四种模式是组合使用构造函数模式和原型模式，这是创建自定义类型的最常见方式。因为构造函数模式和原型模式分开使用都存在一些问题，因此我们可以组合使用这两种模式，通过构造函数来初始化对象的属性，通过原型对象来实现函数方法的复用。这种方法很好的解决了两种模式单独使用时的缺点，但是有一点不足的就是，因为使用了两种不同的模式，所以对于代码的封装性不够好。

    （5）第五种模式是动态原型模式，这一种模式将原型方法赋值的创建过程移动到了构造函数的内部，通过对属性是否存在的判断，可以实现仅在第一次调用函数时对原型对象赋值一次的效果。这一种方式很好地对上面的混合模式进行了封装。

    （6）第六种模式是寄生构造函数模式，这一种模式和工厂模式的实现基本相同，我对这个模式的理解是，它主要是基于一个已有的类型，在实例化时对实例化的对象进行扩展。这样既不用修改原来的构造函数，也达到了扩展对象的目的。它的一个缺点和工厂模式一样，无法实现对象的识别。

    嗯我目前了解到的就是这么几种方式。
```

[红宝书:第6章]()

### 14.2.JavaScript 继承的几种实现方式

```
    我了解的 js 中实现继承的几种方式有：

    （1）第一种是以原型链的方式来实现继承，但是这种实现方式存在的缺点是，在包含有引用类型的数据时，会被所有的实例对象所共享，容易造成修改的混乱。还有就是在创建子类型的时候不能向超类型传递参数。

    （2）第二种方式是使用借用构造函数的方式，这种方式是通过在子类型的函数中调用超类型的构造函数来实现的，这一种方法解决了不能向超类型传递参数的缺点，但是它存在的一个问题就是无法实现函数方法的复用，并且超类型原型定义的方法子类型也没有办法访问到。

    （3）第三种方式是组合继承，组合继承是将原型链和借用构造函数组合起来使用的一种方式。通过借用构造函数的方式来实现类型的属性的继承，通过将子类型的原型设置为超类型的实例来实现方法的继承。这种方式解决了上面的两种模式单独使用时的问题，但是由于我们是以超类型的实例来作为子类型的原型，所以调用了两次超类的构造函数，造成了子类型的原型中多了很多不必要的属性。

    （4）第四种方式是原型式继承，原型式继承的主要思路就是基于已有的对象来创建新的对象，实现的原理是，向函数中传入一个对象，然后返回一个以这个对象为原型的对象。这种继承的思路主要不是为了实现创造一种新的类型，只是对某个对象实现一种简单继承，ES5 中定义的 Object.create() 方法就是原型式继承的实现。缺点与原型链方式相同。

    （5）第五种方式是寄生式继承，寄生式继承的思路是创建一个用于封装继承过程的函数，通过传入一个对象，然后复制一个对象的副本，然后对象进行扩展，最后返回这个对象。这个扩展的过程就可以理解是一种继承。这种继承的优点就是对一个简单对象实现继承，如果这个对象不是我们的自定义类型时。缺点是没有办法实现函数的复用。

    （6）第六种方式是寄生式组合继承，组合继承的缺点就是使用超类型的实例做为子类型的原型，导致添加了不必要的原型属性。寄生式组合继承的方式是使用超类型的原型的副本来作为子类型的原型，这样就避免了创建不必要的属性。
```

[红宝书:第6章]()

### 14.3.谈谈 This 对象的理解

```
    this 是执行上下文中的一个属性，它指向最后一次调用这个方法的对象。在实际开发中，this 的指向可以通过四种调用模式来判断。

    （1）第一种是函数调用模式，当一个函数不是一个对象的属性时，直接作为函数来调用时，this 指向全局对象。

    （2）第二种是方法调用模式，如果一个函数作为一个对象的方法来调用时，this 指向这个对象。

    （3）第三种是构造器调用模式，如果一个函数用 new 调用时，函数执行前会新创建一个对象，this 指向这个新创建的对象。
    
    （4）第四种是 apply 、 call 和 bind 调用模式，这三个方法都可以显示的指定调用函数的 this 指向。其中 apply方法接收两个参数：一个是 this 绑定的对象，一个是参数数组。call 方法接收的参数，第一个是 this 绑定的对象
，后面的其余参数是传入函数执行的参数。也就是说，在使用 call() 方法时，传递给函数的参数必须逐个列举出来。bind 方法通过传入一个对象，返回一个 this 绑定了传入对象的新函数。这个函数的 this 指向除了使用 new 时会被改变，其他情况下都不会改变。

    这四种方式，使用构造器调用模式的优先级最高，然后是 apply 、 call 和 bind 调用模式，然后是方法调用模式，然后是函数调用模式。
```

[《JavaScript 深入理解之 this 详解》](http://cavszhouyou.top/JavaScript%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3%E4%B9%8Bthis%E8%AF%A6%E8%A7%A3.html)

## other

### 1.手写 call、apply 及 bind 函数

```javascript
call函数实现

Function.prototype.myCall = function (context) {
  // 判断调用对象
  if (typeof this !== "function") {
    console.error("type error");
  }

  // 获取参数
  let args = [...arguments].slice(1),
    result = null;

  // 判断 context 是否传入，如果未传入则设置为 window
  context = context || window;

  // 将调用函数设为对象的方法
  context.fn = this;

  // 调用函数
  result = context.fn(...args);

  // 将属性删除
  delete context.fn;

  return result;
}
```


```javascript
apply 函数实现

Function.prototype.myApply = function (context) {
  // 判断调用对象是否为函数
  if (typeof this !== "function") {
    throw new TypeError("Error");
  }

  let result = null;

  // 判断 context 是否存在，如果未传入则为 window
  context = context || window;

  // 将函数设为对象的方法
  context.fn = this;

  // 调用方法
  if (arguments[1]) {
    result = context.fn(...arguments[1]);
  } else {
    result = context.fn();
  }

  // 将属性删除
  delete context.fn;

  return result;
}
```


```javascript
bind 函数实现

Function.prototype.myBind = function (context) {
  // 判断调用对象是否为函数
  if (typeof this !== "function") {
    throw new TypeError("Error");
  }

  // 获取参数
  var args = [...arguments].slice(1),
    fn = this;

  return function Fn() {

    // 根据调用方式，传入不同绑定值
    return fn.apply(this instanceof Fn ? this : context, args.concat(...arguments));
  }

}
call 函数的实现步骤：

（1）判断调用对象是否为函数，即使我们是定义在函数的原型上的，但是可能出现使用 call 等方式调用的情况。

（2）判断传入上下文对象是否存在，如果不存在，则设置为 window 。

（3）处理传入的参数，截取第一个参数后的所有参数。

（4）将函数作为上下文对象的一个属性。

（5）使用上下文对象来调用这个方法，并保存返回结果。

（6）删除刚才新增的属性。

（7）返回结果。
```


    apply 函数的实现步骤：
    
    （1）判断调用对象是否为函数，即使我们是定义在函数的原型上的，但是可能出现使用 call 等方式调用的情况。
    
    （2）判断传入上下文对象是否存在，如果不存在，则设置为 window 。
    
    （3）将函数作为上下文对象的一个属性。
    
    （4）判断参数值是否传入
    
    （4）使用上下文对象来调用这个方法，并保存返回结果。
    
    （5）删除刚才新增的属性
    
    （6）返回结果


    bind 函数的实现步骤：
    
    （1）判断调用对象是否为函数，即使我们是定义在函数的原型上的，但是可能出现使用 call 等方式调用的情况。
    
    （2）保存当前函数的引用，获取其余传入参数值。
    
    （3）创建一个函数返回
    
    （4）函数内部使用 apply 来绑定函数调用，需要判断函数作为构造函数的情况，这个时候需要传入当前函数的 this 给 apply 调
        用，其余情况都传入指定的上下文对象。
  [《手写 call、apply 及 bind 函数》](https://juejin.im/book/5bdc715fe51d454e755f75ef/section/5bdd0d8e6fb9a04a044073fe)
 [《JavaScript 深入之 call 和 apply 的模拟实现》](https://github.com/mqyqingfeng/Blog/issues/11)

### 2.函数柯里化的实现

```javascript
函数柯里化指的是一种将使用多个参数的一个函数转换成一系列使用一个参数的函数的技术。

function curry(fn, args) {

  // 获取函数需要的参数长度
  let length = fn.length;

  args = args || [];

  return function () {

    let subArgs = args.slice(0);
    
    // 拼接得到现有的所有参数
    for (let i = 0; i < arguments.length; i++) {
      subArgs.push(arguments[i]);
    }
    
    // 判断参数的长度是否已经满足函数所需参数的长度
    if (subArgs.length >= length) {
      // 如果满足，执行函数
      return fn.apply(this, subArgs);
    } else {
      // 如果不满足，递归返回柯里化的函数，等待参数的传入
      return curry.call(this, fn, subArgs);
    }

  }
}

// es6 实现
function curry(fn, ...args) {
  return fn.length <= args.length ? fn(...args) : curry.bind(null, fn, ...args);
}
```
### 3.Object.is() 与原来的比较操作符 “===”、“==” 的区别

    使用双等号进行相等判断时，如果两边的类型不一致，则会进行强制类型转化后再进行比较。
    
    使用三等号进行相等判断时，如果两边的类型不一致时，不会做强制类型准换，直接返回 false。
    
    使用 Object.is 来进行相等判断时，一般情况下和三等号的判断相同，它处理了一些特殊的情况，比如 -0 和 +0 不再相等，两个 NaN 认定为是相等的。 
### 4.escape,encodeURI,encodeURIComponent 有什么区别

   ```
    encodeURI 是对整个 URI 进行转义，将 URI 中的非法字符转换为合法字符，所以对于一些在 URI 中有特殊意义的字符不会进行转义。

    encodeURIComponent 是对 URI 的组成部分进行转义，所以一些特殊字符也会得到转义。 

    escape 和 encodeURI 的作用相同，不过它们对于 unicode 编码为 0xff 之外字符的时候会有区别，escape 是直接在字符的unicode 编码前加上 %u，而 encodeURI 首先会将字符转换为 UTF-8 的格式，再在每个字节前加上 %。
   ```

   [《escape,encodeURI,encodeURIComponent 有什么区别?》](https://www.zhihu.com/question/21861899)

### 5.js 的事件循环是什么

   ```
    因为 js 是单线程运行的，在代码执行的时候，通过将不同函数的执行上下文压入执行栈中来保证代码的有序执行。在执行同步代码的时候，如果遇到了异步事件，js 引擎并不会一直等待其返回结果，而是会将这个事件挂起，继续执行执行栈中的其他任务。当异步事件执行完毕后，再将异步事件对应的回调加入到与当前执行栈中不同的另一个任务队列中等待执行。任务队列可以分为宏任务对列和微任务对列，当当前执行栈中的事件执行完毕后，js 引擎首先会判断微任务对列中是否有任务可以执行，如果有就将微任务队首的事件压入栈中执行。当微任务对列中的任务都执行完成后再去判断宏任务对列中的任务。

    微任务包括了 promise 的回调、node 中的 process.nextTick 、对 Dom 变化监听的 MutationObserver。

    宏任务包括了 script 脚本的执行、setTimeout ，setInterval ，setImmediate 一类的定时事件，还有如 I/O 操作、UI 渲染等。
   ```

   详细资料可以参考：
   [《浏览器事件循环机制（event loop）》](https://juejin.im/post/5afbc62151882542af04112d)
   [《详解 JavaScript 中的 Event Loop（事件循环）机制》](https://zhuanlan.zhihu.com/p/33058983)
   [《什么是 Event Loop？》](http://www.ruanyifeng.com/blog/2013/10/event_loop.html)
   [《这一次，彻底弄懂 JavaScript 执行机制》](https://juejin.im/post/59e85eebf265da430d571f89)

### 6.为什么 0.1 + 0.2 != 0.3？如何解决这个问题？

   ```
    当计算机计算 0.1+0.2 的时候，实际上计算的是这两个数字在计算机里所存储的二进制，0.1 和 0.2 在转换为二进制表示的时候会出现位数无限循环的情况。js 中是以 64 位双精度格式来存储数字的，只有 53 位的有效数字，超过这个长度的位数会被截取掉
    这样就造成了精度丢失的问题。这是第一个会造成精度丢失的地方。在对两个以 64 位双精度格式的数据进行计算的时候，首先会进行对阶的处理，对阶指的是将阶码对齐，也就是将小数点的位置对齐后，再进行计算，一般是小阶向大阶对齐，因此小阶的数在对齐的过程中，有效数字会向右移动，移动后超过有效位数的位会被截取掉，这是第二个可能会出现精度丢失的地方。当两个数据阶码对齐后，进行相加运算后，得到的结果可能会超过 53 位有效数字，因此超过的位数也会被截取掉，这是可能发生精度丢失的第三个地方。
    对于这样的情况，我们可以将其转换为整数后再进行运算，运算后再转换为对应的小数，以这种方式来解决这个问题。
    我们还可以将两个数相加的结果和右边相减，如果相减的结果小于一个极小数，那么我们就可以认定结果是相等的，这个极小数可以使用 es6 的 Number.EPSILON
   ```

   详细资料可以参考：
   [《十进制的 0.1 为什么不能用二进制很好的表示？》](https://blog.csdn.net/Lixuanshengchao/article/details/82049191)
   [《十进制浮点数转成二进制》](https://blog.csdn.net/zhengyanan815/article/details/78550073)
   [《浮点数的二进制表示》](http://www.ruanyifeng.com/blog/2010/06/ieee_floating-point_representation.html)
   [《js 浮点数存储精度丢失原理》](https://juejin.im/post/5b372f106fb9a00e6714aa21)
   [《浮点数精度之谜》](https://juejin.im/post/594a31d0a0bb9f006b0b2624)
   [《JavaScript 浮点数陷阱及解法》](https://github.com/camsong/blog/issues/9)
   [《0.1+0.2 !== 0.3？》](https://juejin.im/post/5bd2f10a51882555e072d0c4)
   [《JavaScript 中奇特的~运算符》](https://juejin.im/entry/59cdd7fb6fb9a00a600f8eef)

### 7.原码、反码和补码的介绍

   ```
     原码是计算机中对数字的二进制的定点表示方法，最高位表示符号位，其余位表示数值位。优点是易于分辨，缺点是不能够直接参与运算。

     正数的反码和其原码一样；负数的反码，符号位为1，数值部分按原码取反。
     如 [+7]原 = 00000111，[+7]反 = 00000111； [-7]原 = 10000111，[-7]反 = 11111000。

     正数的补码和其原码一样；负数的补码为其反码加1。

     例如 [+7]原 = 00000111，[+7]反 = 00000111，[+7]补 = 00000111； 
         [-7]原 = 10000111，[-7]反 = 11111000，[-7]补 = 11111001

     之所以在计算机中使用补码来表示负数的原因是，这样可以将加法运算扩展到所有的数值计算上，因此在数字电路中
     我们只需要考虑加法器的设计就行了，而不用再为减法设置新的数字电路。
   ```

   详细资料可以参考：
   [《关于2的补码》](http://www.ruanyifeng.com/blog/2009/08/twos_complement.html)

### 8.toPrecision 和 toFixed 和 Math.round 、ceil、floor？

   ```
    toPrecision 用于处理精度，精度是从左至右第一个不为 0 的数开始数起。
    toFixed 是对小数点后指定位数取整，从小数点开始数起。
    Math.round 是将一个数字四舍五入到一个整数。
    Math.ceil() === 向上取整，函数返回一个大于或等于给定数字的最小整数。
    Math.floor() === 向下取整，函数返回一个小于或等于给定数字的最大整数
   ```

### 9.js中命名规则

```
（1）第一个字符必须是字母、下划线（_）或美元符号（$）
（2）余下的字符可以是下划线、美元符号或任何字母或数字字符

一般我们推荐使用驼峰法来对变量名进行命名，因为这样可以与 ECMAScript 内置的函数和对象命名格式保持一致。
```

### 10.如何获取一个元素相对于浏览器窗口左上角的位置

```
  可以利用Dom元素上的getBoundingClientRect()方法，该方法直接返回一个对象，可以拿到距离浏览器的距离
```

### 11.[-4,-3,0,0,0,6,2,3]这样一个有规律的数列，0的左边都是负数，0的右边都是整数，要求找到与0相邻的正数和负数（-3,6），要求复杂度为log n

```
采用两次二分查找找出整数和负数
```

### 12.怎么判断一个数组

```
Array.isArray，instance of，Object.prototype.toString.call(arr)
```

### 13.实现一个合并数组的方法，使其合并后的数组仍是有序的

```
利用双指针，每次比较指针的位置，当一个指针完时，将另一个数组直接添加到结果数组之后即可（此处拓展了一下数组的concat和slice方法
```

### 14.[1,2,3].map(parseInt)的结果

    parseInt() 函数能解析一个字符串，并返回一个整数，需要两个参数 (val, radix)，其中 radix 表示要解析的数字的基数。（该值介于 2 ~ 36 之间，并且字符串中的数字不能大于 radix 才能正确返回数字结果值）。
    
    此处 map 传了 3 个参数 (element, index, array)，默认第三个参数被忽略掉，因此三次传入的参数分别为
    
    "1-0", "2-1", "3-2"
    
    因为字符串的值不能大于基数，因此后面两次调用均失败，返回 NaN ，第一次基数为 0 ，按十进制解析返回 1。
    
    由于map()接收的回调函数可以有3个参数：callback(currentValue, index, array)，通常我们仅需要第一个参数，而忽略了传入的后面两个参数。不幸的是，parseInt(string, radix)没有忽略第二个参数，导致实际执行的函数分别是：
    
    parseInt('1', 0); // 1, 按十进制转换
    
    parseInt('2', 1); // NaN, 没有一进制
    
    parseInt('3', 2); // NaN, 按二进制转换不允许出现3
    
    可以改为r = arr.map(Number);，因为Number(value)函数仅接收一个参数。
[《为什么 ["1", "2", "3"].map(parseInt) 返回 [1,NaN,NaN]？》](https://blog.csdn.net/justjavac/article/details/19473199)

### 15.array reduce 实现 filter

```javascript
Array.prototype._filter = function (fn) {
    const self = this
    return this.reduce(function(pre, cur, index) {
        // console.log(fn.call(self, cur, index))
        if (fn.call(self, cur, index, self)) {
            pre.push(cur)
        }
        return pre
    }, [])
}

;{
    const arr = [11, 22, 33, 44, 55]
    const res = arr._filter((item, index, arr) => {
        console.log(item, index, arr)
        return item > 33
    })
    console.log(res)
}
```

[reduce实现filter](http://js.jsrun.net/ywyKp/edit)

### 16.instanceof 的作用？

   ```js
    instanceof 运算符用于判断构造函数的 prototype 属性是否出现在对象的原型链中的任何位置。

    实现：

    function myInstanceof(left, right) {
      let proto = Object.getPrototypeOf(left), // 获取对象的原型
        prototype = right.prototype; // 获取构造函数的 prototype 对象

      // 判断构造函数的 prototype 对象是否在对象的原型链上
      while (true) {
        if (!proto) return false;
        if (proto === prototype) return true;

        proto = Object.getPrototypeOf(proto);
      }
    }
   ```

 [《instanceof》](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/instanceof)

### 17.Set 和 WeakSet 、Map 和 WeakMap 结构？

   ```
（1）ES6 提供了新的数据结构 Set。它类似于数组，但是成员的值都是唯一的，没有重复的值。
（2）WeakSet 结构与 Set 类似，也是不重复的值的集合。但是 WeakSet 的成员只能是对象，而不能是其他类型的值。WeakSet 中的对象都是弱引用，即垃圾回收机制不考虑 WeakSet 对该对象的引用，
 
（1）Map 数据结构。它类似于对象，也是键值对的集合，但是“键”的范围不限于字符串，各种类型的值（包括对象）都可以当作键。
（2）WeakMap 结构与 Map 结构类似，也是用于生成键值对的集合。但是 WeakMap 只接受对象作为键名（ null 除外），不接受其他类型的值作为键名。而且 WeakMap 的键名所指向的对象，不计入垃圾回收机制。
   ```

### 18.基本数据类型与引用数据类型，堆、栈区别联系

```
   js 一共有7种基本数据类型，分别是 Undefined、Null、Boolean、Number、String，object,还有在 ES6 中新增的 Symbol 类型，代表创建后独一无二且不可变的数据类型，它的出现我认为主要是为了解决可能出现的全局变量冲突的问题。
   
   js 可以分为两种类型的值，一种是基本数据类型，一种是复杂数据类型。
   基本数据类型....（上面）
   复杂数据类型指的是 Object 类型，所有其他的如 Array、Date 等数据类型都可以理解为 Object 类型的子类。

   两种类型间的主要区别是它们的存储位置不同，基本数据类型的值直接保存在栈中，而复杂数据类型的值保存在堆中，通过使用在栈中保存对应的指针来获取堆中的值。
```

[《JavaScript 有几种类型的值？》](https://blog.csdn.net/lxcao/article/details/52749421)
[《JavaScript 有几种类型的值？能否画一下它们的内存图；》](https://blog.csdn.net/jiangjuanjaun/article/details/80327342)

### 19.哪些操作会造成内存泄漏

   ```
（1）意外的全局变量
（2）被遗忘的计时器或回调函数
（3）脱离 DOM 的引用
（4）闭包

第一种情况是我们由于使用未声明的变量，而意外的创建了一个全局变量，而使这个变量一直留在内存中无法被回收。

第二种情况是我们设置了 setInterval 定时器，而忘记取消它，如果循环函数有对外部变量的引用的话，那么这个变量会被一直留在内存中，而无法被回收。

第三种情况是我们获取一个 DOM 元素的引用，而后面这个元素被删除，由于我们一直保留了对这个元素的引用，所以它也无法被回收。

第四种情况是不合理的使用闭包，从而导致某些变量一直被留在内存当中。
   ```

   详细资料可以参考：
   [《JavaScript 内存泄漏教程》](http://www.ruanyifeng.com/blog/2017/04/memory-leak.html)
   [《4类 JavaScript 内存泄漏及如何避免》](https://jinlong.github.io/2016/05/01/4-Types-of-Memory-Leaks-in-JavaScript-and-How-to-Get-Rid-Of-Them/)
   [《杜绝 js 中四种内存泄漏类型的发生》](https://juejin.im/entry/5a64366c6fb9a01c9332c706)
   [《javascript 典型内存泄漏及 chrome 的排查方法》](https://segmentfault.com/a/1190000008901861)

### 20. js 中的深浅拷贝实现

```javascript
浅拷贝的实现

function shallowCopy(object) {

  // 只拷贝对象
  if (!object || typeof object !== "object") return;

  // 根据 object 的类型判断是新建一个数组还是对象
  let newObject = Array.isArray(object) ? [] : {};

  // 遍历 object，并且判断是 object 的属性才拷贝
  for (let key in object) {
    if (object.hasOwnProperty(key)) {
      newObject[key] = object[key];
    }
  }

  return newObject;
}

深拷贝的实现

function deepCopy(object) {

  if (!object || typeof object !== "object") return;

  let newObject = Array.isArray(object) ? [] : {};

  for (let key in object) {
    if (object.hasOwnProperty(key)) {
      newObject[key] = typeof object[key] === "object" ? deepCopy(object[key]) : object[key];
    }
  }

  return newObject;
}
```
    浅拷贝指的是将一个对象的属性值复制到另一个对象，如果有的属性的值为引用类型的话，那么会将这个引用的地址复制给对象，因此两个对象会有同一个引用类型的引用。浅拷贝可以使用  Object.assign 和展开运算符来实现。
    
    深拷贝相对浅拷贝而言，如果遇到属性值为引用类型的时候，它新建一个引用类型并将对应的值复制给它，因此对象获得的一个新的引用类型而不是一个原有类型的引用。深拷贝对于一些对象可以使用 JSON 的两个函数来实现，但是由于 JSON 的对象格式比 js 的对象格式更加严格，所以如果属性值里边出现函数或者 Symbol 类型的值时，会转换失败。
[《JavaScript 专题之深浅拷贝》](https://github.com/mqyqingfeng/Blog/issues/32)
[《前端面试之道》](https://juejin.im/book/5bdc715fe51d454e755f75ef/section/5bed40d951882545f73004f6)

### 20.Js 动画与 CSS 动画区别及相应实现

    CSS3 的动画的优点
    
    在性能上会稍微好一些，浏览器会对 CSS3 的动画做一些优化
    代码相对简单
    
    缺点
    
    在动画控制上不够灵活
    兼容性不好
    
    JavaScript 的动画正好弥补了这两个缺点，控制能力很强，可以单帧的控制、变换，同时写得好完全可以兼容 IE6，并且功能强大。对于一些复杂控制的动画，使用 javascript 会比较靠谱。而在实现一些小的交互动效的时候，就多考虑考虑 CSS 吧
### 21.get 请求传参长度的误区

   ```
    误区：我们经常说 get 请求参数的大小存在限制，而 post 请求的参数大小是无限制的。

    实际上 HTTP 协议从未规定 GET/POST 的请求长度限制是多少。对 get 请求参数的限制是来源与浏览器或web 服务器，浏览器或 web 服务器限制了 url 的长度。为了明确这个概念，我们必须再次强调下面几点:

    （1）HTTP 协议未规定 GET 和 POST 的长度限制
    （2）GET 的最大长度显示是因为浏览器和 web 服务器限制了 URI 的长度
    （3）不同的浏览器和 WEB 服务器，限制的最大长度不一样
    （4）要支持 IE，则最大长度为 2083byte，若只支持 Chrome，则最大长度 8182byte
   ```

### 22.get 和 post 请求在缓存方面的区别

   相关知识点：

   ```
    get 请求类似于查找的过程，用户获取数据，可以不用每次都与数据库连接，所以可以使用缓存。

    post 不同，post 做的一般是修改和删除的工作，所以必须与数据库交互，所以不能使用缓存。因此 get 请求适合于请求缓存。
   ```

   回答：

   ```
    缓存一般只适用于那些不会更新服务端数据的请求。一般 get 请求都是查找请求，不会对服务器资源数据造成修改，而 post 请求一般都会对服务器数据造成修改，所以，一般会对 get 请求进行缓存，很少会对 post 请求进行缓存。    
   ```

[《HTML 关于 post 和 get 的区别以及缓存问题的理解》](https://blog.csdn.net/qq_27093465/article/details/50479289)

### 23.什么是 Proxy

    Proxy 用于修改某些操作的默认行为，等同于在语言层面做出修改，所以属于一种“元编程”，即对编程语言进行编程。
    
    Proxy 可以理解成，在目标对象之前架设一层“拦截”，外界对该对象的访问，都必须先通过这层拦截，因此提供了一种机制，可以
    对外界的访问进行过滤和改写。Proxy 这个词的原意是代理，用在这里表示由它来“代理”某些操作，可以译为“代理器”。
[Proxy](https://es6.ruanyifeng.com/?search=instanceof&x=0&y=0#docs/proxy)

### 24.\__proto__

```javascript
var Student = {
    name: 'Robot',
    height: 1.2,
    run: function () {
        console.log(this.name + ' is running...');
    }
};

var xiaoming = {
    name: '小明'
};

xiaoming.__proto__ = Student;
```

注意最后一行代码把`xiaoming`的原型指向了对象`Student`，看上去`xiaoming`仿佛是从`Student`继承下来的：

### 25.强缓存/协商缓存

强缓存

```
当浏览器去请求某个文件的时候，服务端就在respone header里面对该文件做了缓存配置。缓存的时间、缓存类型都由服务端控制，具体表现为：respone header 的cache-control，常见的设置是max-age public private no-cache no-store等
max-age表示缓存的时间是315360000秒（10年），public表示可以被浏览器和代理服务器缓存，代理服务器一般可用nginx来做。immutable表示该资源永远不变，但是实际上该资源并不是永远不变，它这么设置的意思是为了让用户在刷新页面的时候不要去请求服务器！如果没有 immutable 的话则用户刷新页面，即使资源没有过期，浏览器也会去请求服务器

1.cache-control: max-age=xxxx，public
  客户端和代理服务器都可以缓存该资源；
  客户端在xxx秒的有效期内，如果有请求该资源的需求的话就直接读取缓存,statu code:200 ，如果用户做了刷新操作，就	 向服务器发起http请求

2.cache-control: max-age=xxxx，private
  只让客户端可以缓存该资源；代理服务器不缓存
  客户端在xxx秒内直接读取缓存,statu code:200

3.cache-control: max-age=xxxx，immutable
  客户端在xxx秒的有效期内，如果有请求该资源的需求的话就直接读取缓存,statu code:200 ，即使用户做了刷新操作，也	 不向服务器发起http请求

4.cache-control: no-cache
  跳过设置强缓存，但是不妨碍设置协商缓存；一般如果你做了强缓存，只有在强缓存失效了才走协商缓存的，设置了no- 	     cache就不会走强缓存了，每次请求都回询问服务端。

5.cache-control: no-store
  不缓存，这个会让客户端、服务器都不缓存，也就没有所谓的强缓存、协商缓存了
```

协商缓存

```
上面说到的强缓存就是给资源设置个过期时间，客户端每次请求资源时都会看是否过期；只有在过期才会去询问服务器。所以，强缓存就是为了给客户端自给自足用的。而当某天，客户端请求该资源时发现其过期了，这是就会去请求服务器了，而这时候去请求服务器的这过程就可以设置协商缓存。这时候，协商缓存就是需要客户端和服务器两端进行交互的
设置协商缓存:response header里面的设置
  etag: '5c20abbd-e2e8'
  last-modified: Mon, 24 Dec 2018 09:49:49 GMT
也就是说，每次请求返回来 response header 中的 etag和 last-modified，在下次请求时在 request header 就把这两个带上，服务端把你带过来的标识进行对比，然后判断资源是否更改了，如果更改就直接返回新的资源，和更新对应的response header的标识etag、last-modified。如果资源没有变，那就不变etag、last-modified，这时候对客户端来说，每次请求都是要进行协商缓存了，即：
	发请求-->看资源是否过期-->过期-->请求服务器-->服务器对比资源是否真的过期-->没过期-->返回304状态码-->客户端用缓存的老资源
	发请求-->看资源是否过期-->过期-->请求服务器-->服务器对比资源是否真的过期-->过期-->返回200状态码-->客户端如第一次接收该资源一样，记下它的cache-control中的max-age、etag、last-modified等

所以协商缓存步骤总结：

请求资源时，把用户本地该资源的 etag 同时带到服务端，服务端和最新资源做对比。
如果资源没更改，返回304，浏览器读取本地缓存。
如果资源有更改，返回200，返回最新的资源。
```

[彻底弄懂强缓存与协商缓存](https://www.jianshu.com/p/9c95db596df5)

### 26.进程与线程的区别

```
进程是资源分配的最小单位，线程是CPU调度的最小单位
做个简单的比喻：进程=火车
1.线程=车厢线程在进程下行进（单纯的车厢无法运行）
2.一个进程可以包含多个线程（一辆火车可以有多个车厢）
3.不同进程间数据很难共享（一辆火车上的乘客很难换到另外一辆火车，比如站点换乘）
4.同一进程下不同线程间数据很易共享（A车厢换到B车厢很容易）
5.进程要比线程消耗更多的计算机资源（采用多列火车相比多个车厢更耗资源）
6.进程间不会相互影响，一个线程挂掉将导致整个进程挂掉（一列火车不会影响到另外一列火车，但是如果一列火车上中间的一节车厢着火了，将影响到所有车厢）
7.进程可以拓展到多机，进程最多适合多核（不同火车可以开在多个轨道上，同一火车的车厢不能在行进的不同的轨道上）
8.进程使用的内存地址可以上锁，即一个线程使用某些共享内存时，其他线程必须等它结束，才能使用这一块内存。（比如火车上的洗手间）－"互斥锁"
9.进程使用的内存地址可以限定使用量（比如火车上的餐厅，最多只允许多少人进入，如果满了需要在门口等，等有人出来了才能进去）－“信号量”
```

[线程和进程的区别是什么](https://www.zhihu.com/question/25532384)

### 27.JS new的时候发生了什么

```
1、创建一个新对象
2、将构造函数的作用域赋值给新对象（this指向这个新对象）
3、执行构造函数中的代码（为这个新对象添加属性）
4、返回新对象
```

[JS new的时候发生了什么](https://www.jianshu.com/p/9bf81eea03b2)

### 28.DNS是什么

概念

```
DNS 的全称是 Domain Name System 或者 Domain Name Service，它主要的作用就是将人们所熟悉的网址 (域名) “翻译”成电脑可以理解的 IP 地址，这个过程叫做 DNS 域名解析。 打个比方，我们登百度的地址的时候，都是www.baidu.com，进行登陆，难道你会去敲IP地址登百度？明显，域名容易记忆。

大致就是:浏览器输入地址，然后浏览器这个进程去调操作系统某个库里的gethostbyname函数(例如，Linux GNU glibc标准库的gethostbyname函数)，然后呢这个函数通过网卡给DNS服务器发UDP请求，接收结果，然后将结果给返回给浏览器。

(1)我们在用chrome浏览器的时候，其实会先去浏览器的dns缓存里头查询，dns缓存中没有，再去调用gethostbyname函数
(2)gethostbyname函数在试图进行DNS解析之前首先检查域名是否在本地 Hosts 里，如果没找到再去DNS服务器上查
```

拓展问题：浏览器输入URL到返回页面全过程

```
其实回答很简单(俗称天龙八步)
1.根据域名，进行DNS域名解析；
2.拿到解析的IP地址，建立TCP连接；
3.向IP地址，发送HTTP请求；
4.服务器处理请求；
5.返回响应结果；
6.关闭TCP连接；
7.浏览器解析HTML；
8.浏览器布局渲染；
```

[面试官:讲讲DNS的原理](https://zhuanlan.zhihu.com/p/79350395?utm_source=ZHShareTargetIDMore)

### 28.cdn是什么

```
CDN是将源站内容分发至最接近用户的节点，使用户可就近取得所需内容，提高用户访问的响应速度和成功率。解决因分布、带宽、服务器性能带来的访问延迟问题，适用于站点加速、点播、直播等场景。

最简单的CDN网络由一个DNS服务器和几台缓存服务器组成：
1.当用户点击网站页面上的内容URL，经过本地DNS系统解析，DNS系统会最终将域名的解析权交给CNAME指向的CDN专用DNS服务器。
2.CDN的DNS服务器将CDN的全局负载均衡设备IP地址返回用户。
3.用户向CDN的全局负载均衡设备发起内容URL访问请求。
4.CDN全局负载均衡设备根据用户IP地址，以及用户请求的内容URL，选择一台用户所属区域的区域负载均衡设备，告诉用户向这台设备发起请求。
5.区域负载均衡设备会为用户选择一台合适的缓存服务器提供服务，选择的依据包括：根据用户IP地址，判断哪一台服务器距用户最近；根据用户所请求的URL中携带的内容名称，判断哪一台服务器上有用户所需内容；查询各个服务器当前的负载情况，判断哪一台服务器尚有服务能力。基于以上这些条件的综合分析之后，区域负载均衡设备会向全局负载均衡设备返回一台缓存服务器的IP地址。
6.全局负载均衡设备把服务器的IP地址返回给用户。
7.用户向缓存服务器发起请求，缓存服务器响应用户请求，将用户所需内容传送到用户终端。如果这台缓存服务器上并没有用户想要的内容，而区域均衡设备依然将它分配给了用户，那么这台服务器就要向它的上一级缓存服务器请求内容，直至追溯到网站的源服务器将内容拉到本地
```

[闲话 CDN](https://zhuanlan.zhihu.com/p/39028766)

[CDN是什么？使用CDN有什么优势？](https://www.zhihu.com/question/36514327/answer/193768864?utm_source=wechat_session&utm_medium=social&utm_oi=986191292573048832)

### 29.tcp的连接断开吧（三次握手 + 四次握手）

<img src="./img/三次握手.png" style="zoom:80%;" />

```
第一次握手：建立连接时，客户端发送syn包（syn=x）到服务器，并进入SYN_SENT状态，等待服务器确认；SYN：同步序列编号（Synchronize Sequence Numbers）。

第二次握手：服务器收到syn包，必须确认客户的SYN（ack=x+1），同时自己也发送一个SYN包（syn=y），即SYN+ACK包，此时服务器进入SYN_RECV状态；

第三次握手：客户端收到服务器的SYN+ACK包，向服务器发送确认包ACK(ack=y+1），此包发送完毕，客户端和服务器进入ESTABLISHED（TCP连接成功）状态，完成三次握手。

```

<img src="./img/四次挥手.png" style="zoom:80%;" />

```
1）客户端进程发出连接释放报文，并且停止发送数据。释放数据报文首部，FIN=1，其序列号为seq=u（等于前面已经传送过来的数据的最后一个字节的序号加1），此时，客户端进入FIN-WAIT-1（终止等待1）状态。 TCP规定，FIN报文段即使不携带数据，也要消耗一个序号。
2）服务器收到连接释放报文，发出确认报文，ACK=1，ack=u+1，并且带上自己的序列号seq=v，此时，服务端就进入了CLOSE-WAIT（关闭等待）状态。TCP服务器通知高层的应用进程，客户端向服务器的方向就释放了，这时候处于半关闭状态，即客户端已经没有数据要发送了，但是服务器若发送数据，客户端依然要接受。这个状态还要持续一段时间，也就是整个CLOSE-WAIT状态持续的时间。
3）客户端收到服务器的确认请求后，此时，客户端就进入FIN-WAIT-2（终止等待2）状态，等待服务器发送连接释放报文（在这之前还需要接受服务器发送的最后的数据）。
4）服务器将最后的数据发送完毕后，就向客户端发送连接释放报文，FIN=1，ack=u+1，由于在半关闭状态，服务器很可能又发送了一些数据，假定此时的序列号为seq=w，此时，服务器就进入了LAST-ACK（最后确认）状态，等待客户端的确认。
5）客户端收到服务器的连接释放报文后，必须发出确认，ACK=1，ack=w+1，而自己的序列号是seq=u+1，此时，客户端就进入了TIME-WAIT（时间等待）状态。注意此时TCP连接还没有释放，必须经过2∗∗MSL（最长报文段寿命）的时间后，当客户端撤销相应的TCB后，才进入CLOSED状态。
6）服务器只要收到了客户端发出的确认，立即进入CLOSED状态。同样，撤销TCB后，就结束了这次的TCP连接。可以看到，服务器结束TCP连接的时间要比客户端早一些。
```

```
问题1】为什么连接的时候是三次握手，关闭的时候却是四次握手？

答：因为当Server端收到Client端的SYN连接请求报文后，可以直接发送SYN+ACK报文。其中ACK报文是用来应答的，SYN报文是用来同步的。但是关闭连接时，当Server端收到FIN报文时，很可能并不会立即关闭SOCKET，所以只能先回复一个ACK报文，告诉Client端，"你发的FIN报文我收到了"。只有等到我Server端所有的报文都发送完了，我才能发送FIN报文，因此不能一起发送。故需要四步握手。

【问题2】为什么TIME_WAIT状态需要经过2MSL(最大报文段生存时间)才能返回到CLOSE状态？

答：虽然按道理，四个报文都发送完毕，我们可以直接进入CLOSE状态了，但是我们必须假象网络是不可靠的，有可以最后一个ACK丢失。所以TIME_WAIT状态就是用来重发可能丢失的ACK报文。在Client发送出最后的ACK回复，但该ACK可能丢失。Server如果没有收到ACK，将不断重复发送FIN片段。所以Client不能立即关闭，它必须确认Server接收到了该ACK。Client会在发送出ACK之后进入到TIME_WAIT状态。Client会设置一个计时器，等待2MSL的时间。如果在该时间内再次收到FIN，那么Client会重发ACK并再次等待2MSL。所谓的2MSL是两倍的MSL(Maximum Segment Lifetime)。MSL指一个片段在网络中最大的存活时间，2MSL就是一个发送和一个回复所需的最大时间。如果直到2MSL，Client都没有再次收到FIN，那么Client推断ACK已经被成功接收，则结束TCP连接。

【问题3】为什么不能用两次握手进行连接？

答：3次握手完成两个重要的功能，既要双方做好发送数据的准备工作(双方都知道彼此已准备好)，也要允许双方就初始序列号进行协商，这个序列号在握手过程中被发送和确认。

       现在把三次握手改成仅需要两次握手，死锁是可能发生的。作为例子，考虑计算机S和C之间的通信，假定C给S发送一个连接请求分组，S收到了这个分组，并发 送了确认应答分组。按照两次握手的协定，S认为连接已经成功地建立了，可以开始发送数据分组。可是，C在S的应答分组在传输中被丢失的情况下，将不知道S 是否已准备好，不知道S建立什么样的序列号，C甚至怀疑S是否收到自己的连接请求分组。在这种情况下，C认为连接还未建立成功，将忽略S发来的任何数据分 组，只等待连接确认应答分组。而S在发出的分组超时后，重复发送同样的分组。这样就形成了死锁。

【问题4】如果已经建立了连接，但是客户端突然出现故障了怎么办？

TCP还设有一个保活计时器，显然，客户端如果出现故障，服务器不能一直等下去，白白浪费资源。服务器每收到一次客户端的请求后都会重新复位这个计时器，时间通常是设置为2小时，若两小时还没有收到客户端的任何数据，服务器就会发送一个探测报文段，以后每隔75秒钟发送一次。若一连发送10个探测报文仍然没反应，服务器就认为客户端出了故障，接着就关闭连接。
```



[TCP的三次握手与四次挥手理解及面试题（很全面](https://blog.csdn.net/qq_38950316/article/details/81087809)



### flex实现圣杯布局(即上下固定，中间自适应，左右固定，中间自适应)

```
利用flex: 0 100px来设置一个固定大小，flex:1来实现自适应大小。上下固定时将flex-direction设置为column即可
```



### 移动端的项目和pc端的不同

### 不使用setTimeout/setInterval等js的api，实现每隔一秒输出数组的一个元素



