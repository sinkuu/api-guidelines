# 相互運用性


<a id="c-common-traits"></a>
## 積極的に一般的なトレイトを型に実装している (C-COMMON-TRAITS)

Rustのトレイトシステムは _孤立_ を許容しません。
大まかに言うと、全ての`impl`はトレイトのあるクレートか、実装対象の型があるクレートに置かれる必要があります。
従って、型を定義するクレートは積極的に、可能な限り全ての一般的なトレイトを実装すべきです。

次のような状況を考えてみてください。

* `std`クレートが`Display`トレイトを宣言。
* `url`クレートが`Url`型を宣言、`Display`を実装せず。
* `webapp`クレートが`std`と`url`をインポート。

この場合、`url`が`Display`を実装していない以上、`webapp`にはどうすることもできません。
(newtypeパターンは有効な回避策ですが、不便です。)

最も重要な、一般的なトレイトを`std`から挙げます。

- [`Copy`](https://doc.rust-lang.org/std/marker/trait.Copy.html)
- [`Clone`](https://doc.rust-lang.org/std/clone/trait.Clone.html)
- [`Eq`](https://doc.rust-lang.org/std/cmp/trait.Eq.html)
- [`PartialEq`](https://doc.rust-lang.org/std/cmp/trait.PartialEq.html)
- [`Ord`](https://doc.rust-lang.org/std/cmp/trait.Ord.html)
- [`PartialOrd`](https://doc.rust-lang.org/std/cmp/trait.PartialOrd.html)
- [`Hash`](https://doc.rust-lang.org/std/hash/trait.Hash.html)
- [`Debug`](https://doc.rust-lang.org/std/fmt/trait.Debug.html)
- [`Display`](https://doc.rust-lang.org/std/fmt/trait.Display.html)
- [`Default`](https://doc.rust-lang.org/std/default/trait.Default.html)

ある型に`Default`を実装し、引数を取らない`new`コンストラクタを追加するのは
一般的かつ開発者の予期するところであることに注意してください。
`new`は慣用的なコンストラクタであり、利用者はそれが存在することを期待します。
従ってコンストラクタが引数を取らないのが自然であれば、
`default`と機能的に同一であったとしてもコンストラクタが存在しているべきです。


<a id="c-conv-traits"></a>
## 変換に標準のトレイト`From`, `AsRef`, `AsMut`を用いている (C-CONV-TRAITS)

以下の変換用トレイトは理にかなう限り実装されているべきです。

- [`From`](https://doc.rust-lang.org/std/convert/trait.From.html)
- [`TryFrom`](https://doc.rust-lang.org/std/convert/trait.TryFrom.html)
- [`AsRef`](https://doc.rust-lang.org/std/convert/trait.AsRef.html)
- [`AsMut`](https://doc.rust-lang.org/std/convert/trait.AsMut.html)

以下の変換用トレイトは実装されるべきではありません。

- [`Into`](https://doc.rust-lang.org/std/convert/trait.Into.html)
- [`TryInto`](https://doc.rust-lang.org/std/convert/trait.TryInto.html)

上記のトレイトは`From`, `TryFrom`を基にしたブランケット実装を持つので、
こちらを代わりに実装してください。

### 標準ライブラリでの例

- `From<u16>`が`u32`に実装されています。幅の小さい整数は常に幅の大きい整数に変換可能であるためです。
- `From<u32>`は`u16`に実装されて*いません*。数値が大きすぎると変換できないためです。
- `TryFrom<u32>`が`u16`に実装されており、`u16`に収まらないほど数値が大きければエラーを返します。
- [`From<Ipv6Addr>`]が[`IpAddr`]に実装されています。この型はv4とv6の両方のIPアドレスを表すことができます。

[`From<Ipv6Addr>`]: https://doc.rust-lang.org/std/net/struct.Ipv6Addr.html
[`IpAddr`]: https://doc.rust-lang.org/std/net/enum.IpAddr.html


<a id="c-collect"></a>
## コレクションが`FromIterator`と`Extend`を実装している (C-COLLECT)

[`FromIterator`]と[`Extend`]を実装すると、
そのコレクションは以下のイテレータメソッドと共に使うことができるようになります。

[`FromIterator`]: https://doc.rust-lang.org/std/iter/trait.FromIterator.html
[`Extend`]: https://doc.rust-lang.org/std/iter/trait.Extend.html

- [`Iterator::collect`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.collect)
- [`Iterator::partition`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.partition)
- [`Iterator::unzip`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.unzip)

`FromIterator`はイテレータが含んでいる値からコレクションを生成するものです。
`Extend`は既存のコレクションにイテレータの含む値を追加します。

### 標準ライブラリでの例

- [`Vec<T>`]は`FromIterator<T>`と`Extend<T>`の双方を実装しています。

[`Vec<T>`]: https://doc.rust-lang.org/std/vec/struct.Vec.html


<a id="c-serde"></a>
## データ構造がSerdeの`Serialize`と`Deserialize`を実装している (C-SERDE)

データ構造として使われる型は[`Serialize`]及び[`Deserialize`]を実装しているべきです。

[`Serialize`]: https://docs.serde.rs/serde/trait.Serialize.html
[`Deserialize`]: https://docs.serde.rs/serde/trait.Deserialize.html

明らかにデータ構造である型とそうでない型の間にはグレーゾーンが存在します。
[`LinkedHashMap`]や[`IpAddr`]はデータ構造です。
`LinkedHashMap`や`IpAddr`をJSONファイルから読み取りたい、
あるいはIPCを用いて他のプロセスに送りたいというのは当然の要求でしょう。
一方で[`LittleEndian`]はデータ構造ではりません。
これは`byteorder`の使うマーカ型で、 最適化によって消えるため実行時には存在しません。
これらは違いが明らかな例です。曖昧な例に遭遇した場合、必要ならば
IRCの#rustあるいは#serdeチャンネルで助言を求めることができます。

[`LinkedHashMap`]: https://docs.rs/linked-hash-map/0.4.2/linked_hash_map/struct.LinkedHashMap.html
[`IpAddr`]: https://doc.rust-lang.org/std/net/enum.IpAddr.html
[`LittleEndian`]: https://docs.rs/byteorder/1.0.0/byteorder/enum.LittleEndian.html

クレートが他の用途でSerdeに依存していなければ、
Cargoのfeatureを使ってSerdeへの依存をオプショナルにしたいと思われるかもしれません。
そうすることで、下流のライブラリがSerdeへの依存を必要とする場合のみSerdeがコンパイルされるようになります。

他のSerdeベースのライブラリとの一貫性を保つため、Cargoのfeatureの名前は単に`"serde"`とすべきです。
その他、`"serde_impls"`や`"serde_serialization"`のような名前は使わないでください。

deriveを使わない場合、典型的な例は以下のようになります。

```toml
[dependencies]
serde = { version = "1.0", optional = true }
```

```rust
#[cfg(feature = "serde")]
extern crate serde;

struct T { /* ... */ }

#[cfg(feature = "serde")]
impl Serialize for T { /* ... */ }

#[cfg(feature = "serde")]
impl<'de> Deserialize<'de> for T { /* ... */ }
```

そして、deriveを使う場合は以下のようになります。

```toml
[dependencies]
serde = { version = "1.0", optional = true, features = ["derive"] }
```

```rust
#[cfg(feature = "serde")]
#[macro_use]
extern crate serde;

#[cfg_attr(feature = "serde", derive(Serialize, Deserialize))]
struct T { /* ... */ }
```


<a id="c-send-sync"></a>
## 型が可能な限り`Send`,`Sync`である (C-SEND-SYNC)

[`Send`]および[`Sync`]はコンパイラが可能だと判断できれば自動的に実装されます。

[`Send`]: https://doc.rust-lang.org/std/marker/trait.Send.html
[`Sync`]: https://doc.rust-lang.org/std/marker/trait.Sync.html

生ポインタを操作する型の場合、`Send`および`Sync`の実装が
その型のスレッドセーフ特性を正確に表すように注意してください。
以下のようなテストを追加することで、
`Send`や`Sync`の実装が意図せず外れるような退行バグの発生を防ぐことができます。

```rust
#[test]
fn test_send() {
    fn assert_send<T: Send>() {}
    assert_send::<MyStrangeType>();
}

#[test]
fn test_sync() {
    fn assert_sync<T: Sync>() {}
    assert_sync::<MyStrangeType>();
}
```


<a id="c-good-err"></a>
## エラー型の意味が分かりやすく、行儀の良い実装となっている (C-GOOD-ERR)

エラー型とは、パブリックな関数から返される`Result<T, E>`における、任意の`E`のことです。 
エラー型は常に[`std::error::Error`]トレイトを実装しているべきです。
これは例えば[`error-chain`]のようなエラー処理ライブラリによって異なるエラー型を抽象化して扱うために使われており、
その型を別の型のエラーの[`cause()`]として使うことができるようになります。

[`std::error::Error`]: https://doc.rust-lang.org/std/error/trait.Error.html
[`error-chain`]: https://docs.rs/error-chain
[`cause()`]: https://doc.rust-lang.org/std/error/trait.Error.html#method.cause

加えて、エラー型は[`Send`]、[`Sync`]トレイトを実装するべきです。
`Send`でないエラー型は[`thread::spawn`]によって作られたスレッドから返すことはできません。
そして`Sync`でない型は[`Arc`]を用いて複数のスレッドから参照することかできません。
これらはマルチスレッディングアプリケーションにおける、基本的なエラー処理での一般的な要求です。

[`Send`]: https://doc.rust-lang.org/std/marker/trait.Send.html
[`Sync`]: https://doc.rust-lang.org/std/marker/trait.Sync.html
[`thread::spawn`]: https://doc.rust-lang.org/std/thread/fn.spawn.html
[`Arc`]: https://doc.rust-lang.org/std/sync/struct.Arc.html

`Send`と`Sync`は、`Send + Sync + Error`を要求する[`std::io::Error::new`]を用いて
カスタムエラーをIOエラーに変換する際にも必要です。

[`std::io::Error::new`]: https://doc.rust-lang.org/std/io/struct.Error.html#method.new

この指針について、注意が必要なのは[`reqwest::Error::get_ref`]のようなErrorトレイトオブジェクトを返す関数です。
普通は`Error + Send + Sync + 'static`が呼び出し側にとって最も有用です。
`'static`を追加することにより、[`Error::downcast_ref`]を使うことが可能になります。

[`reqwest::Error::get_ref`]: https://docs.rs/reqwest/0.7.2/reqwest/struct.Error.html#method.get_ref
[`Error::downcast_ref`]: https://doc.rust-lang.org/std/error/trait.Error.html#method.downcast_ref-2

例えエラーとして返せる有用な情報が無かったとしても、決して`()`をエラー型として使わないでください。

- `()`は`Error`を実装していないため、`error-chain`のようなエラー処理ライブラリと共に使うことができません。
- `()`は`Display`を実装していないため、利用者が自分でエラー時のメッセージを記述する必要があります。
- `()`は`Debug`を実装していますが、`unwrap()`した際に役に立たないものです。
- 下流のクレートが`?`演算子を使うためには、意味の不明瞭な`From<()>`をエラー型に実装しければなりません。

代わりに、クレートや関数毎に意味の分かるエラー型を定義し、
適切な`Error`および`Display`の実装を追加してください。
返すべき情報がないならばユニット構造体型として定義することができます。

```rust
use std::error::Error;
use std::fmt::Display;

// これの代わりに……
fn do_the_thing() -> Result<Wow, ()>

// こうして下さい
fn do_the_thing() -> Result<Wow, DoError>

#[derive(Debug)]
struct DoError;

impl Display for DoError { /* ... */ }
impl Error for DoError { /* ... */ }
```

`Display`によって与えられるエラーメッセージは小文字とし、最後に約物は付けないで下さい。
そして普通は簡潔なものにしてください。

[`Error::description()`]の返すメッセージはあまり気にする必要はありません。
`description()`ではなく常に`Display`を使ってエラーを表示すべきですから、
適当に`"JSON error"`のようなもので十分です。

[`Error::description()`]: https://doc.rust-lang.org/std/error/trait.Error.html#tymethod.description

### 標準ライブラリでの例

- [`ParseBoolError`]は、文字列をboolとしてパースするのに失敗した際、返される型です。

[`ParseBoolError`]: https://doc.rust-lang.org/std/str/struct.ParseBoolError.html

### エラーメッセージの例

- "unexpected end of file"
- "provided string was not \`true\` or \`false\`"
- "invalid IP address syntax"
- "second time provided was later than self"
- "invalid UTF-8 sequence of {} bytes from index {}"
- "environment variable was not valid unicode: {:?}"


<a id="c-num-fmt"></a>
## バイナリ数値型が`Hex`, `Octal`, `Binary`によるフォーマットをサポートしている (C-NUM-FMT)

- [`std::fmt::UpperHex`](https://doc.rust-lang.org/std/fmt/trait.UpperHex.html)
- [`std::fmt::LowerHex`](https://doc.rust-lang.org/std/fmt/trait.LowerHex.html)
- [`std::fmt::Octal`](https://doc.rust-lang.org/std/fmt/trait.Octal.html)
- [`std::fmt::Binary`](https://doc.rust-lang.org/std/fmt/trait.Binary.html)

これらのトレイトはフォーマット指定子`{:X}`、`{:x}`、`{:o}`、および`{:b}`を使用した際の
表現を制御します。

`|`や`&`などビット単位での操作が行われる数値型にはこれらのトレイトを実装してください。
これは特にビットフラグ型において妥当です。
`struct Nanoseconds(u64)`のような、量を表す数値型においては必要ないでしょう。

<a id="c-rw-value"></a>
## 読み書きを行うジェネリックな関数が`R: Read`と`W: Write`を値渡しで受け取っている (C-RW-VALUE)

標準ライブラリは次の2つの実装を含んでいます。

```rust
impl<'a, R: Read + ?Sized> Read for &'a mut R { /* ... */ }

impl<'a, W: Write + ?Sized> Write for &'a mut W { /* ... */ }
```

従って、`R: Read`あるいは`W: Write`という境界を持つジェネリックなパラメータを値で受け取る関数は、
必要ならばミュータブルな参照を受け取ることも可能になっています。

そういった関数のドキュメントには、ミュータブルな参照を渡すこともできると簡単に書いてあるべきです。
新規のRustユーザが頻繁にこの点に引っ掛かるためです。
例えば、複数回に渡ってあるファイルからデータを読み出したいにも関わらず、
関数がreaderを値で受け取るという場合、対処方法の分からないユーザがいます。
この場合、上記の実装を利用して`f`の代わりに`&mut f`を代わりに渡せるというのが答えです。

### 例

- [`flate2::read::GzDecoder::new`]
- [`flate2::write::GzEncoder::new`]
- [`serde_json::from_reader`]
- [`serde_json::to_writer`]

[`flate2::read::GzDecoder::new`]: https://docs.rs/flate2/0.2/flate2/read/struct.GzDecoder.html#method.new
[`flate2::write::GzEncoder::new`]: https://docs.rs/flate2/0.2/flate2/write/struct.GzEncoder.html#method.new
[`serde_json::from_reader`]: https://docs.serde.rs/serde_json/fn.from_reader.html
[`serde_json::to_writer`]: https://docs.serde.rs/serde_json/fn.to_writer.html
