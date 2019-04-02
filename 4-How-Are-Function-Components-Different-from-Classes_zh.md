## React 函数式组件和类组件有何不同？

> - 原文地址：https://overreacted.io/how-are-function-components-different-from-classes/
> - 原文作者：[Dan Abramov](https://github.com/gaearon)
> - Markdown 地址：https://github.com/gaearon/overreacted.io/edit/master/src/pages/how-are-function-components-different-from-classes/index.md
> - 译者：[Yangfan2016](https://github.com/Yangfan2016)

一段时间内，权威的答案是 “类” 可以提供更多的特性的访问（比如，`state`）。而 [Hooks](https://reactjs.org/docs/hooks-intro.html) 就不一样了

或许你已了解到一种最佳实践。哪一种内？大多数基准测试都[不完美](https://medium.com/@dan_abramov/this-benchmark-is-indeed-flawed-c3d6b5b6f97f?source=your_stories_page---------------------------)，因此我得小心谨慎从它们中得出结论。性能本质上取决于代码做了什么而不是你选择函数还是类。我们观察到，它们的性能差别微不足道，尽管优化策略有些[不同](https://reactjs.org/docs/hooks-faq.html#are-hooks-slow-because-of-creating-functions-in-render)

无论哪种情况下，我们都[不推荐](https://reactjs.org/docs/hooks-faq.html#should-i-use-hooks-classes-or-a-mix-of-both)重写你已存在的组件，除非你有其他原因，和不介意做第一个 “吃螃蟹” 的人。`Hooks` 仍然是新概念（就像 2014 年的 React），并且那些 “最佳实践” 至今都没有在教程里找到

那么给我们留下了什么内？React 函数和类有本质的区别吗？当然，它们有（在心智模型层面上）。**这篇文章，我将找到它们之间的最大的不同。** 它自从 2015 的函数式组件被[引入](https://reactjs.org/blog/2015/09/10/react-v0.14-rc1.html#stateless-function-components)就存在了，但是它经常被忽视：

>**函数式组件捕获渲染的值**

让我们拆开这个概念来理解

---

**注意：这篇文章并不做类或函数的价值评判。我只是描述下这两种编程模式在 React 中的不同。更多采用函数式的问题，请查阅 [Hooks FAQ](https://reactjs.org/docs/hooks-faq.html#adoption-strategy)**

---

仔细看看这个组件：

```jsx
function ProfilePage(props) {
  const showMessage = () => {
    alert('Followed ' + props.user);
  };

  const handleClick = () => {
    setTimeout(showMessage, 3000);
  };

  return (
    <button onClick={handleClick}>Follow</button>
  );
}
```

它会显示一个用 `setTimeout` 模拟网络请求的按钮，并且之后会显示一个确认框。例如，如果 `props.user` 的值是 `'Dan'`，那么在 3 秒之后会显示 `'Followed Dan'`。足够简单

*（注意，在这个例子中，不论我使用箭头函数还是声明式函数都没有关系。`function handleClick()` 都会以相同的方式正确执行）*

我们如何用 “类” 重写内？原生翻译看起来可能是这样的：

```jsx
class ProfilePage extends React.Component {
  showMessage = () => {
    alert('Followed ' + this.props.user);
  };

  handleClick = () => {
    setTimeout(this.showMessage, 3000);
  };

  render() {
    return <button onClick={this.handleClick}>Follow</button>;
  }
}
```

通常认为这两个代码片段是等价的。人们总是在这些模式进行自由的重构，而从来没有注意到它们的意义：

![找出两个版本的不同](https://overreacted.io/wtf-1d3c7a341ee3fcadc79df00e7d872e4b.gif)

**然而，这两个代码片段只有细微的差别。** 仔细看看它们，你发现不同了吗？就我个人而言，我花了一些时间才找到

**前方剧透，如果你要自己弄明白的话，请看这个[在线 demo](https://codesandbox.io/s/pjqnl16lm7)** ，文章的剩下的部分都在解释它们的差别和为何它如此重要

---

在继续之前，我想强调下这篇文章描述的区别和 React Hooks 半毛钱关系都没有。甚至上面的例子里都没有提及 Hooks！

文章都是关于 React 中所有函数和 “类” 的差异。如果你计划在 React app 中更频繁的使用函数，你可能更想理解它

---

**我们将用一个 React 应用中常见的 "类" bug 来图解这个区别**

打来这个 **[sandbox 例子](https://codesandbox.io/s/pjqnl16lm7)**，有一个简介选择器和两个 `信息页面`（每个都有一个关注按钮） 

试着按下面的顺序触发这两个按钮：

1. **单击** 其中一个关注按钮  
2. 在 3 秒过去之前 **改变** 已选的简介  
3. **读取** 警告框的内容  

你会发现一个奇怪的差异：

* 使用上面的 **function** `信息页面` ，点击关注 Dan 的简介后，然后导航到 Sophie 的简介依然弹框显示 `'Followed Dan'`

* 使用上面的 **class** `信息页面` ，它会弹框显示 `'Followed Sophie'`：

![Demonstration of the steps](https://overreacted.io/bug-386a449110202d5140d67336a0ade5a0.gif)

---


在这个例子中，第一个行为是正确的。**如果我关注了一个人，然后导航到另一个人的简介页面，我的组件不应该困惑我到底关注了谁。** 这个  “类” 实现明显是个 bug 

*（虽然你完全可以这样关注 [Sophie](https://mobile.twitter.com/sophiebits)）*

---

那么为何我们的 “类” 例子会如此表现内？

让我们仔细看看我们 “类” 方法 `showMessage`：

```jsx{3}
class ProfilePage extends React.Component {
  showMessage = () => {
    alert('Followed ' + this.props.user);
  };
```

这个 “类” 方法从 `this.props.user` 读取。Props 在 React 里是不可变的，因此它们永远也不会改变。**然而，`this` *是*，总是多变的** 

事实上，这就是 “类” 里 `this` 的全部目的。React 自己会随着时间改变，以至于你可以在 `render` 和生命周期方法获取到最新的版本

因此如果我们在请求期间重新渲染我们的组件，`this.props` 会改变。`showMessage` 方法获取到 `user` 将是 “更新的” `props`

这个例子揭露出一个关于用户界面本质的有趣的观察。如果我们说 UI 概念上是当前应用状态的函数，**那么事件处理器就是渲染结果的一部分（就像可视化输出一样）**。我们事件处理器 “属于” 一个拥有特别 props 和 state 的特别渲染

然而，这些回调读取 `this.props` 超时会断开这个联系。我们的 `showMessage` 回调不能 “绑” 到任何特定的渲染，那么它就会 “丢失” 正确的 props。读取 `this` 的链接就会被切断  

---

**我们假设函数式组件不存在。** 我们该如何解决这个问题内？

我们想以某种方式 “修复” 带着正确的 props 的 `render` 和 `showMessage` 回调读取它们的联系。在某个地方，`props` 可能会丢失

一种方式是在事件处理更早读取 `this.props`，并且显示通过超时完成处理器传递进去：

```jsx{2,7}
class ProfilePage extends React.Component {
  showMessage = (user) => {
    alert('Followed ' + user);
  };

  handleClick = () => {
    const {user} = this.props;
    setTimeout(() => this.showMessage(user), 3000);
  };

  render() {
    return <button onClick={this.handleClick}>Follow</button>;
  }
}
```

起[作用](https://codesandbox.io/s/3q737pw8lq)了。然而，这种方式随着时间变化显著造成代码更冗余和容易出错。如果我们需要不止一个 prop？如果我们也需要访问这个 state？**如果 `showMessage` 调用其他方法，而且这个方法也需要读取 `this.props.something` 或 `this.state.something`，我们又遇到了同样的问题。** 因此我们把 `this.props` 和 `this.state` 作为参数从 `showMessage` 传递给每个它调用的方法


这么做会破坏 “类” 正常提供的工程学。它也难以记住和执行，这就是为何人们常常满足于 bug 的原因

同样，在 `handleClick` 里内嵌 `alert` 代码并不能解决更大的问题。我们想用一种方式结构化代码允许被更多方法拆分，*但是* 还是要读取调用相应渲染的 props 和 state。**这个问题甚至都不是 React 独有的（你可以把这个可变对象如 `this`，放到任何一个 UI 库里都可以重现）**

或许，我们可以在构造器中 *绑定* 这个方法？

```jsx{4-5}
class ProfilePage extends React.Component {
  constructor(props) {
    super(props);
    this.showMessage = this.showMessage.bind(this);
    this.handleClick = this.handleClick.bind(this);
  }

  showMessage() {
    alert('Followed ' + this.props.user);
  }

  handleClick() {
    setTimeout(this.showMessage, 3000);
  }

  render() {
    return <button onClick={this.handleClick}>Follow</button>;
  }
}
```

不，它不会解决任何问题。记住，问题所在是我们读取 `this.props` 太迟了（不是我们使用的语法有错！）**然而，如果我们完全依赖 JavaScript 闭包可以解决这个问题** 

闭包总是被回避，因为它[难](https://wsvincent.com/javascript-closure-settimeout-for-loop/)以理解，值会随着时间变化。但是在 React 中，props 和 state 是不可变的！（或者至少，它是强烈推荐）消除了
闭包的主要阻碍

这意味着如果你在一个特别的渲染靠近 props 或 state，你总是可以指望它们完全相同：

```jsx{3,4,9}
class ProfilePage extends React.Component {
  render() {
    // 捕获这个 props！
    const props = this.props;

    // 注意：我们在 *render 内部*。
    // 这些不是类方法
    const showMessage = () => {
      alert('Followed ' + props.user);
    };

    const handleClick = () => {
      setTimeout(showMessage, 3000);
    };

    return <button onClick={handleClick}>Follow</button>;
  }
}
```


**你已经捕获了渲染时的 props：**

这种方法，任何代码内置在它里面（包含在 `showMessage` 里），保证看到特定渲染的 props。React 不再 “移动我们的奶酪”

**我们可以在里面添加我们想要的任何数量的辅助函数，并且它们都将使用捕获的 prop 和 state。** 闭包营救了我们！


---

[上面这个例子](https://codesandbox.io/s/oqxy9m7om5)是对的，但是看起来有点怪，如果你在 `render` 里定义了函数而不是使用 “类” 方法，那么在 “类” 里这么做有什么意义内？

事实上，我们可以通过移除它周围的 “壳” 来简化代码：

```jsx
function ProfilePage(props) {
  const showMessage = () => {
    alert('Followed ' + props.user);
  };

  const handleClick = () => {
    setTimeout(showMessage, 3000);
  };

  return (
    <button onClick={handleClick}>Follow</button>
  );
}
```

就像上面这样，`props` 依然可以被捕获（React 把它们作为参数传递进去）。**不像 `this`，`props` 对象永远不可能因 React 而发生改变**

如果你把函数定义里的 `props` 结构的话，会更清晰：

```jsx{1,3}
function ProfilePage({ user }) {
  const showMessage = () => {
    alert('Followed ' + user);
  };

  const handleClick = () => {
    setTimeout(showMessage, 3000);
  };

  return (
    <button onClick={handleClick}>Follow</button>
  );
}
```

当父组件携带不同的 props 渲染 `ProfilePage` 时，React 会再次调用 `ProfilePage` 函数。但是我们已经点击 “属于” 前一次渲染它自己的 `user` 的值和读取 `showMessage` 回调那个事件处理程序。它们都原封不动

这就是为何在这个版本的 [demo](https://codesandbox.io/s/pjqnl16lm7) 函数中，点击关注 Sophie 的简介和改变选项到 Sunil 依然弹出 `'Followed Sophie'`：

这个行为是对的。*（尽管你可能也想关注 [Sunil](https://mobile.twitter.com/threepointone)）*

---

现在我们就理解了 React 中 “类” 和函数的最大差别：

>**函数组件捕获到已渲染的值**

**对于 Hooks，同样的原则也适用于 state。** 仔细看看下面这个例子：

```jsx
function MessageThread() {
  const [message, setMessage] = useState('');

  const showMessage = () => {
    alert('You said: ' + message);
  };

  const handleSendClick = () => {
    setTimeout(showMessage, 3000);
  };

  const handleMessageChange = (e) => {
    setMessage(e.target.value);
  };

  return (
    <>
      <input value={message} onChange={handleMessageChange} />
      <button onClick={handleSendClick}>Send</button>
    </>
  );
}
```

（这里有一个[在线 demo](https://codesandbox.io/s/93m5mz9w24)）


虽然这个消息 app UI 并不是很好，但它阐述了相同的原理：如果我发送了一个特别的消息，那么组件不应该困惑发送出去的消息实际是什么。这个函数组件的 `message` 捕获了 “属于” 返回浏览器调用的点击处理程序的渲染的 state。因此，`message` 被设置为我点击 “发送” 那一时刻的输入框的值

---

So we know functions in React capture props and state by default. **But what if we *want* to read the latest props or state that don’t belong to this particular render?** What if we want to [“read them from the future”](https://dev.to/scastiel/react-hooks-get-the-current-state-back-to-the-future-3op2)?

In classes, you’d do it by reading `this.props` or `this.state` because `this` itself is mutable. React mutates it. In function components, you can also have a mutable value that is shared by all component renders. It’s called a “ref”:

```js
function MyComponent() {
  const ref = useRef(null);
  // You can read or write `ref.current`.
  // ...
}
```

However, you’ll have to manage it yourself.

A ref [plays the same role](https://reactjs.org/docs/hooks-faq.html#is-there-something-like-instance-variables) as an instance field. It’s the escape hatch into the mutable imperative world. You may be familiar with “DOM refs” but the concept is much more general. It’s just a box into which you can put something.

Even visually, `this.something` looks like a mirror of `something.current`. They represent the same concept.

By default, React doesn’t create refs for latest props or state in function components. In many cases you don’t need them, and it would be wasted work to assign them. However, you can track the value manually if you’d like:

```jsx{3,6,15}
function MessageThread() {
  const [message, setMessage] = useState('');
  const latestMessage = useRef('');

  const showMessage = () => {
    alert('You said: ' + latestMessage.current);
  };

  const handleSendClick = () => {
    setTimeout(showMessage, 3000);
  };

  const handleMessageChange = (e) => {
    setMessage(e.target.value);
    latestMessage.current = e.target.value;
  };
```

If we read `message` in `showMessage`, we’ll see the message at the time we pressed the Send button. But when we read `latestMessage.current`, we get the latest value — even if we kept typing after the Send button was pressed.

You can compare the [two](https://codesandbox.io/s/93m5mz9w24) [demos](https://codesandbox.io/s/ox200vw8k9) to see the difference yourself. A ref is a way to “opt out” of the rendering consistency, and can be handy in some cases.

Generally, you should avoid reading or setting refs *during* rendering because they’re mutable. We want to keep the rendering predictable. **However, if we want to get the latest value of a particular prop or state, it can be annoying to update the ref manually.** We could automate it by using an effect:

```js{4-8,11}
function MessageThread() {
  const [message, setMessage] = useState('');

  // Keep track of the latest value.
  const latestMessage = useRef('');
  useEffect(() => {
    latestMessage.current = message;
  });

  const showMessage = () => {
    alert('You said: ' + latestMessage.current);
  };
```

(Here’s a [demo](https://codesandbox.io/s/yqmnz7xy8x).)

We do the assignment *inside* an effect so that the ref value only changes after the DOM has been updated. This ensures our mutation doesn’t break features like [Time Slicing and Suspense](https://reactjs.org/blog/2018/03/01/sneak-peek-beyond-react-16.html) which rely on interruptible rendering.

Using a ref like this isn’t necessary very often. **Capturing props or state is usually a better default.** However, it can be handy when dealing with [imperative APIs](/making-setinterval-declarative-with-react-hooks/) like intervals and subscriptions. Remember that you can track *any* value like this — a prop, a state variable, the whole props object, or even a function.

This pattern can also be handy for optimizations — such as when `useCallback` identity changes too often. However, [using a reducer](https://reactjs.org/docs/hooks-faq.html#how-to-avoid-passing-callbacks-down) is often a [better solution](https://github.com/ryardley/hooks-perf-issues/pull/3). (A topic for a future blog post!)

---

In this post, we’ve looked at common broken pattern in classes, and how closures help us fix it. However, you might have noticed that when you try to optimize Hooks by specifying a dependency array, you can run into bugs with stale closures. Does it mean that closures are the problem? I don’t think so.

As we’ve seen above, closures actually help us *fix* the subtle problems that are hard to notice. Similarly, they make it much easier to write code that works correctly in the [Concurrent Mode](https://reactjs.org/blog/2018/03/01/sneak-peek-beyond-react-16.html). This is possible because the logic inside the component closes over the correct props and state with which it was rendered.

In all cases I’ve seen so far, **the “stale closures” problems happen due to a mistaken assumption that “functions don’t change” or that “props are always the same”**. This is not the case, as I hope this post has helped to clarify.

Functions close over their props and state — and so their identity is just as important. This is not a bug, but a feature of function components. Functions shouldn’t be excluded from the “dependencies array” for `useEffect` or `useCallback`, for example. (The right fix is usually either `useReducer` or the `useRef` solution above — we will soon document how to choose between them.)

When we write the majority of our React code with functions, we need to adjust our intuition about [optimizing code](https://github.com/ryardley/hooks-perf-issues/pull/3) and [what values can change over time](https://github.com/facebook/react/issues/14920).

As [Fredrik put it](https://mobile.twitter.com/EphemeralCircle/status/1099095063223812096):

>The best mental rule I’ve found so far with hooks is ”code as if any value can change at any time”.

Functions are no exception to this rule. It will take some time for this to be common knowledge in React learning materials. It requires some adjustment from the class mindset. But I hope this article helps you see it with fresh eyes.

React functions always capture their values — and now we know why.


> - 本文仅代表原作者个人观点，译者不发表任何观点
> - 版权由原作者所有   
Copyright (c) Dan Abramov and the contributors.  
All rights reserved.

