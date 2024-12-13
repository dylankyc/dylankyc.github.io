# Redux Typescript 大型软件实践

- [Intro](#intro)
  - [当我们谈到 pattern matching 时，是真的要讲 `一只螃蟹向左走向右走` 的问题吗？](#%E5%BD%93%E6%88%91%E4%BB%AC%E8%B0%88%E5%88%B0-pattern-matching-%E6%97%B6%E6%98%AF%E7%9C%9F%E7%9A%84%E8%A6%81%E8%AE%B2-%E4%B8%80%E5%8F%AA%E8%9E%83%E8%9F%B9%E5%90%91%E5%B7%A6%E8%B5%B0%E5%90%91%E5%8F%B3%E8%B5%B0-%E7%9A%84%E9%97%AE%E9%A2%98%E5%90%97)
  - [Type-safe action creator in Redux](#type-safe-action-creator-in-redux)
  - [后记](#%E5%90%8E%E8%AE%B0)

# Intro

大多数程序员都将类型安全理解为编程语言的一个特性，TypeScript 作为 JavaScript 的超集，它可以更严格地执行类型检查。

我们拿一个螃蟹 🦀️ 举例，我们知道它只能向左走或向右走，于是我们定义它的方向：

```typescript
type Direction = "left" | "right";
```

然后定义它的类：

```typescript
class Crab {
  name: string;
  constructor(name: string) {
    this.name = name;
  }
  move(direction: Direction, distanceInMeters: number = 0) {
    console.log(`${this.name} moved ${distanceInMeters} m.`);
  }
}
```

然后我们去了夏威夷，抓了一只小螃蟹：

```typescript
const crab = new Crab("a just catched crab from Hawaii");
```

让它向前走：

```typescript
crab.move("forward", 20);
```

于是出错了！

```bash
error TS2345: Argument of type '"forward"' is not assignable to parameter of type 'Direction'.

14 crab.move("forward", 20);
             ~~~~~~~~~
```

这个出错是什么意思呢？它告诉我们 `forward` 不可以赋值给类型是 `Direction` 的参数，到这里我们知道类型检查开始工作了。

![forward](images/pattern-matching-1.png)

我们只能让我们的小螃蟹向左或向右走：

```typescript
crab.move("left", 20);
crab.move("right", 20);
```

针对 `Direction` 类型，我们还可以用模式匹配来进行类型检查。

假设我们的左岸是大海，右岸是陆地，而我们的螃蟹智商极高，当它移动时，它知道食物在海里，从而远离陆地。

vscode 会提示我们，只能向左走向右走，其他方向时不允许的

![forward](images/pattern-matching-2.png)

## 当我们谈到 pattern matching 时，是真的要讲 `一只螃蟹向左走向右走` 的问题吗？

Either Type

为什么要使用 Either Type?

Either Type 算是一种容器，它可以存储两种类型：A 和 B，这里是 Left 和 Right，一种表示失败，一种表示成功。

学过 Rust 的人都知道，Rust 有个 Result 类型：

```rust
pub enum Result<T, E> {
    /// Contains the success value
    Ok(T),

    /// Contains the error value
    Err(E),
}
```

它在 pattern match 的时候很有用，出错处理几乎离不开它：

```rust
fn halves_if_even(i: i32) -> Result<i32, Error> {
    if i % 2 == 0 { Ok(i/2) } else { Err(/* something */) }
}

fn do_the_thing(i: i32) -> Result<i32, Error> {
    let i = match halves_if_even(i) {
        Ok(i) => i,
        e => return e,
    };

    // use `i`
}
```

我们可以效仿 Rust 实现 TypeScript 的 match on Either

我们使用 `union type` 来定义 Either

```typescript
type Left<T> = { type: "left"; value: T };
type Right<T> = { type: "right"; value: T };
type Either<L, R> = Left<L> | Right<R>;
```

Either 定义了一个容器，实际编码中，我们需要从 Either 容器里提取结果，为了调用者的方便，我们允许传入 callback 来处理不同的情况

```typescript
function match<T, L, R>(
  input: Either<L, R>,
  left: (left: L) => T,
  right: (right: R) => T
) {
  switch (input.type) {
    case "left":
      return left(input.value);
    case "right":
      return right(input.value);
  }
}
```

调用者此时可以定义自己的函数，返回类型是个 Either，失败返回 Error，成功得到运动的方向

```typescript
function validateCrabMoveDirection(
  crab: Crab
): Either<Error, { direction: Direction }> {
  if (crab.name === "strange crab") {
    // return Left type
    return { type: "left", value: Error("x") };
  }
  // return Right type
  return { type: "right", value: { direction: crab.smartmove("right") } };
}
```

于是可以用 match 来获取上述函数的运行结果：

```typescript
{
  const direction = match(
    validateCrabMoveDirection(crab),
    (_) => null,
    (right) => right.direction
  );
  // output: right
  console.log(direction);
}

{
  const crab = new Crab("strange crab");
  const direction = match(
    validateCrabMoveDirection(crab),
    (_) => null,
    (right) => right.direction
  );
  // output: null
  console.log(direction);
}
```

## Type-safe action creator in Redux

讲到这里不得不讲下 `TypeScript` 在 `Redux` 中的应用，当我们给 redux 的 action type 定义很多类型时，一个显著的问题是，不同 action creator 的函数类型不能动态获取，此时我们可以利用 `TypeScript` 的 `ReturnType` 来解决

以一个 Notes 应用为例

首先定义 Notes 的 `interface`

```typescript
interface Note {
  id: number;
  title: string;
  content: string;
  creationDate: string;
}
```

然后定义 action type，可以用 const

```typescript
const FETCH_REQUEST = "FETCH_REQUEST";
const FETCH_SUCCESS = "FETCH_SUCCESS";
const FETCH_ERROR = "FETCH_ERROR";
```

或者使用 enum

```typescript
const enum NotesActionTypes {
  FETCH_REQUEST = "@@notes/FETCH_REQUEST",
  FETCH_SUCCESS = "@@notes/FETCH_SUCCESS",
  FETCH_ERROR = "@@notes/FETCH_ERROR",
}
```

然后定义我们的 action creator，此时用到 `typesafe-actions` 这个库

```typescript
const fetchRequest = createAction(NotesActionTypes.FETCH_REQUEST);
const fetchSuccess = createAction(NotesActionTypes.FETCH_SUCCESS, (action) => {
  return (data: Note[]) => action(data);
});
const fetchError = createAction(NotesActionTypes.FETCH_ERROR, (action) => {
  return (message: string) => action(message);
});
```

每个 action creator 的返回类型不同，此时 `ReturnType` 登场

```typescript
// 利用 ReturnType 定义 action 减少代码冗余

const actions = { fetchRequest, fetchSuccess, fetchError };
type Action = ReturnType<(typeof actions)[keyof typeof actions]>;
```

定义了上述 Action，我们就可以给我们的 reducer 中的 action 做类型检查了

```typescript
// 定义 redux state
type State = { notes: Note[]; state: string; errorMessage?: string };

// 定义 redux reducer
const reducer: Reducer<State> = (state: State, action: Action) => {
  switch (action.type) {
    case getType(fetchRequest): {
      return { ...state, state: "LOADING" };
    }
    case getType(fetchSuccess): {
      return { ...state, state: "LOADED", notes: action.payload };
    }
    case getType(fetchError): {
      return {
        ...state,
        state: "ERROR",
        notes: [],
        errorMessage: action.payload,
      };
    }
    default: {
      return state;
    }
  }
};
```

简单测试

```typescript
let state = { notes: [], state: "INIT" };
state = reducer(state, fetchRequest());
// { notes: [], state: 'LOADING' }
console.log(state);
```

## 后记

关于为何螃蟹要横向走？来自维基百科

> 因为腿关节构造的缘故，螃蟹横著走会比较迅速，因此它们一般都是横著行进的，另外，蛙蟹科的一些生物也会直着或倒退着行进。

> 螃蟹富含优质蛋白质，蟹肉较细腻，肌肉纤维中含有 10 余种游离氨基酸，其中谷氨酸、脯氨酸、精氨酸含量较多，对术后、病后、慢性消耗性疾病等需要补充营养的人大有益处。螃蟹脂肪含量很低，但维生素 A、E 和 B 族较高，特别是蟹黄中富含维生素 A，有益于视力及皮肤健康。蟹富含矿物元素钙、镁以及锌、硒、铜等人体必需的微量元素。但由于螃蟹高胆固醇、高嘌呤，痛风患者食用时应自我节制，患有感冒、肝炎、心血管疾病的人不宜食蟹。死蟹不能吃，会带有大量细菌和毒素。

题图 https://pixabay.com/users/skylark-201564/
