# vue官方学习笔记
学习完官网基础知识后，有以下理解，并产生部分疑问
## 传统和现代
### 传统开发  
引入script（还有一些版本问题比较复杂）[说明](https://cn.vuejs.org/v2/guide/installation.html)

编写组件对象
// js脚本中定义一个名为 button-counter 的新组件
`var button-counter={
  data: function () {
    return {
      count: 0
    }
  },
  template: '<button v-on:click="count++">You clicked me {{ count }} times.</button>'
}`

注册组件
`Vue.component('button-counter', button-counter)`

使用组件
`<div id="components-demo">
  <button-counter></button-counter>
</div>
new Vue({ el: '#components-demo' })`  
之后vue框架就会使用定义的组件模板替换html标签。
更神奇的是，组件模板里面也可以使用其他组件，所以，只需要手动渲染根节点，所有子节点的自定义组件就能全部渲染出来。  
这是一种把界面写到js的思想。
### 现代模块化
因为浏览器的版本问题是开发体验的最大阻力，于是为了统一大家的开发体验，nodejs上的前端开发模式出现了。
跟java开发类似，我们使用更自然、更可维护、更可读的语言（es6语法）编写一些高级代码，在node上的经过各种框架编译出兼容性强的
浏览器可以解析的代码。
vue可以使用不同的模块化框架webpack、Browserify、Rollup[官方说明](https://cn.vuejs.org/v2/guide/deployment.html)推荐webpack
如果你使用 webpack，并且喜欢分离 JavaScript 和模板文件，你可以使用 vue-template-loader，它也可以在构建过程中把模板文件转换成为 JavaScript 渲染函数。

## 现代模块化开发
### webpack快速入门
[解释](https://segmentfault.com/a/1190000006178770)  
需要了解热加载中间件、编译阶段报错特性
### 模块化必知exports、require和export、import
[解释](https://segmentfault.com/a/1190000010426778)

## vue难理解点
### v-bind和v-mode区别  
v-bind是单向的，js修改数据，界面就变，用于数据异步加载。
界面变动单向修改数据使用事件机制。
v-mode是双向的，数据变动影响界面展示，同时界面变动，数据也变动，用于修改表单。原理是结合上面两个

## 语法困惑
注意在 ES2015+ 中，在对象中放一个类似 ComponentA 的变量名其实是 ComponentA: ComponentA 的缩写，即这个变量名同时是：

## 事件修饰符
还没搞懂。。。


