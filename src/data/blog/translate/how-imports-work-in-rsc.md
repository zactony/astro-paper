---
pubDatetime: 2025-07-25T14:33:04.170Z
title: RSC 中的 Import 如何工作
featured: true
tags:
  - React
  - 翻译
  - RSC
description:
  介绍了 React Server Components（RSC）如何通过扩展模块系统（import/export）及 'use client' / 'use server' 指令，实现前端与后端代码的清晰分界与协作，同时保证模块行为直观和高可维护性。
---


## Table of contents

## 前言
介绍了 React Server Components（RSC）如何通过扩展模块系统（import/export）及 'use client' / 'use server' 指令，实现前端与后端代码的清晰分界与协作，同时保证模块行为直观和高可维护性。[原文在这](https://overreacted.io/how-imports-work-in-rsc/)。

译文从这开始～～

React Server Components（RSC）是一种编程范式，让你能够将客户端/服务器应用以一个跨越两个环境的单一程序方式来表达。具体来说，RSC 扩展了模块系统（即 `import` 和 `export` 关键字），用新的语义让开发者能够控制前后端的分界。

我之前已经写过[《'use client' 和 'use server' 有什么用？》](https://overreacted.io/what-does-use-client-do/)，介绍了这两个指令如何标记“两端”之间的分割点。本文我想专注讲讲这些指令和 `import`、`export` 关键词的交互方式。

这是一篇深入讲解，适合想要建立准确 RSC 心智模型，以及对模块系统原理感兴趣的人。你也许会发现 RSC 的方法令人惊讶——它其实比你想象的更简单。

和往常一样，本文 90% 内容其实不是直接讲 RSC，而是讲一般情况下 import 如何工作、当我们要让后端和前端共享代码时会发生什么。我的目标是展示 RSC 如何自然地解决我们在编写跨前后端代码时最后10%的难题。

我们先从基本概念开始。



## 什么是模块系统？

计算机执行一个程序时，其实并不需要“模块”。计算机只需要代码和数据都**完整加载到内存里**，才能运行和处理它们。其实是**我们人类**需要将代码划分为模块：

- 模块可以让我们把复杂程序拆分为易于理解的片段。
- 模块能让我们声明哪些代码要暴露（`export`）出去，哪些应该是实现细节。
- 模块便于我们复用自己和他人的代码。

**我们希望按模块化方式“创作”程序——但执行一个程序时，其模块内容都需要被“摊平”，装入内存。模块系统的任务，就是把人写代码的方式和计算机执行代码所需的方式桥接起来。**

具体地说，**模块系统**就是一套规则，定义程序能如何分割到不同文件、开发者如何控制代码片段之间的可见性，以及这些片段最终如何链接为内存中的一个程序。

在 JavaScript 里，模块系统主要通过 `import` 和 `export` 暴露出来。



## Import 很像复制粘贴……

假设有下面这两个文件，叫 `a.js` 和 `b.js`：

```js
// a.js
export function a() {
  return 2;
}
```
```js
// b.js
export function b() {
  return 2;
}
```

本身它们只是定义了函数，啥也没做。

再假设有个 `index.js`：

```js
import { a } from './a.js';
import { b } from './b.js';

const result = a() + b(); // 4
console.log(result);
```

现在，这个模块把它们组合成了一个程序！

JavaScript 的模块系统规则很复杂，有很多细节。但可以用一个简单的直觉理解：**上面的程序在运行时，应该和如下不含模块、写在一个文件里的代码表现完全一样：**

```js
function a() {
  return 2;
}

function b() {
  return 2;
}

const result = a() + b(); // 4
console.log(result);
```

换句话说，`import` 和 `export` 的设计就是**像“复制粘贴”那样工作**——最终程序里所有代码都得被 JS 引擎“摊平”到内存里。



## ……但其实又不是

上面说 import 像复制粘贴，实际上不完全准确。要理解原因，我们需要回顾下 C 语言的 `#include` 指令。

`#include` 比 JavaScript 的 import 早了 40 年，是真的[和复制粘贴很像](https://stackoverflow.com/a/5735389/458193)！举个例子，这是一个 C 程序：

```c
#include "a.h"
#include "b.h"

int main() {
  return a() + b();
}
```

在 C 里，`#include` 指令会**把 a.h 和 b.h 的全部内容原样嵌入到本文件**。这样很简单，但有两个大问题：

1.  `#include` 有个问题是，[不同文件里的无关函数名字会冲突](https://softwareengineering.stackexchange.com/a/202156/3939)。现代模块系统则能让每个标识符只对本文件可见。
2.  另一个问题是，同一个文件如果被多个地方 include，就会在输出程序里重复了多份。解决方法是加[编译时“只 include 一次”](https://stackoverflow.com/a/12928949/458193)的保护。现代模块系统比如 import，会自动帮你处理这点。

这个点很重要，我们展开说说。



## JavaScript 模块是“单例”

假设我们新加了一个 `c.js`：

```js
export function c() {
  return 2;
}
```

然后把 `a.js` 和 `b.js` 都改成各自 import c：

```js
// a.js
import { c } from './c.js';
export function a() {
  return c() * 2;
}
```
```js
// b.js
import { c } from './c.js';
export function b() {
  return c() * 3;
}
```

如果 import 是字面意义的 “复制粘贴”（类似 #include），程序里就会有两个 c 函数副本。但事实并非如此！

JavaScript 模块系统保证，以上代码跟之前的 `index.js`，无论模块导入了多少次、从哪里导入，**最终都会只有一个 c 函数定义**，如下：

```js
function c() {
  return 2;
}

function a() {
  return c() * 2;
}

function b() {
  return c() * 3;
}

const result = a() + b(); // (2 * 2) + (2 * 3) = 10
console.log(result);
```

换句话说，现代模块系统比如 JavaScript 的模块，**保证单个模块里的代码，无论被 import 多少次、从多少地方 import，都只会执行一次。**



这是设计上的一个关键点。为什么？因为这样大家才能放心地在模块顶层写副作用代码——例如做初始化、设置连接池、配置全局变量等——不用担心这些代码会被执行多次而造成混乱。



## 导入值 ≠ 只是一份副本

还有一个更易被忽视但很重要的点：**导入的值并不是一份拷贝，而是对导出值本身的引用。**

来看一个例子。`c.js` 变成这样：

```js
export let count = 0;

export function increment() {
  count += 1;
}
```

`a.js` 这样写：

```js
import { count, increment } from './c.js';

export function a() {
  increment();
  return count;
}
```

`b.js` 也写类似的代码：

```js
import { count, increment } from './c.js';

export function b() {
  increment();
  return count;
}
```

`index.js`：

```js
import { a } from './a.js';
import { b } from './b.js';

console.log(a()); // 1
console.log(b()); // 2
```

你如果认为 import 拿到的是 count 的一份独立副本，那输出可能都是 1。但实际上，无论从 `a.js`、`b.js` 还是 `index.js` 导入，**大家拿到的都是同一个 `count` 变量。**

`index.js` 最终会输出：

```
1
2
```

这展示了“导入值共享同一份内存”，这是现代模块系统的第二大核心点。



## 前端与后端代码分界问题

上面这些讨论假定只有**一个执行环境和一份内存**：不管 import 来 import 去，大家操作的都是内存里的同一块数据。

但是用了前后端分离的框架或者像 React Server Components 这样的范式后，问题来了：**“同一个模块”是不是可能运行于不同环境下？**

- 在客户端 bundle 里，这段代码会作为由用户浏览器加载的 JavaScript 运行，拥有一份内存。
- 在服务器端 bundle 里，这段代码又会在服务器进程里跑，拥有另一份完全独立的内存。

即便我们 import/import 到的是同一个文件，在客户端和服务端，**其实拿到的是完全不同的模块副本**，它们彼此不共享内存。这就是“跨环境”写代码时的关键难题！



## RSC 如何解决分界问题

React Server Components 通过 `'use client'` 和 `'use server'` 指令，让你能在代码文件级别声明：**这个文件是属于客户端（浏览器）还是服务器。**

- `'use client'` 文件及其所有被直接/间接 import 的代码（只要未碰到 `'use server'` 或 `'use client'` 边界）都被打包到客户端 bundle，在浏览器执行。
- `'use server'` 文件及其导入链都被打包到服务器 bundle，只在服务器执行。

在 RSC 里，整个代码树是一棵包含了“客户端分支”和“服务器分支”的大的依赖树。**只有在遇到指令切换点时，代码才会“跨界”。**（比如，从 `'use server'` 文件里 import 一个 `'use client'` 文件，反之同理。）



这种模型非常直观：

- **默认所有导入都在本地环境内解决，**只在遇到边界时才发生“跨端”调用。
- 导入同一个“跨端”组件时，每一端有各自独立的实例和内存。
- 这样可以避免副作用和状态穿越端的混乱，也让代码行为变得容易理解。



## 代码示例：RSC 下的 Import 行为

（原文有代码片段说明实际行为，中文直译略。如果需要代码完整含注释翻译版本可指出。）



## 为什么不用 RPC？为什么不用 API？

“后端调用前端（或前端调用后端）”听起来像 RPC 或 API，但 RSC 故意不是这么做的：

- 你写代码的感觉仍是“模块系统”，让你有清晰依赖关系和封装边界。
- 只在跨端边界自动帮你序列化/反序列化 props 和数据，而不是暴露整个后端接口。
- 这让代码复用和重构变得更自然——因为是模块而不是 API。



## 总结

- **模块系统**的核心目的是：“把人类写的代码结构和计算机运行时的物理代码布局桥接起来”。
- `import`/`export` 让我们像“复制粘贴”一样组织代码，不过它自动实现了单例和共享引用。
- 在 RSC 下，一份代码可以拆分运行在两份完全独立的内存中（服务器/客户端各一份）。
- RSC 让你通过 `'use client'` / `'use server'` 很自然地控制分界点。
