<!-- omit in toc -->
# Reactの基本動作

<!-- omit in toc -->
## 目次
- [Reactを動かしてみる](#reactを動かしてみる)
  - [Reactの記法（JSX）](#reactの記法jsx)
- [Reactのコンセプト](#reactのコンセプト)
- [Reactアプリ開発の始め方（create-react-app）](#reactアプリ開発の始め方create-react-app)
- [コンポーネントの作成](#コンポーネントの作成)
- [コンポーネント間の値の受け渡し（props）](#コンポーネント間の値の受け渡しprops)

## Reactを動かしてみる
以下のhtmlファイルを作成して、reactを動かしてみましょう。ここでの記述方法は、通常の開発で使うことはありませんので、覚える必要はありません。ここでは、Reactがどのような手順でhtmlに変更を加えているのかを確認してください。好きなディレクトリに以下の`index.html`を作成してください。

`index.html`
```html
<!DOCTYPE html>
<html>
<head>
  <script src="https://unpkg.com/react@18/umd/react.development.js"></script>
  <script src="https://unpkg.com/react-dom@18/umd/react-dom.development.js"></script>
  <script src="https://unpkg.com/babel-standalone@6/babel.min.js"></script>
</head>
<body>
    <div id="app"></div>
    <script>
        const appElement = document.querySelector('#app');
        const root = ReactDOM.createRoot(appElement);
        root.render("Hello React");
    </script>
</body>
</html>
```

作成したindex.htmlをWebブラウザで開いてください。「Hello React」の文字が表示されれば成功です。


`head`タグ内では、Reactを動かすためのライブラリを読み込んでいます。

`body`タグ内に記述したscriptによって、`<div id="app"></div>`の中に、`root.render`の引数が挿入されます。
`root.render`の引数を色々変えてみましょう。

### Reactの記法（JSX）
Reactでは、HTMLタグ(に似たもの)をjavascriptに直接記述する特殊な記法を使います。一般的にこの記法は、JSXと呼ばれています。
先ほどのhtmlをJSXの記法を使って書き換えてみましょう。

`sample.html`の`body`タグ
```html
<body>
    <div id="app"></div>
    <script>
        const appElement = document.querySelector('#app');
        const root = ReactDOM.createRoot(appElement);
        root.render(<h1>Hello</h1>);
    </script>
</body>
```

このように変更すると、エラーが発生していることが分かります。これは、通常のjavascriptでは、`<h1>Hello React</h1>`のようなhtmlタグのような記述ができないためです。

Reactでは、babelというライブラリが、Reactのコードをjavascriptに変換するようになっています。babelがReactのコードを変換してくれるように、`sample.html`を以下のように編集しましょう。


`sample.html`の`body`タグ
```html
<body>
    <div id="app"></div>
    <script type="text/babel">
        const appElement = document.querySelector('#app');
        const root = ReactDOM.createRoot(appElement);
        root.render(<h1>Hello React</h1>);
    </script>
</body>
```

これで、正しく動作するはずです。

## Reactのコンセプト

Reactでは、コンポーネントと呼ばれる単位で、コードを記述していきます。コンポーネントとは、画面の各構成要素をReactで定義したものです。
これによって、コードが整理され、使い回しができ、疎結合になります。

それでは、先ほどのhtmlを編集して、コンポーネントを定義していきましょう。

`sample.html`の`body`タグ
```html
<body>
    <div id="app"></div>
    <script type="text/babel">
        const appElement = document.querySelector('#app');
        const root = ReactDOM.createRoot(appElement);

        const Hello = () => {
            return <h1>Hello React</h1>
        }

        root.render(<Hello />);
    </script>
</body>
```

コンポーネントは、以下のように関数で定義し、必ず最初の文字を大文字で定義します。

```javascript
const Hello = () => {
    return <h1>Hello React</h1>
}
```

※コンポーネントをクラスで記述するものもありますが、これは以前のReactの記法です。すごく特殊な場合を除いては、関数でコンポーネントを定義しましょう。

## Reactアプリ開発の始め方（create-react-app）
ここから本格的にReactのプロジェクトを作っていきましょう。reactのプロジェクトは、`react-create-app`を利用することで、簡単に始めることができます。
以下ようなコマンドで、Reactのtemplateを生成することができます。(node.jsをインストールする必要があります。)

```shell
npx create-react-app プロジェクト名
```

それでは、実際にcreate-react-appを試してみましょう。
これ以降のコードは、typescriptを使用していきますので、templateオプションとしてtypescriptを指定します。`--template typescript`

```shell
npx create-react-app my-app --template typescript
```

これを実行すると、`my-app`というディレクトリが作成され、その中にReactのテンプレートが作成されます。
`my-app`のディレクトリ構成は、以下のようになっているはずです。
```
my-app
├── .git
├── .gitignore
├── README.md
├── node_modules
├── package-lock.json
├── package.json
├── public
├── src
└── tsconfig.json
```

それでは、`my-app`ディレクトリに移動して、実行してみましょう。

```shell
cd my-app
npm start
```

`npm start`を実行すると、開発サーバが立ち上がり、ブラウザで`http://localhost:3000`が開かれ、以下のような表示になります。

![create_react_app default](../images/ch2_create_react_app.png)

それでは、ファイルの中身を少しみてみましょう。`public/index.html`を開いてみてください。以下のような内容になっているはずです。（コメントは削除しています。）

`public/index.html`
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <link rel="icon" href="%PUBLIC_URL%/favicon.ico" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <meta name="theme-color" content="#000000" />
    <meta
      name="description"
      content="Web site created using create-react-app"
    />
    <link rel="apple-touch-icon" href="%PUBLIC_URL%/logo192.png" />
    <link rel="manifest" href="%PUBLIC_URL%/manifest.json" />
    <title>React App</title>
  </head>
  <body>
    <noscript>You need to enable JavaScript to run this app.</noscript>
    <div id="root"></div>
  </body>
</html>
```

これが、ベースとなるhtmlファイルです。`body`タグにある`<div id="root"></div>`の中は、Reactで構成していく部分になります。
`head`タグ内は、必要に応じて変更してください。

次に`src`ディレクトリの中身を見てみましょう。ここには、ソースコードが格納されています。通常、typescriptのソースコードの拡張子は、`.ts`ですが、Reactでは、上で紹介したJSXという記法を使うため、コンポーネントを記述する場合は、拡張子も変わり、`.tsx`となります。(javascriptの場合は、`.jsx`)


`src/index.tsx`を開いてみてください。

`src/index.tsx`
```javascript
import React from 'react';
import ReactDOM from 'react-dom/client';
import './index.css';
import App from './App';
import reportWebVitals from './reportWebVitals';

const root = ReactDOM.createRoot(
  document.getElementById('root') as HTMLElement
);
root.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);

// If you want to start measuring performance in your app, pass a function
// to log results (for example: reportWebVitals(console.log))
// or send to an analytics endpoint. Learn more: https://bit.ly/CRA-vitals
reportWebVitals();
```

ここの内容は、HTMLに直接記述したものと同じような内容であることが分かると思います。
このスクリプトによって、`public/index.html`の`<div id="root"></div>`の中に、`root.render`の引数が挿入されます。
`root.render`の引数である`React.StrictMode`, `App`は、コンポーネントになります。`React.StrictMode`は、開発時にバグを見つけやすくするようにするもので、画面は何も変わりませんが、プログラムの挙動が変わります。

`React.StrictMode`によって以下のような挙動が追加されます。（参考：[<StrictMode> - React](https://ja.react.dev/reference/react/StrictMode)）
- レンダー（画面描画）を追加で1回行う。
- エフェクト(useEffectで指定した内容)の実行を追加で1回行う。
- 非推奨のAPIを使用していないかチェックする。

次は、`App`コンポーネントについてみてみましょう。`src/App.tsx`を開いてください。

`src/App.tsx`
```javascript
import React from 'react';
import logo from './logo.svg';
import './App.css';

function App() {
  return (
    <div className="App">
      <header className="App-header">
        <img src={logo} className="App-logo" alt="logo" />
        <p>
          Edit <code>src/App.tsx</code> and save to reload.
        </p>
        <a
          className="App-link"
          href="https://reactjs.org"
          target="_blank"
          rel="noopener noreferrer"
        >
          Learn React
        </a>
      </header>
    </div>
  );
}

export default App;
```

ここには、実際にブラウザの画面上に表示されているものが定義されていることが分かると思います。
それでは、この`App.tsx`を書き換えてみましょう。

```javascript
function App() {
  return (
    <h1>Hello React</h1>
  );
}

export default App;
```

これで、HTMLに直接記述したときと同じ内容になっているはずです。ブラウザ画面を見てみてください。
`npm start`を終了させた人は、`npm start`を実行して、[http://localhost:3000/](http://localhost:3000/)をブラウザで開いてください。

以下のような画面になっていれば、成功です。

![](../images/ch2_hello_react.png)


## コンポーネントの作成

それでは、コンポーネントを作成してみましょう。

コンポーネントを定義する際には、基本的に以下のルールがあります。
- コンポーネントは、パスカルケース
- １つのファイルにつき、１つのコンポーネント
- コンポーネントのファイル名　＝　コンポーネント名

１つのファイルにつき、１つのコンポーネントというルールは慣習的で、実際に１つのファイルに複数のコンポーネントを定義しても、エラーにはなりません。
ディレクトリ構成は様々ありますが、１つのファイルに１つのコンポーネントというルールは、守られていることが多いようです。

ここでは、とりあえず`components`というディレクトリを作成し、その配下にコンポーネントを定義したファイルを配置していきましょう。
それでは、以下のディレクトリ、ファイルを作成し、`Hello`コンポーネントを作成していきましょう。

`src`ディレクトリ配下に、`components`ディレクトリを作成し、その中に`Hello.tsx`を作成してください。
`Hello`コンポーネントを作成するので、`Hello.tsx`というファイル名になります。

```
my-app
└── src
    └──components
        └── Hello.tsx
```

`Hello.tsx`
```javascript
const Hello = () => {
    return <h1>Hello React</h1>
}

export default Hello
```

## コンポーネント間の値の受け渡し（props）
Reactはコンポーネントという単位で、細かくコードを分けていくので、コンポーネント間で値を共有したい場合がよくあります。(今はまだかもしれませんが)
コンポーネント間での値の受け渡しには、propsを使います。
