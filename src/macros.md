# マクロ


<a id="c-evocative"></a>
## 入力の構文から結果をイメージできるようになっている (C-EVOCATIVE)

Rustのマクロの入力ではお好きな構文を実装することができますが、
特異なシンタックスを導入するのではなく、
Rustの構文に似せることでマクロ以外のコードとの統一性を保てるようにしてください。
キーワードや記号類の選択・配置には注意してください。

良い指針は、マクロの出力に似せた構文(特にキーワードと記号類)を使用することです。

例えば、マクロが与えられた名前の構造体型を宣言するならば、
マクロの入力において名前の前にキーワード`struct`を付けさせるようにし、
コードを読む人に対して構造体型が宣言されるということを知らせてください。

```rust
// このようにしてください...
bitflags! {
    struct S: u32 { /* ... */ }
}

// ...キーワードがなかったり...
bitflags! {
    S: u32 { /* ... */ }
}

// ...その場その場で勝手な単語を導入したりしないでください
bitflags! {
    flags S: u32 { /* ... */ }
}
```

もう一つの例はセミコロンとコンマの問題です。
Rustにおける定数の宣言では、最後にセミコロンを付けます。
複数の定数を宣言するマクロの場合、Rustの構文と違っているとしても同様にセミコロンを付けるようにすべきでしょう。

```rust
// 普通の定数の宣言にはセミコロンが使われます
const A: u32 = 0b000001;
const B: u32 = 0b000010;

// なので、こうすべきです...
bitflags! {
    struct S: u32 {
        const C = 0b000100;
        const D = 0b001000;
    }
}

// ...こうではなく
bitflags! {
    struct S: u32 {
        const E = 0b010000,
        const F = 0b100000,
    }
}
```

マクロには幅広い用例があるため、こういった特定の例はそのまま通用しないでしょう。
ですが、同じような考え方を適用できないか考えてみて下さい。


<a id="c-macro-attr"></a>
## アイテムを宣言するマクロが属性と衝突しない (C-MACRO-ATTR)

アイテムを宣言するマクロは、それぞれに属性を付与できるようになっているべきです。
よくある例は特定のアイテムをcfgで条件コンパイルするような場合です。

```rust
bitflags! {
    struct Flags: u8 {
        #[cfg(windows)]
        const ControlCenter = 0b001;
        #[cfg(unix)]
        const Terminal = 0b010;
    }
}
```

構造体型や列挙型を生成するマクロも、deriveを使えるように属性をサポートすべきです。

```rust
bitflags! {
    #[derive(Default, Serialize)]
    struct Flags: u8 {
        const ControlCenter = 0b001;
        const Terminal = 0b010;
    }
}
```


<a id="c-anywhere"></a>
## アイテムを宣言するマクロがアイテムを宣言できる場所のどこでも使える (C-ANYWHERE)

Rustではアイテムをモジュールレベルから関数のような狭いスコープまで配置することができます。
アイテムを宣言するマクロも同様に、いずれの場所でも動くようになっているべきです。
そして、少なくともモジュールレベルおよび関数レベルでマクロを呼び出すテストを追加すべきです。

```rust
#[cfg(test)]
mod tests {
    test_your_macro_in_a!(module);

    #[test]
    fn anywhere() {
        test_your_macro_in_a!(function);
    }
}
```

モジュールスコープでは動くものの、関数スコープでは動かないマクロの簡単な例を挙げます。

```rust
macro_rules! broken {
    ($m:ident :: $t:ident) => {
        pub struct $t;
        pub mod $m {
            pub use super::$t;
        }
    }
}

broken!(m::T); // 問題なくTおよびm::Tに展開される

fn g() {
    broken!(m::U); // コンパイルに失敗する、super::Uがgではなく外側のモジュールの方を指すため
}
```


<a id="c-macro-vis"></a>
## アイテムを宣言するマクロが可視性の指定をサポートしている (C-MACRO-VIS)

マクロによって宣言されるアイテムの可視性は、Rustの可視性の構文に沿ってください。
デフォルトではプライベートで、`pub`が指定されたらパブリックにします。

```rust
bitflags! {
    struct PrivateFlags: u8 {
        const A = 0b0001;
        const B = 0b0010;
    }
}

bitflags! {
    pub struct PublicFlags: u8 {
        const C = 0b0100;
        const D = 0b1000;
    }
}
```


<a id="c-macro-ty"></a>
## 型の指定が柔軟である (C-MACRO-TY)

マクロが`$t:ty`のようにして型を受け取る場合、
以下の全てに対応できるべきです。

- プリミティブ型: `u8`, `&str`
- 相対パス: `m::Data`
- 絶対パス: `::base::Data`
- 上位を参照する相対パス: `super::Data`
- ジェネリクス: `Vec<String>`

これができないマクロの簡単な例を挙げます。
次のマクロはプリミティブ型や絶対パスではうまく動きますが、相対パスでは動きません。

```rust
macro_rules! broken {
    ($m:ident => $t:ty) => {
        pub mod $m {
            pub struct Wrapper($t);
        }
    }
}

broken!(a => u8); // okay

broken!(b => ::std::marker::PhantomData<()>); // okay

struct S;
broken!(c => S); // fails to compile
```
