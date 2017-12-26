# デバッガビリティ


<a id="c-debug"></a>
## 全てのパブリックな型に`Debug`を実装する (C-DEBUG)

例外が必要なことは稀なはずです。


<a id="c-debug-nonempty"></a>
## `Debug`表現を空にしない (C-DEBUG-NONEMPTY)

`Debug`表現は空にするべきではありません。概念的に空である値に対しても同様です。

```rust
let empty_str = "";
assert_eq!(format!("{:?}", empty_str), "\"\"");

let empty_vec = Vec::<bool>::new();
assert_eq!(format!("{:?}", empty_vec), "[]");
```
