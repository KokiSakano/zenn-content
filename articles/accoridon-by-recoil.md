---
title: "アコーディオンのベースのコンポーネントをRecoilを使って作る"
emoji: "🪗"
type: "tech"
topics: ["a11y", "React", "accordion", "Recoil", "tailwindcss" ]
published: true
---
## はじめに
この記事では定義したアプリケーションで汎用的に使えるアコーディオンコンポーネントを紹介します。
具体的には[Chakra UI](https://chakra-ui.com/)や[Material UI](https://mui.com/)が提供するアコーディオンのような高い再利用性を保ちつつ、スタイリングの観点における自由度が小さいものを作ります。
このように作ることで、1アプリケーションにおける様々な場面で利用可能で、デザインや体験の揺れが小さい、特定のアプリ内で利用するには最適な共通コンポーネントを作ることができます。

この記事で作成するアコーディオンは[ARIA Authoring Practices Guide (APG)](https://www.w3.org/WAI/ARIA/apg/patterns/accordion/examples/accordion/)を元に作成しました。このサイトでは他の様々なパターンの実装がアクセシビリティの観点と合わせて紹介されているので他のコンポーネントを作る際も参照するのがおすすめです。

作成するコンポーネントは`Accordion`、`AccordionItem`、`AccordionButton`、`AccordionPanel`の4つです。これらのコンポーネントを組み合わせてアコーディオンを作ります。
`Summary1`コンポーネントを持つボタンを操作することで`Detail1`コンポーネントの表示が切り替えられらるアコーディオンと、`Summary2`コンポーネントを持つボタンを操作することで`Detail2`コンポーネントの表示が切り替えられるアコーディオンの２つを持つ1連のコンポーネントを作る時は以下のように組み立てます。

```tsx
<Accordion>
  <AccordionItem>
    <AccordionButton>
      <Summary1 />
    </AccodionButton>
    <AccordionPanel>
      <Detail1 />
    </AccodionPanel>
  </AccodionItem>
  <AccordionItem>
    <AccordionButton>
      <Summary2 />
    </AccodionButton>
    <AccordionPanel>
      <Detail2 />
    </AccodionPanel>
  </AccodionItem>
</Accodion>
```
実際にこれから作るコンポーネントを利用して以下のようにアコーディオンを表現できます。

@[codesandbox](https://codesandbox.io/embed/accordion-xvzptr?fontsize=14&hidenavigation=1&theme=dark)

## 利用する技術
アコーディオンで用いる状態管理は[recoil](https://recoiljs.org)を用いました。Reactの`Context`でも実装可能ですが、アプリケーション全体の状態管理としてrecoilを用いることが好きなのでここでも用いました。recoilが持つ特別な機能を用いているわけではないので、自由に置き換えてご覧ください。
スタイリングには[tailwindcss](https://tailwindcss.com/)を、iconには[heroicons](https://heroicons.com/)を利用しました。
tailwindcssは指定するclassの量が大きく読みづらいことがあるので、見やすさのために[`clsx`](https://github.com/lukeed/clsx/releases)も利用しました。

## Accordion
一連のアコーディオンをまとめる役割を持つコンポーネントです。このコンポーネントが持つアコーディオンがフォーカスされているとき、このコンポーネント自身の見た目も変化させることでよりフォーカスされている箇所を視覚に訴えかけます。

```tsx
import { FC, ReactNode } from 'react';
import clsx from 'clsx';

export const Accordion: FC<{ children: ReactNode }> = ({ children }) => {
  return (
    <div
      className={clsx(
        'rounded-md border-2 p-2',
        'focus-within:border-transparent focus-within:outline-none focus-within:ring-2 focus-within:ring-blue-500',
      )}
    >
      {children}
    </div>
  );
};
```
ここでは記述しませんでしたが、このコンポーネントが持つアコーディオンのうち一つしか開くようにできない制約を設ける場合など、一連のアコーディオンが互いに依存して動きを作り出すための状態はこのコンポーネントで管理します。

## AccordionItem
単一のアコーディオンを管理するコンポーネントです。開閉の状態のようなそれぞれのアコーディオンが持つ基本的な状態を管理します。具体的には`RecoilRoot`を設けて状態をこのコンポーネント内にスコープさせます。これによってコンポーネントの呼び出しごとにrecoilの状態が分割され、それぞれで独立した状態として扱えます。`Accordion`で管理する状態がある場合は`RecoilRoot`の引数に`override`を調整するなどの調整が必要になります。

```tsx
'use client';

import { FC, PropsWithChildren, useId } from 'react';
import { RecoilRoot } from 'recoil';
import { itemIdState, openState } from './state';

export const AccordionItem: FC<
  PropsWithChildren<{ defaultOpen?: boolean }>
> = ({ children, defaultOpen = false }) => {
  const id = useId();
  return (
    <RecoilRoot
      initializeState={(mutableSnapshot) => {
        mutableSnapshot.set(openState, defaultOpen);
        mutableSnapshot.set(itemIdState, id);
      }}
    >
      <div className="border-b">{children}</div>
    </RecoilRoot>
  );
};

// state.ts
import { atom } from 'recoil';

export const openState = atom<boolean>({
  key: 'open',
  default: false,
});

export const itemIdState = atom<string>({
  key: 'itemId',
  default: '',
});
```
このコンポーネント内で取り扱う状態は2つです。
1つはアコーディオンの開閉の状態です。これは単純に型が`boolean`の値です。
もう１つはアコーディオン内で扱う`id`が重複しないための値です。今回実装するアコーディオンは開閉ボタンとパネルに指定された`id`をもとにaria属性のやり取り行うためここで状態として扱うようにしています。

どちらも`RecoilRoot`を呼び出す際に`initializeState`で初期化を行っています。`initializeState`ではrecoilで扱う状態のミュータブルなスナップショットが提供され、それを用いて状態の更新ができます。のこでは`openState`をコンポーネントの引数をもとに初期化、
`itemIdState`は`useId`で取得した値を初期値しました。

## AccordionButton
アコーディオンを開閉するためのボタンです。
ホバー時に背景色を変更したり、フォーカス時にボタンを強調させることでボタンの状態をユーザーの視覚にわかりやすく示しています。
また、aria属性の`aria-expanded`で開閉の状態を伝え、`aria-controls`でボタンが与える影響先を指定しています。影響先として渡す文字列はこの後に紹介する`AccordionPanel`のもつ`id`です。

```tsx
'use client';

import {
  ChevronDownIcon,
  ChevronUpIcon,
} from '@heroicons/react/24/solid';
import { FC, PropsWithChildren } from 'react';
import { useRecoilState, useRecoilValue } from 'recoil';
import { itemIdState, openState } from './state';
import clsx from 'clsx';

export const AccordionButton: FC<PropsWithChildren<{}>> = ({
  children,
}) => {
  const [open, setOpen] = useRecoilState(openState);
  const id = useRecoilValue(itemIdState);
  return (
    <button
      type="button"
      className={clsx(
        'flex w-full flex-row items-center justify-between rounded-md p-2',
        'hover:bg-gray-100',
        'focus:first:border-transparent focus:first:outline-none focus:first:ring-2 focus:first:ring-blue-500',
      )}
      aria-expanded={open}
      aria-controls={`${id}-panel`}
      id={`${id}-button`}
      onClick={() => setOpen((open) => !open)}
    >
      {children}
      {open ? (
        <ChevronUpIcon className="h-4 w-4" />
      ) : (
        <ChevronDownIcon className="h-4 w-4" />
      )}
    </button>
  );
};
```
このコンポーネントは開閉に合わせて右側に表示するアイコンを明示的に指定しています。アコーディオンの開閉を示すアイコンを自由に入れ替え可能にすると、アプリケーション内でアイコンが表す意図がぼやけてしまうので固定しています。

## AccordionPanel
アコーディオンを開いたときに表示される内容を管理するコンポーネントです。

```tsx
'use client';

import { FC, PropsWithChildren } from 'react';
import { useRecoilValue } from 'recoil';
import { itemIdState, openState } from './state';
import clsx from 'clsx';

export const AccordionPanel: FC<PropsWithChildren<{}>> = ({
  children,
}) => {
  const id = useRecoilValue(itemIdState);
  const open = useRecoilValue(openState);
  return (
    <div
      id={`${id}-panel`}
      role="region"
      aria-labelledby={`${id}-button`}
      hidden={!open}
      className={clsx({ hidden: !open }, 'p-2')}
    >
      {children}
    </div>
  );
};
```
`AccrdionButton`から`aria-control`を介して参照されることもあり、非表示の場合にDOMを消すのではなくCSSを用いて表示の管理を行なっております。DOMが非表示であることを示すために非表示の時は要素に`hidden`を渡しています。
また、`role`に`region`を渡してランドマーク化を行い、`aria-labelledby`で`AccordionButton`を参照してランドマークに名付けています。

## 組み合わせる
最初にも紹介しましたが、これらのコンポーネントを用いてアコーディオンを形成すると以下のようになります。

@[codesandbox](https://codesandbox.io/embed/accordion-xvzptr?fontsize=14&hidenavigation=1&theme=dark)

キーボードで操作できることを確認してください。ボタンにフォーカスがあるときはスペースやエンターで内容の開閉ができ、タブで次のボタンに移動、シフトとタブの同時押しで前のボタンに移動するように操作できるようになっています。

## さいごに
ReactでRecoilを用いてAccordionを作成する方法を紹介しました。W3Cが提供する例に従ってアクセシビリティに考慮しつつ、アプリケーション内だけで利用することを活かすことに留意してコンポーネントを作りました。
他のコンポーネントも同じように作ることでより簡単により良いアプリケーションを作成できるので参考になれば幸いです。
