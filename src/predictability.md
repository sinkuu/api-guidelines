# 予測性


<a id="c-smart-ptr"></a>
## スマートポインタがinherentメソッドを持っていない (C-SMART-PTR)

例えば、[`Box::into_raw`]は次のように定義されています。

[`Box::into_raw`]: https://doc.rust-lang.org/std/boxed/struct.Box.html#method.into_raw

```rust
impl<T> Box<T> where T: ?Sized {
    fn into_raw(b: Box<T>) -> *mut T { /* ... */ }
}

let boxed_str: Box<str> = /* ... */;
let ptr = Box::into_raw(boxed_str);
```

もしこれがinherentメソッドであったら、呼び出そうとしているメソッドが`T`のものなのか
`Box<T>`のものなのか区別が付かなくなります。

```rust
impl<T> Box<T> where T: ?Sized {
    // Do not do this.
    fn into_raw(self) -> *mut T { /* ... */ }
}

let boxed_str: Box<str> = /* ... */;

// スマートポインタのDerefを経由してstrのメソッドにアクセスしている
boxed_str.chars()

// これは`Box<str>`のメソッド……?
boxed_str.into_raw()
```


<a id="c-conv-specific"></a>
## 変換メソッドは最も関係の深い型に付ける (C-CONV-SPECIFIC)

迷ったら`_from`よりも`to_`/`as_`/`into_`を選んでください。
後者の方がより使いやすく、また他のメソッドにチェーンすることもできるからです。

2つの型の間の変換において、多くの場合どちらか一方が明らかに特徴的です。
すなわち、他方にはない不変条件や解釈が追加されています。
例えば[`str`]はUTF-8でエンコードされたバイト列ですから、単なるバイト列である`&[u8]`より特徴的です。

[`str`]: https://doc.rust-lang.org/std/primitive.str.html

変換メソッドは、関係する型の中で、より特徴的なものが持つべきです。
従って、`str`は[`as_bytes`]メソッド及び[`from_utf8`]コンストラクタを持つのです。
この方が直感的であるだけでなく、`&[u8]`のような型が無数の変換メソッドで汚染されていくという事態が避けられます。

[`as_bytes`]: https://doc.rust-lang.org/std/primitive.str.html#method.as_bytes
[`from_utf8`]: https://doc.rust-lang.org/std/str/fn.from_utf8.html


<a id="c-method"></a>
## 明確なレシーバを持つ関数はメソッドにする (C-METHOD)

特定の型と強く関連した操作についてはメソッドにしてください。

```rust
impl Foo {
    pub fn frob(&self, w: widget) { /* ... */ }
}
```

関数にしてはいけません。

```rust
pub fn frob(foo: &Foo, w: widget) { /* ... */ }
```


関数でなくメソッドを選ぶことには多数の利点があります。

* インポートしたり関数へのパスを記述したりする必要がない。その型の値さえあれば必要な操作ができます。
* 呼び出し時に自動借用が働きます。 (可変借用も含めて)
* 「この型`T`で何ができるんだろう」という疑問への答えが簡単になります。 (特にrustdocを使用している場合)
* `self`記法が使われるため、より簡潔かつ明白に所有権の区別が示されます。


<a id="c-no-out"></a>
## 関数がoutパラメータを持たない (C-NO-OUT)

例えば複数の`Bar`を返すときはこのようにしてください。

```rust
fn foo() -> (Bar, Bar)
```

このようにoutパラメータのようなものを取ってはいけません。

```rust
fn foo(output: &mut Bar) -> Bar
```

タプルや構造体を使って複数の値を返しても効率のよいコードにコンパイルされますし、
ヒープの確保も行われません。複数の値を返す必要があるならこれらの型を利用すべきです。

例外は関数が呼び出し側の所有するデータを変更する場合です。
例えば、バッファの再利用をする場合は次のようになるでしょう。

```rust
fn read(&mut self, buf: &mut [u8]) -> io::Result<usize>
```


<a id="c-overload"></a>
## 奇妙な演算子オーバーロードを行っていない (C-OVERLOAD)

組み込みの演算子(`*`や`|`など)は[`std::ops`]にあるトレイトを実装することで使えるようになります。
これらの演算子には元から意味が付与されています。
例えば、`Mul`は乗算のような(そして結合性などの特性を共有した)演算にのみ実装されるべきです。

[`std::ops`]: https://doc.rust-lang.org/std/ops/index.html#traits


<a id="c-deref"></a>
## `Deref`と`DerefMut`を実装しているのはスマートポインタだけである (C-DEREF)

`Deref`トレイトはコンパイラによって様々な状況で暗黙的に使われ、メソッドの解決と関わります。
その周辺の規則はスマートポインタを念頭において設計されているため、
これらのトレイトはスマートポインタに対してのみ実装されるべきです。

### 標準ライブラリでの例

- [`Box<T>`](https://doc.rust-lang.org/std/boxed/struct.Box.html)
- [`String`](https://doc.rust-lang.org/std/string/struct.String.html)は[`str`](https://doc.rust-lang.org/std/primitive.str.html)を指すスマートポインタ
- [`Rc<T>`](https://doc.rust-lang.org/std/rc/struct.Rc.html)
- [`Arc<T>`](https://doc.rust-lang.org/std/sync/struct.Arc.html)
- [`Cow<'a, T>`](https://doc.rust-lang.org/std/borrow/enum.Cow.html)


<a id="c-ctor"></a>
## コンストラクタはスタティックなinherentメソッドである (C-CTOR)

Rustにおいて、「コンストラクタ」は単なる慣習に過ぎません。
コンストラクタの命名には様々な慣習があり、その区別が分かり辛いことが多々あります。

最も基本的なコンストラクタの形は引数のない`new`メソッドです。

```rust
impl<T> Example<T> {
    pub fn new() -> Example<T> { /* ... */ }
}
```

コンストラクタは、その生成する型のスタティック(`self`を取らない)なinherentメソッドです。
型をインポートする慣習と併せれば、分かりやすく簡潔にその型を生成することができます。

```rust
use example::Example;

// Construct a new Example.
let ex = Example::new();
```

`new`という名前は、1つ目の最も重要なコンストラクタに使われるべきです。
上記の例のように引数を取らないこともあれば、`Box`に入れる値を取る[`Box::new`]のように、
引数を取ることもあります。

主にI/Oリソースを表す型では、[`File::open`]、[`Mmap::open`]、
[`TcpStream::connect`]、あるいは [`UpdSocket::bind`]のように
コンストラクタの命名が異なっていることがあります。
これらは、各々の領域で適した名前が選ばれています。

ある型の値を生成する方法が複数存在することは多くあります。
そういった場合、2個目以降のコンストラクタには`_with_foo`などと名前の最後に付けることが一般的です。
例えば、[`Mmap::open_with_offset`]などです。
複数のオプションがあるならばビルダーパターン([C-BUILDER])の使用も考えてください。

別の型の値を取って変換を行うコンストラクタというものもあります。
それらは[`std::io::Error::from_raw_os_error`]のように、一般に`from_`から始まる名前を持ちます。
ここで、よく似たものに`From`トレイト([C-CONV-TRAITS])が存在します。
`from_`の付いた変換コンストラクタと`From<T>`実装の間には3つの相違点があります。

- `from_`コンストラクタはunsafeにすることができますが、`From`の実装ではできません。
  例: [`Box::from_raw`]。
- `from_`コンストラクタは、[`u64::from_str_radix`]のように
  元になるデータを区別するための追加の引数を取ることができます。
- `From`実装は元のデータから出力の型のエンコード方法を決定できる場合にのみ適しています。
  [`u64::from_be`]や[`String::from_utf8`]のように入力の型が単なるデータ列であるとき、
  コンストラクタの名前によってその意味を伝えることが可能です。

[`Box::from_raw`]: https://doc.rust-lang.org/std/boxed/struct.Box.html#method.from_raw
[`u64::from_str_radix`]: https://doc.rust-lang.org/std/primitive.u64.html#method.from_str_radix
[`u64::from_be`]: https://doc.rust-lang.org/std/primitive.u64.html#method.from_be
[`String::from_utf8`]: https://doc.rust-lang.org/std/string/struct.String.html#method.from_utf8

[C-BUILDER]: type-safety.html#c-builder
[C-CONV-TRAITS]: interoperability.html#c-conv-traits

### 標準ライブラリでの例

- [`std::io::Error::new`]はIOエラーの生成に使われるコンストラクタ
- [`std::io::Error::from_raw_os_error`]は
  OSから与えられたエラーコードを変換するコンストラクタ
- [`Box::new`]は引数を1つとり、コンテナ型を生成するコンストラクタ
- [`File::open`]はファイルをオープンする
- [`Mmap::open_with_offset`]は指定されたオプションでメモリーマップをオープンする

[`File::open`]: https://doc.rust-lang.org/stable/std/fs/struct.File.html#method.open
[`Mmap::open`]: https://docs.rs/memmap/0.5.2/memmap/struct.Mmap.html#method.open
[`Mmap::open_with_offset`]: https://docs.rs/memmap/0.5.2/memmap/struct.Mmap.html#method.open_with_offset
[`TcpStream::connect`]: https://doc.rust-lang.org/stable/std/net/struct.TcpStream.html#method.connect
[`UpdSocket::bind`]: https://doc.rust-lang.org/stable/std/net/struct.UdpSocket.html#method.bind
[`std::io::Error::new`]: https://doc.rust-lang.org/std/io/struct.Error.html#method.new
[`std::io::Error::from_raw_os_error`]: https://doc.rust-lang.org/std/io/struct.Error.html#method.from_raw_os_error
[`Box::new`]: https://doc.rust-lang.org/stable/std/boxed/struct.Box.html#method.new

