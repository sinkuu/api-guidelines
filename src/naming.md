# 命名


<a id="c-case"></a>
## 大文字・小文字の使い分けがRFC430に従っている (C-CASE)

Rustにおける基本的な命名規則は[RFC 430]に記述されています。

Rustは「型レベル」のもの(型やトレイト)に`CamelCase`、
「値レベル」のものに`snake_case`を使用する傾向があります。
正確には、次表のように命名します。

| アイテム | 規則 |
| ---- | ---------- |
| クレート | [不明](https://github.com/rust-lang-nursery/api-guidelines/issues/29) |
| モジュール | `snake_case` |
| 型 | `CamelCase` |
| トレイト | `CamelCase` |
| Enumのバリアント | `CamelCase` |
| 関数 | `snake_case` |
| メソッド | `snake_case` |
| 一般のコンストラクタ | `new` or `with_more_details` |
| 変換を行うコンストラクタ | `from_some_other_type` |
| マクロ | `snake_case!` |
| ローカル変数 | `snake_case` |
| スタティック変数 | `SCREAMING_SNAKE_CASE` |
| 定数 | `SCREAMING_SNAKE_CASE` |
| 型パラメータ | concise `CamelCase`, usually single uppercase letter: `T` |
| ライフタイム | short `lowercase`, usually a single letter: `'a`, `'de`, `'src` |
| Feature | [不明](https://github.com/rust-lang-nursery/api-guidelines/issues/101)、ただし[C-FEATURE]を参照 |

頭字語や複合語は、`CamelCase`では一語に数えます。例えば`UUID`ではなく`Uuid`を使い、`USize`ではなく`Usize`、`StdIn`ではなく`Stdin`を使って下さい。
`snake_case`では`is_xid_start`のように小文字にします。

`snake_case`または`SCREAMING_SNAKE_CASE`では、
それが最後の文字を除いて一文字で区切ってはいけません。
例えば、`b_tree_map`ではなく`btree_map`としますが、`PI2`ではなく`PI_2`とします。

クレート名の前後に`-rs`や`-rust`を付けるべきではありません。
全てのクレートがRust製であることは分かりきっています!
そのことをユーザに常に知らせ続ける必要はありません。
(訳注: レポジトリ名等ではなく、Cargo.tomlで指定するクレート名についての指針です)

[RFC 430]: https://github.com/rust-lang/rfcs/blob/master/text/0430-finalizing-naming-conventions.md
[C-FEATURE]: #c-feature

### 標準ライブラリでの例

この規則は標準ライブラリの全体に渡って使用されています。


<a id="c-conv"></a>
## 変換メソッドに`as_`, `to_`, `into_`を使っている (C-CONV)

変換メソッドの名前には次のプレフィクスを付けるべきです。

| プレフィクス | コスト | 所有権 |
| ------ | ---- | --------- |
| `as_` | 低い | 借用 -\> 借用 |
| `to_` | 高い | 借用 -\> 借用<br>借用 -\> 所有 (Copyでない型)<br>所有 -\> 所有 (Copy型) |
| `into_` | 可変 | 所有 -\> 所有 (Copyでない型) |

以下に例を挙げます。

- [`str::as_bytes()`]はコストなしに、`str`をUTF-8バイト列として見たビューを返します。
  入力は借用された`&str`で出力は借用された`&[u8]`です。
- [`Path::to_str`]はOSの与えるパスのバイト列に対し、コストの高いUTF-8チェックを行います。 
  入出力はともに借用されています。小さくない実行時コストがあるため、これを`as_str`と呼ぶことはできません。
- [`str::to_lowercase()`]はUnicode規格に沿って`str`を小文字に変換したものを返します。
  この処理は文字列のデコードを含み、またメモリの確保も行うでしょう。
  入力は借用された`&str`で出力は所有された`String`です。
- [`f64::to_radians()`]は浮動小数点数で表された角度を度からラジアンに変換します。
  入力は`f64`です。これが`&f64`でないのは、コピーに殆どコストが掛からないためです。
  入力が消費されないので、このメソッドを`into_radians`と呼ぶのはミスリーディングです。
- [`String::into_bytes()`]は`String`がラップしている`Vec<u8>`を取り出します。
  この処理にコストは掛かりません。このメソッドは`String`の所有権を得て、所有された`Vec<u8>`を返します。
- [`BufReader::into_inner()`]はバッファリングされたreaderの所有権を得て、ラップされていたreaderを取り出します。
  この処理にコストは掛かりません。バッファ内のデータは破棄されます。
- [`BufWriter::into_inner()`]はバッファリングされたwriterの所有権を得て、ラップされていたwriterを取り出します。
  この処理にはコストの掛かるバッファのフラッシングが必要なことがあります。

[`str::as_bytes()`]: https://doc.rust-lang.org/std/primitive.str.html#method.as_bytes
[`Path::to_str`]: https://doc.rust-lang.org/std/path/struct.Path.html#method.to_str
[`str::to_lowercase()`]: https://doc.rust-lang.org/std/primitive.str.html#method.to_lowercase
[`f64::to_radians()`]: https://doc.rust-lang.org/std/primitive.f64.html#method.to_radians
[`String::into_bytes()`]: https://doc.rust-lang.org/std/string/struct.String.html#method.into_bytes
[`BufReader::into_inner()`]: https://doc.rust-lang.org/std/io/struct.BufReader.html#method.into_inner
[`BufWriter::into_inner()`]: https://doc.rust-lang.org/std/io/struct.BufWriter.html#method.into_inner

`as_`や`into_`の付くような変換メソッドは一般に抽象を弱めます。
内部表現を公開したり(`as`)、データを内部表現に分解したり(`into`)することになるからです。
一方で、`to_`と付くような変換メソッドは一般に抽象レベルを保てることが多いですが、
内部で何らかの処理を行いデータの表現方法を変換する必要があります。

ある値をラップして高レベルの意味を持たせるような型において、
ラップされた型にアクセスさせる手段は`into_inner()`メソッドによって提供されるべきです。
これは例えば、バッファリングを提供する[`BufReader`]や、
エンコード・デコードを行う[`GzDecoder`]、
アトミックなアクセスを提供する[`AtomicBool`]といった型のようなセマンティクスを持つ型に適用できます。

[`BufReader`]: https://doc.rust-lang.org/std/io/struct.BufReader.html#method.into_inner
[`GzDecoder`]: https://docs.rs/flate2/0.2.19/flate2/read/struct.GzDecoder.html#method.into_inner
[`AtomicBool`]: https://doc.rust-lang.org/std/sync/atomic/struct.AtomicBool.html#method.into_inner

変換メソッドの名前に`mut`を含めるときは、返り値の型の記述と同じ順番にしてください。
例えば[`Vec::as_mut_slice`]はミュータブルは名前の通りスライスを返します。
例えば`as_slice_mut`等よりもこのような命名が推奨されます。

[`Vec::as_mut_slice`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.as_mut_slice

```rust
// Return type is a mut slice.
fn as_mut_slice(&mut self) -> &mut [T];
```

##### 標準ライブラリでのさらなる例

- [`Result::as_ref`](https://doc.rust-lang.org/std/result/enum.Result.html#method.as_ref)
- [`RefCell::as_ptr`](https://doc.rust-lang.org/std/cell/struct.RefCell.html#method.as_ptr)
- [`slice::to_vec`](https://doc.rust-lang.org/std/primitive.slice.html#method.to_vec)
- [`Option::into_iter`](https://doc.rust-lang.org/std/option/enum.Option.html#method.into_iter)


<a id="c-getter"></a>
## Getterの名前がRustの規則に従っている (C-GETTER)

後述するような例外を除き、getterの名前を`get_`で始めるべきではありません。

```rust
pub struct S {
    first: First,
    second: Second,
}

impl S {
    // Not get_first.
    pub fn first(&self) -> &First {
        &self.first
    }

    // Not get_first_mut, get_mut_first, or mut_first.
    pub fn first_mut(&mut self) -> &mut First {
        &mut self.first
    }
}
```

`get`という命名は、getterによって得られるものが自明である場合にのみ使用されるべきです。
例えば、[`Cell::get`]は`Cell`の中身を返します。

[`Cell::get`]: https://doc.rust-lang.org/std/cell/struct.Cell.html#method.get

境界検査といった、実行時のバリデーションが必要なgetterには、
unsafeな`_unchecked`版も用意できないか検討してください。
そのようなメソッドの宣言はふつう、以下のようになります。

```rust
fn get(&self, index: K) -> Option<&V>;
fn get_mut(&mut self, index: K) -> Option<&mut V>;
unsafe fn get_unchecked(&self, index: K) -> &V;
unsafe fn get_unchecked_mut(&mut self, index: K) -> &mut V;
```

getterと変換([C-CONV](#c-conv))の間の違いは往々にして微かなもので、
常に境界がハッキリしている訳ではありません。
例えば[`TempDir::path`]は一時ディレクトリのファイルシステム上のパスのgetterとして捉えられますが、
[`TempDir::into_path`]は一時ディレクトリを削除する責任を呼び出し側に移す変換メソッドです。
このような場合`path`はgetterなので、`get_path`や`as_path`と呼ぶことは正しくありません。

[`TempDir::path`]: https://docs.rs/tempdir/0.3.5/tempdir/struct.TempDir.html#method.path
[`TempDir::into_path`]: https://docs.rs/tempdir/0.3.5/tempdir/struct.TempDir.html#method.into_path

### 標準ライブラリでのさらなる例

- [`std::io::Cursor::get_mut`](https://doc.rust-lang.org/std/io/struct.Cursor.html#method.get_mut)
- [`std::ptr::Unique::get_mut`](https://doc.rust-lang.org/std/ptr/struct.Unique.html#method.get_mut)
- [`std::sync::PoisonError::get_mut`](https://doc.rust-lang.org/std/sync/struct.PoisonError.html#method.get_mut)
- [`std::sync::atomic::AtomicBool::get_mut`](https://doc.rust-lang.org/std/sync/atomic/struct.AtomicBool.html#method.get_mut)
- [`std::collections::hash_map::OccupiedEntry::get_mut`](https://doc.rust-lang.org/std/collections/hash_map/struct.OccupiedEntry.html#method.get_mut)
- [`<[T]>::get_unchecked`](https://doc.rust-lang.org/std/primitive.slice.html#method.get_unchecked)


<a id="c-iter"></a>
## イテレータを生成するメソッドの名前が`iter`, `iter_mut`, `into_iter`となっている ([C-ITER])

[RFC 199]に従ってください。

`U`という型の要素を持つコンテナの場合、イテレータを生成するメソッドは次のように命名するべきです。

```rust
fn iter(&self) -> Iter             // Iter implements Iterator<Item = &U>
fn iter_mut(&mut self) -> IterMut  // IterMut implements Iterator<Item = &mut U>
fn into_iter(self) -> IntoIter     // IntoIter implements Iterator<Item = U>
```

この指針は全ての要素が同質であると意味付けられたコレクションに対して適用されます。
例えば`str`はバイト列ですが、有効なUTF-8であるという保証があるため、この指針は適用できません。
従って`iter`/`iter_mut`/`into_iter`といったメソッド群ではなく、
バイト列としてイテレートする[`str::bytes`]、キャラクタ列としてイテレートする[`str::chars`]を持ちます。

[`str::bytes`]: https://doc.rust-lang.org/std/primitive.str.html#method.bytes
[`str::chars`]: https://doc.rust-lang.org/std/primitive.str.html#method.chars

このガイドラインはメソッドにのみ適用され、関数は対象外です。
例えば`url`クレートの[`percent_encode`]は文字列をパーセントエンコーディングしていくイテレータを返します。
この場合、`iter`/`iter_mut`/`into_iter`といった命名を使うことに利点はありません。

[`percent_encode`]: https://docs.rs/url/1.4.0/url/percent_encoding/fn.percent_encode.html
[RFC 199]: https://github.com/rust-lang/rfcs/blob/master/text/0199-ownership-variants.md

### 標準ライブラリでの例

- [`Vec::iter`](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.iter)
- [`Vec::iter_mut`](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.iter_mut)
- [`Vec::into_iter`](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.into_iter)
- [`BTreeMap::iter`](https://doc.rust-lang.org/std/collections/struct.BTreeMap.html#method.iter)
- [`BTreeMap::iter_mut`](https://doc.rust-lang.org/std/collections/struct.BTreeMap.html#method.iter_mut)


<a id="c-iter-ty"></a>
## イテレータの型名が、それを生成するメソッドと揃っている (C-ITER-TY)

`into_iter`というメソッドは`IntoIter`という型を返すべきです。
他のイテレータを返すメソッドでも同様です。

この指針は主にメソッドに対して適用されますが、関数に対しても大抵は適用できます。
例えば`url`クレートの[`percent_encode`]関数は[`PercentEncode`][PercentEncode-type]
という型名のイテレータを返します。

[PercentEncode-type]: https://docs.rs/url/1.4.0/url/percent_encoding/struct.PercentEncode.html

このような命名法は[`vec::IntoIter`]のようにモジュール名を付けて呼ぶ際に最も有用です。

[`vec::IntoIter`]: https://doc.rust-lang.org/std/vec/struct.IntoIter.html

### 標準ライブラリでの例

* [`Vec::iter`] は [`Iter`][slice::Iter] を返す。
* [`Vec::iter_mut`] は [`IterMut`][slice::IterMut] を返す。
* [`Vec::into_iter`] は [`IntoIter`][vec::IntoIter] を返す。
* [`BTreeMap::keys`] は [`Keys`][btree_map::Keys] を返す。
* [`BTreeMap::values`] は [`Values`][btree_map::Values] を返す。

[`Vec::iter`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.iter
[slice::Iter]: https://doc.rust-lang.org/std/slice/struct.Iter.html
[`Vec::iter_mut`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.iter_mut
[slice::IterMut]: https://doc.rust-lang.org/std/slice/struct.IterMut.html
[`Vec::into_iter`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.into_iter
[vec::IntoIter]: https://doc.rust-lang.org/std/vec/struct.IntoIter.html
[`BTreeMap::keys`]: https://doc.rust-lang.org/std/collections/struct.BTreeMap.html#method.keys
[btree_map::Keys]: https://doc.rust-lang.org/std/collections/btree_map/struct.Keys.html
[`BTreeMap::values`]: https://doc.rust-lang.org/std/collections/struct.BTreeMap.html#method.values
[btree_map::Values]: https://doc.rust-lang.org/std/collections/btree_map/struct.Values.html


<a id="c-feature"></a>
## Featureの名前に余計な単語が入っていない (C-FEATURE)

[Cargoのfeature]の名前に意味のない単語を付けないでください。
`use-abc`や`with-abc`などとせず、単に`abc`とするべきです。

[Cargoのfeature]: http://doc.crates.io/manifest.html#the-features-section

標準ライブラリへの依存がオプションである場合が最もよく目にする例でしょう。
これを指針に従って行うと、以下のようになります。

```toml
# In Cargo.toml

[features]
default = ["std"]
std = []
```

```rust
// In lib.rs

#![cfg_attr(not(feature = "std"), no_std)]
```

このfeatureに`std`以外の、`use-std`や`with-std`あるいはその他の独創的な名前を付けないでください。
そうすることで、Cargoが暗黙的に追加するオプショナルな依存性のfeatureと沿った形になります。
例えば`x`というクレートがSerdeと標準ライブラリに対する依存をオプションとして持つとき、


```toml
[package]
name = "x"
version = "0.1.0"

[features]
std = ["serde/std"]

[dependencies]
serde = { version = "1.0", optional = true }
```

のようになります。そして、さらに`x`に依存するとき、Serdeへの依存を
`features = ["serde"]`で有効化できます。
また、同じように標準ライブラリへの依存を`features = ["std"]`で有効化できます。
Cargoによって暗黙的に追加されるfeatureは`serde`であり、`use-serde`でも`with-serde`でもありません。
ですから、明示的なfeatureに対しても同じようにするべきなのです。

関連事項として、Cargoのfeatureは追加式ですから、
`no-abc`といったfeatureは大きな間違いです。


<a id="c-word-order"></a>
## 命名時に単語を並べる順番が揃っている (C-WORD-ORDER)

標準ライブラリにおけるエラー型をいくつか示します。

- [`JoinPathsError`](https://doc.rust-lang.org/std/env/struct.JoinPathsError.html)
- [`ParseBoolError`](https://doc.rust-lang.org/std/str/struct.ParseBoolError.html)
- [`ParseCharError`](https://doc.rust-lang.org/std/char/struct.ParseCharError.html)
- [`ParseFloatError`](https://doc.rust-lang.org/std/num/struct.ParseFloatError.html)
- [`ParseIntError`](https://doc.rust-lang.org/std/num/struct.ParseIntError.html)
- [`RecvTimeoutError`](https://doc.rust-lang.org/std/sync/mpsc/enum.RecvTimeoutError.html)
- [`StripPrefixError`](https://doc.rust-lang.org/std/path/struct.StripPrefixError.html)

全て、動詞-オブジェクト-エラーの順番で並んでいます。
もし新たにアドレスのパースに失敗したことを表すエラーを追加するならば、
`AddrParseError`等ではなく、
一貫性を考えて動詞-オブジェクト-エラーの順に並べ`ParseAddrError`とすべきです。

どの順番を選ぶかは大して重要ではありませんが、クレート内での一貫性、
標準ライブラリの似た機能との整合性には注意してください。
