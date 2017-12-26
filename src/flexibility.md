# 柔軟性


<a id="c-intermediate"></a>
## 重複した処理を行わなくて済むように中間生成物を公開している (C-INTERMEDIATE)

何らかの処理を行って結果を返す関数の多くは、関連したデータを途中で生成しています。
もしユーザにとって有益なものがあれば、それを公開するAPIの追加を検討してください。

### 標準ライブラリでの例

- [`Vec::binary_search`]は目的の値が見つかったか否かを`bool`で表したり、
  目的の値が見つかった位置を`Option<usize>`で返すようにはなっていません。
  代わりに、もし見つかればその位置を、そして見つからなければその値が挿入されるべき位置を返します。

- [`String::from_utf8`]は入力が正しいUTF-8でなければ失敗します。
  そのとき、入力のどこまでが正しいUTF-8であったかを返し、また入力されたバイト列の所有権も返します。

- [`HashMap::insert`]はそのキーの場所に元から値が存在していたら、その値を`Option<T>`で返します。
  ユーザがこの値を必要としているとき、この挙動がなければハッシュテーブルへのルックアップを二度繰り返さなければなりません。

[`Vec::binary_search`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.binary_search
[`String::from_utf8`]: https://doc.rust-lang.org/std/string/struct.String.html#method.from_utf8
[`HashMap::insert`]: https://doc.rust-lang.org/stable/std/collections/struct.HashMap.html#method.insert


<a id="c-caller-control"></a>
## 呼び出し側がデータをコピーするタイミングを決める (C-CALLER-CONTROL)

引数の所有権を要する関数は、借用して複製するのではなく所有権を受け取るべきです。

```rust
// 良い例:
fn foo(b: Bar) {
    /* use b as owned, directly */
}

// 悪い例:
fn foo(b: &Bar) {
    let b = b.clone();
    /* use b as owned after cloning */
}
```

もし関数が引数の所有権を必要としないのなら、
所有権を取って最終的にdropする代わりに可変あるいは非可変借用を受け取るべきです。

```rust
// 良い例:
fn foo(b: &Bar) {
    /* use b as borrowed */
}

// 悪い例:
fn foo(b: Bar) {
    /* use b as borrowed, it is implicitly dropped before function returns */
}
```

`Copy`トレイトは本当に必要な場合のみ使用してください。
単に低コストでコピーが可能であると伝えるために使用してはいけません。


<a id="c-generic"></a>
## ジェネリクスを用いて関数の引数に対する制限を最小にしている (C-GENERIC)

関数の引数に対する制約が少ないほど、その関数は幅広く使えるようになります。

単にイテレーションが必要なだけであれば、このようにジェネリクスを用いてください。

```rust
fn foo<I: IntoIterator<Item = i64>>(iter: I) { /* ... */ }
```

特定の型を指定しないでください。

```rust
fn foo(c: &[i64]) { /* ... */ }
fn foo(c: &Vec<i64>) { /* ... */ }
fn foo(c: &SomeOtherCollection<i64>) { /* ... */ }
```

もっと言えば、ジェネリクスを用い、関数の引数に対して必要な制約を正確に示してください。

### ジェネリクスの利点

* _再利用性_。ジェネリックな関数は受け取る型への明確な要件を示しつつ、
  多くの型に対して適用できます。

* _スタティックディスパッチと最適化_。ジェネリック関数を呼び出すと特定の型に特殊化("monomorphized")されます。
  トレイトメソッドの呼び出しはスタティックかつ実装を直接呼び出す形に変換され、
  コンパイラは呼び出しをインライン化し最適化することが可能です。

* _レイアウトのインライン化_。構造体型や列挙型がジェネリックな型`T`を持つとき、
  型`T`の値へは間接的なアクセスを挟むことなく、構造体型や列挙型の内部にインライン化されます。

* _推論_。ジェネリック関数の型パラメータは大抵推論が可能であるため、
  明示的な変換やその他のメソッド呼び出しといったコード中の冗長な部分を削減することが可能です。

* _正確な型_。ジェネリクスによって型に名前を与えられるため、
  正確にその型を受け取りあるいは生成する箇所を指定することができます。
  例えば、次の関数は全く同じ型`T`を受け取り、返すことが保証されます。
  `Trait`を実装する異なる型を用いて呼び出したりすることはできません。

  ```rust
  fn binary<T: Trait>(x: T, y: T) -> T
  ```


### ジェネリクスの欠点

* _コードサイズ_。ジェネリック関数の特殊化により、関数の中身は複製されます。
  コードサイズの増大がパフォーマンスと釣り合うかどうか検討するべきです。

* _一様な型_。これは型が正確であることの裏返しです。型パラメータ`T`は一つの実際の型をもちます。
  例えば`Vec<T>`は単一の具体型のコレクションです(そして内部的にはメモリ上に隣り合って並べられます)。
  一様でないコレクションが有用な場合もあります。[trait objects][C-OBJECT]を参照してください。

* _関数の定義の複雑化_。ジェネリクスを多用すると、関数の定義を読んだり理解することが難しくなります。

[C-OBJECT]: #c-object

### 標準ライブラリの例

- [`std::fs::File::open`]は`AsRef<Path>`というジェネリックな型を取ります。
  これにより、文字列リテラル`"f.txt"`や[`Path`]、あるいは[`OsString`]などを渡して
  ファイルを開くことができます。

[`std::fs::File::open`]: https://doc.rust-lang.org/std/fs/struct.File.html#method.open
[`Path`]: https://doc.rust-lang.org/std/path/struct.Path.html
[`OsString`]: https://doc.rust-lang.org/std/ffi/struct.OsString.html


<a id="c-object"></a>
## トレイトオブジェクトとして有用なトレイトがオブジェクトセーフになっている (C-OBJECT)

トレイトオブジェクトには大きな制約があります。それは、トレイトオブジェクト経由で呼ばれるメソッドは
ジェネリクスを使用できず、また`Self`をレシーバ以外の引数で使用できないことです。

トレイトを設計する際、そのトレイトがオブジェクトとして使用されるのか
ジェネリクスの境界として使用されるのかを決めておく必要があります。

そのトレイトがオブジェクトとして使用されることを念頭に置くならば、
トレイトメソッドではジェネリクスの代わりにトレイトオブジェクトを使用すべきです。

`Self: Sized`という`where`節を用いることで、特定のメソッドをトレイトオブジェクトから除外することができます。
次のトレイトはジェネリックなメソッドを持つためオブジェクトセーフではありません。

```rust
trait MyTrait {
    fn object_safe(&self, i: i32);

    fn not_object_safe<T>(&self, t: T);
}
```

ジェネリックなメソッドに`Self: Sized`を要求させることでトレイトオブジェクトから除外し、
そのトレイトをオブジェクトセーフにすることが可能です。

```rust
trait MyTrait {
    fn object_safe(&self, i: i32);

    fn not_object_safe<T>(&self, t: T) where Self: Sized;
}
```

### トレイトオブジェクトの利点

* _一様性_。これ無しでは解決できない問題もあります。
* _コードサイズ_。ジェネリクスと異なり、トレイトオブジェクトは特殊化(monomorphized)されたコードを生成しないため、
  コードサイズを大幅に削減することができます。

### トレイトオブジェクトの欠点

* _ジェネリックなメソッドが使えない_。トレイトオブジェクトは今のところジェネリックなメソッドを
  持つことができません。
* _動的ディスパッチとファットポインタ_。トレイトオブジェクトは、パフォーマンスに影響する可能性のある
  間接アクセスと仮想関数テーブルによるディスパッチを引き起こします。
* _Selfが使えない_。メソッドのレシーバ引数を除いて`Self`型を取ることはできません。

### 標準ライブラリでの例

- [`io::Read`]と[`io::Write`]は頻繁にオブジェクトとして使われます。
- [`Iterator`]には、トレイトオブジェクトとして使用できるようにするため、
  `Self: Sized`が指定されたジェネリックなメソッドがあります。

[`io::Read`]: https://doc.rust-lang.org/std/io/trait.Read.html
[`io::Write`]: https://doc.rust-lang.org/std/io/trait.Write.html
[`Iterator`]: https://doc.rust-lang.org/std/iter/trait.Iterator.html
