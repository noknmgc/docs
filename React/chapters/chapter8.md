---
title: Chapter8 カスタムフック
---
<!-- {% raw %} -->

<!-- omit in toc -->
# カスタムフック

<!-- omit in toc -->
## 目次
- [カスタムフックとは](#カスタムフックとは)
- [カスタムフックの利点](#カスタムフックの利点)
  - [ロジックの分離](#ロジックの分離)
  - [ロジックの使い回し](#ロジックの使い回し)
  - [利点まとめ](#利点まとめ)
- [カスタムフックの実装](#カスタムフックの実装)
  - [準備](#準備)
  - [useCounterの実装](#usecounterの実装)
- [Next: Chapter9 グローバルな状態管理 zustand](#next-chapter9-グローバルな状態管理-zustand)
- [Prev: Chapter7 DOM操作 useRef, createPortal](#prev-chapter7-dom操作-useref-createportal)


## カスタムフックとは
カスタムフックとは、独自で実装するフックのことです。フックとは、コンポーネントで使うロジックを再利用可能な関数にしたものです。これまでの章で出てきた`useState`や`useEffect`、`useRef`などがそれに当たります。

カスタムフックができることを簡単に説明すると、以下の`Counter`コンポーネントのロジック部分を関数`useCounter`としてまとめることができます。

```javascript
import { useLayoutEffect, useState } from "react";

const Counter: React.FC = () => {
  const [count, setCount] = useState(0);

  useLayoutEffect(() => {
    if (count >= 10) setCount(0);
  }, [count]);

  return (
    <div>
      <div className="text-lg">{count}</div>
      <button
        className="rounded-lg bg-gray-300 px-2"
        onClick={() => {
          setCount((prev) => prev + 1);
        }}
      >
        +
      </button>
    </div>
  );
};

export default Counter;
```

上記コードでカスタムフック`useCounter`を自分で定義すると、以下のようにできます。

```jsx
import { useCounter } from "../hooks/useCounter";

const Counter: React.FC = () => {
  const { count, add } = useCounter();

  return (
    <div>
      <div className="text-lg">{count}</div>
      <button
        className="rounded-lg bg-gray-300 px-2"
        onClick={() => {
          add();
        }}
      >
        +
      </button>
    </div>
  );
};

export default Counter;
```

## カスタムフックの利点

### ロジックの分離
カスタムフックを利用すれば、コンポーネントはUI、カスタムフックはロジックというように分離することができます。こうすることでテストが容易になるという利点があります。

### ロジックの使い回し
大規模な開発になってくると、同じようなロジックを様々なコンポーネントで使う場合が出てきます。そのような場合に、カスタムフックを定義すれば、コンポーネントごとにロジックを定義する必要はなく、カスタムフックを利用するだけで良くなります。

もし、その処理でバグがあった場合、カスタムフックを利用していた場合いれば、該当するカスタムフックのみを修正するだけでよくなります。

### 利点まとめ
以上のことから、カスタムフックは、以下の場合に使うのが良いでしょう。

- 複数のコンポーネントで同じロジックを利用する
- ロジックが複雑でコードが長くなっている

プロジェクトによっては、ロジックは全てカスタムフックに記述し、ロジックを完全に分離させる場合もあるようです。どこまでカスタムフックを使うかは、プロジェクトごとに決めるのが良いですが、最低限でも上記の場合は、利用するのがよさそうです。

## カスタムフックの実装
それでは、実際に上で紹介した`useCounter`を実装していきましょう。

### 準備
まずは、カスタムフックの実装する前のコンポーネントを実装しましょう。以下のコンポーネント`CounterWithCustomHook`を作成してください。

`src/components/CounterWithCustomHook.tsx`
```jsx
import { useLayoutEffect, useState } from "react";

const CounterWithCustomHook: React.FC = () => {
  const [count, setCount] = useState(0);

  useLayoutEffect(() => {
    if (count >= 10) setCount(0);
  }, [count]);

  return (
    <div>
      <div className="text-lg">{count}</div>
      <button
        className="rounded-lg bg-gray-300 px-2"
        onClick={() => {
          setCount((prev) => prev + 1);
        }}
      >
        +
      </button>
    </div>
  );
};

export default CounterWithCustomHook;
```

このコンポーネントの実行結果が分かるように、`App`コンポーネントを以下のように修正してください。

`src/App.tsx`
```jsx
import CounterWithCustomHook from "./components/CounterWithCustomHook";

function App() {
  return (
    <div className="m-4 space-y-2">
      <CounterWithCustomHook />
    </div>
  );
}

export default App;
```

`npm run dev`を実行して結果をブラウザで確認してください。

これは、6章で作成した上限が9のカウンターと同じものです。表示されている数字が9の時に、+ボタンを押すと数字が0に戻ります。

このコンポーネントをカスタムフックを用いてリファクタリングを行っていきます。

### useCounterの実装

`src`ディレクトリ内に`hooks`ディレクトリを作成し、このなかにカスタムフックのソースコードを配置していきます。

カスタムフックは、慣習的に以下のようなルールがあります。
- `use`から始まる名称であること
- `hooks`を利用していること
  
  ※ここでの`hooks`は、React公式のフック(useState, useEffectなど)やライブラリが実装しているカスタムフックのことです。

それでは、`useCounter`を実装していきます。以下のファイルを作成してください。このカスタムフックは、jsxを使わないので、拡張子は`.ts`で良いです。

`src/hooks/useCounter.ts`
```typescript
export const useCounter = () => {
  // ここにロジックを記述
};
```

この`useCounter`に`CounterWithCustomHook`コンポーネントのロジックを書いていきます。

どこまでカスタムフックに入れるのかは、特にルールはありませんが、ここではReactのhooksを利用している箇所をカスタムフックに入れていきます。

`CounterWithCustomHook`コンポーネントでReactのhooksを使っているのは、以下の箇所です。

- `useState`を使っている行
- `useLayoutEffect`を使っている処理
- `button`タグの`onClick`イベントの`setCount((prev) => prev + 1);`

これをカスタムフックに含めると、`useCounter`は以下のようになります。

```typescript
import { useLayoutEffect, useState } from "react";

export const useCounter = () => {
  const [count, setCount] = useState(0);

  useLayoutEffect(() => {
    if (count >= 10) setCount(0);
  }, [count]);

  const add = () => {
    setCount((prev) => prev + 1);
  };

  return { count, add };
};
```

次に、このカスタムフックを使って`CounterWithCustomHook`コンポーネントをリファクタリングしましょう。

`CounterWithCustomHook`コンポーネントは、以下のように変更してください。

`src/components/CounterWithCustomHook.tsx`
```jsx
import { useCounter } from "../hooks/useCounter";

const CounterWithCustomHook: React.FC = () => {
  const { count, add } = useCounter();

  return (
    <div>
      <div className="text-lg">{count}</div>
      <button
        className="rounded-lg bg-gray-300 px-2"
        onClick={() => {
          add();
        }}
      >
        +
      </button>
    </div>
  );
};

export default CounterWithCustomHook;
```

`npm run dev`を実行して結果をブラウザで確認してください。
リファクタリング前と動作が変わらなければ、成功です。

## [Next: Chapter9 グローバルな状態管理 zustand](../chapters/chapter9.md)

## [Prev: Chapter7 DOM操作 useRef, createPortal](../chapters/chapter7.md)

<!-- {% endraw %} -->