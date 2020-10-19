---
title: wtf-javascript-01
tags:
  - javascript
date: 2020-10-20 01:03:28
---

<!-- more -->

## 前言

这是参见 [javascript-questions](https://github.com/lydiahallie/javascript-questions/blob/master/zh-CN/README-zh_CN.md) 库做出的一些笔记，记录一下 js 中一些奇奇怪怪的现象和注意事项~

本篇主题有：

- IIFE
- 令人迷惑的 new
- 浏览器事件知识

<!-- more -->

## IIFE

> 关联至[问题 2]

题目给出了两个相似的 for 循环，它们唯一的区别就是循环变量的声明方法，一个是使用的是 `var` 关键字，而另外一个使用的是 ES6 中新增的 `let` 关键字。在两个循环的循环体内都是使用 `setTimeout` 方法在延迟一段同一段长度的时间后将循环变量 `i` 打印到控制台。

奇怪的是，使用 `var` 声明循环变量的那个循环最终会输出 '3' 三次，而使用 `let` 声明循环变量会按照我们的期望输出 '1', '2', '3'。

这主要是因为两种变量声明方式不同导致的差异：

- `var` 这种声明方式会使变量提升，并且分配内存空间，放在这个[问题 2]中，在 for 循环执行之后访问循环变量 `i`，由于这个变量已经成为全局变量了，并且哪怕将这个定时器的延迟设置为 0，`setTimeout` 的回调也始终会在循环执行完成之后再执行，因此回调执行时，访问到的循环变量 `i` 的值已经是 3 了。
- `let` 则会绑定到块级作用域，也就是 for 循环的循环体，每次循环都会将 `i` 的新值绑定到循环体，这么说可能不是很让人明白，但是经过试验可以证明这个绑定仅仅是浅拷贝而已，不过对于基本数据类型还是会发生真正意义上的拷贝，下面补充这个例子证明了这一点：

    ```javascript
    const l = [];
    for (let a = { i: 0 }; a.i < 3; a.i++) {
        setTimeout(() => l.push(a), 1);
    }
    console.log(l[0] === l[1] && l[1] === l[2]); // true
    ```

这个问题体现出 ES6 中 `let` 关键字与 `var` 关键字声明变量的差异，但是在 ES6 之前，JS 程序员该如何实现或模拟块级作用域绑定呢？答案就是 IIFE（立即调用函数表达式）：

```javascript
for (var i = 0; i < 3; i++) {
    ((j) => setTimeout(() => console.log(j)))(i);
}
// output: 1 2 3
```

上面代码片段中的匿名函数拥有独立的作用域，参数 `j` 是每轮遍历变量 `i` 值的一份拷贝，因此执行结果和[问题 2]中使用 ES6 的 `let` 关键字来声明循环变量的情形一致。

## 令人迷惑的 new

> 关联至[问题 12]

这个问题定义了一个函数，并在函数内部使用了 `this` 关键字。最关键的在于下方「调用」了两次这个函数，最重要的区别在于一个在调用之前使用了 `new` 关键字，而另外一个没有使用。

如果有一定的 JS 基础，对 `this` 关键字比较了解的话，很容易意识到，在**没有**使用 `new` 关键字的调用中，`this` 就会指向上一级对象，在这里就是全局对象，因此赋值过程并不会有任何问题，但是执行结果（返回值）是什么呢？

或许是因为这两种相似的用法摆在了一起，再加上受 Python 的实例化对象的语法的影响，我错误地认为返回值是 `{}`，也就是一个空对象，但事实上这个没有使用 `new` 关键字的调用就是普通的函数调用，因为这个函数没有显式返回任何值，因此返回值就是 `undefined`！

> 如果读者没有学习过 Python，或许就不会在这道问题上出错……在 Python 中无需使用 `new` 关键字，直接调用类构造器即可实例化一个相应的对象。

在使用了 `new` 关键字的调用中，`this` 就是指向新创建的空对象的，而所被调用的函数也摇身一变被称为「构造函数」，无论「构造函数」中是否返回值，实例化对象之后的返回值一定是这个创建对象的引用。在 ES6 之前，定义构造函数的写法就是如[问题 12]中这般简单，不得不让 C 系程序员感到十分新奇。而在 ES6 之后，也引入了 `class` 关键字，提供了其它面向对象语言类似的语法来定义类。

### new.target

`new.target` 属性允许你检测函数或构造方法是否是通过 `new` 运算符被调用的。

- 在通过 `new` 运算符被初始化的函数或构造方法中，`new.target` 返回一个指向构造方法或函数的引用。
- 在普通的函数调用中，`new.target` 的值是 `undefined`。

嗯，这是 MDN 上关于 `new.target` 的介绍，但是转念一想，这些功能基本可以通过 `this` 关键字来实现嘛，这又是 ES6 搞出的语法糖？不过由于这个语法的设计初衷就是为了判断构造方法是否是通过 `new` 运算符调用的，因此如果想要在继承后的子类的构造函数中的 `super()` 调用之前即执行相关判断会变得相当有用：

```javascript
class Parent {
    constructor() {
        console.log(new.target) // Child!
    }
}
class Child extends Parent {
    constructor() {
        // console.log(this);  // 这里不能使用 this，会报错
        console.log(new.target) // 这里可以正常使用，Child!
        super(); // this = Reflect.construct(Parent, [], new.target);
        console.log(this);
    }
}
new Child;
```

> `new.target` 不能在构造函数或普通函数的外部使用

## 浏览器事件知识

> 关联至[问题 13]

当一个事件发生在具有父元素的元素上（例如，在我们的例子中是&lt;video&gt;元素）时，现代浏览器运行两个不同的阶段 - 捕获阶段和冒泡阶段。 在捕获阶段：

- 浏览器检查元素的最外层祖先&lt;html&gt;，是否在捕获阶段中注册了一个onclick事件处理程序，如果是，则运行它。
- 然后，它移动到&lt;html&gt;中单击元素的下一个祖先元素，并执行相同的操作，然后是单击元素再下一个祖先元素，依此类推，直到到达实际点击的元素。

在冒泡阶段，恰恰相反:

- 浏览器检查实际点击的元素是否在冒泡阶段中注册了一个onclick事件处理程序，如果是，则运行它
- 然后它移动到下一个直接的祖先元素，并做同样的事情，然后是下一个，等等，直到它到达&lt;html&gt;元素。

![冒泡和捕获](https://mdn.mozillademos.org/files/14075/bubbling-capturing.png)

<!-- 问题引用 -->

[问题 2]: https://github.com/lydiahallie/javascript-questions/blob/master/zh-CN/README-zh_CN.md#2-%E8%BE%93%E5%87%BA%E6%98%AF%E4%BB%80%E4%B9%88
[问题 12]: https://github.com/lydiahallie/javascript-questions/blob/master/zh-CN/README-zh_CN.md#12-%E8%BE%93%E5%87%BA%E6%98%AF%E4%BB%80%E4%B9%88
[问题 13]: https://github.com/lydiahallie/javascript-questions/blob/master/zh-CN/README-zh_CN.md#13-%E4%BA%8B%E4%BB%B6%E4%BC%A0%E6%92%AD%E7%9A%84%E4%B8%89%E4%B8%AA%E9%98%B6%E6%AE%B5%E6%98%AF%E4%BB%80%E4%B9%88
