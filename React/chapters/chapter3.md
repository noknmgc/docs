<!-- omit in toc -->
# スタイリング

<!-- omit in toc -->
## 目次
- [スタイリングを行うコンポーネントの追加](#スタイリングを行うコンポーネントの追加)
- [Raw CSS](#raw-css)
- [style props](#style-props)
- [CSS Moldules](#css-moldules)
  - [CSS Modulesの注意点](#css-modulesの注意点)
- [CSS in JS](#css-in-js)
- [Tailwind](#tailwind)
- [まとめ](#まとめ)

## スタイリングを行うコンポーネントの追加
スタイリングの対象となるボタンコンポーネントを作りましょう。

以下の`src/components/StyledButton.tsx`を作成してください。
`src/components/StyledButton.tsx`
```javascript
const StyledButton: React.FC = () => {
  return (
    <>
      <button>Push</button>
    </>
  );
};

export default StyledButton;
```

このボタンが画面に表示されるように、`App`コンポーネントも変更しましょう。
`src/App.tsx`
```javascript
import StyledButton from "./components/StyledButton";

function App() {
  return (
    <>
      <StyledButton />
    </>
  );
}

export default App;
```

また、viteでプロジェクトのテンプレートを作成した場合、デフォルトのスタイルを`main.tsx`で読み込んでいるので、`main.tsx`でcssを読み込んでいる行をコメントアウトまたは、削除しましょう。

```javascript
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App.tsx";
//import './index.css'

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

`npm run dev`を実行し、[http://localhost:5173/](http://localhost:5173/)をブラウザで開いてください。
以下のようなボタンが表示されます。

![default button](../images/ch3_button.png)

## Raw CSS
スタイリングのやり方について説明します。まずは、一番シンプルなcssファイルを読み取る方法です。

まず、cssを定義しましょう。以下のcssファイルを作成してください。
`src/components/cutom-button.css`
```css
.custom-button {
  padding-top: 0.625rem;
  padding-bottom: 0.625rem;
  padding-left: 1.25rem;
  padding-right: 1.25rem;
  margin-bottom: 0.5rem;
  border-radius: 0.5rem;
  font-size: 0.875rem;
  line-height: 1.25rem;
  font-weight: 500;
  color: #ffffff;
  background-color: #1d4ed8;
}

.custom-button:hover {
  background-color: #1e40af;
}

.custom-button:focus {
  outline-style: solid;
  outline-width: 4px;
  outline-color: #93c5fd;
}
```

それでは、`StyledButton.tsx`のbutton要素にクラス名をつけましょう。htmlでは、通常`class`でクラス名を指定しますが、Reactでは、`class`がjavascriptのクラスと名前が被ってしまうため、`className`で指定します。

また、定義したcssも読み込む必要がありますので、cssファイルをimportしましょう。

`StyledButton.tsx`を以下のように修正しましょう。
`src/components/StyledButton.tsx`
```javascript
import "./custom-button.css";

const StyledButton: React.FC = () => {
  return (
    <>
      <button className="custom-button">Push</button>
    </>
  );
};

export default StyledButton;
```

ブラウザを確認すると以下のようなボタンが表示されているはずです。
また、少し触ってみて、hoverやfocus時のスタイルも確認してみましょう。

![styled button by css file](../images/ch3_css_styled_button.png)

ただし、cssファイルを読み込ませた場合、グローバルに適用されます。そのため、**全体に適用したいスタイルのみ記述**するようにしましょう。

全体に適用したいスタイルは、この`index.css`に定義するようにし、`main.tsx`で読み込むようにしましょう。

各コンポーネントに適用するスタイルに、cssを使用することもできますが、このやり方だと名前の衝突が問題になります。もし、同じ名前のスタイルが定義されていた場合、後から読み込まれたスタイルが上書きされてしまいます。

**各コンポーネントで使うスタイルは、以下で紹介するCSS Modules, CSS in JS, Tailwindのいずれかを利用してください。**

## style props
ReactでHTML要素を使う場合に、style propsを渡すことができます。このやり方だと、HTML要素に直接スタイルを記述するので、名前の衝突がありません。また、javascriptの変数も使うことができるので、動的なスタイルも適用できます。

style propsは以下のように適用します。`StyledButton.tsx`にボタンを追加し、cssで記述したスタイルをそのままstyle propsに記述していきます。コンポーネントを以下のように修正しましょう。

また、今後ボタンが複数並ぶので、みやすいようにgridも作成しています。

```javascript
const StyledButton: React.FC = () => {
  return (
    <div
      style={{
        display: "grid",
        gridTemplateColumns: "repeat(4, minmax(0, 1fr))",
        gap: "1rem",
      }}
    >
      <button className="custom-button">Push</button>
      <button
        style={{
          paddingTop: "0.625rem",
          paddingBottom: "0.635rem",
          paddingLeft: "1.25rem",
          paddingRight: "1.25rem",
          marginBottom: "0.5rem",
          borderRadius: "0.5rem",
          fontSize: "0.875rem",
          lineHeight: "1.25rem",
          fontWeight: 500,
          color: "#ffffff",
          backgroundColor: "#1d4ed8",
          borderStyle: "none",
        }}
      >
        Push
      </button>
    </div>
  );
};
```

こうすることで、同じようなボタンが2つ並びます。

![styled button by style props](../images/ch3_css_styled_button2.png)

ただし、style propsでは、hoverとfocusは、後で紹介する状態管理を使って行う必要があるので、ここでは適用していません。

style propsはパフォーマンスが悪いため、あまり多用しない方が良いでしょう。名前の衝突の回避や動的なスタイルの適用は、後に紹介する方法でもできます。

## CSS Moldules
cssをそのまま使った場合にあった問題：名前の衝突を回避するために使われるのが、css modulesです。

css modulesでは、cssは通常通り記述するのですが、最終的に適用されるクラス名が自動生成されたものになるので、名前の衝突が回避できるようになります。

[Raw CSS](#raw-css)で作成したcssファイルと中身が同じものを用意します。
ファイル名は、`custom-button.module.css`としてください。cssの前に`module`と入れることで、CSS Modulesとして認識してくれます。

以下のファイルを作成してください。

`src/components/custom-button.module.css`
```css
.custom-button {
  padding-top: 0.625rem;
  padding-bottom: 0.625rem;
  padding-left: 1.25rem;
  padding-right: 1.25rem;
  margin-bottom: 0.5rem;
  border-radius: 0.5rem;
  font-size: 0.875rem;
  line-height: 1.25rem;
  font-weight: 500;
  color: #ffffff;
  background-color: #1d4ed8;
  border-style: none;
}

.custom-button:hover {
  background-color: #1e40af;
}

.custom-button:focus {
  outline-style: solid;
  outline-width: 4px;
  outline-color: #93c5fd;
}
```

それでは、作成したファイルを読み込んでいきましょう。
CSS Modulesでは、以下のようにimportして使います。
```javascript
import styles from "./custom-button.module.css"
```
このimportした`styles`を見てみると
```json
{"custom-button":"_custom-button_1317u_1"}
```
となっており、`"_custom-button_1317u_1"`は、自動生成されたクラス名です。したがって、`styles["custom-button"]`を`className`に指定しましょう。
```javascript
<button className={styles["custom-button"]}>Push</button>
```

`StyledButton.tsx`を以下のように修正しましょう。

```javascript
import "./custom-button.css";
import styles from "./custom-button.module.css";

const StyledButton: React.FC = () => {
  return (
    <div
      style={{
        display: "grid",
        gridTemplateColumns: "repeat(4, minmax(0, 1fr))",
        gap: "1rem",
      }}
    >
      <button className="custom-button">Push</button>
      <button
        style={{
          paddingTop: "0.625rem",
          paddingBottom: "0.635rem",
          paddingLeft: "1.25rem",
          paddingRight: "1.25rem",
          marginBottom: "0.5rem",
          borderRadius: "0.5rem",
          fontSize: "0.875rem",
          lineHeight: "1.25rem",
          fontWeight: 500,
          color: "#ffffff",
          backgroundColor: "#1d4ed8",
          borderStyle: "none",
        }}
      >
        Push
      </button>
      <button className={styles["custom-button"]}>Push</button>
    </div>
  );
};

export default StyledButton;
```

cssの書き方は、変わらないので、すでにcssを使い慣れている人におすすめの方法です。

### CSS Modulesの注意点

CSS Modulesは、css-loaderのメンテナーがCSS Moduleを非推奨にしたいと意思表明があり、近々非推奨になるのではないかと言われていたり、一方でNext.jsでは公式ドキュメントでCSS Moduleを推奨にするようにしたいと言うIssueが立てられたりなど、CSS Modulesが、今後どうなっていくのか不透明な状態です。

[CSS Modulesの歴史、現在、これから - Hatena Developer Blog](https://developer.hatenastaff.com/entry/2022/09/01/093000#CSS-Modules%E3%81%AE%E3%83%A1%E3%83%B3%E3%83%86%E3%83%8A%E3%83%B3%E3%82%B9%E7%8A%B6%E6%B3%81)


ここからは、筆者個人の意見ですが、CSS Modulesは、cssに使い慣れている人にとっては、非常に強力なツールですので、既にcssで使い慣れている人は、使ってもいいかと思います。

逆にcssを使い慣れていない人は、次に紹介するCSS-in-JSやTailwindを使うのが安全かと思います。


## CSS in JS
CSS-in-JSとは、外部ファイル(cssファイル)にスタイルを定義するのではなく、Javascriptにスタイルを定義するものです。

CSS-in-JSのメリット
- ファイル数が減る。
- cssの記述の方法がほぼ同じ
- 動的なスタイルが作りやすい。
- スタイルの適用対象がわかりやすい。

CSS-in-JSのライブラリは、複数ありますが、ここでは、最も利用されているStyled Componentsを取り扱います。

## Tailwind

Tailwindのメリット
- ファイル数が減る。
- クラス名を考える必要がない。
- クラス名からスタイルが分かる。

まず、Tailwindのインストール手順です。

## まとめ
全体に適用するスタイルは、cssで記述しましょう。各コンポーネントで適用するスタイルは、CSS Modules, CSS in JS, Tailwindの中から自分に合ったものを選びましょう。

本資料では、Tailwind CSSを使っていきます。
