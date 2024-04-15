---
title: "Reactコンポーネントでchildrenに対してpropsを渡す"
emoji: "🫠"
type: "tech"
topics: [react, children, props, cloneElement, renderItem]
published: true
---

任意のコンポーネント(`ReactNode`)を子として持つコンポーネントを開発するとき、子コンポーネントの`props`を指定したいことがあります。
```tsx
import { FC, ReactNode } from 'react';

type Props = {
  children: ReactNode;
};

const ParentComponent: FC<Props> = ({
  children,
}) => (
  <div>
    {/* childrenにonClick等のpropsを親から付与したい */}
    {children}
  </div>
);
```
この記事ではこのようなパターンを実装する2つの方法と、それを活用したツールチップの実装例を紹介します。

## `cloneElement`
:::message alert
ここで紹介する`cloneElement`と`isValidElement`はReactのレガシーAPIです。
推奨されるAPIではないため新しく実装する場合はこの後に紹介するrender propパターンを使ってください。
:::
最初に紹介する方法は`cloneElement`が対象の`ReactElement`を複製する時に、第2引数に渡したオブジェクトで`props`を上書きする機能を利用した実装です。
子コンポーネントの`props`に`onClick`を付与したものを子要素としてレンダリングするコンポーネントは以下のように書けます。

```tsx
import { cloneElement, FC, isValidElement, ReactNode } from 'react';

const ParentComponent: FC<Props> = ({ children }) => {
  const cloneChildren = cloneElement(
    isValidElement(children) ? children : <button>{children}</button>,
    {
      onClick: () => alert('親から付与されたclickイベントです。'),
    }
  );
  return <div>{cloneChildren}</div>;
};
```

`cloneElement`の第1引数は`ReactElement`を渡したいので、`isValidElement`を用いて`ReactElement`を抽出しました。`children`が`ReactElement`ではなかった場合は`button`要素で囲んだものを渡すようにしています。
そして、第2引数は`props`に`onClick`を付与するようなオブジェクトを渡しました。第2引数にオブジェクトを渡すと`children`自身が持っていた`props`が全て置き換えられることに注意して下さい。
元の`props`を保持させたい場合は以下のようにします。
```ts
const cloneChildren = isValidElement(children)
  ? cloneElement(children, {
      // 子要素のpropsを優先したいときは順番を入れ替える
      ...children.props,
      onClick: () => alert('親から付与されたclickイベントです。'),
    })
  : cloneElement(<button>{children}</button>, {
      // ReactElementじゃないときは`button`に対して付与されるので考慮不要
      onClick: () => alert('親から付与されたclickイベントです。'),
    });
```

### 例
例として素の`button`要素を宣言したものと、`ParentComponent`によって素の`button`要素に`onClick`を付与したもの、`ParentComponent`によって`onClick`を持つ`button`要素に`onClick`を付与した3つを準備しました。
@[stackblitz](https://stackblitz.com/edit/vitejs-vite-j7zhmo?embed=1&file=src%2FApp.tsx&view=preview)

## render propパターン
続いてはrender propパターン呼ばれる`props`を利用した実装です。
コンポーネントから子コンポーネントを組み上げる`renderItem`というツールキットを提供して、付与したい`props`を用いた子コンポーネントを宣言させます。

```tsx
import { FC, ReactElement } from 'react';

type Props = {
  renderItem: (props: { onClick: () => void }) => ReactElement;
};

const ParentComponent: FC<Props> = ({ renderItem }) => {
  return (
    <div>
      {renderItem({
        onClick: () => alert('親から付与されたclickイベントです。'),
      })}
    </div>
  );
};
```
`renderItem`は付与させたい`props`を引数として持ち、`ReactElement`を返すような関数です。利用する側では`props`を受け取って、それを付与したコンポーネントを返すようにします。
```tsx
<ParentComponent
  renderItem={(props) => (
    <button {...props}>
      親コンポーネントから渡ってきたonClickを付与したbutton要素
    </button>
  )}
/>
```
この方法では指定したい`props`を子コンポーネント側に明示的に渡せることや子コンポーネントの実装を自身でハンドリングできるところに利点があります。

### 例
`cloneElement`で紹介したものと同じような例です。
@[stackblitz](https://stackblitz.com/edit/vitejs-vite-t4zd3z?embed=1&file=src%2FApp.tsx&view=preview)

## ツールチップを実装する
render propを有効活用できる例としてツールチップを実装します。

この記事ではツールチップを開閉等の状態をもつ`Root`コンポーネントと、ツールチップの開閉のトリガーとなる`Trigger`コンポーネント、表示されるツールチップ自体である`Content`コンポーネントから構築します。
### 状態
まずはツールチップの開閉等の情報を司る状態を定義します。
必要な状態は開閉についての情報である`isOpen`と開閉を切り替える`onOpen`・`onClose`、そして各コンポーネントを連携するとき活用する`contextId`の4つです。
```ts
type Context = {
  isOpen: boolean;
  onOpen: () => void;
  onClose: () => void;
  contextId: string;
};
```
これらの情報を各コンポーネントで扱えるように`createContext`でコンテキストを定義します。
```ts
const TooltipContext = createContext<Context | null>(null);

const useTooltipContext = (): Context => {
  const context = useContext(TooltipContext);
  if (context === null) {
    throw new Error('TooltipContextのProviderを定義して下さい。');
  }
  return context;
};
```
コンテキストを初期値の`null`のままで活用されないように`useTooltipContext`としてcustom hooksに切り出しました。

### Tooltip.Root
次に、他のコンポーネントの親となる`Root`コンポーネントを実装します。
```tsx
const Root: FC<{
  children: ReactNode;
}> = ({ children }) => {
  const contextId = useId();
  const [isOpen, setIsOpen] = useState(false);

  const onOpen = useCallback(() => {
    setIsOpen(true);
  }, []);

  const onClose = useCallback(() => {
    setIsOpen(false);
  }, []);

  // escキーを押したらツールチップを閉じさせる
  useEffect(() => {
    const abortController = new AbortController();
    document.addEventListener(
      'keydown',
      (e) => {
        if (e.key === 'Escape') {
          onClose();
        }
      },
      {
        signal: abortController.signal,
      }
    );
    return () => {
      abortController.abort();
    };
  }, [onClose]);

  return (
    <TooltipContext.Provider
      value={{
        isOpen,
        onOpen,
        onClose,
        contextId,
      }}
    >
      <div style={{ position: 'relative' }}>{children}</div>
    </TooltipContext.Provider>
  );
};
```
先ほど定義した`TooltipContext`に必要な情報を定義して、`TooltipContext.Provider`で情報を流し込んでいます。
今回は`Content`コンポーネントを`absolute`要素で実装しようと考えているため、`position: relative`な`div`も配置しています。

### Tooltip.Content
続いて、`Tooltip`の本体である`Content`コンポーネントを実装します。
```tsx
const Content: FC<{ children: ReactNode }> = ({ children }) => {
  const { isOpen, contextId } = useTooltipContext();

  return (
    <div
      id={contextId}
      role="tooltip"
      style={{
        top: '50%',
        color: 'white',
        backgroundColor: '#333',
        borderRadius: '10px',
        left: 0,
        opacity: isOpen ? 1 : 0,
        padding: '8px',
        pointerEvents: 'none',
        position: 'absolute',
        transform: 'translate(-25%, -150%)',
        width: 'max-content',
        zIndex: 1,
      }}
    >
      {children}
    </div>
  );
};
```
`Trigger`コンポーネントから参照させるために`TooltipContext`の`contextId`を`id`として付与しました。他は`role`や`style`の付与を行なっています。
`style`は汎用的なものではないので、参考にしないでください。[`floating-ui`](https://floating-ui.com/)を用いて表示位置を求めるのがおすすめです。

### Tooltip.Trigger
最後に`Trigger`の実装です。ここでrender propパターンを用います。
```tsx
const Trigger: FC<{
  renderItem: (props: {
    'aria-describedby'?: string;
    onMouseEnter: () => void;
    onMouseLeave: () => void;
    onFocus: () => void;
    onBlur: () => void;
  }) => ReactElement;
}> = ({ renderItem }) => {
  const { isOpen, onOpen, onClose, contextId } = useTooltipContext();

  return renderItem({
    ['aria-describedby']: isOpen ? contextId : undefined,
    onMouseEnter: onOpen,
    onMouseLeave: onClose,
    onFocus: onOpen,
    onBlur: onClose,
  });
};
```
`TooltipContext`から受け取った開閉を切り替える関数をもとに`onMouseEnter`、`onMouseLeave`、`onFocus`、`onBlur`を子コンポーネントに付与するように実装しています。`onMouseEnter`、`onMouseLeave`でマウスカーソルを合わせることによる開閉、`onFocus`、`onBlur`でフォーカス時の開閉を実装しています。
`Trigger`コンポーネントを説明する内容が`Content`コンポーネントとして表示されるはずなので、`Content`の`id`として渡した`contextId`を`aria-describedby`に付与しました。

### ツールチップを利用する
作成したツールチップは以下のように使います。
```tsx
<Tooltip.Root>
  <Tooltip.Content>tooltip content</Tooltip.Content>
  <Tooltip.Trigger
    renderItem={(props) => <button {...props}>tooltip</button>}
  />
</Tooltip.Root>
```
`renderItem`でトリガーするコンポーネントの宣言と、そこで必要な`props`の付与を行なっています。
これによって、特定の見た目から発火するツールチップではなく利用する側でコンポーネントの見た目や振る舞いを選択できるツールチップができました。

実際に動いている様子は下から確認できます。
@[stackblitz](https://stackblitz.com/edit/vitejs-vite-pzx8az?embed=1&file=src%2FApp.tsx)
