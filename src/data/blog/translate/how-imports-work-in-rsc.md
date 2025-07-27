---
pubDatetime: 2025-07-25T14:33:04.170Z
title: RSC 中的 import 工作原理
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

# RSC 中的 import 工作原理

React Server Components（RSC）是一种让你将前端/后端应用作为跨两个环境的“单一程序”来表达的编程范式。RSC 扩展了模块系统（即 JavaScript 的 `import` 和 `export` 关键字），让开发者能更好地控制前后端代码的分隔。

## 什么是模块系统？

计算机在执行程序时，其实并不需要“模块”；它只需要代码和数据被完整加载到内存中即可。模块更多是为了我们人类更好地组织代码：
- 可以将复杂程序拆分为易于理解的部分
- 限定哪些代码可以被外部访问（export），哪些作为内部实现细节
- 方便复用自己和他人的代码

**模块系统的职责，就是桥接“人类的代码拆分方式”和“计算机实际的执行方式”之间的差距。**

以 JavaScript 为例，模块系统通过 `import` 和 `export` 关键字向开发者开放。


## import 其实很像“复制粘贴”……

比如有如下两个文件，a.js 和 b.js：

```js
export function a() {
  return 2;
}
```

```js
export function b() {
  return 2;
}
```

各自只定义了一个函数。再看一个 index.js：

```js
import { a } from './a.js';
import { b } from './b.js';

const result = a() + b(); // 4
console.log(result);
```

这时，JavaScript 模块系统确保，**当这个程序运行时，它应该和以下没有用模块的单文件效果一致：**

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

也就是说，`import` 和 `export` 在本质逻辑上有点类似最终“复制粘贴”到一起，因为最后 JS 引擎会把它们都“展开”在内存里。


## ……但和真正复制粘贴又不完全一样

早期 C 语言的 `#include` 指令基本就是原始的“粘贴”行为，例如：

```c
#include "a.h"
#include "b.h"

int main() {
  return a() + b();
}
```

C 的 `#include` 会把 a.h 和 b.h 的内容原样嵌入当前文件。然而这种方式有两个大问题：
1. 不同文件如果有同名函数，内容就会冲突。现代模块系统里，各标识符只在各自文件本地可见，避免了此问题。
2. 一个文件如果被多次 include，会被重复插入多次。为此，需要在每个可被 include 的文件加“只引入一次”判断。

现在的 `import` 机制自动避免了这些问题。


## JavaScript 模块是单例的

假设我们又加了个 c.js：

```js
export function c() {
  return 2;
}
```

然后 a.js 和 b.js 都各自引入 `c` 并调用：

```js
import { c } from './c.js';

export function a() {
  return c() * 2;
}
```

```js
import { c } from './c.js';

export function b() {
  return c() * 3;
}
```

**如果 import 真的是复制粘贴，最后会有两份 c 函数。但其实不会！**

JavaScript 模块系统确保每个模块内容，在整个程序中**无论被导入多少次，只会执行一次**。最终的运行效果等同于：

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


## （以下略去局部说明，聚焦核心内容）

### RSC 的核心分隔点

以前，单页面应用（SPA）或传统 Node.js 后端的模块导入逻辑只涉及一个运行环境；在 RSC 里，`'use client'` 和 `'use server'` 分别标记某个模块是否运行于客户端还是服务端。

- **如果没有 `'use client'` 或 `'use server'`，那么该模块默认为服务端执行（RSC 默认行为）**
- **子模块可以继承父模块的“环境标签”，除非自身显式指定**

这让你可以细粒度控制某个文件运行在客户端、服务端，还是两者都用。


## RSC 下的 import/workflow 细节

1. **同一个模块，可以被服务端和客户端分别加载、拥有各自的实例**
2. 如果一个不带 `'use client'` 的服务端模块导入了一个带 `'use client'` 的客户端模块，那么二者运行环境分离，互不影响
3. 某些文件如果被标为 `'use client'`，它们 import 的其它文件也必须能在客户端运行（否则打包报错）


## 总结理解

- 传统 JS 模块系统让多文件只在“同一个环境内”做单例展开；
- RSC 通过 `'use client'` 和 `'use server'` 开辟边界，让开发者自由安排哪些模块在前/后端环境加载，并保证同一模块不同环境可以拥有独立实例；
- RSC 的 `import` 行为本质上类似传统 JS 模块系统的“单例”，但多了“环境维度”的区分。