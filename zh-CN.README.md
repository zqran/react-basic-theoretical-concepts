# React - 基础理论概念

> 注：机翻版

这篇文档是我试图正式解释我的 React 心智模型的尝试。目的是用演绎推理来描述这一点，从而引导我们进行这种设计。

肯定有一些前提是有争议的，这个例子的实际设计可能有错误和差距。这只是正式化的开始。如果您对如何正式化有更好的想法，请随时发送拉取请求。从简单 -> 复杂的过程应该是有意义的，没有太多的库细节闪耀。

React.js 的实际实现充满了实用的解决方案、增量步骤、算法优化、遗留代码、调试工具以及使它真正有用所需的东西。 这些东西更短暂，如果它足够有价值并且具有足够高的优先级，则可以随着时间而改变。实际的实现更难推理。

我喜欢有一个更简单的心智模型，我可以让自己立足于其中。

## Transformation - 转换

React 的核心前提是 UI 只是将数据投影到不同形式的数据中。相同的输入给出相同的输出。一个简单的纯函数。

```js
function NameBox(name) {
  return { fontWeight: 'bold', labelContent: name };
}
```

```
'Sebastian Markbåge' ->
{ fontWeight: 'bold', labelContent: 'Sebastian Markbåge' };
```

## Abstraction - 抽象

但是，您无法在单个函数中容纳复杂的 UI。重要的是，UI 可以被抽象为不会泄露其实现细节的可重用部分。例如从另一个函数调用一个函数。

```js
function FancyUserBox(user) {
  return {
    borderStyle: '1px solid blue',
    childContent: [
      'Name: ',
      NameBox(user.firstName + ' ' + user.lastName)
    ]
  };
}
```

```
{ firstName: 'Sebastian', lastName: 'Markbåge' } ->
{
  borderStyle: '1px solid blue',
  childContent: [
    'Name: ',
    { fontWeight: 'bold', labelContent: 'Sebastian Markbåge' }
  ]
};
```

## Composition - 组合

要实现真正可重用的特性，仅仅重用叶子并为它们构建新容器是不够的。您还需要能够从*组合*其他抽象的容器构建抽象。 我对“组合”的看法是它们将两个或多个不同的抽象组合成一个新的抽象。

```js
function FancyBox(children) {
  return {
    borderStyle: '1px solid blue',
    children: children
  };
}

function UserBox(user) {
  return FancyBox([
    'Name: ',
    NameBox(user.firstName + ' ' + user.lastName)
  ]);
}
```

## State - 状态

UI 不仅仅是服务器/业务逻辑状态的复制。实际上有很多状态特定于精确投影而不是其他状态。例如，如果您开始在文本字段中输入内容。这可能会或可能不会复制到其他选项卡或您的移动设备。滚动位置是一个典型的例子，你几乎不想在多个投影中复制它。

我们倾向于更喜欢我们的数据模型是不可变的。我们将函数贯穿其中将状态更新为顶部的单个原子。

```js
function FancyNameBox(user, likes, onClick) {
  return FancyBox([
    'Name: ', NameBox(user.firstName + ' ' + user.lastName),
    'Likes: ', LikeBox(likes),
    LikeButton(onClick)
  ]);
}

// Implementation Details

var likes = 0;
function addOneMoreLike() {
  likes++;
  rerender();
}

// Init

FancyNameBox(
  { firstName: 'Sebastian', lastName: 'Markbåge' },
  likes,
  addOneMoreLike
);
```

*注意：这些示例使用副作用来更新状态。我对此的实际心理模型是它们在“更新”过程中返回下一个版本的状态。如果没有它，解释起来会更简单，但我们将来会想要更改这些示例。*

## Memoization - 记忆（缓存）

如果我们知道函数是纯函数，一遍又一遍地调用同一个函数是浪费的。我们可以创建一个函数的记忆版本来跟踪最后一个参数和最后一个结果。 这样，如果我们继续使用相同的值，我们就不必重新执行它。

```js
function memoize(fn) {
  var cachedArg;
  var cachedResult;
  return function(arg) {
    if (cachedArg === arg) {
      return cachedResult;
    }
    cachedArg = arg;
    cachedResult = fn(arg);
    return cachedResult;
  };
}

var MemoizedNameBox = memoize(NameBox);

function NameAndAgeBox(user, currentTime) {
  return FancyBox([
    'Name: ',
    MemoizedNameBox(user.firstName + ' ' + user.lastName),
    'Age in milliseconds: ',
    currentTime - user.dateOfBirth
  ]);
}
```

## Lists - 列表

大多数 UI 是某种形式的列表，然后为列表中的每个项目生成多个不同的值。这创建了一个自然的层次结构。

为了管理列表中每个项目的状态，我们可以创建一个 Map 来保存特定项目的状态。

```js
function UserList(users, likesPerUser, updateUserLikes) {
  return users.map(user => FancyNameBox(
    user,
    likesPerUser.get(user.id),
    () => updateUserLikes(user.id, likesPerUser.get(user.id) + 1)
  ));
}

var likesPerUser = new Map();
function updateUserLikes(id, likeCount) {
  likesPerUser.set(id, likeCount);
  rerender();
}

UserList(data.users, likesPerUser, updateUserLikes);
```

*注意：我们现在有多个不同的参数传递给 FancyNameBox。这打破了我们的记忆，因为我们一次只能记住一个值。更多内容如下。 *

## Continuations - 延续

不幸的是，由于 UI 中到处都是列表列表，因此显式管理它变得相当多的样板文件。

我们可以通过延迟执行一个函数来将一些样板从我们的关键业务逻辑中移出。例如，通过在 JavaScript 中使用“currying”（[`bind`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind)）。 然后我们从我们现在没有样板的核心函数之外传递状态。

这并没有减少样板文件，但至少将其移出关键业务逻辑。

```js
function FancyUserList(users) {
  return FancyBox(
    UserList.bind(null, users)
  );
}

const box = FancyUserList(data.users);
const resolvedChildren = box.children(likesPerUser, updateUserLikes);
const resolvedBox = {
  ...box,
  children: resolvedChildren
};
```

## State Map - 状态映射

我们从前面知道，一旦我们看到重复的模式，我们可以使用组合来避免一遍又一遍地重新实现相同的模式。我们可以将提取和传递状态的逻辑转移到我们经常重用的低级别函数中。

```js
function FancyBoxWithState(
  children,
  stateMap,
  updateState
) {
  return FancyBox(
    children.map(child => child.continuation(
      stateMap.get(child.key),
      updateState
    ))
  );
}

function UserList(users) {
  return users.map(user => {
    continuation: FancyNameBox.bind(null, user),
    key: user.id
  });
}

function FancyUserList(users) {
  return FancyBoxWithState.bind(null,
    UserList(users)
  );
}

const continuation = FancyUserList(data.users);
continuation(likesPerUser, updateUserLikes);
```

## Memoization Map - 缓存映射

一旦我们想在一个列表中记忆多个项目，记忆就会变得更加困难。您必须找出一些复杂的缓存算法来平衡内存使用与频率。

幸运的是，UI 在相同的位置上往往相当稳定。树中的相同位置每次都获得相同的值。这棵树被证明是一种非常有用的记忆策略。

我们可以使用我们用于状态的相同技巧，并通过可组合函数传递一个记忆缓存。

```js
function memoize(fn) {
  return function(arg, memoizationCache) {
    if (memoizationCache.arg === arg) {
      return memoizationCache.result;
    }
    const result = fn(arg);
    memoizationCache.arg = arg;
    memoizationCache.result = result;
    return result;
  };
}

function FancyBoxWithState(
  children,
  stateMap,
  updateState,
  memoizationCache
) {
  return FancyBox(
    children.map(child => child.continuation(
      stateMap.get(child.key),
      updateState,
      memoizationCache.get(child.key)
    ))
  );
}

const MemoizedFancyNameBox = memoize(FancyNameBox);
```

## Algebraic Effects - 代数效应

事实证明，通过多个抽象级别传递您可能需要的每一个小值是一种 PITA。有时有一种快捷方式可以在两个抽象之间传递东西而不涉及中间体，这有点好。在 React 中，我们称之为“上下文”。

有时数据依赖关系并没有整齐地遵循抽象树。例如，在布局算法中，您需要先了解孩子的大小，然后才能完全满足他们的要求。

现在，这个例子有点“不合时宜”。我将使用 [代数效应](http://math.andrej.com/eff/) 作为 [为 ECMAScript 提出的](https://esdiscuss.org/topic/one-shot-delimited-continuations-with-effect-handlers)。 如果你熟悉函数式编程，他们会避免单子强加的中间仪式。

```js
function ThemeBorderColorRequest() { }

function FancyBox(children) {
  const color = raise new ThemeBorderColorRequest();
  return {
    borderWidth: '1px',
    borderColor: color,
    children: children
  };
}

function BlueTheme(children) {
  return try {
    children();
  } catch effect ThemeBorderColorRequest -> [, continuation] {
    continuation('blue');
  }
}

function App(data) {
  return BlueTheme(
    FancyUserList.bind(null, data.users)
  );
}
```
