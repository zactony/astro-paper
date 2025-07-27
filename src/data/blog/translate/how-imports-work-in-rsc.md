---
pubDatetime: 2025-07-25T14:33:04.170Z
title: RSC 中的导入机制原理
featured: true
tags:
  - React
  - 翻译
  - RSC
  - dan abramov
description:
  介绍了 React Server Components（RSC）如何通过扩展模块系统（import/export）及 'use client' / 'use server' 指令，实现前端与后端代码的清晰分界与协作，同时保证模块行为直观和高可维护性。
---


## Table of contents

## 前言
介绍了 React Server Components（RSC）如何通过扩展模块系统（import/export）及 'use client' / 'use server' 指令，实现前端与后端代码的清晰分界与协作，同时保证模块行为直观和高可维护性。[原文在这](https://overreacted.io/how-imports-work-in-rsc/)。

译文从这开始～～

## RSC 中的导入机制原理

React Server Components（RSC）是一种编程范式，它允许你使用单一程序表达涵盖前端和后端的应用。这种方式实际上扩展了模块系统（`import` 和 `export` 关键字）的语义，帮助开发者精确控制前后端的边界。

我之前已经[写过](https://overreacted.io/what-does-use-client-do/)关于 `'use client'` 和 `'use server'` 指令，它们用于标记两个环境之间的“分界点”。本文将重点讨论这些指令如何与 `import` 和 `export` 关键字协同工作。

此文是一次深入探讨，适合希望建立准确 RSC 心智模型的开发者，也适合对模块系统本身感兴趣的技术同仁。你可能会发现 RSC 的方法既令人惊讶，又比想象中简单。

和往常一样，文章的大部分内容其实并不是纯讲 RSC，而是讲解“导入”本身如何运作，以及当我们试图在前后端之间共享代码时会发生什么。我的目标是展示 RSC 如何为跨端代码开发带来一种自然优雅的解决方案，特别是帮助解决那最后 10% 难题。

让我们从基础概念讲起。

---

### 什么是模块系统？

当计算机执行程序时，其实并不需要“模块”。它只需要把程序代码和数据**加载到内存中**，即可运行和处理。实际上，是**我们人类**希望将代码拆分为模块：

- 模块让我们能把复杂的程序分解为头脑可以装下的部分。
- 模块让我们能约束哪些代码应该对外可见（*导出*），哪些应该只作为实现细节隐藏。
- 模块让我们能复用其他人（以及自己）写的代码。

**我们希望以“分块”的方式编写程序，但“执行”程序时必须将所有模块都“展开”在内存里。模块系统的任务，就是桥接人类编写习惯和计算机执行需求之间的差距。**

具体来说，*模块系统*是一组规则，规定了程序如何被拆分为文件、开发者如何控制各部分的可见性，以及这些部分如何被链接成最终可加载到内存的一个整体程序。

对于 JavaScript，模块系统就是通过 `import` 和 `export` 关键字呈现的。

---

### 导入就像复制粘贴……

假设有如下两个文件，分别是 `a.js` 和 `b.js`：

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

它们单独看起来只是定义了函数，并没有实际执行任何操作。

再看 `index.js` 文件：

```js
import { a } from './a.js';
import { b } from './b.js';

const result = a() + b(); // 4

console.log(result);
```

这个模块把它们串联成了一个完整的程序！

JavaScript 的模块系统规则其实很复杂。但我们可以用一种简单的直觉来理解——模块系统会确保**在程序运行前，模块被展开后和下面这个单文件程序的行为完全一致**（即根本没有用模块）：

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

换句话说，`import` 和 `export` 的本质设计思路**类似“复制粘贴”**，因为最终 JavaScript 引擎运行时，确实需要所有代码都“铺开”进内存。

---

### ……但又不是复制粘贴

刚才说“导入就像复制粘贴”，但其实并不完全一样。为了说明这一点，让我们回忆一下 C 语言中的 `#include` 指令。

C 中的 `#include` 指令，在实现上真的[就是复制粘贴](https://stackoverflow.com/a/5735389/458193)。例如：

```c
#include "a.h"
#include "b.h"

int main() {
  return a() + b();
}
```

`#include` 会**直接把 `a.h` 和 `b.h` 的全部内容嵌入**上面的文件。这虽然简单，但有明显的两个缺点：

1. **全局名称冲突**：如果多个文件有同名函数，会发生冲突。现代模块系统（如 import）很自然地避免了这一点，各自作用域隔离。
2. **重复包含问题**：同一个文件可能被不同地方反复 include，导致输出代码里有多份相同内容。为解决这一点，最佳实践是加“只包含一次”的条件宏。现代模块系统自动帮我们做了类似的事情。

这个点很重要，下面详细解释它。

---

### JavaScript 模块是单例的

假如新增了 `c.js`：

```js
export function c() {
  return 2;
}
```

现在假设 `a.js` 和 `b.js` 都要用到 `c.js`：

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

如果 `import` 真的和复制粘贴一样，那最终程序里会有两份 `c` 函数。幸运的是，事实并非如此！

JavaScript 模块系统保证了，即便多个位置导入同一个模块，**模块内容只被定义一次**，就像下文的单一文件：

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

也就是说，现代模块系统保证**每个模块里的代码至多只执行一次**，不管被多少地方、多少次 import 进来。

这是模块系统极其关键的设计点，带来了很多好处：

- 最终程序（不论是可执行文件、bundle 还是内存）不会因为重复引用而体积膨胀。
- 每个模块可以在顶层定义“私有状态”，被反复导入时都不会被重建。
- 心智模型更简单——每个模块都是“单例”。如果想某段代码只执行一次，只需写在模块顶层。

底层实现通常就是维护一个 `Map`，用模块文件名作为 key，缓存已加载模块及其所有导出值。任何一个 JS 模块系统实现都会有类似逻辑，例如 [Node.js 源码](https://github.com/nodejs/node/blob/ed2c6965d2f901f3c786f9d24bcd57b2cd523611/lib/internal/modules/esm/loader.js#L114-L139)、[webpack 源码](https://github.com/webpack/webpack/blob/19ca74127f7668aaf60d59f4af8fcaee7924541a/lib/javascript/JavascriptModulesPlugin.js#L1435-L1446)、[Metro (RN) 源码](https://github.com/facebook/metro/blob/15fef8ebcf5ae0a13e7f0925a22d4211dde95e02/packages/metro-runtime/src/polyfills/require.js#L204-L209)。

总结：每个 JavaScript 模块都是单例。无论被多少地方导入，只会执行一次。

刚才讲了多模块，下一步来看多台计算机时会怎样。

---

### 一台计算机上的一个程序

大多数 JavaScript 程序仅在一台计算机执行。

这台机器可能是浏览器，也可能是 Node.js 服务器，甚至某些 JS 运行时本身。但绝大多数 JS 程序都只针对单机环境设计。程序加载、运行、结束。

JavaScript 的模块系统也是为**单机单程序**而设计。流程可以总结如下：

1. 有一个文件作为入口，之前的例子就是 `index.js`，由 JS 引擎从这里启动。
2. 入口文件 import 其他模块，这些模块还可能 import 更多模块。JS 引擎执行这些模块代码，并缓存导出值。
3. 如果遇到已经加载过的模块（比如第二次 import `c.js`），不会重复执行，直接从缓存取出导出内容。

最终，效果等价于把所有模块复制粘贴到一个大文件中，根据需要自动重命名变量，确保每个模块内容只出现一次：

```js
/* c.js */ function c() { return 2; }
/* a.js */ function a() { return c() * 2; }
/* b.js */ function b() { return c() * 3; }

const result = a() + b(); // (2 * 2) + (2 * 3) = 10

console.log(result);
```

所以说，`import` 实质是把某些代码**引入**你的程序。

但如果我们希望前后端都用 JavaScript 呢？（或者说，给应用增加 [JS BFF（Backend For Frontend）](https://overreacted.io/jsx-over-the-wire/#backend-for-frontend) 更好？）

---

### 两台计算机、两个程序

传统模式下，JS 前端和后端对应两套程序，运行在不同计算机上。很多时候甚至分别由两支团队维护。

具体来看，*后端* 负责服务端渲染 HTML 页面（及部分 API）；*前端* 负责页面中的交互逻辑。

后端代码可能如下（`backend/index.js`）：

```js
function server() {
  return (
    `<html>
      <body>
        <button onClick="sayHello()">
          Press me
        </button>
        <script src="/frontend/index.js type="module"></script>
      </body>
    </html>`
  );
}
```

前端代码（`frontend/index.js`）：

```js
function sayHello() {
  alert('Hi!');
}

window['sayHello'] = sayHello;
```

为了体现二者独立性，下面并排写出两个文件：

```js
function server() {
  return (
    `<html>
      <body>
        <button onClick="sayHello()">
          Press me
        </button>
        <script src="/frontend/index.js type="module"></script>
      </body>
    </html>`
  );
}
```

```js
function sayHello() {
  alert('Hi!');
}

window['sayHello'] = sayHello;
```

看看从任意一侧 import 会发生什么。

假如我们从 `backend/index.js` 导入 `a.js` 和 `b.js`：

```js
import { a } from '../a.js';
import { b } from '../b.js';

function server() {
  return (
    `<html>
      <body>
        <button onClick="sayHello()">
          Press me
        </button>
        <script src="/frontend/index.js type="module"></script>
      </body>
    </html>`
  );
}
```

```js
function sayHello() {
  alert('Hi!');
}

window['sayHello'] = sayHello;
```

这时模块被**引入后端**：

```js
/* c.js */ function c() { return 2; }
/* a.js */ function a() { return c() * 2; }
/* b.js */ function b() { return c() * 3; }

function server() {
  return (
    `<html>
      <body>
        <button onClick="sayHello()">
          Press me
        </button>
        <script src="/frontend/index.js type="module"></script>
      </body>
    </html>`
  );
}
```

```js
function sayHello() {
  alert('Hi!');
}

window['sayHello'] = sayHello;
```

假设前端也进行同样的导入：

```js
import { a } from '../a.js';
import { b } from '../b.js';

function sayHello() {
  alert('Hi!');
}

window['sayHello'] = sayHello;
```

这时模块被**引入前端**：

```js
/* c.js */ function c() { return 2; }
/* a.js */ function a() { return c() * 2; }
/* b.js */ function b() { return c() * 3; }

function sayHello() {
  alert('Hi!');
}

window['sayHello'] = sayHello;
```

需要注意**前后端并不共享模块系统**！

这点非常关键。无论从哪边导入，代码都只是**被带入自己那一边**，没有更多特权。两个侧的模块系统是**完全独立**的。每个模块在各自环境下仍然表现为单例，但只限于本环境。

虽然 a.js、b.js、c.js 表面上“复用”了，但其实更准确的说法是前后端各自“拥有”一份这些模块。

这就是一贯的 fullstack 代码共享方式。只是环境代码重用变多后，很容易不小心复用到**不该“跨端”的内容**。

如何约束并控制代码重用？

---

### 构建报错其实是好事

假如有人编辑了 `c.js`，加了只能在后端用的代码，比如用 `fs` 读取文件：

```js
import { readFileSync } from 'fs';

export function c() {
  return Number(readFileSync('./number.txt', 'utf8'));
}
```

这不会影响后端代码：

```js
/* fs.js */ function readFileSync() { /* ... */ }
/* c.js */  function c() { return Number(readFileSync('./number.txt', 'utf8')); }
/* a.js */  function a() { return c() * 2; }
/* b.js */  function b() { return c() * 3; }

function server() { ... }
```

但如果前端编译时遇到 `fs` 会直接报错：

```js
import { readFileSync } from 'fs'; // 🔴 构建错误，无法导入 'fs'

/* c.js */ function c() { return Number(readFileSync('./number.txt', 'utf8')); }
/* a.js */ function a() { return c() * 2; }
/* b.js */ function b() { return c() * 3; }

function sayHello() {
  alert('Hi!');
}

window['sayHello'] = sayHello;
```

其实这正是我们理想中的结果！

当试图“跨端”复用代码时，我们希望能够及早发现哪些代码实际上无法两边都工作。

遇到 API 只在一端可用（比如 `fs` 属于服务端），我们**希望**构建立刻失败，这样就能更早决策如何修正：

1. 可以把 fs 调用移到别的地方。
2. 可以重构 a.js、b.js，让它们不依赖 c.js。
3. 可以让前端不再依赖 a.js、b.js。

**上述方案全都成立**，要怎么选取其实由实际需求决定，没有万能自动方案，这很像解决一次 Git 冲突：过程或许可略闹心，但选择权掌握在开发者手上。

“复用”代码的代价就是如此。幸运的是这带来了很大的灵活性。唯一的副作用，就是报错时你需要决定**要修改哪一个模块**。

本例中我们很幸运，“导入错误”导致了早期构建失败，问题立刻暴露。但如果没报错会怎样？

---

### 服务端专用代码

假如别人修改 `c.js`，直接导入了后端的机密信息：

```js
import { secret } from './secrets.js';

export function c() {
  return secret;
}
```

这样情况可比刚才糟糕得多：前端构建不会报错，`secret` 被无声地加入了前后端代码：

```js
/* secrets.js */ const secret = 12345;
/* c.js */       function c() { return secret; }
/* a.js */       function a() { return c() * 2; }
/* b.js */       function b() { return c() * 3; }

function server() { ... }
```

```js
/* secrets.js */ const secret = 12345;
/* c.js */       function c() { return secret; }
/* a.js */       function a() { return c() * 2; }
/* b.js */       function b() { return c() * 3; }

function sayHello() {
  alert('Hi!');
}

window['sayHello'] = sayHello;
```

这种情况极其危险，但很多 fullstack 应用并没有任何防护，开发者一不小心就把敏感信息拉入了前端！

如何杜绝这种风险？

一个好主意是：仿照上面 fs 案例，引入一个特殊包，比如叫 `server-only`，用于**标记哪些代码永远不能流向前端**。这个包本身没有功能，仅仅是“毒丸”占位。我们让前端打包工具只要碰见它就立刻报错。

标记 secrets.js 只允许在后端使用：

```js
import 'server-only';

export const secret = 12345;
```

这样只要有人间接把 secrets.js 加进前端 bundle，构建就会失败。比如 a.js/b.js → c.js → secrets.js → server-only，这条依赖链一路往上传播“毒丸”：

```js
/* server-only */ /* （后端无实际影响） */
/* secrets.js */  const secret = 12345;
/* c.js */        function c() { return secret; }
/* a.js */        function a() { return c() * 2; }
/* b.js */        function b() { return c() * 3; }
function server() { ... }
```

```js
/* server-only */ /* 🔴 （在前端构建时报错） */
/* secrets.js */  const secret = 12345;
/* c.js */        function c() { return secret; }
/* a.js */        function a() { return c() * 2; }
/* b.js */        function b() { return c() * 3; }

function sayHello() { alert('Hi!'); }
window['sayHello'] = sayHello;
```

这样我们就可以从根源上控制哪些代码**绝不能离开后端**！（具体实现，见 [Next.js 相关代码](https://github.com/vercel/next.js/blob/f684e973f1ddbbdc99cdda9a89070d6d228a1dd7/crates/next-custom-transforms/src/transforms/react_server_components.rs#L640)）

和前文 fs 案例一样，可以采取三种措施：

1. 把 secrets.js import 移到别处。
2. 让 a.js/b.js 不依赖 c.js。
3. 让前端不需要 a.js/b.js。

关键在于，这种机制**能沿依赖链自动传播**。只需要在“敏感文件”加“毒丸”，而不必上层每一层都写。仅当某个模块本地确有特殊兼容性需求时，才需特殊标记。

---

### 仅限客户端的代码

类似于 `server-only` “毒丸”，我们还可以设置镜像版 `client-only`，让服务端构建遇到它就失败（如果后端不打包则可以单独检查）。

假如我们在 c.js 用了浏览器专属 API，可以做如下决定：c.js 只允许前端用：

```js
import 'client-only';

export function c() {
  return Number(prompt('How old are you?'));
}
```

虽然这种场景没泄密那么关键，但有助于更快发现问题。目的是让“某些代码只该在客户端用”这种需求及时变成构建期报错，不等到运行时报晦涩兼容性 bug：

```js
/* client-only */ /* 🔴 （后端构建时报错） */
/* c.js */        function c() { return Number(prompt('How old are you?')); }
/* a.js */        function a() { return c() * 2; }
/* b.js */        function b() { return c() * 3; }
function server() { ... }
```

```js
/* client-only */ /* （前端无实际影响） */
/* c.js */        function c() { return Number(prompt('How old are you?')); }
/* a.js */        function a() { return c() * 2; }
/* b.js */        function b() { return c() * 3; }
function sayHello() { alert('Hi!'); }
window['sayHello'] = sayHello;
```

这里同样三种常规处理方式：

1. 修改 c.js 兼容后端（移除“毒丸”）。
2. 重构 a.js/b.js 不依赖 c.js。
3. 让后端不要依赖 a.js/b.js。

也可以想象更细粒度的 `client-only`、`server-only`，甚至直接由第三方包自身声明哪些 API 只允许在哪端用。比如 React 就通过 [`package.json` 的条件导出（Conditional Exports）](https://nodejs.org/api/packages.html#conditional-exports)实现了类似隔离。

通过上述机制，前后端“边界”变得由 build 系统严格把控，本地声明影响依赖链，风险大为降低。

需要理解的是，这些“毒丸”**不是决定代码导去哪边**，只是防止代码流向不合适的环境。它们仅仅是“限制”，“不准来的自爆荆棘”。

到这一步，我们其实就快得到 RSC 了。

还差最后一点点……

---

### “一个程序，两台计算机”

回顾下我们传统两端各自独立的结构：

```js
function server() { ... }
```

```js
function sayHello() { alert('Hi!'); }
window['sayHello'] = sayHello;
```

现在我们已经形成如下高清晰度心智模型：

- 从任意一侧 import，只是把代码引入本侧。
- 两侧模块系统完全独立。两个侧各自维护单例 cache，互不影响。
- 默认假设代码可重用，特殊文件可用“毒丸”精确声明“禁止进本环境”。这些“毒丸”不起自身迁移作用，只负责限制。

用这种方法，其实已经比传统“混搭”方案安全很多了。

但还留下一点点问题：前后端二者仍然是靠**人为约定**来保证同步，比如后端引用 sayHello，其实是假设前端一定会存在 window.sayHello，但语法层面无法强约束。

```js
function server() { ... /* 假定 sayHello 存在 */ }
```

```js
function sayHello() { alert('Hi!'); }
window['sayHello'] = sayHello;
```

这种模式有点脆弱。

作为读者你可能已经意识到，后端**不能直接导入** sayHello，因为那样会把它连到后端代码里。

有没有办法，语法上**引用** sayHello，但实际并不 import 进来？这正是 [`'use client'` 指令](https://overreacted.io/what-does-use-client-do/) 的“魔法”作用：

```js
import { sayHello } from '../frontend/index.js';

function Server() {
  return (
    <html>
      <body>
        <button onClick={sayHello}>
          Press me
        </button>
      </body>
    </html>
  );
}
```

```js
'use client';

export function sayHello() {
  alert('Hi.');
}
```

这就是 RSC 所带来的“那最后 10%”。

在 RSC 语义下，两边的 import 默认和普通 import“工作方式一致”，但加上 `'use client'` 后，就变成了“打开通往前端的大门”。

加上 `'use client'`，你就告诉 React：“如果我被后端 import，不要把我的代码真的带进服务端，只需要在后端提供一个引用，这样 React 可以最终自动生成 `<script>`，并将其前端激活。”

同理，[`'use server'`](https://overreacted.io/what-does-use-client-do/#use-server) 让前端能“打开后端的大门”，引用服务器模块而不真的把它带入前端代码。

**这些指令不是要求每个前端或后端模块都加一遍——完全没必要也没意义！**它们只是在需要“开门”时打通环境，让你可以“引用”对端的模块。

如果你要把数据从后端传到前端（通过 `<script>`），需要 `'use client'`。如果你要让前端调用后端（API 请求），需要 `'use server'`。剩下的情况，正常 import 即可，仍然局限在本方环境。

---

### 总结

RSC 并没有回避前端和后端各自独立的模块系统。与传统混合 JavaScript 代码共享方案完全一致：要共享的代码实际各自存在于两端。RSC 提供了两套关键机制：

- `import 'client-only'` 和 `import 'server-only'` “毒丸”，让特定文件声明“绝不能跨界”。
- `'use client'` 和 `'use server'` 指令，允许你**引用对端模块**，并在需要时隐式传递数据，但不会真的导入代码。

这样一来，你可以把 RSC 应用视为同时运行于两台计算机的“单程序”：两套独立模块系统，两枚“毒丸”，两扇通道“门”。

等建立起这种“分层”的肌肉记忆，你会慢慢发现，传统的 frontend/backend 目录结构已经完全多余了，因为边界早已清晰、本地声明自动沿依赖链传递，代码布局可以随项目演化灵活变更。

“毒丸”阻止错误代码跨界，指令让你安全高效穿梭于两个世界，其余 import 依然按常规规则运作。

最后要做的，就是修复构建报错。

据说现在的 LLM 也越来越会做这件事了。

