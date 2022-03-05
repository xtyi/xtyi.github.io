---
title: JavaScript 的 this 是什么？
date: 2018-04-12T13:38:28+08:00
draft: false
tags:
  - js
---


# this

理解 JS 的 `this` 实际上非常简单，只需要了解 `call` 就行了

接下来，我们都使用 `call` 来调用函数，因为这才是 JS 最原始的函数调用方式

```js
function foo() {
	'use strict'
	console.log(this)
	console.log(arguments)
}

foo.call()
foo.call({name: 'nada'})
foo.call({name: 'nada'}, 1, 2, 3)
```

上面三次调用打印的内容如下
![js call](https://img-blog.csdnimg.cn/20200228235903207.png)
可以看到，当我们什么都不传入的时候 `this` 为 `undefined`，而当我们传入一个对象作为第一个参数时，`this` 就是这个对象

> `Arguments` 对象是其它参数组成的一个伪数组

现在，你已经知道 `this` 就是 `call` 调用时传入的第一个参数，那么 `this` 到底有什么用呢 ？正常调用函数时为什么没传 `this` ？

我们不使用 `this` 写一段代码感受一下

```js
const person = {
	name: 'nada',
	greet(person) {
		console.log(`Hey, I'm ${person.name}`)
	}
}

person.greet(person)
```

是不是非常麻烦，`person.greet(person)` 这种代码怎么看都是重复的

正常的代码应该是这样的 `person.greet()`

但是如果这样的话，没有参数传递给 `greet()` 这个函数，函数中当然也不能访问 `person` 对象

这种问题在编程语言层面很容易解决，当你以这样的语法调用函数时 `person.greet()` ，JS 将前面那个对象作为参数传给函数就好了

所以，`fn.call()` 这种方式才是最存粹的函数调用，`fn.call()` 需要将 `this` 显式的作为参数传入，而我们平时写的 `fn()` 都由 JS 施加了魔法，`this` 被隐藏了起来，你不能指定 `this` 代表的对象，而是 JS 按照规则自动传入

既然调用时没传参数，那声明时就也不要写 `this` 了，不然不一致，语言的一致性是很重要的，最后就变成了下面这样，这也是我们平时代码的样子

```js
const person = {
	name: 'nada',
	greet() {
		console.log(`Hey, I'm ${this.name}`)
	}
}

person.greet()
```

> Python 在这种问题上就采取了不同的设计，Python 中类的方法的第一个参数永远是 `self`，这样的话，调用和声明就出现了不一致，不过这种不一致还可以接受吧


函数和对象本无关系，函数负责逻辑，对象负责数据，但是面向对象编程中，将对象(数据)和函数封装在一起，这种情况下，函数只为某些对象服务，函数和对象就需要有一定的关联，函数就需要一个上下文

然而函数本质只有输入和输出，没有上下文

因此 `this` 就作为函数的参数为函数带来了上下文，这个上下文就是一个对象，函数内部就可以通过 `this` 关键字访问这个对象的属性

`this` 其实就是连接函数和对象的桥梁

最后再来看一些示例吧

```js
const person = {
	name: 'nada',
	foo() {
		console.log(this)
	}
}

person.greet()
```

这里调用 `person.foo()` 后打印的 `this` 当然是 `person` 对象

```js
const bar = person.foo
bar()
```

调用 `bar()` 打印的 `this` 是 `Window` 对象

```js
person.foo.call({name: 'nono'})
```

这时，`this` 是 `{name: 'nono'}`

这很清楚的说明了，`this` 正像上面所述，是动态传入的参数

而不是将一个函数作为一个对象的属性，这个函数的 `this` 就一定是这个对象

> 函数和对象本质上并无关联

`this` 是什么，只有调用时才能确定

如果面试官拿着下面这样的只有声明的代码问你 `this` 是什么，那你只能回答说没人能知道

```js
const person = {
	name: 'nada',
	foo() {
		console.log(this)
	}
}
```

# 回调函数

回调函数的 `this` 指向是一个问题，当我们调用 `person.foo()` 时就知道 `this` 就是前面的 `person` ，当我们调用 `bar()` 时，也知道 `this` 是 `undefined`

但是回调函数的 `this` 是什么，我们是不能推断出来的，不过看下文档或是直接打印一下看看就好了

```js
document.addEventListener('click', function () {
	console.log(this)
})
```

# 箭头函数
在一些情况下，一不小心就到导致错误的 this

```js
const person = {
	name: 'nada',
	foo() {
		return function () {
			console.log(this.name)
		}
	}
}
const bar = person.foo()
bar() // undefined
```

这里调用 `bar()` 传给 `this` 的是 `Window` 对象

但 ES6 的箭头函数帮助我们解决了这些问题

箭头函数的 `this` 是在创建时就绑定的

```js
const person = {
	name: 'nada',
	foo() {
		return () => {
			console.log(this.name)
		}
	}
}
const bar = person.foo()
bar() // 'nada'
```

上面的箭头函数是在 `person` 对象中的，所以它的 `this` 就绑定了 `person` 对象

