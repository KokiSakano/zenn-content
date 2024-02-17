---
title: "React Canaryバージョンにおけるtitle要素の新たな挙動"
emoji: "📃"
type: "tech"
topics: ['react', 'canary', 'title']
published: true
---
Reactの[カナリアバージョン](https://react.dev/community/versioning-policy#canary-channel)ではコンポーネントから`title`要素を用いることでドキュメントのタイトルを設定可能になります。

## 最新バージョン
まずは、現在の最新バージョンであるv18.2.0で挙動を確認します。
コンポーネント内で`title`要素を用いると、用いた場所通りに`title`要素が描画されます。

```tsx:tsx
<div>
  <p>⬇︎⬇︎⬇︎タイトル⬇︎⬇︎⬇︎</p>
  <title>タイトル</title>
  <p>⬆︎⬆︎⬆︎タイトル⬆︎⬆︎⬆︎</p>
</div>
```
上記のようなJSXを記述した場合、描画結果も同じようになります。
```html:html
<div>
  <p>⬇︎⬇︎⬇︎タイトル⬇︎⬇︎⬇︎</p>
  <title>タイトル</title>
  <p>⬆︎⬆︎⬆︎タイトル⬆︎⬆︎⬆︎</p>
</div>
```

検索エンジンやブラウザの挙動によっては`body`要素内に`title`がある場合はそれがドキュメントのタイトルと読み取ってくれないことが多いのでこのような結果は避けたいです。そもそも、`title`要素はHTMLの標準では`head`要素内に配置するように規定されているので、`body`要素内に配置されるのは相応しくありません。
さらに、描画先のHTMLの`head`要素にすでに`title`が設定済みな場合はドキュメントのタイトルを更新できないのでこの方法でドキュメントのタイトルを設定することは難しいです(そしてそのケースに該当することは多いと思います)。

:::message
>The title element of a document is the first title element in the document (in tree order), if there is one, or null otherwise.

[HTML Standard](https://html.spec.whatwg.org/multipage/dom.html#document.title)によれば最初の`title`要素が`document.title`となるように定められています。
この仕様によって`head`要素内にすでに`title`要素が設定された場合に`body`要素内で`title`要素を設定してもドキュメントのタイトルは変更されません。
:::

実際にコンポーネント内で`title`を設定するデモは以下の通りです。

@[stackblitz](https://stackblitz.com/edit/vitejs-vite-c49eyy?embed=1&file=src%2FApp.tsx&view=preview)

## カナリアバージョン
次にReactのカナリアバージョンにおける挙動を確認しましょう。

```tsx:tsx
<div>
  <p>⬇︎⬇︎⬇︎タイトル⬇︎⬇︎⬇︎</p>
  <title>タイトル</title>
  <p>⬆︎⬆︎⬆︎タイトル⬆︎⬆︎⬆︎</p>
</div>
```
先ほどと同じように上記のようなJSXを記述した場合、描画結果は以下のようになります。
```html:html
<div>
  <p>⬇︎⬇︎⬇︎タイトル⬇︎⬇︎⬇︎</p>e>
  <p>⬆︎⬆︎⬆︎タイトル⬆︎⬆︎⬆︎</p>
</div>
```
該当するコンポーネントから`title`要素の部分が消えています。消えた`title`要素は`head`要素内に追加されます。既に`title`要素が設定されていた場合は先頭に追加されます。
先頭に追加されるため、ドキュメントのタイトルが設定したものに変更されます。

先程と同じようなデモですが、今回はタイトルが変更されていることが確認できると思います。

@[stackblitz](https://stackblitz.com/edit/vitejs-vite-iesjot?embed=1&file=src%2FApp.tsx&view=preview)

## 注意点
### 複数宣言する
`title`要素は他の要素と同じように複数のコンポーネントで複数の箇所で宣言可能です。それらが描画されると、`head`要素内に`title`要素が次々追加さます。

単一のコンポーネントで複数宣言する場合のデモは以下のとおりです。

@[stackblitz](https://stackblitz.com/edit/vitejs-vite-xxmbyb?embed=1&file=src%2FApp.tsx&view=preview)

レンダリングされる順番通りに`head`要素内`title`要素が追加されて行くように見えます。

単一のコンポーネントでしたのでどのように`title`要素が追加されるかわかりやすかったですが、複数のコンポーネントでは`title`要素がどのように追加されるか想像するのが難しくドキュメントのタイトルをうまく設定できません。
さらに今後の改修によって`title`要素がどのように追加されるかは変化する可能性もありますし、`title`要素が多いことで検索エンジンやブラウザ等の解釈に悪影響があるかもしれません。レンダリング毎に宣言する`title`要素はできるだけ1つとなるようにしてください。

このような背景があるので、Reactで`title`要素を設定するようにする場合は描画先のHTMLの`head`要素に`title`要素を配置しない方が良いと思います。
コンポーネントで宣言した`title`要素はライフサイクルに合わせて`head`要素から取り除かれます。同じライフサイクルでレンダリングされないコンポーネントから`title`要素を宣言するのは問題ありません。

### children
`title`要素の`children`は単一の文字列である必要があります。

下記のデモを見てください(動作を確認するために`title`要素を複数宣言しています)。

@[stackblitz](https://stackblitz.com/edit/vitejs-vite-drp9hb?embed=1&file=src%2FApp.tsx)

`title`要素をそれぞれ`hello {world}`と`{'hello ' + world}`で宣言しました。`head`要素内を見てみると、前者は`<title></title>`として設定され、後者は`<title>hello world</title>`と設定されています。
`console`を見てください。それぞれがどのようにレンダリングされているかを確認できます。多くは変わりませんが、`props`の`children`が配列になっているものと文字列になっているものに分かれていると思います。
```js
// 前者のconsole logを抜粋
{
  $$typeof: Symbol(react.element)
  key: null,
  props: {
    children: ['hello ', 'world'],
  },
  ref: null,
  type: 'title',
}

// 後者のconsole logを抜粋
{
  $$typeof: Symbol(react.element)
  key: null,
  props: {
    children: 'hello world',
  },
  ref: null,
  type: 'title',
}
```
`title`要素を利用するときは`children`が文字列となるようにする必要があります。
自明な仕様ではありませんが、利用する際は注意してください。

### 例外
以下のパターンで`title`が利用された場合は挙動が最新のバージョンと同じになります。

- `svg`要素内の`title`要素
- `title`要素が`props`に`itemProp`を持つ(itempropについては[仕様書](https://html.spec.whatwg.org/multipage/microdata.html#names:-the-itemprop-attribute)を確認してください)

どちらもドキュメントのタイトルに設定するとは異なる意味を持つのでこのような仕様になっています。むしろ通常の利用が例外的に`head`要素内に追加されると考えた方が良いと思います。
