##设计模式

###1.单例模式

​	单例模式的定义是：保证一个类仅有一个实例，并提供一个访问它的全局的访问点。

#### 1.1 实现单例模式

​	根据定义，实现单例模式。

```javascript
var Singleton = function(name) {
  this.name = name;
}
Singleton.getInstance = function(name) {
  if(!this.instance){
    this.instance = new Singleton(name);
  }
  return this.instance;
}
var a = Singleton.getInstance('s1');
var b = Singleton.getInstance('s2');

console.log(a == b);//true
```

​	或者：

```javascript
var Singleton = function(name) {
  this.name = name;
}
Singleton.getInstance = (function() {
  var instance;
  return function(name) {
    if(!instance){
      instance = new Singleton(name);
    }
    return instance;
  }
})();
var a = Singleton.getInstance('s1');
var b = Singleton.getInstance('s2');

console.log(a == b);
```

​	这两种方法是按照定义实现的。但是存在一个问题，不透明性。即使用者必须知道这是一个单例类，然后通过`Singleton.getInstance`获取对象。

#### 1.2 透明的单例类

​	在下面的例子中，我们将使用`CreateDiv`单例类，它的作用是在页面创建唯一的div节点。

```javascript
var CreateDiv = (function() {
  var instance;
  var CreateDiv = function(html) {
    if(instance) {
      return instance;
    }
    this.innerHTML = html;
    this.init;
    return instance = this;
  }
  CreateDiv.prototype.init = function() {
    var div = document.createElement('div');
    div.innerHTML = this.innerHTML;
    document.body.appendChild(div);
  }
  return CreateDiv;
})()
var a = new CreateDiv( 'sven1' );
var b = new CreateDiv( 'sven2' );
console.log( a === b ); // true
```

​	这样我们就完成了一个透明式单例类的编写，它可以通过new构造函数的方式创建唯一对象。

​	但是，在这段代码中，`CreateDiv`实际负责两件事，一是创建和初始化`div`，另一件是保证单例。这是一种不好的做法，与“单一职责原则”不符。

#### 1.3 用代理实现单例模式

​	基于上述问题，我们引入了代理模式。

​	首先实现`CreateDiv`类，它负责创建和初始化`div`。

```javascript
var CreateDiv = function(html) {
  this.innerHtml = html;
  this.init();
}
CreateDiv.prototype.init = function() {
  var div = document.createElement('div');
  div.innerHTML = this.innerHtml;
  document.body.appendChild(div)
}
```

​	接下来引入代理类`ProxySingletonCreateDiv`。

```javascript
var ProxySingletonCreateDiv = (function() {
  var instance;
  return function(html) {
    if(!instance){
      instance = new CreateDiv(html)
    }
    return instance;
  }
})()

var a = new ProxySingletonCreateDiv('s1');
var b = new ProxySingletonCreateDiv('s2');

console.log(a == b);
```

​	我们将负责管理单例模式的逻辑转移到了代理类`ProxySingletonCreateDiv`中。如此，`CreateDiv`仍是一个普通的类，它与`ProxySingletonCreateDiv`结合起来达到了单例模式的效果。

​	本例是缓存代理的应用之一。

#### 1.4 惰性单例

​	惰性单例指的是在有需要的时候才创建对象实例。惰性单例是单例模式的重点。并且非常实用。如百度的登录窗口，一般不需要创建，但是在我们点击的时候才创建，并且永远只有这一个。

```javascript
var CreateLoginLayer = (function() {
  var div;
  return function() {
    if(!div){
      div = document.createElement('div');
      div.innerHTML = '我是登录浮窗';
      div.style.display = 'none';
      document.body.appendChild(div)
    }
    return div;
  }
})();
document.getElementById('loginBtn').onclick = function() {
  var loginLayer = new CreateLoginLayer();
  loginLayer.style.display = 'block';
}
```

####1.5 通用的惰性单例

​	上面的代码还有一些问题。	

​	1. 违反单一职责原则。

​	2.如果我们下次需要创建页面中唯一的`iframe`或者`script`，就得按照这个代码几乎复制一份。

​	我们将如何管理单例模式的代码抽离出来，将创建对象的方法fn当作参数动态传入。

```javascript
var getSingle = function(fn) {
  var result;
  return function() {
    return result || (result = new fn(arguments))
  }
}
var createLoginLayer = function(){
  var div = document.createElement( 'div' );
  div.innerHTML = '我是登录浮窗';
  div.style.display = 'none';
  document.body.appendChild( div );
  return div;
};
var createSingleLoginLayer = getSingle( createLoginLayer );
	document.getElementById( 'loginBtn' ).onclick = function(){
	var loginLayer = createSingleLoginLayer();
	loginLayer.style.display = 'block';
};
```

​	接下来将`CreateLoginLayer`以参数的形式传入，我们不仅可以传入`CreateLoginLayer`,还可以传入`CreateIframe`等。

###2.策略模式

 	策略模式的定义是：定义一系列的算法，把它们一个个封装起来，并且使它们可以相互替换。

​	在程序设计中，要实现某一个功能有多种方案可以选择。比如一个压缩文件的程序，既可以选择 zip算法，也可以选择 gzip算法。
​	这些算法灵活多样，而且可以随意互相替换。这种解决方案就是本章将要介绍的策略模式。

#### 2.1 使用策略模式计算奖金

​	很多公司的年终奖是根据员工的工资基数和年底绩效情况来发放的。例如，绩效为 S的人年终奖有 4倍工资，绩效为 A的人年终奖有 3倍工资，而绩效为 B的人年终奖是 2倍工资。假设财务部要求我们提供一段代码，来方便他们计算员工的年终奖。

​	**1.最初的代码实现**

```javascript
var calculateBonus = function( performanceLevel, salary ){
  if ( performanceLevel === 'S' ){
  	return salary * 4;
  }
  if ( performanceLevel === 'A' ){
  	return salary * 3;
  }
  if ( performanceLevel === 'B' ){
  	return salary * 2;
  }
};
calculateBonus( 'B', 20000 ); // 输出：40000
calculateBonus( 'S', 6000 ); // 输出：24000
```

​	这段代码存在显而易见的缺点。

	1. 函数比较庞大，包含很多if-else语句。
 	2. 缺乏可扩展性，如果增加C等级或者改变等级对应的奖金系数，必须深入函数内部，违反开闭原则。
 	3. 复用性差

	#### 2.2 使用策略模式重构代码

​	策略模式指的是定义一系列的算法，把它们一个个封装起来。将不变的部分和变化的部分隔开是每个设计模式的主题，策略模式也不例外，**策略模式的目的就是将算法的使用与算法的实现分离开来。**

​	**一个基于策略模式的程序至少由两部分组成。第一个部分是一组策略类，策略类封装了具体的算法，并负责具体的计算过程。 第二个部分是环境类 Context，Context接受客户的请求，随后把请求委托给某一个策略类。要做到这点，说明 Context中要维持对某个策略对象的引用。**

​	第一个版本是模仿传统面向对象语言中的实现。我们先把每种绩效的计算规则都封装在对应的策略类里面：

```javascript
var personformaceS = function(){}
personformaceS.prototype.calculate = function(salary) {
  return salary * 4;
}
var personformaceA = function(){}
personformaceA.prototype.calculate = function(salary) {
  return salary * 3;
}
var personformaceB = function(){}
personformaceB.prototype.calculate = function(salary) {
  return salary * 2;
}
```

​	接下来定义奖金类：

```javascript
function Bonus() {
  this.salary = 0;
  this.strategy = null;
}
Bonus.prototype.setSalary = function(salary) {
  this.salary = salary;
}
Bonus.prototype.setStrategy = function(strategy) {
  this.strategy = strategy;
}
Bonus.prototype.getBonus = function() {
  return this.strategy.calculate(this.salary);
}
```

​	回顾一下策略模式的思想：

​	定义一系列的算法，把它们一个个封装起来，并且使它们可以相互替换。

​	详细一点：定义一系列的算法，并把它们各自封装成策略类，算法被封装在策略类中，在用户对Context请求的时候，Context总是把请求委托给策略对象中的某一个进行计算。

```javascript
var myBonus = new Bonus;
myBonus.setSalary(1000);
myBonus.setStrategy(new performanceS());
myBonus.getBonus() // 4000
```

#### 2.3 Javascript版本的策略模式

​	策略模式由上一节可知分为策略类，负责算法的实现，另一类是Context类，负责算法的使用。在`javascript`中,可以直接用对象来包装这些算法。

```javascript
var strategies ={
  S:function(salary) {
    return salary *4;
  },
  A:function(salary) {
    return salary *3;
  },
  B:function(salary) {
    return salary *2;
  } 
}
```

​	同样，我们可以不必用Bonus类来模仿用户请求操作，直接用一个函数来充当。

```javascript
function calculateBonus(level, salary) {
  return strategies[level](salary);
}
```

#### 2.3 更广义的“算法”

​	策略模式的定义是封装一系列的算法，把它们封装起来，并且可以互相替换。

​	从定义上来看，策略模式就是用来封装算法的。在实际开发中，我们可以把算法的含义扩散开来，使策略模式亦可以封装一系列的“业务规则”，只要这些业务规则的目标一致，并且可以互相替换，就可以应用策略模式。

#### 2.3 表单校验

​	假设我们正在编写一个注册的页面，在点击注册按钮之前，有如下几条校验逻辑。
 用户名不能为空。
 密码长度不能少于 6位。
 手机号码必须符合格式。

##### 2.3.1 表单校验的第一个版本

不引入策略模式。

```html
<body>
  <form action="XXX" method="post" id="registerForm">
    请输入用户名：<input type='text' name='username' />
    请输入密码：<input type='text' name='password' />
    请输入手机：<input type='text' name='photo' />
  </form>
</body>
```

```javascript
var registerForm = document.getElementById('registerForm');
registerForm.onsubmit = function() {
  if(register.username.value === ""){
    alert('用户名不能为空!');
    return false;
  }
  if(register.password.value.length < 6){
    alert('密码不能少于6位!');
    return false;
  }
  if(!(/^1[3578][0-9]{9}$/.test(registerForm.photo.value))){
    alert('手机号码必须符合格式!');
    return false;
  }
}
```

缺点与未使用策略模式计算奖金相同。

##### 2.3.2 使用策略模式

```javascript
var strategies = {
	isNotEmpty: function(value, errorMsg) {
    if(value === ''){
      return errorMsg;
    }
  },
  minLength: function(value, minLength, errorMsg) {
    if(value.length < minLength) {
      return errorMsg;
    }
  },
  isMobile: function(value, errorMsg) {
    if(!(/^1[3578][0-9]{9}$/.test(value))){
      return errorMsg;
    }
  }
}

function Validator () {
  this.errorMsgs = [];
}

Validator.prototype.add = function (type,...rest) {
  if(strategies[type](...rest)){
    this.errorMsgs.push(strategies[type](...rest))
  }
}

Validator.prototype.getErrorMsgs = function () {
  return this.errorMsgs;
}

registerForm.onsubmit = function () {
  var validateForm = new Validator;
  validateForm.add('isNotEmpty', registerForm.username.value,'用户名不能为空!');
  validateForm.add('minLength', registerForm.password.value, 6, '密码不能少于6位!');
  validateForm.add('isMobile', registerForm.photo.value,'号码格式不正确!');
  errorMsg = validateForm.getErrorMsgs()
  if(errorMsg.length > 0) {
    errorMsgs.forEach(err => {
      console.log(err)
    })
    return false;
  }
}
```

#### 2.4 策略模式的优缺点

优点：

+ 利用组合，委托，多态的思想，避免多重条件语句
+ 支持开放-封闭原则，将算法封装在strategy对象中，易于扩展。
+ 可复用

缺点：

+ 增加策略类或者策略对象，但是也比逻辑堆砌在Context中好
+ 要使用策略类，必须了解所有的strategy，所有的strategy的不同点。

#### 2.5 一等函数对象和策略模式

​	实际上在 JavaScript这种将函数作为一等对象的语言里，策略模式已经融入到了语言本身
当中，我们经常用高阶函数来封装不同的行为，并且把它传递到另一个函数中。当我们对这些函数发出“调用”的消息时，不同的函数会返回不同的执行结果。在 JavaScript中，“函数对象的多态性”来得更加简单。
​	在前面的学习中，为了清楚地表示这是一个策略模式，我们特意使用了 strategies 这个名字。如果去掉 strategies ，我们还能认出这是一个策略模式的实现吗？代码如下：

```javascript
var S = function( salary ){
return salary * 4;
};
var A = function( salary ){
return salary * 3;
};
var B = function( salary ){
return salary * 2;
};
var calculateBonus = function( func, salary ){
return func( salary );
};
calculateBonus( S, 10000 ); // 输出：40000
```

​	在 JavaScript语言的策略模式中，策略类往往被函数所代替，这时策略模式就
成为一种“隐形”的模式。

### 3.代理模式

​	代理模式的定义是：为一个对象提供一个代替品或占位符，以便控制对它的访问。

​	代理模式的关键是，当用户不便直接访问或者不满足需要访问一个对象时，提供一个替代对象来控制对对象的访问。用户实际访问的是替代对象，替代对象对请求做出一些处理后，再由替代对象去访问本体对象。

#### 3.1 代理模式初探

​	窈窕淑女，君子好逑。

​	小明看上了A，想通过送花的方式跟A表白。

​	假设A心情好的时候小明表白通过的几率较高，心情差的时候必定被拒绝。B了解A，知道A什么时候心情好。

```javascript
var Flower = function () {}
var xiaoming = {
	sendFlower:function (target) {
    var flower = new Flower();
    target.receivedFlower(flower);
  }
}

var B = {
  reveivedFlower:function (flower) {
    A.listenGoodMood(function(){
      A.receivedFlower(flower);
    })
  }
}

var A = {
  listenGoodMood(fn) {
    setTimeout(() => { fn() }, 1000);
  },
  receivedFlower(flower) {
    console.log('我收到花了' + flower);
  }
}

xiaoming.sendFlower()
```

#### 3.2 保护代理和虚拟代理

​	**保护代理**：代理B可以帮助A过滤掉一些请求，对不符合要求的请求过滤，直接在B这里拒绝，这种代理叫保护代理。

​	B可以帮A过滤掉一些没有宝马或者年龄太大的，A和B一个充当白脸，一个黑脸，A继续保持良好的女神形象，由B来拒绝，控制对A的访问。

​	**虚拟代理**：把一些开销很大的操作，延迟到真正需要的时候才去执行叫虚拟代理。

​	假设现实中的花价格不菲，导致在程序世界里， `new Flower` 也是一个代价昂贵的操作，那么我们可以把 `new Flower` 的操作交给代理 B 去执行，代理 B 会选择在 A 心情好时再执行 `new Flower`。

#### 3.3 虚拟代理实现图片预加载

```javascript
var myImg = (function() {
	var myImg = new Image();
	document.body.appendChild('myImg');
	return {
		setSrc: function(src) {
			myImg.src = src;
		}
	}
})()

var proxyImage = (function() {
  var proxyImg = new Image();
  proxyImg.onload = function(src){
    myImg.setSrc(this.src);
  }
  
  return {
    setSrc: function (src) { //先给实际图片节点设置一个本地图片，让proxyImg去加载图片，加载完成，触发onload,把src赋给实际图片。
      myImg.setSrc('loading.gif');
     	proxyImg.src = src;
    }
  }
})()

proxyImg.setSrc('logo.png')
```

####3.4 代理的意义

​	如果不使用代理模式，我们也可以完成一个图片预加载功能。

```javascript
var myImg = (funciton() {
	var imgNode = new Image();
	document.body.appendChild(imgNode);

	var img = new Image();
	img.onload = function () {
    imgNode.src = this.src;
  }
	return {
    setSrc: function (src) {
      imgNode.src = 'loading.gif';
      img.src = src;
    }
  }
})()

myImg.set('logo.png');
```



​	为了说明代理的意义，下面我们引入一个面向对象设计的原则——单一职责原则。

​	单一职责原则指的是，就一个类（通常也包括对象和函数等）而言，应该仅有一个引起它变化的原因。如果一个对象承担了多项职责，就意味着这个对象将变得巨大，引起它变化的原因可能会有多个。面向对象设计鼓励将行为分布到细粒度的对象之中，如果一个对象承担的职责过多，等于把这些职责耦合到了一起，这种耦合会导致脆弱和低内聚的设计。当变化发生时，设计可能会遭到意外的破坏。

​	职责被定义为“引起变化的原因”。上段代码中的 `MyImage `对象除了负责给 `img `节点设置 `src`外，还要负责预加载图片。我们在处理其中一个职责时，有可能因为其强耦合性影响另外一个职责的实现。

​	另外，在面向对象的程序设计中，大多数情况下，若违反其他任何原则，同时将违反开放 —封闭原则。如果我们只是从网络上获取一些体积很小的图片，或者 5年后的网速快到根本不再需要预加载，我们可能希望把预加载片的这段代码从 `MyImage `对象里删掉。这时候就不得不改动`MyImage `对象了。

​	实际上，我们需要的只是给 `img `节点设置 `src `，预加载图片只是一个锦上添花的功能。如果能把这个操作放另一个对象里面，自然是一个非常好的方法。于是代理的作用在这里就体现出来了，代理负责预加载图片，预加载的操作完成之后，把请求重新交给本体 `MyImage `。

​	纵观整个程序，我们并没有改变或者增加 `MyImage `的接口，但是通过代理对象，实际上给系统添加了新的行为。这是符合开放 — 封闭原则的。给 `img `节点设置 `src `和图片预加载这两个功能，被隔离在两个对象里，它们可以各自变化而不影响对方。何况就算有一天我们不再需要预加载，那么只需要改成请求本体而不是请求代理对象即可。

#### 3.4 代理和本体接口的一致性

​	上节提到当我们有一天不需要预加载时，只需要改成请求本体即可。其中的关键是代理对象和本体都实现了`setSrc`方法，在用户看来，代理对象和本体是一致的，用户并不知道代理和本体的区别。这样做的好处：

+ 用户可以放心的请求代理，他只关心是否能得到结果。
+ 在任何使用本体的地方都能替换成代理对象。

​	在`JavaScript`中，我们有时通过鸭子类型检测本体和对象都实现了相同的接口。（凡是都能嘎嘎叫的都是鸭子，凡是具有某一特定功能都是同一类型）。

​	另外值得一提的是，如果代理对象和本体对象都为一个函数（函数也是对象），函数必然都能被执行，则可以认为它们也具有一致的“接口”。（本体和代理都是函数直接执行视为具有相同的接口，就是执行代理函数也能得到本体对象的结果。在上文都是执行本体和代理对象的相同方法。）

#### 3.5 虚拟代理合并http请求

​	假设我们在做一个文件同步的功能，选中`checkbox`的时候文件就会被上传到后台服务器。

![image-20191209095127005](C:\Users\a02021\AppData\Roaming\Typora\typora-user-images\image-20191209095127005.png)

##### 3.5.1 上传文件第一版

```html
<div>
  <input type="checkbox" id="1" />
  <input type="checkbox" id="2" />
  <input type="checkbox" id="3" />
  <input type="checkbox" id="4" />
  <input type="checkbox" id="5" />
  <input type="checkbox" id="6" />
  <input type="checkbox" id="7" />
</div>
```

```javascript
function uploadFiles (id) {
  console.log('文件已被上传。')
}

var checkboxs = document.getElementsByTagName('input');
for(var item in checkboxs) {
  item.onclick = () => {
    if(this.checked = "true"){
      uploadFiles(this.id);
    }
  }
}
```

##### 3.5.2 使用虚拟代理

```javascript
function uploadFiles (id) {
  console.log('文件已被上传。')
}

var checkboxs = document.getElementsByTagName('input');
for(var item in checkboxs) {
  item.onclick = () => {
    if(this.checked = "true"){
      proxyUploadFiles(this.id); //请求只需更换成代理接口
    }
  }
}

function proxyUploadFiles(id) {
  var cache = [];
  var timer;
  return function(id) {
    cache.push(id);
    
    if(timer) {
      return;
    }
    
    var timer = setTimeout(() => {
      uploadFiles(cache.join(','));
      clearTimer(timer);
      timer = null;
      cache.length = 0;
    }, 2000)
  }
} 
```

​	本例做到了收集两秒内的请求，然后统一发送。第一个请求到来，`push`进数组，`timer`被赋值，此后一直存在，后续请求`push`进数组，但因为存在`timer`返回，2s之后统一发送请求，`timer`和`cache`被置空，重复该过程。

#### 3.6 虚拟代理在惰性加载中的应用

​	有个可以帮助开发者新建一个控制台，输出打印日志的脚本`miniConsole.js`，调用`miniConsole.log(1)`，这句话会在页面创建一个`div`，然后把`log`显示在`div`中。

​	并不是每个用户都需要加载这个脚本。只有当用户按下`F2`才会加载这个脚本，并且唤出控制台。

​	在`miniConsole.js`被加载之前，为了让用户正常使用里面的`API`（即使我们不打开控制台，我们的脚本一样调用了`miniConsole.log`来打印日志，只是这时还不存在`miniConsole.js`这个脚本。暂未真正执行打印的这个过程，但是我们需要把这个操作先存储起来，当用户点击`F2`，加载脚本，打开控制台，就真正去执行这些操作。）解决方案是用一个占位的`miniConsole`代理对象来给用户提前使用，它对外暴露了一样的接口。

​	打印`log`的请求我们包裹在一个函数中，这个包裹了请求的函数相当于命令模式中的`Command`对象，随后将这些函数都放在缓存队列中，这些逻辑都是在代理对象中完成的，当按下F2，脚本被加载，加载完成之后遍历代理对象中的缓存队列，并且依次执行。

​	整理一下代理对象的代码，使它成为一个标准的虚拟代理对象。

```javascript
var miniConsole = (function () {
  var cache = [];
  var handler = function (ev) {
    if (ev.keyCode === 113) {
      var script = document.createElement('script');
      script.onload = function () {
        for (var i = 0, fn; fn = cache[i++];) {
          fn();
        }
      };
      script.src = 'miniConsole.js';
      document.getElementsByTagName('head')[0].appendChild(script);
      document.body.removeEventListener('keydown', handler);// 只加载一次 miniConsole.js
    }
  };
  document.body.addEventListener('keydown', handler, false);
  return {
    log: function () {
      var args = arguments;
      cache.push(function () {
        return miniConsole.log.apply(miniConsole, args);
      });
    }
  }
})();
miniConsole.log(11); // 开始打印 log
// miniConsole.js 代码
miniConsole = {
  log: function () {
    // 真正代码略
    console.log(Array.prototype.join.call(arguments));
  }
};
```

#### 3.7 缓存代理

​	缓存代理可以为一些开销大的运算结果提供暂时的存储，下次运算时，如果传入的参数与之前的一致，则直接返回之前的计算结果。

#####3.7.1 缓存代理用于计算乘积

```javascript
var mult = function() {
  var a = 1;
  for(var i in arguments){
    a = a * i;
  }
  return a;
} 
```

​	假设这是很复杂的计算。现在加入缓存代理函数。

```javascript
var proxyMult = function() {
  var cache = {};
  return function() {
    for(var args in cache) {
      if(Array.prototype.join.call(arguments)===args){
        return cache[args];
      }
      var result = mult.apply(this, arguments);
      cache[Array.prototype.join.call(arguments)] = result;
      return result;
    }
  }
}
```

通过增加缓存代理的方式，`mult`函数可以继续专注于本身的计算，缓存的功能由代理对象实现。

#####3.7.2 缓存代理用于ajax异步请求数据

​	我们在常常在项目中遇到分页的需求，同一页的数据理论上只需要去后台拉取一次，这些已经拉取到的数据在某个地方被缓存后，下次再请求同一页的时候，便可以直接使用之前的数据。
​	显然这里也可以引入缓存代理，实现方式跟计算乘积的例子差不多，唯一不同的是，请求数据是个异步的操作，我们无法直接把计算结果放到代理对象的缓存中，而是要通过回调的方式。

####3.8 用高阶函数动态创建缓存代理

​	将计算方法传入一个专门用于创建缓存代理的工厂中，就可以为各种计算方法创建缓存代理。

```javascript
var createProxyFactory = function( fn ){
  var cache = {};
  return function(){
    var args = Array.prototype.join.call( arguments, ',' );
    if ( args in cache ){
    	return cache[ args ];
    }
  	return cache[ args ] = fn.apply( this, arguments );
  }
};
var proxyMult = createProxyFactory( mult ),
```

#### 3.9 其他代理模式

​	防火墙代理：控制网络资源的访问，保护主题不让“坏人”接近。
​	远程代理：为一个对象在不同的地址空间提供局部代表，在 Java 中，远程代理可以是另
一个虚拟机中的对象。

### 4.迭代器模式

​	迭代器模式是指提供一种方法顺序访问一个聚合对象的各个元素，而又不暴露该对象的内部表示。迭代器模式可以把迭代的过程从业务逻辑中分离出来，即不关系对象的内部构造，也可以按顺序访问其中的元素。

###5.ES6中的遍历器

####5.1 遍历器的概念

​	遍历器是一种接口，为不同的数据结构提供统一的访问机制。任何数据只要部署`Iterator`接口，即可完成遍历操作。

​	`Iterator`的作用有三个：

+ 为各种数据结构，提供统一的，简便的访问操作。
+ 使数据结构的成员能够按某种次序排列。
+ 供for...of消费

​	`Iterator`的遍历过程：

+ 创建一个指针对象，指向数据结构的起始位置，也就是说，遍历器对象其实是一个指针对象。
+ 第一次调用指针对象的next方法，指针指向数据结构的第一位成员。
+ 第二次调用指向第二位成员。
+ 不断调用，直至指向数据成员的结束位置。

​	每一次调用`next`方法，都会返回当前成员的信息。即一个包括`value`和`done`属性的对象。

```javascript
function myIterator (array) {
  var curIndex = 0;
  return {
    next: function() {
      let done = !(curIndex < array.length);
      return done?{
        value:undefined,done:done
      }:{value:array[curIndex++],done:done}
    }
  }
}
var i = myIterator([1,2,3]);
console.log(i.next());
console.log(i.next());
console.log(i.next());
console.log(i.next());
//{ value: 1, done: false }
//{ value: 2, done: false }
//{ value: 3, done: false }
//{ value: undefined, done: true }
```

#### 5.2 默认`Iterator`接口

​	默认的`Iterator`接口部署在数据结构的`Symbol.iterator`属性。该属性是一个函数，执行该函数返回一个遍历器，调用遍历器的`next`方法，即可返回数据。

​	原生具备`iterator`接口的数据结构。

		+ Array
		+ Map
		+ Set
		+ String
		+ TypedArray
		+ 函数的arguments对象
		+ NodeList对象

```javascript
Object.prototype[Symbol.iterator] = function () {
  const keys = Object.keys(this); //该方法返回自身的属性
  const self = this;
  const len = keys.length;
  let curIndex = 0;
  return {
    next:function(){
      return curIndex < len ? { value: [keys[curIndex], self[keys[curIndex++]]], done: false } : { value: undefined, done: true }
    }
  }
}

var obj = {
  name: 'xy',
  sex: 'male',
  age: 12
}

for (var [prop,value] of obj) {
  console.log(prop, value)
}

//name xy
//sex male
//age 12
```

​	类数组（键名为数值和具有length属性）可以直接借用数组的`Symbol.iterator`方法。

```javascript
let iterable = {
0: 'a',
1: 'b',
2: 'c',
length: 3,
[Symbol.iterator]: Array.prototype[Symbol.iterator]
};
for (let item of iterable) {
console.log(item); // 'a', 'b', 'c'
}
```

####5.3 调用`iterator`接口的场合

​	除了`for...of`会默认调用`Iterator`接口，还有一些场合会调用。

1. 解构赋值

   对数组和`Set`结构进行解构赋值时，会默认调用`Symbol.iterator`接口。

   ```javascript
   [a,...rest] = [1,2,3]
   //a=1,rest=[2,3]
   ```

   

2. 扩展运算符

3. yield*

   yield* 后面跟的是一个可遍历的结构，它会调用该结构的遍历器接口。

   ```javascript
   let generator = function* () {
   yield 1;
   yield* [2,3,4];
   yield 5;
   };
   var iterator = generator();
   iterator.next() // { value: 1, done: false }
   iterator.next() // { value: 2, done: false }
   iterator.next() // { value: 3, done: false }
   iterator.next() // { value: 4, done: false }
   iterator.next() // { value: 5, done: false }
   iterator.next() // { value: undefined, done: true }
   ```

4. 其他场合

​	由于数组的遍历会调用遍历器接口，所以任何接受数组作为参数的场合，其实都调用了遍历器接口。下面是一些例子。

+ `for...of`
+ `Array.from()`
+ `Map(), Set(), WeakMap(), WeakSet()（比如 new Map([['a',1]['b',2]]) ）`
+ `Promise.all()`
+ `Promise.race()`

### 6.发布订阅模式

​	发布订阅模式定义了对象间一种一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都将得到通知。

#### 6.1 发布订阅模式的通用实现

```javascript
var pubSub = {
  events: {},
  listen:function(type,fn){
    if(!this.events[type]){
      this.events[type] = [];
    }
    this.events[type].push(fn);
  },
  trigger:function() {
    var type = Array.prototype.shift.call(arguments);
    if(!this.events.type||!this.events.type.length === 0){
      return;
    }
    for(var fn of this.events[type]){
      fn.apply(this, arguments)
    }
  },
  remove:function(type, fn) {
    var fns = this.events[type];
    if(!fns){
      return false;
    }
    for(var i= 0;i<fns.length;i++){
      if(fn == fns[i]){
        this.events[type].splice(i,1)
      }
    }
  }
}
```

#### 6.2 必须先订阅再发布吗

	我们所了解到的发布 — 订阅模式，都是订阅者必须先订阅一个消息，随后才能接收到发布者
​	发布的消息。如果把顺序反过来，发布者先发布一条消息，而在此之前并没有对象来订阅它，这条消息无疑将消失在宇宙中。

​	在某些情况下，我们需要先将这条消息保存下来，等到有对象来订阅它的时候，再重新把消息发布给订阅者。就如同 QQ中的离线消息一样，离线消息被保存在服务器中，接收人下次登录上线之后，可以重新收到这条消息。
这种需求在实际项目中是存在的，比如在之前的商城网站中，获取到用户信息之后才能渲染用户导航模块，而获取用户信息的操作是一个 ajax 异步请求。当 ajax 请求成功返回之后会发布一个事件，在此之前订阅了此事件的用户导航模块可以接收到这些用户信息。但是这只是理想的状况，因为异步的原因，我们不能保证 ajax请求返回的时间，有时候它返回得比较快，而此时用户导航模块的代码还没有加载好（还没有订阅相应事件），特别是在用了
一些模块化惰性加载的技术后，这是很可能发生的事情。也许我们还需要一个方案，使得我们的发布 — 订阅对象拥有先发布后订阅的能力。

​	为了满足这个需求，我们要建立一个存放离线事件的堆栈，当事件发布的时候，如果此时还没有订阅者来订阅这个事件，我们暂时把发布事件的动作包裹在一个函数里，这些包装函数将被存入堆栈中，等到终于有对象来订阅此事件的时候，我们将遍历堆栈并且依次执行这些包装函数，也就是重新发布里面的事件。当然离线事件的生命周期只有一次，就像 QQ的未读消息只会被重新阅读一次，所以刚才的操作我们只能进行一次。

#### 6.3 发布模式的优缺点

​	发布 — 订阅模式的优点非常明显，一为时间上的解耦，二为对象之间的解耦。它的应用非常广泛，既可以用在异步编程中，也可以帮助我们完成更松耦合的代码编写。

当然，发布 — 订阅模式也不是完全没有缺点。创建订阅者本身要消耗一定的时间和内存，而且当你订阅一个消息后，也许此消息最后都未发生，但这个订阅者会始终存在于内存中。另外，发布 — 订阅模式虽然可以弱化对象之间的联系，但如果过度使用的话，对象和对象之间的必要联系也将被深埋在背后，会导致程序难以跟踪维护和理解。特别是有多个发布者和订阅者嵌套到一起的时候，要跟踪一个 bug不是件轻松的事情。

#### 6.4 发布订阅模式和观察者模式的区别

​	发布订阅模式更像一个中介，发布者和订阅者并无直接联系，你不知道我，我也不知道你。订阅者像第三方中介订阅信息，发布者通过第三方中介发布消息，发布订阅模式就是这个中介。

​	观察者模式则是两个对象的直接联系，B订阅A，A发布消息，直接通知B。

​	发布订阅与观察者的区别https://www.cnblogs.com/viaiu/p/9939301.html

