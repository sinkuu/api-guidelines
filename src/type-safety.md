# 型安全性


<a id="c-newtype"></a>
## newtypeを使って静的に値を区別する (C-NEWTYPE)

実際の型は同じでも、newtypeを用いることでその異なる解釈の間を静的に区別することができます。

例えば、`f64`の値はマイルとキロメートルの何れかの量を示しているかもしれません。
newtypeを使うことで、どちらが正しい解釈なのかを示すことが可能です。

```rust
struct Miles(pub f64);
struct Kilometers(pub f64);

impl Miles {
    fn to_kilometers(self) -> Kilometers { /* ... */ }
}
impl Kilometers {
    fn to_miles(self) -> Miles { /* ... */ }
}
```

このように型を分けることで、混ざってしまわないことを静的に保証できます。
例えば、

```rust
fn are_we_there_yet(distance_travelled: Miles) -> bool { /* ... */ }
```

は間違って`Kilometers`の値で呼ばれることはありません。
コンパイラが適切な変換を施すことを思い出させてくれるため、[恐ろしいバグ]も回避できます。

[恐ろしいバグ]: http://en.wikipedia.org/wiki/Mars_Climate_Orbiter


<a id="c-custom-type"></a>
## `bool`や`Option`の代わりに意味のある型を使っている (C-CUSTOM-TYPE)

こうして下さい。

```rust
let w = Widget::new(Small, Round)
```

次は駄目な例です。

```rust
let w = Widget::new(true, false)
```

`bool`や`u8`、`Option`のような汎用的な型には様々な解釈が考えられます。

専用の型(列挙型、構造体、あるいはタプル)を用い、その意味と不変条件を伝えるようにしてください。
上記の駄目な例では、引数の名前を調べない限り`true`と`false`がどういった意味なのか分かりません。
`Small`や`Round`といった型を使った場合ではその意味が明確になっています。

専用の型を使うことで今後の拡張も楽になります。上記の例で言うと、
ここに`ExtraLarge`というバリアントを追加したりできますね。

既存の型にゼロコストで区別された名前を付ける方法についてはnewtypeパターン([C-NEWTYPE])を参照してください。

[C-NEWTYPE]: #c-newtype


<a id="c-bitflag"></a>
## フラグの集合を列挙型ではなく`bitflags`で表している (C-BITFLAG)

Rustでは`enum`において値を明示することが可能です。

```rust
enum Color {
    Red = 0xff0000,
    Green = 0x00ff00,
    Blue = 0x0000ff,
}
```

値の指定は他のシステムや言語向けに整数値としてシリアライゼーションしないといけない場合に便利です。
数値でなく`Color`を関数が取れば、数値への変換が可能であると同時に不正な値の入力が防げるため
「型安全性」に貢献します。

`enum`は複数の選択肢のうちの1つを要求するAPIに使われます。
しかし、APIの入力がフラグの集合である場合もあります。
C言語のコードでは各々のフラグを特定のビットに割当てることで、1つの整数値によって32あるいは64等のフラグを表せるようにします。
Rustでは[`bitflags`]クレートがこのパターンの型安全な実装を提供しています。

[`bitflags`]: https://github.com/rust-lang-nursery/bitflags

```rust
#[macro_use]
extern crate bitflags;

bitflags! {
    struct Flags: u32 {
        const FLAG_A = 0b00000001;
        const FLAG_B = 0b00000010;
        const FLAG_C = 0b00000100;
    }
}

fn f(settings: Flags) {
    if settings.contains(FLAG_A) {
        println!("doing thing A");
    }
    if settings.contains(FLAG_B) {
        println!("doing thing B");
    }
    if settings.contains(FLAG_C) {
        println!("doing thing C");
    }
}

fn main() {
    f(FLAG_A | FLAG_C);
}
```


<a id="c-builder"></a>
## 複雑な値の生成にビルダーパターンを使っている (C-BUILDER)

以下のような理由によって、データ構造の生成に複雑な手順が必要なことがあります。

* 多数の入力がある
* 複合的なデータである (例えばスライス)
* オプショナルな設定データがある
* 選択肢が複数ある

こんなとき、異なった引数をもつ多数のコンストラクタを追加してしまいがちです。

そういったデータ構造`T`があるとき、`T`に _ビルダー_ を用意することを検討してください。

1. インクリメンタルに`T`を設定するための別のデータ型、`TBuilder`を作ります。
   可能であれば、より明瞭な名前を選んでください。例えば、[子プロセス]を作成するビルダーの
   [`Command`]、[`Url`]を作成するビルダーの[`ParseOptions`]のようにです。
2. ビルダー自体のコンストラクタは、`T`を生成するために _必須_ のパラメータのみ取るべきです。
3. ビルダーは設定を行うための便利なメソッドを提供するべきです。
   例えば、複合的なデータをインクリメンタルに設定できるようなものです。
   メソッドは`self`を返して、メソッドチェーンを行えるようにすべきです。
4. ビルダーは実際に`T`の値を生成する「終端」メソッドを1つ以上提供すべきです。

[`Command`]: https://doc.rust-lang.org/std/process/struct.Command.html
[子プロセス]: https://doc.rust-lang.org/std/process/struct.Child.html
[`Url`]: https://docs.rs/url/1.4.0/url/struct.Url.html
[`ParseOptions`]: https://docs.rs/url/1.4.0/url/struct.ParseOptions.html

ビルダーパターンは特に、`T`の生成においてプロセスやタスクの立ち上げのような副作用が生じる場合にも適しています。

Rustでのビルダーパターンの作り方には、所有権の取り扱いによって以下で示す2通りの方法があります。

### 非消費ビルダー (推奨)

最終的な`T`の生成にビルダー自身を必要としない場合があります。
例として[`std::process::Command`]を挙げます。

[`std::process::Command`]: https://doc.rust-lang.org/std/process/struct.Command.html

```rust
// NOTE: 実際のCommand APIはStringを所有しません。
// これは単純化されたバージョンです

pub struct Command {
    program: String,
    args: Vec<String>,
    cwd: Option<String>,
    // etc
}

impl Command {
    pub fn new(program: String) -> Command {
        Command {
            program: program,
            args: Vec::new(),
            cwd: None,
        }
    }

    /// Add an argument to pass to the program.
    pub fn arg(&mut self, arg: String) -> &mut Command {
        self.args.push(arg);
        self
    }

    /// Add multiple arguments to pass to the program.
    pub fn args(&mut self, args: &[String]) -> &mut Command {
        self.args.extend_from_slice(args);
        self
    }

    /// Set the working directory for the child process.
    pub fn current_dir(&mut self, dir: String) -> &mut Command {
        self.cwd = Some(dir);
        self
    }

    /// Executes the command as a child process, which is returned.
    pub fn spawn(&self) -> io::Result<Child> {
        /* ... */
    }
}
```

ビルダーの設定データを使って実際にプロセスを立ち上げる`spawn`メソッドが、
ビルダーを不変参照で受け取っていることに注目してください。
これはプロセスの立ち上げにビルダーの設定データの所有権が必要ないからです。

終端メソッドである`spawn`が参照しか必要としないのですから、
設定メソッドは`self`を参照で受け取り、返すべきです。

#### 利点

借用を使うことで、`Command`はワンライナーでも、もっと複雑なことをする場合でも
どちらにでも使うことができます。

```rust
// One-liners
Command::new("/bin/cat").arg("file.txt").spawn();

// Complex configuration
let mut cmd = Command::new("/bin/ls");
cmd.arg(".");
if size_sorted {
    cmd.arg("-S");
}
cmd.spawn();
```

### 消費ビルダー

ビルダーが`T`を生成する際に、所有権を移動しなければならない場合があります。
つまり終端メソッドが`&self`ではなく`self`を取るときです。

```rust
impl TaskBuilder {
    /// Name the task-to-be.
    pub fn named(mut self, name: String) -> TaskBuilder {
        self.name = Some(name);
        self
    }

    /// Redirect task-local stdout.
    pub fn stdout(mut self, stdout: Box<io::Write + Send>) -> TaskBuilder {
        self.stdout = Some(stdout);
        self
    }

    /// Creates and executes a new child task.
    pub fn spawn<F>(self, f: F) where F: FnOnce() + Send {
        /* ... */
    }
}
```

ここで、`stdout`を設定するときには`io::Write`を実装する型の所有権が渡されるため、
タスクの生成を行う際にその所有権を移動する必要があります(`spawn`内)。

ビルダーの終端メソッドが所有権を要求するとき、大きなトレードオフが存在します。

* もし他のビルダーメソッドが可変借用を受け渡すと、
  前述した複雑な設定の場合は上手く動きますが、ワンライナーでの設定は不可能になります。

* もし他のビルダーメソッドが`self`の所有権を受け渡すと、
  ワンライナーはそのまま動きますが、複雑な設定を行う際には不便です。

簡単なものは簡単なまま保つべきですから、
消費ビルダーのビルダーメソッドは全て`self`を取り、返すべきです。
このビルダーを使用するコードは次のようになります。

```rust
// One-liners
TaskBuilder::new("my_task").spawn(|| { /* ... */ });

// Complex configuration
let mut task = TaskBuilder::new();
task = task.named("my_task_2"); // must re-assign to retain ownership
if reroute {
    task = task.stdout(mywriter);
}
task.spawn(|| { /* ... */ });
```

所有権が`spawn`に消費されるまで各メソッド間で次々と受け渡され、ワンライナーは今までどおり動きます。
一方で、複雑な設定は記述量が増えています。各メソッドを呼ぶ際に再束縛が必要になります。
