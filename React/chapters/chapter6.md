---
title: Chapter6 useEffect
---

<!-- omit in toc -->
# useEffect

<!-- omit in toc -->
## 目次
- [関数コンポーネントの処理による制限](#関数コンポーネントの処理による制限)
- [useEffectとは](#useeffectとは)
- [useEffectの具体的な用途](#useeffectの具体的な用途)
  - [データフェッチ](#データフェッチ)
  - [イベントの登録](#イベントの登録)
- [useLayoutEffect](#uselayouteffect)

## 関数コンポーネントの処理による制限
一度だけ実行したい内容がある場合、コンポーネントの先頭に処理を書くと、コンポーネントが再描画される度に実行されてしまいます。

少し例を見てみましょう。以前作成した`Counter`コンポーネントに2秒後に`count`を10増やすという処理を加えます。以下のように修正してください。

`src/components/Counter.tsx`
```javascript
import { useState } from "react";

const Counter: React.FC = () => {
  const [count, setCount] = useState(0);

  setTimeout(() => {
    setCount((prev) => prev + 10);
  }, 2000);

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

結果を確認するために、`App`コンポーネントを以下のように書き換えます。

`src/App.tsx`
```javascript
import Counter from "./components/Counter";

function App() {
  return (
    <div className="m-4 space-y-2">
      <Counter />
    </div>
  );
}

export default App;
```

`npm run dev`を実行して、ブラウザで結果を見てましょう。

![カウンター](../images/ch6_counter.png)

まず、`StrictMode`が有効なので、コンポーネントが2回実行されます。その関係で、何も触らなくても2秒ごとに20ずつ増加します。

さらに+ボタンを何度かクリックしてみてください。そうすると、どんどん増加する値が増えていくかと思います。

これは、`counter`の値が更新される度に`Counter`コンポーネントが再実行されるので、その度に`setTimeout`が実行されることが原因です。

一度だけ実行したい場合、`useEffect`を利用することで実現できます。`useEffect`の使い方を見ていきましょう。

## useEffectとは
関数コンポーネントで一度だけ実行したい処理がある場合や、何らかの値に合わせて変更される別の値がある場合などに使用します。

`useEffect`は以下のように記述します。

```javascript
import { useEffect } from "react";

const MyComponent: React.FC = () => {
  useEffect(
    () => {
      // 処理を記述
      // ...

      // 返値には、クリーアップ関数を指定。
      return () => {};
    },
    [
      /* 依存する変数を入れる */
    ],
  );
  return <></>;
};
```

`useEffect`の第一引数には、関数を指定します。関数の中に行いたい処理を記述します。関数の返値には、クリーンアップ関数を記述します。第二引数には、配列を指定します。配列には、依存している変数を入れます。依存している変数が変更されるたびに、`useEffect`に指定した関数の中身が実行されます。1度だけ実行したい場合は、空の配列を指定します。

[関数コンポーネントの処理による制限](#関数コンポーネントの処理による制限)で作成した`Counter`コンポーネントを`useEffect`を使って`setTimeout`の処理を1度だけ実行するようにしましょう。`Counter`コンポーネントを以下のように変更します。

`src/components/Counter.tsx`
```javascript
import { useEffect, useState } from "react";

const Counter: React.FC = () => {
  const [count, setCount] = useState(0);

  useEffect(() => {
    setTimeout(() => {
      setCount((prev) => prev + 10);
    }, 1000);
  }, []);

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

1度だけ実行するので、`useEffect`の第二引数は空の配列を指定します。

`npm run dev`を実行し、ブラウザで結果を見てみましょう。`StrictMode`が有効なため、2秒後にカウントは20増えますが、それ以降は増えないはずです。


## useEffectの具体的な用途


### データフェッチ
APIから取得した値によって、画面の表示や選択肢が変わることは、よくあることです。しかし大抵の場合、APIからのデータ取得は、再描画ごとに行う必要はありません。そこで`useEffect`が利用されます。

実際に、[国民の祝日API](https://national-holidays.jp/about.html)を利用して、入力された日付の祝日の名前を取得するコンポーネントを作ってみましょう。

```javascript
import { useEffect, useState } from "react";

const Holiday = () => {
  const [date, setDate] = useState("");
  const [name, setName] = useState("");

  const fetchHoliday = async (date: string) => {
    const response = await fetch(`http://api.national-holidays.jp/${date}`);
    const data = await response.json();
    setName(data.name ?? "該当なし");
  };

  useEffect(() => {
    const parsedDate = new Date(date);
    // Dateがparseできた場合のみ処理を行う
    if (!Number.isNaN(parsedDate.getTime())) {
      fetchHoliday(date);
    }
  }, [date]);

  return (
    <div>
      <input
        type="date"
        value={date}
        onChange={(e) => {
          setDate(e.target.value);
        }}
      />
      <div>祝日:{name}</div>
    </div>
  );
};

export default Holiday;
```

このコンポーネントでは、`useEffect`の第二引数の配列には、`date`を指定しています。そのため、`date`を変更するたびに`useEffect`で指定した関数が実行され、祝日の名前を取得します。

`App`コンポーネントに追加して実行結果を確認してみてください。

```javascript
import Holiday from "./components/Holiday";

function App() {
  return (
    <div className="m-4 space-y-2">
      <Holiday />
    </div>
  );
}

export default App;
```

![祝日表示](../images/ch6_holiday.png)

### イベントの登録

```javascript

```

## useLayoutEffect

