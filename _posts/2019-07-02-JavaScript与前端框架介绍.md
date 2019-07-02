---
layout:     post
title:      JavaScript与前端框架介绍
subtitle:   JavaScript与前端框架介绍
date:       2019-07-02
author:     gregorius
header-img: img/post-bg-swift.jpg
catalog: true
tags:
    - 分布式
---

&emsp;&emsp;JavaScript是门优秀的语言，自从它出生开始就有了跨平台的特性，在1995 年 Netscape 一位名为 Brendan Eich 的工程师创造了 JavaScript，随后在 1996 年初，JavaScript 首先被应用于 Netscape 2 浏览器上。最初的 JavaScript 名为 LiveScript，后来因为 Sun Microsystem 的 Java 语言的兴起和广泛使用，Netscape 出于宣传和推广的考虑，将它的名字从最初的 LiveScript 更改为 JavaScript。

&emsp;&emsp;几个月后，Microsoft 随着 IE 3 推出了一个与之基本兼容的语言 JScript。又几个月后，Netscape 将 JavaScript 提交至 [Ecma International](http://www.ecma-international.org/)（一个欧洲标准化组织）， [ECMAScript](https://developer.mozilla.org/en-US/docs/Glossary/ECMAScript "ECMAScript: ECMAScript is the scripting language on which JavaScript is based. Ecma International is in charge of standardizing ECMAScript.")标准第一版便在 1997 年诞生了，随后在 1999 年以 [ECMAScript 第三版](http://www.ecma-international.org/publications/standards/Ecma-262.htm)的形式进行了更新，从那之后这个标准没有发生过大的改动。由于委员会在语言特性的讨论上发生分歧，ECMAScript 第四版尚未推出便被废除，但随后于 2009 年 12 月发布的 ECMAScript 第五版引入了第四版草案加入的许多特性。第六版标准已经于2015年六月发布。

&emsp;&emsp;JavaScript 是一种面向对象的动态语言，它包含类型、运算符、标准内置（ built-in）对象和方法。它是运行web客户端的语言，虽然历史长河中出现过很多远行在web客户端的语言，但是JavaScript 取得了最终的胜出。虽然ECMA发布了多个版本的标准，但是各大浏览器对标准的支持都不一样，导致了之前的前端工程师必须对主流浏览器进行兼容，这一直都是困扰前端工程师的噩梦。

&emsp;&emsp;经过10年的规划，ES5的终于出来了，ES5 与 ES3 基本保持兼容，较大的语法修正和新功能加入，ES5 在 2013 年的年中成为 JavaScript 开发的主流标准，并在此后五年中一直保持这个位置。ES6作为下一代的JavaScript语言，终于引入了[Class和Module的概念](http://www.alloyteam.com/2016/03/es6-front-end-developers-will-have-to-know-the-top-ten-properties/)，同时随着[nodejs](https://nodejs.org/)的推出，JavaScript已经具备了后端面向对象语言的能力，可以完全使用JavaScript开发出复杂的系统。这里是目前各大浏览器对[ES6的支持情况](https://kangax.github.io/compat-table/es6/)。

###当前主流的前端框架

[angular]([https://angularjs.org/](https://angularjs.org/))：
Angular1
AngularJS的工作原理是:HTML模板将会被浏览器解析到DOM中, DOM结构成为AngularJS编译器的输入。AngularJS将会遍历DOM模板, 来生成相应的NG指令,所有的指令都负责针对view(即HTML中的ng-model)来设置数据绑定。因此, NG框架是在DOM加载完成之后, 才开始起作用的。
 [Angular (原本的 Angular 2)](https://cn.vuejs.org/v2/guide/comparison.html#Angular-%E5%8E%9F%E6%9C%AC%E7%9A%84-Angular-2 "Angular (原本的 Angular 2)")
Angular 事实上必须用 TypeScript 来开发，因为它的文档和学习资源几乎全部是面向 TS 的。TS 有很多好处——静态类型检查在大规模的应用中非常有用，同时对于 Java 和 C# 背景的开发者也是非常提升开发效率的。


[React ]([https://facebook.github.io/react/](https://facebook.github.io/react/))

React 的渲染建立在 Virtual DOM 上——一种在内存中描述 DOM 树状态的数据结构。当状态发生变化时，React 重新渲染 Virtual DOM，比较计算之后给真实 DOM 打补丁。


Virtual DOM 提供了函数式的方法描述视图，它不使用数据观察机制，每次更新都会重新渲染整个应用，因此从定义上保证了视图与数据的同步。它也开辟了 JavaScript 同构应用的可能性。

[Vue]([https://cn.vuejs.org/](https://cn.vuejs.org/))

React 和 Vue 有许多相似之处，它们都有：

- 使用 Virtual DOM
- 提供了响应式 (Reactive) 和组件化 (Composable) 的视图组件。
- 将注意力集中保持在核心库，而将其他功能如路由和全局状态管理交给相关的库。

三大主流系统都是支持组件模块化的框架，这里是Vue提供的[对比图](https://cn.vuejs.org/v2/guide/comparison.html)，值得参考。

###框架选择

&emsp;&emsp;对于中小型项目来说，选择Vue可能是最好的选择，它兼固其它两大框架的优势，并且学习曲线相当平坦，劣势在于生态方面弱于两者。官网提供了丰富的文档，完全可以只看官网文档即可入门和精通。如果想深入学习可以参考这个[blog](https://yugasun.com/)。

>本文参考文章如下：
- https://www.25xt.com/html5css3/15161.html
- http://es6.ruanyifeng.com/#docs/intro
- http://caibaojian.com/toutiao/6864
- https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/A_re-introduction_to_JavaScript
- https://cn.vuejs.org/v2/guide/index.html