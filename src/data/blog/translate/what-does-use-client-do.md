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

## "use client" 做了什么？

React Server Components（RSC）以没有 API 接口而著称。它几乎完全是一种来源于两个指令的编程范式：

- `'use client'`
- `'use server'`

我大胆地认为，这两个指令的发明应该和结构化编程（`if` / `while`）、一等函数、[以及 `async` / `await`](https://tirania.org/blog/archive/2013/Aug-15.html) 一样划时代。换句话说，我认为它们会超越 React、成为常识。

服务器*需要*向客户端发送代码（通过 `<script>` 发送）。客户端*需要*与服务器通信（通过 `fetch`）。`'use client'` 和 `'use server'` 指令将这些需求抽象出来，提供了一种一等的、类型化的、可静态分析的方式，把控制权传递到你代码库中另一台计算机上的某部分：

- **`'use client'` 就是类型化的 `<script>`。**
- **`'use server'` 就是类型化的 `fetch()`。**

这两个指令一起，让你能够在模块系统*内部*表达前后端边界。它们让你能够把客户端/服务器应用*视为一个跨越两台机器的整体程序*，同时不会忽略网络和序列化的本质分界。反过来，这就允许了[跨网络的无缝组合](https://overreacted.io/impossible-components/)。

即使你永远不打算用 React Server Components，我也认为你应该了解这些指令以及它们的工作原理。它们甚至和 React 没太大关系。

它们关乎模块系统。

---

### 'use server'

首先，让我们来看 `'use server'`。

假设你正在写一个有一些 API 路由的后端服务器：

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

然后你有一些前端代码去调用这些 API 路由：

```
document.getElementById('likeButton').onclick = async function() { const postId = this.dataset.postId; if (this.classList.contains('liked')) { const response = await fetch('/api/unlike', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ postId }) }); const { likes } = await response.json(); this.classList.remove('liked'); this.textContent = likes + ' Likes'; } else { const response = await fetch('/api/like', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ postId, userId }) }); const { likes } = await response.json(); this.classList.add('liked'); this.textContent = likes + ' Likes'; }}
```

（为了简单起见，这个例子没有处理并发条件和错误。）

这段代码没问题，但它是[“字符串类型的”](https://www.hanselman.com/blog/stringly-typed-vs-strongly-typed)。我们的本意其实是*在另一台计算机上调用一个函数*。但是，因为前端和后端是两个独立的程序，我们只能用 `fetch` 来表达。

现在假设我们把前端和后端看作*在两台机器上分割的一个整体程序*。我们该如何表达“一段代码想调用另一段代码”？最直接的做法是什么？

如果我们暂时不考虑后端和前端“必须”怎么写，我们会发现，自己真正想表达的，其实就是让前端代码去*调用* `likePost` 和 `unlikePost`：

```js
import { likePost, unlikePost } from './backend'; // 这其实不管用 :(

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

问题当然在于，`likePost` 和 `unlikePost` 不能在前端真正执行。我们不能把后端实现“真的”导入到前端。在前端直接导入后端的实现本身就是*无意义*的。

假设我们有方法在模块级别标记 `likePost` 和 `unlikePost` 作为*服务器导出*：

```js
'use server'; // 标记所有导出可以“从前端调用”

export async function likePost(postId) {
  const userId = getCurrentUser();
  await db.likes.create({ postId, userId });
  const count = await db.likes.count({ where: { postId } });
  return { likes: count };
}

export async function unlikePost(postId) {
  const userId = getCurrentUser();
  await db.likes.destroy({ where: { postId, userId } });
  const count = await db.likes.count({ where: { postId } });
  return { likes: count };
}
```

此时我们可以在后台自动建立 HTTP endpoint。现在有了一个导出函数用于网络调用的语法，前端导入它们就有了*意义*——`import` 它们简单地返回会自动发起 HTTP 请求的 `async` 函数：

```js
import { likePost, unlikePost } from './backend';

document.getElementById('likeButton').onclick = async function() {
  const postId = this.dataset.postId;
  if (this.classList.contains('liked')) {
    const { likes } = await unlikePost(postId); // HTTP call
    this.classList.remove('liked');
    this.textContent = likes + ' Likes';
  } else {
    const { likes } = await likePost(postId); // HTTP call
    this.classList.add('liked');
    this.textContent = likes + ' Likes';
  }
};
```

这正是 `'use server'` 指令的作用。

这并不是新发明——[RPC（远程过程调用）已经存在了几十年](https://en.wikipedia.org/wiki/Remote_procedure_call)。这只是对某种特定形式客户端-服务器应用下的 RPC 的实现，服务器代码可以用 `'use server'` 指定“服务器导出”。从服务器代码导入，行为和普通 `import` 一样，但从客户端代码导入，得到的是实际会发 HTTP 请求的 `async` 函数。

再看看这一对文件：

```js
'use server'; // 标记所有导出都可以“从前端调用”

export async function likePost(postId) {
  const userId = getCurrentUser();
  await db.likes.create({ postId, userId });
  const count = await db.likes.count({ where: { postId } });
  return { likes: count };
}

export async function unlikePost(postId) {
  const userId = getCurrentUser();
  await db.likes.destroy({ where: { postId, userId } });
  const count = await db.likes.count({ where: { postId } });
  return { likes: count };
}
```

```js
import { likePost, unlikePost } from './backend';

document.getElementById('likeButton').onclick = async function() {
  const postId = this.dataset.postId;
  if (this.classList.contains('liked')) {
    const { likes } = await unlikePost(postId); // HTTP call
    this.classList.remove('liked');
    this.textContent = likes + ' Likes';
  } else {
    const { likes } = await likePost(postId); // HTTP call
    this.classList.add('liked');
    this.textContent = likes + ' Likes';
  }
};
```

你可能有疑虑——确实，它不支持多个 API 消费者（除非全在同一个代码库）；确实，需要考虑版本和部署问题；确实，比写 `fetch` 更隐式。

但如果你把后端和前端看成*在两台计算机上分裂的一个程序*，你就很难再回头。这时这两个模块之间是直接的联系。你可以加类型更严格地约束它们的协议（并保证类型是可序列化的）。你可以用“查找所有引用”看到服务端有哪些函数在客户端被用到。未用到的 endpoint 能被自动标记或通过无用代码分析移除。

最重要的是，你现在可以创建完全自包含的双端抽象——一个“前端”紧密绑定相应的“后端”实现。你不用担心路由爆炸——服务端和客户端的分割可以像你的抽象结构一样细粒度。也不用全局命名，只用你需要的地方 `export` 和 `import` 就行了。

`'use server'` 指令让前端和后端的联系变为*语法层面*的——不再依赖约定，而是*体现在模块系统里*。

它打开了指向服务器的一扇*门*。

---

### 'use client'

现在假设你想要把一些信息从后端传递到前端代码。例如，你可以用 `<script>` 渲染一些 HTML：

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

浏览器会加载 `<script>`，并挂载交互逻辑：

```js
document.getElementById('likeButton').onclick = async function() {
  const postId = this.dataset.postId;
  if (this.classList.contains('liked')) {
    // ...
  } else {
    // ...
  }
};
```

这样也能跑，但还不够理想。

首先，你可能不想让前端逻辑成为“全局”——理想情况下每个 Like 按钮都应该接收自己的数据并维护自己的本地状态。你也可能希望 HTML 模板中的显示逻辑和 JS 事件处理逻辑能统一。

我们知道该怎么做。这正是组件库要解决的！让我们把前端逻辑重写成声明式的 `LikeButton` 组件：

```js
function LikeButton({ postId, likeCount, isLiked }) {
  function handleClick() {
    // ...
  }
  return (
    <button className={isLiked ? 'liked' : ''}>
      {likeCount} Likes
    </button>
  );
}
```

为了简单，这里先采用纯客户端渲染。这样服务器要做的，就是传递初始 props：

```js
app.get('/posts/:postId', async (req, res) => {
  const { postId } = req.params;
  const userId = getCurrentUser();
  const likeCount = await db.likes.count({ where: { postId } });
  const isLiked = await db.likes.count({ where: { postId, userId } }) > 0;
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

`LikeButton` 会用到这些 props：

```js
function LikeButton({ postId, likeCount, isLiked }) {
  function handleClick() {
    // ...
  }
  return (
    <button className={isLiked ? 'liked' : ''}>
      {likeCount} Likes
    </button>
  );
}
```

这种方案其实正是 React 在客户端路由诞生前集成到服务端渲染应用的方法：你需要给页面写一个 `<script>`，包含你的前端代码，再写另一个 `<script>`，把代码需要的初始数据（即 props）注入进去。

我们再想想这段代码。这里很有意思：后端很明显在*传递信息*给前端代码。但信息的传递本质上还是*基于字符串*的！

发生了什么事？

```js
app.get('/posts/:postId', async (req, res) => {
  // ...
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

我们实际是在说：让浏览器加载 `frontend.js`，然后从中找到 `LikeButton`，再把这个 JSON 传给它。

所以要是我们能*直接这么写*就好了？

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
'use client'; // 将所有导出标记为“可以从后端渲染”

export function LikeButton({ postId, likeCount, isLiked }) {
  function handleClick() {
    // ...
  }
  return (
    <button className={isLiked ? 'liked' : ''}>
      {likeCount} Likes
    </button>
  );
}
```

我们在这里做了一个概念上的飞跃，但请耐心看下去。我们的意思是，虽然依然有两个独立的运行环境（前端和后端），但我们把它们看成*一个整体程序*，而不是截然分开的两组代码。

这就是为什么我们要在传递信息的地方（后端）和需要接收的地方（前端）建立*语法层面的连接*。最自然的方式还是普通的 `import`。

注意：在后端导入加了 `'use client'` 声明的文件时，拿到的并不是 `LikeButton` 真实函数本体，而是*客户端引用*——底层后续可直接转换为 `<script>`。

我们来看它的原理。

这个 JSX 代码：

```js
import { LikeButton } from './frontend'; // "/src/frontend.js#LikeButton"

<html>
  <body>
    <LikeButton
      postId={42}
      likeCount={8}
      isLiked={true}
    />
  </body>
</html>
```

会生成这样的 JSON：

```js
{
  type: "html",
  props: {
    children: {
      type: "body",
      props: {
        children: {
          type: "/src/frontend.js#LikeButton", // 客户端引用！
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

有了这些信息——这个*客户端引用*——我们就能生成 `<script>`，加载正确的文件、调用正确的方法：

```html
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

事实上，我们拥有了足够的信息，可以在服务器上执行同一个函数，提前生成初始 HTML（这是纯客户端渲染丢失的部分）：

```html
<!-- 可选的初始 HTML -->
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

预渲染初始 HTML 是可选的，但用的是相同原语。

了解了*它的原理*以后，再回看这段代码：

```js
import { LikeButton } from './frontend'; // "/src/frontend.js#LikeButton"

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
'use client'; // 将所有导出标记为“可以从后端渲染”

export function LikeButton({ postId, likeCount, isLiked }) {
  function handleClick() {
    // ...
  }
  return (
    <button className={isLiked ? 'liked' : ''}>
      {likeCount} Likes
    </button>
  );
}
```

如果你放下对前端后端该如何交互的既有观念，你会发现这里发生了一些很特别的事情。

后端代码通过带有 `'use client'` 的 `import`*引用*了前端代码。换句话说，它用模块系统*内部*，表达了一个从“发送 `<script>` 的地方”到“真实存在于 `<script>` 里的地方”的直接联系。因为有了这种直接联系，它可以进行类型检查，你可以使用“查找所有引用”，所有工具都能分析这种引用。

和 `'use server'` 一样，`'use client'` 让前后端的联系变成了*语法层面的*。`'use server'` 是从客户端到服务器开一扇门，`'use client'` 则是从服务器到客户端开一扇门。

就像有两个世界和两扇门。

---

### 两个世界，两扇门

这就是为什么 `'use client'` 和 `'use server'` 不应该被当做“把代码标记为属于客户端或服务器”的手段。它们做的不是这个。

它们让你能够*从一个环境打开通往另一个环境的门*：

- **`'use client'` 将客户端函数*暴露给服务端*。** 底层，后端看到的是像 `'/src/frontend.js#LikeButton'` 这样的引用名。它们可以被渲染成 JSX 标签，最终变成 `<script>` 标签。（你也可以选择性地在服务器上预运行这些脚本，获得初始 HTML。）
- **`'use server'` 将服务器函数*暴露给客户端*。** 底层，前端拿到的是调用后端 HTTP 的 `async` 函数。

这些指令在你的模块系统*内部*表达了网络鸿沟。它们让你把前后端应用作为*跨越两个环境的一个整体程序*来描述。

它们诚实并充分地承认这两个环境没有任何执行上下文共享——这也是 `import` 都不会真的执行代码的原因。相反，它们只是让一侧*引用*另一侧的代码，并向它传递信息。

**它们一起，让你在代码中“编织”两个世界，构建[包含双端逻辑的可复用抽象](https://overreacted.io/impossible-components/)。** 但我认为，这套模式超越了 React，甚至超越了 JavaScript。实质就是“模块层面的 RPC”，有一个镜像机制，用来把更多代码安全地传递到客户端。

服务器和客户端是同一个程序的两面。它们被时间和空间隔开，不能共享执行上下文，不能真的直接 import 彼此。指令能够“跨越时空开门”：服务器可以通过 `'use client'` *渲染*客户端为 `<script>`，客户端通过 `'use server'` *对话*服务器（HTTP 调用）。而 `import` 是表达这一切最直观的方式，指令让你可以直接这么做。

合情合理，不是吗？

---

### P.S.

这里有一张架构小图，可以用于你的幻灯片：

![A black and white yin yang symbol](https://overreacted.io/what-does-use-client-do/diagram.png)
