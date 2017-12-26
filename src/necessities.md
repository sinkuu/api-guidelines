# 必要事項


<a id="c-stable"></a>
## stableなクレートのパブリックな依存クレートがstableである (C-STABLE)

全てのパブリックな依存クレートがstable(>=1.0.0)とならない限り、クレートをstableにすることはできません。

パブリックな依存性は、依存先に由来する型がそのクレートのパブリックなAPIに使われているものです。

```rust
pub fn do_my_thing(arg: other_crate::TheirThing) { /* ... */ }
```

この関数を含むクレートは、`other_crate`がstableにならない限りstableになりません。

パブリックな依存制は思わぬ所に潜んでいることがあるため、注意が必要です。

```rust
pub struct Error {
    private: ErrorImpl,
}

enum ErrorImpl {
    Io(io::Error),
    // ErrorImplはプライベートなので、other_crateがstableでなくても問題ないはず。
    Dep(other_crate::Error),
}

// いや、ここでother_crateをパブリックなAPIの一部にしてしまっている。
impl From<other_crate::Error> for Error {
    fn from(err: other_crate::Error) -> Self {
        Error { private: ErrorImpl::Dep(err) }
    }
}
```


<a id="c-permissive"></a>
## クレートとその依存先がpermissiveなライセンスの下にある (C-PERMISSIVE)

Rustプロジェクトによって公開されているソフトウェアは[MIT]と[Apache 2.0]
のデュアルライセンス下にあります。
Rustエコシステムとの最大の協調性が必要なクレートは同じようにするべきです。
その他の選択肢も以下に提示します。

Rustのライセンスに関しては[Rust FAQ]で幾分かの言及がなされていますが、
このAPIガイドラインでは詳しい説明は行いません。
これはライセンスに関する解説ではなく、Rustとの協調性に関するガイドラインだからです。

[MIT]: https://github.com/rust-lang/rust/blob/master/LICENSE-MIT
[Apache 2.0]: https://github.com/rust-lang/rust/blob/master/LICENSE-APACHE
[Rust FAQ]: https://www.rust-lang.org/en-US/faq.html#why-a-dual-mit-asl2-license

Rustと同等のライセンスをあなたのプロジェクトに適用したい場合、次のように
`Cargo.toml`中の`license`フィールドを設定してください。

```toml
[package]
name = "..."
version = "..."
authors = ["..."]
license = "MIT/Apache-2.0"
```

そして、README.mdの最後に以下を追加してください。

```
## License

Licensed under either of

 * Apache License, Version 2.0
   ([LICENSE-APACHE](LICENSE-APACHE) or http://www.apache.org/licenses/LICENSE-2.0)
 * MIT license
   ([LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT)

at your option.

## Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in the work by you, as defined in the Apache-2.0 license, shall be
dual licensed as above, without any additional terms or conditions.
```

MIT/Apache-2.0のデュアルライセンスの他に、Rustのクレート開発者によってよく使われるのは
MITやBSDのようなpermissiveなライセンスを単体で利用することです。
このやり方でも、小さな制約が加わってしまうだけでRustとの互換性はあります。

Rustとの完全なライセンス互換性が必要なら、Apacheライセンス単体を用いることは推奨されません。
Apacheライセンスはpermissiveではありますが、MITやBSDより大きな制約を課しているため、
デュアルライセンスのRust本体は使うことができるにも関わらず
Apacheライセンスのみのソフトウェアを使うことのできない状況が存在するからです。

クレートの依存先のライセンスはクレート自身の配布にも制約を与える可能性があるため、
permissiveなライセンスのクレートはpermissiveなライセンスのクレートにのみ依存するべきです。
