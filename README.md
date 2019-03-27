## learnMVVM

### 什么是 mvvm

```
graph TB
A[M:模型]-->B[VM:视图模型]
B-->C[V:视图]
B-->A
C-->B
```

MVVM 拆开来即为 Model-View-ViewModel，有 View，ViewModel，Model 三部分组成。它是一种基于前端开发的`架构模式`

- View 层代表的是视图、模版，负责将数据模型转化为 UI 展现出来。
- Model 层代表的是模型、数据，可以在 Model 层中定义数据修改和操作的业务逻辑。
- ViewModel 层连接 Model 和 View。

在 MVVM 的架构下，View 层和 Model 层并没有直接联系，而是通过 ViewModel 层进行交互。ViewModel 层通过双向数据绑定将 View 层和 Model 层连接了起来，使得 View 层和 Model 层的同步工作完全是自动的。因此开发者只需关注业务逻辑，无需手动操作 DOM，复杂的数据状态维护交给 MVVM 统一来管理。

### 什么是数据双向绑定

```
graph LR
A[数据]-->|1.数据变化引起视图变化|B[视图]
B-->|2.视图变化引起数据变化|A
```

而我所理解的双向数据绑定无非就是在单向绑定的基础上给可输入元素（input、textare 等）添加了 change(input)事件，来动态修改 model 和 view，并没有多高深。所以无需太过介怀是实现的单向或双向绑定

### vue.js 和 mvvm 的关系

Vue.js 是一个提供了 MVVM 风格的双向数据绑定的 Javascript 库，专注于 View 层。它的核心是 MVVM 中的 VM，也就是 ViewModel。 ViewModel 负责连接 View 和 Model，保证视图和数据的一致性，这种轻量级的架构让前端开发更加高效、便捷。

许多流行的客户端 JavaScript 框架例如 vue.js,Ember.js,AngularJS 以及 KnockoutJS 都将双向数据绑定作为自己的头号特性。

下图是 vue 实现 mvvm 的思路：

```
graph TB
A[M:模型]-->|data binding|B[VM:视图模型]
B-->|data binding|C[V:视图]
B-->|dom listener|A
C-->|dom listener|B
```

具体来说，这三个方面分别对应如下：

```
// view
<div id="app">
    <input v-model="salary"/>
    <p>{{salary}}</p>
</div>
```

```
// model
let data{
    salary:1000
}
```

```
// viewmodel
let vm = new Vue({
    el:"#app",
    data:data
})
```

前面的理论到此为止，下面我们开始进入实践的环节。
先来看一个小 demo。

[demo](!./demo-mvvm.gif)

```
 <div id="app">
    工资:<input type="text" v-model="salary" /><span>{{ salary }}</span>
    <br />
    奖金:<input type="text" v-model="bonus" />
    <br />
    <span>到手:{{ cTotalMoney }}</span>
    <button @click="hDobule">工资加倍</button>
 </div>
```

````
<script src="https://cdn.bootcss.com/vue/2.6.10/vue.js"></script>
<script>
var vm = new Vue({
    el: "#app",
    data: {
        salary: 10000,
        bonus: 50000
    },
    computed: {
        cTotalMoney() {
        return this.salary + this.bonus;
        }
    },
    methods: {
        hDobule() {
            this.salary *= 2;
        }
    }
});
</script>
```

这是一个比较简单的vue的使用，涉及：
- v-model
- v-bind
- @click
- 计算属性
- vm.salary也能访问，并修改数据

本文的目的就是零开始，一步一步编写代码，来实现这个功能。具体来说，就是实现一个MVVM类，就是把上面的 `new Vue` 改成`new MVVM`,然后，还能让代码正常的功能。



### 实现一个MVVM的基本要素

观察demo代码，先思考几个问题：

1. 【响应式】数据变化了，视图是如何知道的，通知视图的？
2. 【模板引擎】v-model,v-bind是什么？标准的html中是没有这些标签的，它们如何与data中的属性关联起来？
3. 【渲染】vue的模板如何被渲染成html?

- 实现一个数据监听器Observer，能够对数据对象的所有属性进行监听，如有变动可拿到最新值并通知订阅者
- 实现一个指令解析器Compile，对每个元素节点的指令进行扫描和解析，根据指令模板替换数据，以及绑定相应的更新函数
- 实现一个Watcher，作为连接Observer和Compile的桥梁，能够订阅并收到每个属性变动的通知，执行指令绑定的相应回调函数，从而更新视图




1.我们需要一个方法来识别哪个UI元素被绑定了相应的属性
2.我们需要监视属性和UI元素的变化
3.我们需要将所有变化传播到绑定的对象和元素

虽然实现的方法有很多，但是最简单也是最有效的途径是使用发布者-订阅者模式。思想很简单：我们可以使用自定义的data属性在HTML代码中指明绑定。所有绑定起来的JavaScript对象以及DOM元素都将“订阅”一个发布者对象。任何时候如果JavaScript对象或者一个HTML输入字段被侦测到发生了变化，我们将代理事件到发布者-订阅者模式，这会反过来将变化广播并传播到所有绑定的对象和元素。

下面，先来解决第一个问题。
### 响应式

对于如下简单对象，
```
var data =  {
        salary: 10000,
        bonus: 50000
    }
```
当我们去修改属性值时，如何放出消息来告诉外界，某个值被修改了？要实现这个功能，我们可以借助`Object.defineProperty`这个api。它的作用是“精确添加或修改对象的属性”。

下面，我们来看一个简单的例子，

##### defineProperty 基本示例

````

var obj = {}
var \_salary = 10000
Object.defineProperty(obj,"salary",{
get:function(){
console.info("get.....")
return \_salary
},
set:function(newVal){
console.info("set.....")
\_salary = newVal
}
})

obj.salary = 2000; //
obj.salary; //

```
如上代码所示，当我们去修改或者访问salary的值时，就会在控制台中看到对应的输出信息了。如果把上面的`console.info`改成调用函数，或者发出事件等等，就可以达到我们上文

对于这个函数的完整的用法，可以参考如下地址：
[Object.defineProperty](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)
当然，它也是有浏览器的兼容性问题的。如下：
[caniuse](https://caniuse.com/#search=defineProperty) 。具体来说，对于ie9之下的浏览器，它并不支持,这也是vue不支持ie8的原因。

好了，上面的例子介绍了如何`监听`一个属性，那怎么来批量监听一个对象的全部属性呢？ 下面我们来封装一个函数来实现这个功能。

##### 封装函数

```

function Observe(obj) {
let objKeys = Object.keys(obj);
objKeys.forEach(key=>{
var val = obj[key]
Object.defineProperty(obj,key,{
enumerable:true,
set:function(newVal){
console.info( `${obj[key]}----->${newVal}`);
val = newVal;
},
get:function(){
console.info(`get....${key}`)
return val;
}
})
})
}

var data = {salary:10000,bonus:30000}
Observe(data);

data.salary = 20000; // 更新属性值
data.salary;

```

到此为止，对于对象的所有属性，我们都加上`监听者`,这个步骤称之为`数据劫持`. 其实，上面的Object.defineProperty就是vuejs实现的核心原理。你现在对比一下，通过控制台，打印vue实例可以看到到处可见的:`set`,`get`。

vue的核心原理就是这个api`Object.defineProperty()` ，看起来很简单是吧。

本文到此，基础准备工作也就告一段落了。下面的会涉及到 js中的构造器,原型链,等高级一点的内容，坐好车，跟我走吧。


#### MVVM构造器

动手写代码的一步：创建一个名为MVVM的构造器，它接受一个参数，具体的格式就是我们的目标代码中的options。 现在要做的第一步就是对options.data这个对象中的每个属性进行“监听”，就是我们上文提到的数据劫持。

下面来看看代码。

```

function MVVM(options) {
this.$data = options.data;
    Observe(this.$data) // 直接调用前面的函数
}

var vm = new MVVM({
data:{
salary: 10000,
bonus: 50000
}
})

```

#### 把Observe改成MVVM的原型方法
```

function MVVM(options) {
this.$data = options.data;
    this.observe(this.$data) // 直接调用前面的函数
}
MVVM.prototype.observe = function (obj) {
let objKeys = Object.keys(obj);
objKeys.forEach(key=>{
var val = obj[key]
Object.defineProperty(obj,key,{
enumerable:true,
set:function(newVal){
console.info( `${obj[key]}----->${newVal}`);
val = newVal;
},
get:function(){
console.info(`get....${key}`)
return val;
}
})
})
}

var vm = new MVVM({

data:{
salary: 10000,
bonus: 50000
}
})

```
现在的效果，与前面的效果其实是一样的，只不过把代码挪动到了MVVM里面而已。

现在还有一个小问题:我们访问salary时，是通过vm.$data.salary来操作的，这样还是不方便，下面我们把对vm.$data的操作代理到vm上。


### 把vm.$data的属性变化，代理到vm上

改动只有一个地方：Object.defineProperty的第一个参数，从obj改成this。

```

MVVM.prototype.observe = function (obj) {

    let objKeys = Object.keys(obj);
    objKeys.forEach(key=>{

        var val = obj[key]
        Object.defineProperty(this,key,{
            enumerable:true,
            set:function(newVal){
                console.info( `${obj[key]}----->${newVal}`);
                val = newVal;
            },

            get:function(){
                console.info(`get....${key}`)
                return val;
            }
        })
    })

}

```


好了，到此为止，响应式的效果基本实现了。但有一点要说明的是它并不支持多级嵌套的对象，当然，拓展到多级对象并不复杂，这个工作，我们先放一放，把注意力集中在MVVM上。

接下来，我们检查一下任务清单，下一步要做的就是`编译`模板:把v-model,v-bind等在html5本不存在的属性（在vue中它们被称之为指令）检测出来，并与data对象中的属性关联起来。


### 编译模板

在options中，我们传入一个el值，它指向页面上的一个dom节点，我们要做的事情就是分析这个dom节点下面的所有的子节点，检查一下节点中是否有这些v-model，v-bing等属性。

具体来说，给MVVM上添加一个原型方法:compile，如下：

```

function MVVM(options) {

    this.$data = options.data;
    this.$el = document.querySelector(options.el); //
    this.observe(this.$data)
    this.compile()

}
MVVM.prototype.compile = function() {

    console.info(".....compile......");
    let el = this.\$el;
    let vm = this;

    Array.from(el.children).forEach(node => {
        if (node.hasAttribute("v-model")) {
            let exp = node.getAttribute("v-model");
            console.info("v-model", exp);

        } else if (node.hasAttribute("v-bind")) {
            let exp = node.getAttribute("v-bind");
            console.info("v-bind", exp);


        } else if (node.hasAttribute("@click")) {
            let exp = node.getAttribute("@click");
            console.info("@click", exp);

        }
    });

};

var vm = new MVVM({

data:{
salary: 10000,
bonus: 50000
}
})

```
现在把代码跑起来，就可以看到编译之后的效果。 下面我们来分析一下这几个指令在编译的过程中要做的事情：

#####  对于`v-bind`

要做两件事情：显示属性值； 属性值变化时，再次更新。 下面以`salary`属性为例：
（1）把data.salary对象中的属性显示出来.具体来说：
```

node.innerHTML = vm[exp]

```
（2）当data.salary对象值变化时，再次更新。


#####  对于`v-model`

v-model是用来实现双向数据绑定的，一般使用在表单元素上，也只有表单元素提供用户交互效果。 它要做三件事情：显示data对应的属性值； 属性值变化时，再次更新；用户的输入反映到data的属性上。 下面以`salary`属性为例：
（1）把data.salary对象中的属性显示出来.具体来说：
```

node.value = vm[exp]

```
注意这里用的是value，对`v-bind`使用的是`innerHTML`。
（2）当data.salary对象值变化时，再次更新。
（3）让node监听input事件。
```

node.addEventListener("input",(e)=>{
vm[exp] = e.target.value;
})

```

#####  对于`@click`

它加在了button上，要与options中的methods对应起来。要做一件事：给node添加click事件，在事件响应函数中执行methods中的方法。

```

node.addEventListener("click",(e)=>{
vm.\$methods[exp]()
})

```

现在的代码是：
```

function MVVM(options) {

this.$data = options.data;
        this.$el = document.querySelector(options.el);

        this.$methods = options.methods;
        this.observe(this.$data);

        this.compile();
      }
      MVVM.prototype.observe = function(obj) {
        Object.keys(obj).forEach(key => {
          var val = obj[key];
          Object.defineProperty(this, key, {
            enumerable: true,
            set: function(newVal) {
              console.info(`${obj[key]}----->${newVal}`);
              val = newVal;
            },
            get: function() {
              console.info(`get....${key}`);
              return val;
            }
          });
        });
      };

      MVVM.prototype.compile = function() {
        console.info(".....compile......");
        let el = this.$el;
        let vm = this;

        Array.from(el.children).forEach(node => {
          if (node.hasAttribute("v-model")) {
            let exp = node.getAttribute("v-model");
            console.info("v-model", exp);
            node.value = vm[exp];
            node.addEventListener("input", e => {
              console.info("input change.....", e);
              vm[exp] = e.target.value;
            });
          } else if (node.hasAttribute("v-bind")) {
            let exp = node.getAttribute("v-bind");
            console.info("v-bind", exp);
            node.innerHTML = vm[exp];
          } else if (node.hasAttribute("@click")) {
            let exp = node.getAttribute("@click");
            node.addEventListener("click", () => {
              vm.$methods[exp]();
            });
          }
        });
      };

```

现在还剩最后一件事情：“当data.salary对象值变化时，再次更新视图” 。这个问题抽象一下：一件事情发生之后，后续还要做若干其他的操作。要解决这一类问题，我们可以借助一种特殊的设计模式：观察者模式

### 观察者模式

稍微回顾一下观察者模式：

> 观察者模式定义了一种一对多的依赖关系，让多个观察者对象同时监听某一个目标对象，当这个目标对象的状态发生变化时，会通知所有观察者对象，使它们能够自动更新


再详细分析一下我们面临的具体问题。以data.salary为例：
当 data.salary的值变化时，要跟着一起变化的内容有：
- <input v-model="" />中的值
- <span v-bind="salary"></span>中的值。
- {{salary}} 中的值
- 计算属性 的值
.....

现在，这个问题是不是非常适合使用观察者模式？ 真的是再适合不过了。观察者模型有两个重要的概率：目标对象， 观察者。在我们现在面临的问题中，目标对象就是`data.salary`，观察者有多个，可以把它们放在一个列表中： [更新input，更新span,更新{{}},.....]

下面，我们来实现一个典型的观察者模式

#### 基本款的观察者模式

```

function TargetObject() {
this.watcherList = [];
}
TargetObject.prototype.add = function(watch) {
this.watcherList.push(watch);
};
TargetObject.prototype.notify = function() {
this.watcherList.forEach(watcher => watcher());
};

var watch1 = ()=>{console.info("watcher1")}
var watch2 = ()=>{console.info("watcher2")}
var to = new TargetObject()
to.push(watch1)
to.push(watch2)

to.notify()

```

目标对象内部维护了一个list（数组）,并对外暴露了对这个list的增删改查操作（本例中只有add方法）和一个notify(在某个时机对外通知)。 一个观察者就是一个普通的函数，它们之所以称为观察者是因为被加入到目标对象的List中，并且，目标对象可以通过notify来执行这些函数。除此之外，它就是一个函数。

类比理解一下这个场景：你关注一个微信公众号，微信公众号有新消息时，就会通知你。这里：你就是n多个粉丝之一，就是一个观察者。 微信公众号就是目标对象，它的notify方法就是发通知给你。

好了，作为拓展，你可以继续给list增加一个del方法，来模拟取消对微信公众号的关注。


#### 改进款的观察者模式

对于观察者而言，我们可以对它提一些额外的要求，让它能够维护自己的状态，而不只是一个普通的函数。如下：

```

function TargetObject() {
this.watcherList = [];
}
TargetObject.prototype.add = function(watch) {
this.watcherList.push(watch);
};
TargetObject.prototype.notify = function() {
this.watcherList.forEach(watch => watch.update());
};

      function Watcher(name,fn) {
          this.name = name
        this.fn = fn;
      }
      Watcher.prototype.update = function() {
        console.info("观察者this.name 被调用了")
        this.fn();
      };

var watch1 = new Watcher(()=>{console.info("watcher1")})
var watch2 = new Watcher(()=>{console.info("watcher2")})
var to = new TargetObject()
to.push(watch1)
to.push(watch2)

to.notify()

```








```

var data = {
salary:10000,
level:"T4"
}
var vm = {}

    function Observe(obj,vm) {
        Object.keys(obj).forEach(key => {
            var internalValue = obj[key]
            Object.defineProperty(vm, key, {
                get() {
                    console.log(`getting key "${key}": ${internalValue}`)
                    return obj[key]
                },
                set(newVal) {
                    console.log(`setting key "${key}" to: ${internalValue}`)
                    obj[key] = newVal
                }
            })
        })
    }
    Observe(data,vm);

```

参考 [Object.proxy](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy)

#### vue 中如何解析模板？

- 模板
- render 函数
- vdom

模板是什么

```

<div id="app">
    <input v-model="title">
    <button v-on:click="add">sumbit</button>

</div>
```

- 模板是`字符串`
- 模板有逻辑：if,else,for 等
- 和 html 很像，但有区别。上面的代码中 v-model,v-on 在 html 中都是不认识的。
- 模板要转成 js 代码。（字符串）-->(js 代码)。因为要处理其中的逻辑； js 才具备动态渲染 html。
- 模板最终还是要展示成 html 的
- 模板最终还是要转换成一个函数。（render 函数）

```
graph TB
A[模板] --> B[render函数]
```

例如：如下的模板

```
<div id="app">
<p>{{price}}</p>
</div>
```

会转成如下的函数：

```
function render(){
    with(this){
        return _c(
            "div",
            {
                attrs:{"id":"app"}
            },
            [
                _c("p",[_v(_s(price))])
            ]
        )
    }
}
```

- \_c: create
- \_s: toString
- \_v: textVnode
- price 就是 this.price，就是 data 中的 price。

你可以`var vm = new Vue()`，然后，通过 vm.\_c,vm.\_s 来查看返回值。

with 是一个特殊的语法，下面是不用 with 的用法。

```
function render(){

    return vm._c(
        "div",
        {
            attrs:{"id":"app"}
        },
        [
            vm._c("p",[vm._v(vm._s(price))])
        ]
    )

}
```

在 vue 源码中添加 alert()来查看 render.

```
var render =  generate(ast,options)
alert(code.render)
```

在 vue.2 中 开始支持`预编译`： 在开发环境写模板，编译打包之后得到 render()函数。也就是说，在开始运行代码前，就已经提前把 template 转成 reader()函数。

render()函数返回值：vnode 格式。
