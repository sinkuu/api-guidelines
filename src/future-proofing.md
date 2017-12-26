# 将来性


<a id="c-sealed"></a>
## sealedトレイトを使って下流の実装を適切に防いでいる (C-SEALED)

そのクレート内でのみ実装されることを想定したトレイトについて、
sealedトレイトパターンを用いることでユーザのコードを壊すことなしに変更を加えることが可能になります。

```rust
/// このトレイトはsealされているため、他のクレートで実装を追加することはできません。
pub trait TheTrait: private::Sealed {
    // メソッド
    fn ...();

    // ユーザが呼ぶべきでないプライベートメソッド
    #[doc(hidden)]
    fn ...();
}

// 実装
impl TheTrait for usize {
    /* ... */
}

mod private {
    pub trait Sealed {}

    // 同じ型に実装
    impl Sealed for usize {}
}
```

プライベートな空の`Sealed`親トレイトを下流のクレートから参照することはできません。
従って、`Sealed`(そして`TheTrait`)の実装はこのクレート内にのみ存在できます。
トレイトにメソッドを追加することは一般的に破壊的変更となりますが、
sealedトレイトである`TheTrait`にメソッドを追加することは破壊的変更になりません。
また、ドキュメントに掲載されていないメソッドの定義も自由に変更することができます。

sealedトレイトからパブリックなメソッドを取り除いたり、
定義を変更したりすることは依然として破壊的変更であることに注意してください。

混乱したユーザがそれらのトレイトを実装しようとすることを防ぐため、
そのトレイトはsealされており、他のクレートから実装されるべきものではないことを
ドキュメントに記載しておくべきです。

### 例

- [`serde_json::value::Index`](https://docs.serde.rs/serde_json/value/trait.Index.html)
- [`byteorder::ByteOrder`](https://docs.rs/byteorder/1.1.0/byteorder/trait.ByteOrder.html)


<a id="c-struct-private"></a>
## 構造体のフィールドを適切にプライベートにする (C-STRUCT-PRIVATE)

構造体のフィールドをパブリックにすることには重大な責任が伴います。
表現を変更することはできなくなり、またユーザはフィールドを自由に弄ることができるため
値のバリデーションや不変条件の検証などができなくなります。

パブリックなフィールドはC言語的な意味あいの`sturct`、すなわち複合化された受け身のデータ構造には最適ですが、
それ以外ではgetter/setterメソッドを用意しフィールドを隠蔽することを考慮してください。


<a id="c-newtype-hide"></a>
## newtypeを用いて実装詳細を隠蔽している (C-NEWTYPE-HIDE)

newtypeはユーザへの保証を保ちつつ実装詳細を隠蔽するために役立ちます。

例としてこの、イテレータ型を返す`my_transform`関数を見てください。

```rust
use std::iter::{Enumerate, Skip};

pub fn my_transform<I: Iterator>(input: I) -> Enumerate<Skip<I>> {
    input.skip(3).enumerate()
}
```

ユーザから見た際に`Iterator<Item = (usize, T)>`のように見えるよう型を隠したいときは、
newtype型を使うことができます。

```rust
use std::iter::{Enumerate, Skip};

pub struct MyTransformResult<I>(Enumerate<Skip<I>>);

impl<I: Iterator> Iterator for MyTransformResult<I> {
    type Item = (usize, I::Item);

    fn next(&mut self) -> Option<Self::Item> {
        self.0.next()
    }
}

pub fn my_transform<I: Iterator>(input: I) -> MyTransformResult<I> {
    MyTransformResult(input.skip(3).enumerate())
}
```

これにより宣言が簡単になるだけでなく、ユーザへの保証を小さくすることができます。
ユーザは返されたイテレータがどのように生成されたのか、どのような内部表現になっているのかを知ることができません。
したがって、ユーザのコードを壊すこと無く将来的に内部表現を変更できるようになります。

[`impl Trait`]は現在のところunstableですが、将来的にはこれを用いても同じことが達成できるようになります。

[`impl Trait`]: https://github.com/rust-lang/rfcs/blob/master/text/1522-conservative-impl-trait.md

```rust
#![feature(conservative_impl_trait)]

pub fn my_transform<I: Iterator>(input: I) -> impl Iterator<Item = (usize, I::Item)> {
    input.skip(3).enumerate()
}
```


<a id="c-struct-bounds"></a>
## データ構造にderiveしたトレイトの境界を定義で繰り返さない (C-STRUCT-BOUNDS)

ジェネリックなデータ構造はderiveしたトレイト境界をその定義において繰り返すべきではありません。
`derive`属性によって実装されたトレイトは、ジェネリック型がそのトレイトを実装している場合のみ実装される
個別の`impl`ブロックに展開されます。

```rust
// 良い例:
#[derive(Clone, Debug, PartialEq)]
struct Good<T> { /* ... */ }

// 悪い例:
#[derive(Clone, Debug, PartialEq)]
struct Bad<T: Clone + Debug + PartialEq> { /* ... */ }
```

`Bad`のようにderiveしたトレイトを境界として繰り返すのは不要であり、
しかも後方互換性を保つ上で困難となります。
なぜなら、ここで`PartialOrd`をderiveした場合を考えてみて下さい。

```rust
// 非破壊的変更:
#[derive(Clone, Debug, PartialEq, PartialOrd)]
struct Good<T> { /* ... */ }

// 破壊的変更:
#[derive(Clone, Debug, PartialEq, PartialOrd)]
struct Bad<T: Clone + Debug + PartialEq + PartialOrd> { /* ... */ }
```

一般的に、データ構造にトレイト境界を追加すると全ての利用箇所において追加の境界を満たす必要が発生するため、
破壊的変更となります。
しかし、`derive`属性を用いて標準ライブラリのトレイトを実装することは破壊的変更となりません。

以下のトレイトはデータ構造において境界とするべきではありません。

- `Clone`
- `PartialEq`
- `PartialOrd`
- `Debug`
- `Display`
- `Default`
- `Serialize`
- `Deserialize`
- `DeserializeOwned`

`Read`や`Write`のようなderiveできないトレイトの中には、
厳密にはデータ構造によって要求されないグレーゾーンのものが存在します。
これらは型のふるまいを伝える役に立つ可能性がありますが、一方で将来的な拡張性の障害にもなります。
しかし、deriveできるトレイトを境界に追加するよりは問題が少ないでしょう。

### 例外

データ構造にトレイト境界が必要となる、3つの例外があります。

1. データ構造がトレイトの関連型を参照している。
1. `?Sized`境界。
1. データ構造がそのトレイト境界を必要とする`Drop`実装を持っている。Rustは現在、
`Drop`実装の境界がデータ構造自身にもすることを要求します。

### 標準ライブラリでの例

- [`std::borrow::Cow`]は`Borrow`トレイトの関連型を参照しています。
- [`std::boxed::Box`]は暗黙の`Sized`境界を除いています。
- [`std::io::BufWriter`]は`Drop`実装に必要である境界を型に要求します。

[`std::borrow::Cow`]: https://doc.rust-lang.org/std/borrow/enum.Cow.html
[`std::boxed::Box`]: https://doc.rust-lang.org/std/boxed/struct.Box.html
[`std::io::BufWriter`]: https://doc.rust-lang.org/std/io/struct.BufWriter.html
