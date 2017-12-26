# ドキュメンテーション


<a id="c-crate-doc"></a>
## クレートレベルにコード例付きの詳細なドキュメントがある (C-CRATE-DOC)

[RFC 1687]を参照してください。

[RFC 1687]: https://github.com/rust-lang/rfcs/pull/1687


<a id="c-example"></a>
## 全てのアイテムにコード例が付いている (C-EXAMPLE)

全てのパブリックなモジュール、トレイト、構造体型、列挙型、関数、メソッド、マクロ、
およびtypeエイリアスに、その機能を示すコード例を含むrustdocドキュメントが存在するべきです。

この指針は無理のない範囲で適用してください。

適用可能な、その他のアイテムにおけるコード例へのリンクでも十分でしょう。
例えば、ある関数がある型を使うとして、
コード例が関数と型のどちらか一方にあれば、もう一方からはリンクするだけで十分です。

コード例の目的が*どのように*そのアイテムを使うかを示すことであるとは限りません。
読者は関数の呼び出し方、列挙型に対するmatchの使い方といった基本的な事柄は理解していると期待できます。
ですから、コード例の目的は*なぜ*アイテムを使うべきかの提示であることも多くあります。

```rust
// これはclone()を使うコード例の不味い例です。
// *どう*clone()を呼ぶのか機械的に示しているだけであり、
// *なぜ*これを使うべきかがまったく示されていません。
fn main() {
    let hello = "hello";

    hello.clone();
}
```


<a id="c-question-mark"></a>
## コード例が`try!`や`unwrap`ではなく`?`を使っている (C-QUESTION-MARK)

ユーザがコード例を丸写しすることは、その良し悪しは別としてよくあることです。
そして、エラーをunwrapするか否かという判断はユーザが自覚的に判断すべきことです。

エラー処理を含むコード例を構成する一般的な方法を以下に示します。
`#`で始まる行は`cargo test`によってコンパイルされますが、
ユーザに見えるrustdocには表示されません。

```
/// ```rust
/// # use std::error::Error;
/// #
/// # fn try_main() -> Result<(), Box<Error>> {
/// your;
/// example?;
/// code;
/// #
/// #     Ok(())
/// # }
/// #
/// # fn main() {
/// #     try_main().unwrap();
/// # }
/// ```
```


<a id="c-failure"></a>
## 関数のドキュメントにエラー、パニック、安全性に関する事項が含まれている (C-FAILURE)

エラーとなる条件を"Errors"セクションに記載するべきです。
これはトレイトメソッドに対しても同様です。エラーを返す可能性のあるトレイトメソッドには
"Errors"セクションを含んだドキュメントを付けるべきです。

標準ライブラリの例を挙げると、トレイトメソッド[`std::io::Read::read`]の実装のいくつかは
エラーを返す可能性があります。

[`std::io::Read::read`]: https://doc.rust-lang.org/std/io/trait.Read.html#tymethod.read

```
/// Pull some bytes from this source into the specified buffer, returning
/// how many bytes were read.
///
/// ... lots more info ...
///
/// # Errors
///
/// If this function encounters any form of I/O or other error, an error
/// variant will be returned. If an error is returned then it must be
/// guaranteed that no bytes were read.
```

また、パニックを起こす条件を"Panics"セクションに記載するべきです。
これはトレイトメソッドに対しても同様です。パニックを起こす可能性のあるトレイトメソッドには
"Panics"セクションを含んだドキュメントを付けるべきです。

標準ライブラリを例にすると、パニックを起こす可能性のあるメソッドとして[`Vec::insert`]が挙げられます。

[`Vec::insert`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.insert

```
/// Inserts an element at position `index` within the vector, shifting all
/// elements after it to the right.
///
/// # Panics
///
/// Panics if `index` is out of bounds.
```

あらゆるパニックの可能性を網羅する必要はありません。
特に、呼び出し側の提供したロジックの内部でパニックが起こる場合、
例えば以下のような事例で`Display`のパニックをドキュメントに記載するのは過剰です。
ですが、微妙なケースではできる限り多くの可能性を網羅する方が良いでしょう。

```rust
/// # Panics
///
/// This function panics if `T`'s implementation of `Display` panics.
pub fn print<T: Display>(t: T) {
    println!("{}", t.to_string());
}
```

unsafeな関数のドキュメントには、
その関数を正しく使うために呼び出し側が守らなければならない不変条件を記載した
"Safety"セクションを含めてください。

例えば、unsafeな関数である[`std::ptr::read`]は以下の事項を呼び出し側に要求しています。

[`std::ptr::read`]: https://doc.rust-lang.org/std/ptr/fn.read.html

```
/// Reads the value from `src` without moving it. This leaves the
/// memory in `src` unchanged.
///
/// # Safety
///
/// Beyond accepting a raw pointer, this is unsafe because it semantically
/// moves the value out of `src` without preventing further usage of `src`.
/// If `T` is not `Copy`, then care must be taken to ensure that the value at
/// `src` is not used before the data is overwritten again (e.g. with `write`,
/// `zero_memory`, or `copy_memory`). Note that `*src = foo` counts as a use
/// because it will attempt to drop the value previously at `*src`.
///
/// The pointer must be aligned; use `read_unaligned` if that is not the case.
```


<a id="c-link"></a>
## 文章に関係する項目へのリンクを含める (C-LINK)

同じ型の別のメソッドへのリンクはふつう次のようになります。

```md
[`serialize_struct`]: #method.serialize_struct
```

別の型へのリンクはこうなります。

```md
[`Deserialize`]: trait.Deserialize.html
```

親・子モジュールへのリンクは次のようにします。

```md
[`Value`]: ../enum.Value.html
[`DeserializeOwned`]: de/trait.DeserializeOwned.html
```

この指針はRFC 1574の["Link all the things"]によって公式に推奨されています。

["Link all the things"]: https://github.com/rust-lang/rfcs/blob/master/text/1574-more-api-documentation-conventions.md#link-all-the-things


<a id="c-metadata"></a>
## Cargo.tomlが一般的なメタデータを全て含んでいる (C-METADATA)

`Cargo.toml`の`[package]`セクションは以下の値を含むべきです。

- `authors`
- `description`
- `license`
- `repository`
- `readme`
- `keywords`
- `categories`

加えて、2つの任意項目があります。

- `documentation`
- `homepage`

デフォルトで、*crates.io*は[*docs.rs*]上のドキュメントへリンクします。
`documentation`メタデータは*docs.rs*以外の場所でドキュメントをホストしている場合にのみ必要です。
例えば、そのクレートが*docs.rs*のビルド環境に存在しない共有ライブラリを要求している場合などです。

[*docs.rs*]: https://docs.rs

`homepage`メタデータはソースレポジトリやAPIドキュメント以外の独自のウェブサイトがある場合のみ
使用されるべきです。`documentation`や`repository`と重複した値を入れないでください。
例えば、Serdeは`homepage`を専用のウェブサイトである*https://serde.rs*に設定しています。

[C-HTML-ROOT]: #c-html-root
<a id="c-html-root"></a>
### html_root_url属性が設定されている (C-HTML-ROOT)

<!--
Remove this guideline once rustdoc links no-deps documentation with no
html_root_url to docs.rs by default.
https://github.com/rust-lang/rust/issues/42301
-->

docs.rsを主なAPIドキュメントとして使っているならば`"https://docs.rs/CRATE/MAJOR.MINOR.PATCH"`に設定してください。

`html_root_url`属性は、rustdocが下流クレートをコンパイルする際にどのようにURLを作成するべきかを伝えるものです。
これが無ければ、あなたのクレートに依存するクレートにおいて、ドキュメント中のリンクが間違ったものになります。

```rust
#![doc(html_root_url = "https://docs.rs/log/0.3.8")]
```

バージョンがURLに含まれていることから分かる通り、
`Cargo.toml`内のバージョン番号と同期させる必要があります。
`html_root_url`がクレートのバージョンとずれた場合に失敗するインテグレーションテストを追加する
[`version-sync`]クレートが役に立つはずです。

[`version-sync`]: https://crates.io/crates/version-sync

この方式が気に入らなければ、`Cargo.toml`のバージョンの所にメモ書きを残しておくとよいでしょう。

```toml
version = "0.3.8" # html_root_urlの更新 忘れない
```

docs.rs以外にドキュメントをホストしているなら、`html_root_url`の値にクレート名 + index.htmlを付け足すと
あなたのクレートのルートモジュールにたどり着けるように設定してください。
例えば、ルートモジュールが`"https://api.rocket.rs/rocket/index.html"`にあるならば、
`html_root_url`は`"https://api.rocket.rs"`とすべきです。


<a id="c-relnotes"></a>
## 大きな変更が全てリリースノートに記載されている (C-RELNOTES)

クレートのユーザはリリースノートを読むことでバージョン間の変更点を知ることができます。
クレートレベルのドキュメントまたはCargo.tomlからリンクされたリポジトリにリリースノートへのリンクを置いてください。

破壊的変更([RFC 1105]で定義されている)は明確に区別されているべきです。

Gitを用いてソースコードの履歴を管理しているなら、
*crates.io*に発行された全てのリリースは対応するコミットにタグを付けてください。
Git以外のVCSでも同様の処理をしておくべきです。

```bash
# Tag the current commit
GIT_COMMITTER_DATE=$(git log -n1 --pretty=%aD) git tag -a -m "Release 0.3.0" 0.3.0
git push --tags
```

アノテーション付きタグが一つでも存在するとアノテーションのないタグを無視するGitコマンドがあるため、
アノテーション付きタグの使用が推奨されます。

[RFC 1105]: https://github.com/rust-lang/rfcs/blob/master/text/1105-api-evolution.md

### 例

- [Serde 1.0.0 release notes](https://github.com/serde-rs/serde/releases/tag/v1.0.0)
- [Serde 0.9.8 release notes](https://github.com/serde-rs/serde/releases/tag/v0.9.8)
- [Serde 0.9.0 release notes](https://github.com/serde-rs/serde/releases/tag/v0.9.0)
- [Diesel change log](https://github.com/diesel-rs/diesel/blob/master/CHANGELOG.md)


<a id="c-hidden"></a>
## 無用な実装詳細がRustdocに表示されていない (C-HIDDEN)

Rustdocはユーザがそのクレートを使用するのに必要な情報を網羅すべきですが、
それ以上の内容は含むべきではありません。
文章中で関係のある実装詳細を解説するのはよいですが、
実際にドキュメント項目として現れてはいけません。

特に、どのようなimplがドキュメントに表示されるかに関しては精選するようにしてください。
ユーザがクレートを使用するのに必要なものだけ示し、それ以外は隠しましょう。
以下のコードでは、`PublicError`のドキュメントに`From<PrivateError>`が表示されてしまいます。
ユーザが`PrivateError`型を扱うことはないため、この項目はユーザにとっては関係ないものです。
なので、`#[doc(hidden)]`を使って隠します。

```rust
// This error type is returned to users.
pub struct PublicError { /* ... */ }

// This error type is returned by some private helper functions.
struct PrivateError { /* ... */ }

// Enable use of `?` operator.
#[doc(hidden)]
impl From<PrivateError> for PublicError {
    fn from(err: PrivateError) -> PublicError {
        /* ... */
    }
}
```

[`pub(crate)`]も実装詳細をパブリックなAPIから隠す便利なツールです。
同じモジュール以外からもアイテムを使えるようになりますが、他のクレートからは見えません。

[`pub(crate)`]: https://github.com/rust-lang/rfcs/blob/master/text/1422-pub-restricted.md
