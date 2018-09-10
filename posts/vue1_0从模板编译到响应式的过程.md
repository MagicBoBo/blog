---
title: vue1.0从模板编译到响应式的过程
date: 2018-08-21 12:36:33
tags:
---

模板和响应式是vue的核心。关于模板我感觉是个由来已久的概念，比如template.js。实习期间见识到模板的写法是将模板html写到script标签当中，然后拿到模板字符串，用template.js分析，最后用$dom.html()将分析好的模板写回dom之中。

vue中将模板的思想引入了进来，他自己做了模板引擎，这个模板支持{{ }}写法，还支持指令。模板complie后会生成真正的dom与指令集，并在link中将指令与对应的watcher进行绑定。因此我们在defineProperty中做了数据劫持之后，相对应的watcher是持有真实dom的，所以他的update方法可以对真实dom进行修改。

这是一个最简版的vue的数据劫持，在get中订阅主题（只会在第一次get中订阅，具体来说是在你new watcher的时候），然后在set中进行notify
```
function defineReactive (obj, key, val) {
  var dep = new Dep();
  Object.defineProperty(obj, key, {
    get: function () {
      // 添加订阅者 watcher 到主题对象 Dep
      if (Dep.target) dep.addSub(Dep.target);
      return val
    },
    set: function (newVal) {
      if (newVal === val) return
      val = newVal;
      // 作为发布者发出通知
      dep.notify();
    }
  });
}
```
具体来说如何做到只订阅一次，是通过一个全局的Dep对象
```
 function Watcher (vm, node, name, nodeType) {
   //此处缓存watcher
   Dep.target = this;
   this.name = name;
   this.node = node;
   this.vm = vm;
   this.nodeType = nodeType;
   //此处手动调用一次get，然后订阅主题
   this.update();
   //释放缓存，以后劫持的get方法将不会再订阅
   Dep.target = null;
 }
Watcher.prototype = {
  update: function () {
    this.get();
    if (this.nodeType == 'text') {
      this.node.nodeValue = this.value;
    }
    if (this.nodeType == 'input') {
      this.node.value = this.value;
    }
  },
  // 获取 data 中的属性值
  get: function () {
    this.value = this.vm[this.name]; // 触发相应属性的 get
  }
}
```
这是一个最简版的数据劫持，代码来源于[这篇文章](http://www.cnblogs.com/kidney/p/6052935.html)，而真正的watcher创建是在模板编译生成指令集之后，创建一个与指令集绑定的watcher。

