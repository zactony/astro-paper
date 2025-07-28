---
pubDatetime: 2025-07-27T14:01:14.221Z
title: “use client” 到底做了什么？
featured: true
tags:
  - React
  - 翻译
  - dan abramov
description:
  \'use client' 和 'use server' 指令本质上为模块系统开辟了安全、类型化的前后端“传送门”，让你用 import 跨越并显式表达分布式环境下的客户端与服务端逻辑分界。
---

## Table of contents

## 前言
'use client' 和 'use server' 指令本质上为模块系统开辟了安全、类型化的前后端“传送门”，让你用 import 跨越并显式表达分布式环境下的客户端与服务端逻辑分界。[原文在这](https://overreacted.io/what-does-use-client-do/)。

译文从这开始～～

## “use client” 到底做了什么？

React Server Components（RSC）以“无 API 接口”著称。但实际上，它几乎完全基于两条指令实现：

- `'use client'`
- `'use server'`

我想大胆断言，这两个指令的发明，和结构化编程（`if` / `while`）、一等函数、以及 [`async` / `await`](https://tirania.org/blog/archive/2013/Aug-15.html) 属于同一等级。也就是说：这些语法最终会超越 React，成为业界的“常识性”工具。

本质上，服务器*需要*通过 `<script>` 向客户端发送代码，客户端*需要*通过 `fetch` 向服务器请求数据。`'use client'` 和 `'use server'` 指令将这些底层细节抽象出来，带来了一套类型安全、可静态分析的方式，在你的代码里安全地“跳”到另一台计算机上执行某部分逻辑：

- **`'use client'` 本质上是类型安全的 `<script>`。**
- **`'use server'` 本质上是类型安全的 `fetch()`。**

它们一起，让你能在模块系统内表达前后端边界。你的 client/server 应用程序，可以被*当作一个跨两台机器的整体程序组织*，而且不会掩盖网络与序列化的实际鸿沟。这也允许[跨网络的无缝组合](https://overreacted.io/impossible-components/)。

即使你从没打算用 React Server Components，了解这些指令（以及它们的机制）依旧非常值得。它们本质上并不仅限于 React，而真正核心在于**模块系统**本身。

---

### “use server”

先谈 `'use server'`。

假设你有一套后端 API 路由：

```js
async function likePost(postId) {
  const userId = getCurrentUser();
  await db.likes.create({ postId, userId });
  const count = await db.likes.count({ where: { postId } });
  return { likes: count };
}

async function unlikePost(postId) {
  const userId = getCurrentUser();
  await db.likes.destroy({ where: { postId, userId } });
  const count = await db.likes.count({ where: { postId } });
  return { likes: count };
}

app.post('/api/like', async (req, res) => {
  const { postId } = req.body;
  const json = await likePost(postId);
  res.json(json);
});

app.post('/api/unlike', async (req, res) => {
  const { postId } = req.body;
  const json = await unlikePost(postId);
  res.json(json);
});
```

然后前端会调用这些 API 路由:

（为了简化，此处不处理竞态和错误）

典型写法大致如下：

```js
// 字符串调用风格（stringly-typed）
fetch('/api/like', {
  method: 'POST',
  body: JSON.stringify({ postId }),
  // ...
});
```

但是，本质上，我们只是*希望在另一台计算机上调用某个函数*。由于前后端属于两个独立程序，我们唯一能做的就是通过 `fetch` 间接通信。

这时，如果我们将前后端当成是*“同一个程序，分布在两台机器上”*，表达“前端代码想调用后端某个函数”的最直接方式是什么？

其实就是想这样写：

```js
import { likePost, unlikePost } from './backend'; // 这样写在今天的前端并不可行 :(

document.getElementById('likeButton').onclick = async function() {
  const postId = this.dataset.postId;
  if (this.classList.contains('liked')) {
    const { likes } = await unlikePost(postId);
    this.classList.remove('liked');
    this.textContent = likes + ' Likes';
  } else {
    const { likes } = await likePost(postId);
    this.classList.add('liked');
    this.textContent = likes + ' Likes';
  }
};
```

问题是，`likePost` 和 `unlikePost` 不能真的在前端执行；直接 import 后端实现进前端当然不成立。

假设我们有能力在模块级别，**声明导出函数是“服务端可调用”**：

```js
'use server'; // 标记本文件的导出函数可被前端“远程调用”

export async function likePost(postId) { ... }
export async function unlikePost(postId) { ... }
```

后台可以自动帮你完成 HTTP 路由和序列化逻辑。你在前端 import 到的，不是后端真正的函数实现，而是一个自动帮你发 HTTP 的 `async` 封装：

```js
import { likePost, unlikePost } from './backend';

document.getElementById('likeButton').onclick = async function() {
  const postId = this.dataset.postId;
  if (this.classList.contains('liked')) {
    const { likes } = await unlikePost(postId); // 触发 HTTP 调用
    this.classList.remove('liked');
    this.textContent = likes + ' Likes';
  } else {
    const { likes } = await likePost(postId); // 触发 HTTP 调用
    this.classList.add('liked');
    this.textContent = likes + ' Likes';
  }
};
```

这正是 `'use server'` 指令的核心定义。

其实，这一切本质上就是一种为 client-server 场景定制的 [RPC（远程过程调用）](https://en.wikipedia.org/wiki/Remote_procedure_call) 风味：服务器通过 `'use server'` 显式暴露哪些函数可以被前端远程调用。前端 import 到这些函数，实际得到的是一个自动通过 HTTP 向后端请求的 `async` 包裹体。

比如：

```js
// backend.js
'use server';

export async function likePost(postId) { ... }
export async function unlikePost(postId) { ... }
```

```js
// frontend.js
import { likePost, unlikePost } from './backend';

document.getElementById('likeButton').onclick = async function() {
  // ... 逻辑同上
};
```

当然，这种方式有不少值得商榷之处（比如多客户端、版本演化、隐式等问题），但*以“单程序，分布于两台计算机”思维来看*，这种做法赋予了前后端前所未有的直接、类型安全连接。

你可以添加类型来保证契约安全，可以全局查找这些远程 API 是否真正被调用了，甚至能方便地自动检测和清理不再使用的 API。最重要的是，你可以构造完全自包含的双端抽象：“前端”与“后端”紧密耦合，小而美。无需全局命名，代码怎么组织都行，`export` / `import` 随你用。

现在，前端和后端之间的连接，从约定变成了**模块系统层面的语法**。你打开了一扇**通往服务端的“门”**。

---

### “use client”

现在，让我们考虑如何将后端数据传递进前端代码。通常，你会渲染出一个带 `<script>` 的 HTML：

```js
app.get('/posts/:postId', async (req, res) => {
  const { postId } = req.params;
  const userId = getCurrentUser();
  const likeCount = await db.likes.count({ where: { postId } });
  const isLiked = await db.likes.count({ where: { postId, userId } }) > 0;
  const html = `<html>
    <body>
      <button
        id="likeButton"
        className="${isLiked ? 'liked' : ''}"
        data-postid="${Number(postId)}">
        ${likeCount} Likes
      </button>
      <script src="./frontend.js></script>
    </body>
  </html>`;
  res.text(html);
});
```

浏览器加载 `<script>` 后即可附加交互逻辑：

```js
document.getElementById('likeButton').onclick = async function() { ... };
```

但问题来了：

1. 前端代码往往不希望是“全局”的——理想情况下，每个 Like 按钮都能有自己的状态。
2. 显示逻辑最好也能在 HTML 模板和事件处理器间统一。

这很好解决——用组件模式重写即可，例如：

```js
function LikeButton({ postId, likeCount, isLiked }) {
  function handleClick() { ... }
  return (
    <button className={isLiked ? 'liked' : ''}>
      {likeCount} Likes
    </button>
  );
}
```

纯客户端渲染时，后端只需传递初始 props：

```js
app.get('/posts/:postId', async (req, res) => {
  // ...业务逻辑
  const html = `<html>
    <body>
      <script src="./frontend.js></script>
      <script>
        const output = LikeButton(${JSON.stringify({
          postId,
          likeCount,
          isLiked
        })});
        render(document.body, output);
      </script>
    </body>
  </html>`;
  res.text(html);
});
```

此时 LikeButton 组件能用上这些 props：

```js
function LikeButton({ postId, likeCount, isLiked }) {
  function handleClick() { ... }
  return (
    <button className={isLiked ? 'liked' : ''}>
      {likeCount} Likes
    </button>
  );
}
```

你会发现，后端明显是*在向前端传递信息*，但本质上依然还是“字符串拼接”的数据流。

我们实际写的是“让浏览器加载 frontend.js，然后找到里头的 LikeButton，再把 props 组装好的 JSON 传进去”。

所以要是我们能直接用 import 表达这种关系，不用字符串揉合呢？

```js
import { LikeButton } from './frontend';

app.get('/posts/:postId', async (req, res) => {
  // ...
  const jsx = (
    <html>
      <body>
        <LikeButton
          postId={postId}
          likeCount={likeCount}
          isLiked={isLiked}
        />
      </body>
    </html>
  );
  // ...
});
```

```js
'use client'; // 标记：本文件导出可被后端渲染

export function LikeButton({ postId, likeCount, isLiked }) {
  function handleClick() { ... }
  return (
    <button className={isLiked ? 'liked' : ''}>
      {likeCount} Likes
    </button>
  );
}
```

此时虽然跨越了两套运行时环境，但我们在语法层把后端“传递信息”与前端“接收信息”用 import 直连了。

此时，从后端 import 一个带 `'use client'` 的组件，其实你拿到的并不是 LikeButton 真正实现，而是**一个“前端引用”**——底层实际可以转化为 `<script>` 注入：

JSX 如下：

```js
import { LikeButton } from './frontend';

<html>
  <body>
    <LikeButton postId={42} likeCount={8} isLiked={true} />
  </body>
</html>
```

被内部转换为：

```js
{
  type: "html",
  props: {
    children: {
      type: "body",
      props: {
        children: {
          type: "/src/frontend.js#LikeButton", // —> 客户端组件引用
          props: {
            postId: 42,
            likeCount: 8,
            isLiked: true
          }
        }
      }
    }
  }
}
```

如此一来，RSC 就可以根据这个“客户端引用”，自动生成 `<script>`，并调用合适函数：

```html
<script src="./frontend.js"></script>
<script>
  const output = LikeButton({
    postId: 42,
    likeCount: 8,
    isLiked: true
  });
</script>
```

甚至我们可以可选地在服务器预渲染初始 HTML，然后再激活前端 JS：

```html
<!-- 可选：初始 HTML -->
<button class="liked">
  8 Likes
</button>
<!-- 交互逻辑 -->
<script src="./frontend.js"></script>
<script>
  const output = LikeButton({
    postId: 42,
    likeCount: 8,
    isLiked: true
  });
  // ...
</script>
```

综上，这种链路不仅统一类型、对接顺畅，还能让编译器/IDE 直接识别跨端引用、类型校验、全局搜索引用等一系列开发体验的质变提升。

和 `'use server'` 一样，`'use client'` 把“后端引用前端”的能力变成了**模块语法上的一扇门**（不是靠约定，不是靠纯手工 glue code！）。

---

### 两个世界，两个“门”

这就是 `'use client'` 和 `'use server'` 的精髓：**它们不是用来“标记代码属于前端或后端”的！**

它们的设计目的是——**允许你从一个环境“打开通往另一个环境的门”**：

- **`'use client'` 是把客户端函数暴露给服务端。** 服务端 import 到的是“/src/frontend.js#LikeButton”这样的引用，既可渲染为 JSX 标签，本质上底层转为 `<script>` 触发（可选支持 SSR 预渲染）。
- **`'use server'` 是把服务端函数暴露给客户端。** 前端 import 到的其实是一个内部做 HTTP 调用的 `async` 封装。

这些指令在**模块系统内直接表达了“网络鸿沟”。**让你能用代码定义“这其实是分布在两台机器上的单一程序”，同时完全正视并拥抱环境的独立性。

最终，它们**让你能在代码中纵横交织地复用跨端抽象**，最大限度组合两端逻辑，提升可维护性和复用性。这一做法的理论基础，其实可以超越 React 和 JS，成为下一代分布式开发主流范式。

服务端和客户端是同一程序的两面，时空隔绝，不能真正直接 import 对方，但指令帮你“开门”对接。服务器可以通过 `'use client'` 渲染组件到前端（生成 `<script>`），客户端可以通过 `'use server'` 安全发起“函数级别”的精细 API 调用。

import 是最直接、类型安全、可组合的桥梁。指令让这一桥梁成为现实。

是不是很妙？

---

### P.S.

这里有一张架构图，可以用在你的演示文稿里：

![黑白太极图示意（Two Worlds, Two Doors）](https://overreacted.io/what-does-use-client-do/diagram.png)