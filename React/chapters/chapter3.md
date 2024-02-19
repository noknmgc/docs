<!-- omit in toc -->
# スタイリング

<!-- omit in toc -->
## 目次
- [Raw CSS](#raw-css)
- [style props](#style-props)
- [CSS Moldules](#css-moldules)
  - [CSS Modulesの注意点](#css-modulesの注意点)
- [CSS in JS](#css-in-js)
- [Tailwind](#tailwind)


## Raw CSS
スタイリングのやり方について説明します。まずは、一番シンプルなcssファイルを読み取る方法です。

ただし、cssファイルを読み込ませた場合、グローバルに適用されます。そのため、全体に適用したいスタイルのみ記述するようにしましょう。

各コンポーネントに適用するスタイルに、cssを使用することもできますが、このやり方だと名前の衝突が問題になります。もし、同じ名前のスタイルが定義されていた場合、後から読み込まれたスタイルが上書きされてしまいます。

各コンポーネントで使うスタイルは、以下で紹介するCSS Modules, CSS in JS, Tailwindのいずれかを利用してください。

## style props
ReactでHTML要素を使う場合に、style propsを渡すことができます。このやり方だと、HTML要素に直接スタイルを記述するので、名前の衝突がありません。また、javascriptの変数も使うことができるので、動的なスタイルも適用できます。

ただし、style propsはパフォーマンスが悪いため、あまり多用しない方が良いでしょう。名前の衝突の回避や動的なスタイルの適用は、後に紹介する方法でもできます。

## CSS Moldules
cssをそのまま使った場合にあった問題：名前の衝突を回避するために使われるのが、css modulesです。

cssファイルを使いつつ、読み込んだコンポーネントにのみスタイルを適用することができます。

すでにcssを使い慣れている人におすすめの方法です。

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