# Vue.js 源码剖析-响应式原理、虚拟 DOM、模板编译和组件化

## 一、简答题

### 1、请简述 Vue 首次渲染的过程。

1. 首先进行Vue的初始化，初始化Vue的实例成员以及静态成员。
2. 当初始化结束之后，开始调用构造函数，在构造函数中调用this._init()，这个方法相当于我们整个Vue的入口。
3. 在_init()中调用this.\$mount()，共有两个this.\$mount()。
    * 第一个this.\$mount()是entry-runtime-with-compiler.js入口文件，这个\$mount()的核心作用是帮我们把模板编译成render函数，但它首先会判断一下当前是否传入了render选项，如果没有传入的话，它会去获取我们的template选项，如果template选项也没有的话，他会把el中的内容作为我们的模板，然后把模板编译成render函数，它是通过compileToFunctions()函数，帮我们把模板编译成render函数的,当把render函数编译好之后，它会把render函数存在我们的options.render中。
    * 第二个this.\$mount()是runtime/index.js中的this.\$mount()方法，这个方法首先会重新获取el，因为如果是运行时版本的话，是不会走entry-runtime-with-compiler.js这个入口中获取el，所以如果是运行时版本的话，我们会在runtime/index.js的$mount()中重新获取el。
4. 接下来调用mountComponent(),mountComponent()是在src/core/instance/lifecycle.js中定义的，在mountComponent()中，首先会判断render选项，如果没有render，但是传入了模板，并且当前是开发环境的话会发送警告，警告运行时版本不支持编译器。接下来会触发beforeMount这个生命周期中的钩子函数，也就是开始挂载之前。
5. 然后定义了updateComponent()，在这个方法中，定义了_render和_update，_render的作用是生成虚拟DOM，_update的作用是将虚拟DOM转换成真实DOM，并且挂载到页面上来。
6. 再接下来就是创建Watcher对象，在创建Watcher时，传递了updateComponent这个函数，这个函数最终是在Watcher内部调用的。在Watcher创建完之后还调用了get方法，在get方法中，会调用updateComponent()。
7. 然后触发了生命周期的钩子函数mounted,挂载结束，最终返回Vue实例。

### 2、请简述 Vue 响应式原理。

* Vue的响应式是从Vue的实例init()方法中开始的，在init()方法中先调用initState()初始化Vue实例的状态，在initState方法中调用了initData()， initData()是把data属性注入到Vue实例上，并且调用observe(data)将data对象转化成响应式的对象。
* observe是响应式的入口, 在observe(value)中，首先判断传入的参数value是否是对象，如果不是对象直接返回。再判断value对象是否有__ob__这个属性，如果有说明做过了响应式处理，则直接返回，如果没有，创建observer对象，并且返回observer`对象。
* 在创建observer对象时，给当前的value对象定义不可枚举的__ob__属性，记录当前的observer对象，然后再进行数组的响应式处理和对象的响应式处理，数组的响应式处理就是拦截数组的几个特殊的方法，push、pop、shift等，然后找到数组对象中的__ob__对象中的dep,调用dep的notify()方法，再遍历数组中每一个成员，对每个成员调用observer()，如果这个成员是对象的话，也会转换成响应式对象。对象的响应式处理，就是调用walk方法，walk方法就是遍历对象的每一个属性，对每个属性调用defineReactive方法
* defineReactive会为每一个属性创建对应的dep对象，让dep去收集依赖，如果当前属性的值是对象，会调用observe。defineReactive中最核心的方法是getter 和 setter。getter 的作用是收集依赖，收集依赖时, 为每一个属性收集依赖，如果这个属性的值是对象，那也要为子对象收集依赖，最后返回属性的值。在setter 中，先保存新值，如果新值是对象，也要调用 observe ，把新设置的对象也转换成响应式的对象,然后派发更新（发送通知），调用dep.notify()
* 收集依赖时，在watcher对象的get方法中调用pushTarget,记录Dep.target属性，访问data中的成员的时候收集依赖，defineReactive的getter中收集依赖，把属性对应的 watcher 对象添加到dep的subs数组中，给childOb收集依赖，目的是子对象添加和删除成员时发送通知。
* 在数据发生变化的时候，会调用dep.notify()发送通知，dep.notify()会调用watcher对象的update()方法，update()中的调用的queueWatcher()会去判断watcher是否被处理，如果这个watcher对象没有的话添加到queue队列中，并调用flushScheduleQueue()，flushScheduleQueue()触发beforeUpdate钩子函数调用watcher.run()： run()-->get() --> getter() --> updateComponent()
* 然后清空上一次的依赖
* 触发actived的钩子函数
* 触发updated钩子函数



### 3、请简述虚拟 DOM 中 Key 的作用和好处。

在v-for的过程中，为给每一个节点设置key属性的作用：
以便它能够跟踪每个节点的身份，在进行比较的时候，会基于 key 的变化重新排列元素顺序。从而重用和重新排序现有元素，并且会移除 key 不存在的元素。方便让 vnode 在 diff 的过程中找到对应的节点，然后成功复用。

设置key的好处：
可以减少 dom 的操作，减少 diff 和渲染所需要的时间，提升了性能。

### 4、请简述 Vue 中模板编译的过程。

模板编译就是把模板转化成供Vue实例在挂载时可调用的render函数，具体过程：

1. parser解析器将模板字符串解析成AST抽象语法树（一个js对象）
2. optimizer优化器标记静态节点，diff的时候会跳过被标记的静态节点，减少了diff的比较过程，从而优化了 patch 的性能
3. generate代码生成器将AST转换成渲染（render）函数，该函数用于生成虚拟dom