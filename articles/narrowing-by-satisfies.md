---
title: "特定のケースではsatisfies演算子で型が絞り込まれてしまう？？？"
emoji: "🔨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["TypeScript", "satisfies", "narrowing"]
published: true
---

## satisfies演算子
`satisfies`はTypeScript4.9で追加された演算子です。[4.9のドキュメント](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-9.html#the-satisfies-operator)では次のように紹介されています。
> The new `satisfies` operator lets us validate that the type of an expression matches some type, without changing the resulting type of that expression.
>
> (翻訳 by DeepL)新しい `satisfies` 演算子を使うと、式の結果の型を変更することなく、式の型がある型にマッチするかどうかを検証することができる。

以下の3つの`palette`オブジェクトを比較してみます([Playground](https://www.typescriptlang.org/play?#code/FAFwngDgpgBAwgewDYIE4GcYF4YCJVQAmuMAPngOYFQB2J5uARkgK5S4DcoksASgOIAhbDADaBQgC4YNFgFtGUVABoYVKLWmyFS1czZb5i1AF0uwAMYIa6EDAgBDJFBAgoARhEBvYDD8wJaVEAJgBWUNUABiiTZV9-dU08AGJIyIAzdLTcOP8YZigWIOiYErDQk2AAXxgHTCsbEHMG23snFzdg6V4oK1RCAB5EFAxVW1QASxoKMhgBQQA+b3i-QLFyqJjchOoaaVxUjKzInJX81ihizZhyypq6mBam4EtrVsdnVygAZmW8tZC4WukViZ0SexSaUy2W2fn0lzEJTK4TutXqbzs6AcIAm6HSEygmB6fUGwzQ6DGIEm01m8wWHCAA)で確認する)。各例では`blue`を`bleu`と誤って記述しています。
```ts
type Colors = "red" | "green" | "blue";
type RGB = [red: number, green: number, blue: number];

const palette1 = {
  red: [255, 0, 0],
  green: "#00ff00",
  bleu: [0, 0, 255]
} as const;

const palette2: Record<Colors, string | RGB> = {
  red: [255, 0, 0],
  green: "#00ff00",
  bleu: [0, 0, 255]
} as const;

const palette3 = {
  red: [255, 0, 0],
  green: "#00ff00",
  bleu: [0, 0, 255]
} as const satisfies Record<Colors, string | RGB>;
```
`palette1`には型の制約がないため、タイプミスがすぐには分かりません。一方で`palette2`は型注釈を通じて、`palette3`は`satisfies`演算子を使用して制約を課しています。これにより、タイプミスがエラーとして検出されます。

タイプミスを修正するとそれぞれの型は以下のように推論されます。
```ts
declare const palette1: {
  readonly red: readonly [255, 0, 0];
  readonly green: "#00ff00";
  readonly blue: readonly [0, 0, 255];
};

declare const palette2: Record<Colors, string | RGB>;

declare const palette3: {
  readonly red: [255, 0, 0];
  readonly green: "#00ff00";
  readonly blue: [0, 0, 255];
};
```
`palette1`と`palette3`は同じように推論され、`palette2`は型注釈通りの型に推論されます。

このように`satisfies`演算子は対象の式の型を変更せずに、式に対する制約だけを課すような働きを持ちます。

## 対象の式の型が変化するケース
次に以下の`user`を比較してみましょう([Playground](https://www.typescriptlang.org/play?#code/C4TwDgpgBAqgzhATlAvFA3gKClAlgEwC4o5hFcA7AcwG5spRJiByYCAQwGMALJZqAD5RmYAK5hcAG2YAaehXYBbCMVLlqdHLjgB9AG5JcAM1wQiUAEYB7K5I4U6AXzqZOViqSiiEiAIyoMei1zZnZZIIZwFWE2Ll5EcJwcBWUWdigwuSS8XQNyEzNiMlEIOWdMV3dPbyQAJgCsbII0xKTGaNYOHj4spJSO9MyI7X1DAvNi0sxHEnZgbQK4WB8XIA)で確認する)。
```ts
type User = {
  id: string;
  type: 'teacher' | 'pupil',
  name: string;
  is_verified: boolean;
};

const user1 = {
  id: 'a',
  type: 'teacher',
  name: 'a a',
  is_verified: true,
};

const user2 = {
  id: 'a',
  type: 'teacher',
  name: 'a a',
  is_verified: true,
} satisfies User;
```
`satisfies`演算子を使用すると、`User`型の制約が`user2`に適用されますが、推論される型は`user1`と同じになると期待されます。しかし、実際には次のように異なる型情報が推論されます。
```ts
declare const user1: {
  id: string;
  type: string;
  name: string;
  is_verified: boolean;
};

declare const user2: {
  id: string;
  type: "teacher";
  name: string;
  is_verified: true;
};
```
`id`や`name`は同一ですが、`type`と`is_verified`だけ`'teacher'`や`true`のよう型が絞り込まれています。

`satisfies`の定義に反しているように見えますが、オブジェクトの型推論と`satisfies`の制約で整合が取れなくなるためこのような挙動となっています。

`user1`の推論結果から分かるように`'teacher'`のようなリテラルはそれを拡張した`string`のようなプリミティブな型として推論されます。
しかし、そのように推論された場合、`satisfies`演算子における検証がうまく行えません。
例えば`'guardian'`と`'teacher'`の両方ともが`string`として拡張されてしまうと、`satisfies`演算子で検証を行うとき、`string extends 'teacher' | 'pupil'`のようになるので常に制約を満たしていないと判断されます。

制約を満たすことを正しく判別するため、リテラル型のユニオンで制約を課された部分はリテラル型が保持されるように推論されます。
先ほどの例で言えば`'guardian' extends 'teacher' | 'pupil'`や`'teacher' extends 'teacher' | 'pupil'`のようになるので正しく判別できるようになっています。

`is_verified`の制約がプリミティブ型の`boolean`であるにも関わらず、`true`が`boolean`に拡張されない理由は`boolean`が`true | false`のエイリアスであるため、`true`と`false`というリテラル型のユニオンが制約として課されたと判別されるためです。

配列の場合でも同じように型推論されます([Playground](https://www.typescriptlang.org/play?#code/C4TwDgpgBAcgrgWwEYQE4GcoF4oAoCMUAPlAEzFQDMFALAJQDaAugNwBQbAxgPYB26wKL0QoMhHA3wAaMjMqsOPfoOHI06chOmyqTKOgCGwAJboAZsYiZ4ajOyA)で確認する)。
```ts
type Numbers = (1 | 2 | 3 | 4)[];

const numbers1 = [1, 2, 3];

const numbers2 = [1, 2, 3] satisfies Numbers;
```
`numbers1`は単なる`number`型の配列として推論されますが、`number2`はリテラル型のユニオンが制約として設けられているので`(1 | 2 | 3)[]`の特定の数値型として推論されています。
```ts
declare const numbers1: number[];

declare const numbers2: (1 | 2 | 3)[];
```
`as const`を利用した時のような`[1, 2, 3]`のようなタプル型に推論されていないことに気をつけて下さい。

## 似た例
型の制約によって、リテラル型として推論される例は他にもあります。
以下のコードを見てください([Playground](https://www.typescriptlang.org/play?#code/C4TwDgpgBAMglsCAnAhgGwCrmgXigbwCgopEBnYALigHIUaoAfWgIxoG5CBfT0SKAApI4AWwRwAbhCz88+KMVIQK1CsIB2Ac26dCAYwD26ilCTKArmmBQ5i8lVr0dhQgBMIetCjNRDx6-YAjNQAPBhQEAAeiOquZILCYsCS0tgAfAAU3prUGACUNmlQGJzunt7Qfib2AEyh4VExcbAIyOgyEJnZuQU4RSUuVdZmZJbAgTZKFIEZ8vbUdAxceZxDphZWNZO1s1MOi1DLnEA)で確認する)。
```ts
type PrimitiveType = {
  test: string
};
type LiteralType = {
  test: 'a' | 'b';
};

const result = {
  test: 'a'
};

declare const test1: <T extends PrimitiveType>(arg: T) => T;
declare const test2: <T extends LiteralType>(arg: T) => T;

const result1 = test1({ test: 'a' });
const result2 = test2({ test: 'a' });
```
`result`、`result1`、`result2`は以下のように型が推論されます。
```ts
declare const result: {
    test: string;
};

declare const result1: {
    test: string;
};

declare const result2: {
    test: "a";
};
```
`result2`だけ、`string`ではなく`'a'`と推論されています。
`test2`関数は、`T extends LiteralType`の箇所で`'a' | 'b'`であることを課しているためプリミティブ型に拡張されませんでした。

## まとめ
あるリテラルがそのリテラルの対応する型を含むユニオンによって型を文脈的に指定される場合、親プリミティブに拡張されるのではなく実際のリテラル型が保持されます。
`satisfies`によって型の制約を課す場合はこのような挙動をすることを知った上で使っていきたいです。

## 参考

https://github.com/microsoft/TypeScript/issues/55189

https://github.com/microsoft/TypeScript/issues/47920

https://github.com/microsoft/TypeScript/pull/46827

https://github.com/microsoft/TypeScript/issues/48696
