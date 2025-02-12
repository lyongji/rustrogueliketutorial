# 第 1 章：你好，Rust

---

***关于本教程***

*本教程是免费和开源的，所有代码都使用 MIT 许可证 - 因此您可以根据自己的喜好自由使用。我希望您会喜欢这个教程，并制作出伟大的游戏！*

*如果您喜欢这个教程并希望我继续写作，请考虑支持[我的 Patreon](https://www.patreon.com/blackfuture)。*

[![Hands-On Rust](./beta-webBanner.jpg)](https://pragprog.com/titles/hwrust/hands-on-rust/)

---

本教程主要关于学习如何制作 rogue-like 游戏（以及由此扩展的其他游戏），但也应该帮助您熟悉 Rust 和 RLTK - 我们将用来提供输入/输出的 *Roguelike 工具包*。即使您不想使用 Rust，我也希望您能从结构、想法和一般的游戏开发建议中受益。

## 为什么选择 Rust？

Rust 最初出现在 2010 年，但直到最近才达到 "稳定" 状态 - 也就是说，您编写的代码在语言发生变化时不太可能停止工作。开发工作仍在进行中，语言的整个新部分（如异步系统）仍在出现/稳定。本教程将远离开发的尖端 - 它应该是稳定的。

Rust 被设计成 "更好的系统语言" - 即，像 `C++` 一样低级，但更少的犯错机会，专注于避免使 C++ 开发变得困难的许多 "陷阱"，并大量关注内存和线程安全：它被设计成很难编写会破坏其内存或遭受竞态条件的程序（不是不可能，但您必须尝试！）。

Rust 还被设计成比 C++ 有更好的生态系统。`Cargo` 提供了一个完整的包管理器（在 C++ 领域也有 `vcpkg`、`conan` 等，但 cargo 与它们很好地集成），一个完整的构建系统（类似于 `cmake`、`make`、`meson` 等 - 但标准化）。它不像 C 或 C++ 那样在许多平台上运行，但列表在不断扩大。

我尝试了 Rust（在朋友的敦促下），发现虽然它并没有取代我日常工具箱中的 C++ - 但有时它确实帮助我完成了项目。它的语法需要一段时间来适应，但它确实很好地融入了现有的基础设施。

## 学习 Rust

如果您使用过其他编程语言，那么有很多帮助可用！

* [Rust 编程语言书](https://doc.rust-lang.org/book/) 提供了一个优秀的自上而下的语言介绍。
* [通过示例学习 Rust](https://doc.rust-lang.org/rust-by-example/) 更接近我喜欢的学习方式（我已经熟悉多种语言），为大多数您可能会遇到的 topics 提供了常见的用法示例。
* [24 天的 Rust](https://zsiciarz.github.io/24daysofrust/index.html) 提供了一个专注于网络的 24 天课程来学习 Rust。
* [Rust 的所有权模型适用于 JavaScript 开发者](https://blog.thoughtram.io/rust/2015/05/11/rusts-ownership-model-for-javascript-developers.html) 如果您来自 JS 或其他非常高级的语言，这应该会有所帮助。

如果您发现需要一些不在那里的东西，很可能有人编写了一个 `crate`（在每种其他语言中都是 "package"，但 cargo 处理 crates...）来帮助。一旦您有了工作环境，您可以输入 `cargo search <my term>` 来寻找可以帮助的 crates。您还可以前往 [crates.io](https://crates.io/) 查看 Cargo 中提供的完整 crates 列表 - 完整的文档和示例。

如果您完全新手编程，那么有个坏消息：Rust 是一门相对年轻的语言，因此还没有很多 "从零开始学习 Rust" 的材料 - 尚未。您可能会发现从更高级的语言开始，然后 "向下" 移动（更接近金属，如此说来）到 Rust 会更容易。上面链接的教程/指南应该能帮助您入门，如果您决定冒险的话。

## 获取 Rust

在大多数平台上，[rustup](https://rustup.rs/) 足以让您获得一个工作的 Rust 工具链。在 Windows 上，这是一个简单的下载 - 完成后您将获得一个工作的 Rust 环境。在 Unix 派生的系统（如 Linux 和 OS X）上，它提供了一些命令行指令来安装环境。

安装完成后，通过在命令行上输入 `cargo --version` 来验证它是否正常工作。您应该看到类似于 `cargo 1.36.0 (c4fcfb725 2019-05-15)` 的内容（版本会随时间而变化）。

## 获得舒适的开发环境

您想为您的开发工作创建一个目录/文件夹（我个人使用 `users/herbert/dev/rust` - 但这只是个人选择。它实际上可以是任何您喜欢的地方！）。您还需要一个文本编辑器。我是 [Visual Studio Code](https://code.visualstudio.com/) 的粉丝，但您可以使用您习惯的任何编辑器。如果您使用 Visual Studio Code，我推荐以下扩展：

* `Better TOML`：使阅读 toml 文件变得愉快；Rust 经常使用它们
* `C/C++`：使用 C++ 调试系统来调试 Rust 代码
* `Rust (rls)`：不是最快的，但提供全面的语法高亮和错误检查。

一旦您选择了环境，打开一个编辑器并导航到您的新文件夹（在 VS Code 中，选择 `文件 -> 打开文件夹` 并选择文件夹）。

## 创建项目

现在您已经选择好了文件夹，您想要在该文件夹中打开一个终端/控制台窗口。在 VS Code 中，可以通过 `终端 -> 新终端` 来实现。否则，可以像平常一样打开命令行并使用 `cd` 命令切换到您的文件夹。

Rust 拥有一个名为 `cargo` 的内置包管理器。Cargo 可以为您创建项目模板！因此，要创建您的新项目，请输入 `cargo init hellorust`。片刻之后，项目中会出现一个名为 `hellorust` 的新文件夹。
它将包含以下文件和目录：

```
src\main.rs
Cargo.toml
.gitignore
```

这些是：

* 如果您使用 git，`.gitignore` 文件非常有用 - 它可以防止您意外地将不需要的文件放入 git 仓库。如果您不使用 git，可以忽略它。
* `src\main.rs` 是一个简单的 Rust "hello world" 程序源文件。
* `Cargo.toml` 定义了您的项目以及如何构建它。

### 快速 Rust 介绍 - Hello World 的结构

自动生成的 `main.rs` 文件如下所示：

```
fn main() {
    println!("Hello, world!");
}
```

如果您使用过其他编程语言，这应该看起来有点熟悉 - 但语法/关键字可能不同。*Rust* 最初是 [ML](https://en.wikipedia.org/wiki/ML_(programming_language)) 和 C 的混合体，旨在创建一种灵活的 "系统" 语言（意味着：您可以编写用于 CPU 的裸机代码，而无需像 Java 或 C# 那样需要虚拟机）。在这个过程中，它从这两种语言中继承了很多语法。我发现 Rust 的语法在最初的一周看起来很糟糕，但之后很快就变得自然了。就像人类语言一样，让您的头脑适应语法和布局需要一段时间。

那么，这一切意味着什么？

1. `fn` 是 Rust 的 *函数* 关键字。在 JavaScript 或 Java 中，这会写成 `function main()`。在 C 中，会写成 `void main()`（尽管在 C 中 `main` 意味着返回一个 `int`）。在 C# 中，会是 `static void Main(...)`。
2. `main` 是函数的 *名称*。在这种情况下，该名称是一个特殊情况：操作系统需要知道在将程序加载到内存时首先运行什么 - Rust 会额外工作以标记 `main` 为第一个函数。通常，如果您希望程序做任何事情，除非您正在制作一个 *库*（其他程序使用的函数集合），您需要一个 `main` 函数。
3. `()` 是函数的 *参数* 或 *参数*。在这种情况下，没有任何参数 - 因此我们只使用括号。
4. `{` 表示一个 *块* 的开始。在这种情况下，该块是函数的 *主体*。在 `{` 和 `}` 之间的所有内容都是函数的 *内容*：依次执行的指令。块还表示 *作用域* - 因此在函数内部声明的任何内容都仅限于该函数的访问。换句话说，如果您在名为 `cheese` 的函数内部创建一个变量，它将无法从名为 `mouse` 的函数内部看到（反之亦然）。有办法绕过这一点，我们将在构建游戏时介绍它们。
5. `println!` 是一个 *宏*。您可以通过宏名称后面的 `!` 来识别 Rust 宏。您可以在[这里](https://doc.rust-lang.org/1.2.0/book/macros.html)了解有关宏的所有信息；目前，您只需要知道它们是 *特殊* 的函数，在编译期间解析为 *其他代码*。将内容打印到屏幕上可能相当复杂 - 您可能想要打印比 "hello world" 更多的内容 - 而 `println!` 宏涵盖了很多格式化情况。（如果您熟悉 C++，它等同于 `std::fmt`。大多数语言都有自己的字符串格式化系统，因为程序员往往需要输出大量文本！）
6. 最后的 `}` 关闭了在第 4 步开始的块。

前往终端并输入 `cargo run`。经过一些编译后，如果一切正常，您将在终端中看到 "Hello World"。

### 有用的 `cargo` 命令

Cargo 是一个非常有用的工具！您可以从 [Rust 教程](https://doc.rust-lang.org/1.2.0/book/hello-cargo.html) 中了解到一些关于它的信息，如果您感兴趣，可以从 [Cargo 书籍](https://doc.rust-lang.org/cargo/) 中了解到关于它的所有信息。

在 Rust 中工作时，您会经常与 `cargo` 交互。如果您使用 `cargo init` 初始化程序，那么您的程序就是一个 cargo *crate*。编译、测试、运行、更新 - Cargo 可以帮助您完成所有这些工作。它甚至默认为您设置 `git`。

您可能会发现以下 `cargo` 功能很有用：

* `cargo init` 创建一个新项目。这就是您用来创建 hello world 程序的工具。如果您 *真的* 不想使用 `git`，可以输入 `cargo init --vcs none (projectname)`。
* `cargo build` 下载项目的所有依赖项并编译它们，然后编译您的程序。它实际上并不会运行您的程序 - 但这是快速查找编译错误的好方法。
* `cargo update` 将获取您在 `cargo.toml` 文件中列出的 *crates* 的新版本（见下文）。
* `cargo clean` 可以删除项目的所有中间工作文件，释放大量磁盘空间。它们将在您下次运行/构建项目时自动下载并重新编译。偶尔，`cargo clean` 可以帮助解决一些不正常工作的问题 - 特别是 IDE 集成。
* `cargo verify-project` 将告诉您您的 Cargo 设置是否正确。
* `cargo install` 可以通过 Cargo 安装程序。这对于安装您需要的工具很有帮助。

Cargo 还支持 *扩展* - 即插件，使它能够完成更多工作。有一些您可能会发现特别有用的扩展：

* Cargo 可以将所有源代码格式化为 Rust 手册中的标准 Rust。您需要输入 `rustup component add rustfmt` *一次* 来安装该工具。完成后，您可以随时输入 `cargo fmt` 来格式化代码。
* 如果您想使用 `mdbook` 格式 - 用于 [这本书](https://github.com/thebracket/rustrogueliketutorial)! - cargo 也可以帮助您完成这项工作。只需运行 `cargo install mdbook` 一次，即可将工具添加到系统中。之后，`mdbook build` 将构建一个书籍项目，`mdbook init` 将创建一个新的项目，`mdbook serve` 将为您提供一个本地网络服务器来查看您的工作！您可以在他们的 [文档页面](https://rust-lang-nursery.github.io/mdBook/cli/index.html) 上了解到有关 `mdbook` 的所有信息。
* Cargo 还可以与一个 "linter" 集成 - 称为 `Clippy`。Clippy 有点吹毛求疵（就像他的 Microsoft Office 名字一样！）！只需运行 `rustup component add clippy` 一次。您现在可以随时输入 `cargo clippy` 来查看代码可能存在的问题的建议！


### 创建一个新项目

让我们修改新创建的 "hello world" 项目，以使用 [RLTK](https://github.com/thebracket/bracket-lib) - Roguelike Toolkit。

## 配置 Cargo.toml

自动生成的 Cargo 文件如下所示：
```toml
[package]
name = "helloworld"
version = "0.1.0"
authors = ["您的名字，如果它知道的话"]
edition = "2018"

# 有关更多键及其定义，请参见 https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
```

请确保您的名字是正确的！接下来，我们将要求 Cargo 使用 RLTK - Rogue-like 工具包库。Rust 使这变得非常容易。调整 `dependencies` 部分如下所示：
```toml
[dependencies]
rltk = { version = "0.8.0" }
```
我们告诉它包名为 `rltk`，并且在 Cargo 中可用 - 因此我们只需要给出它的版本。您可以执行 `cargo search rltk` 来随时查看最新版本，或访问 [crate webpage](https://crates.io/crates/rltk)。

定期运行 `cargo update` 是一个好主意 - 这将更新程序使用的库。

## Hello Rust - RLTK 风格！

请将 `src\main.rs` 的内容替换为：

```rust
use rltk::{Rltk, GameState};

struct State {}
impl GameState for State {
    fn tick(&mut self, ctx : &mut Rltk) {
        ctx.cls();
        ctx.print(1, 1, "Hello Rust World");
    }
}

fn main() -> rltk::BError {
    use rltk::RltkBuilder;
    let context = RltkBuilder::simple80x50()
        .with_title("Roguelike 教程")
        .build()?;
    let gs = State{ };
    rltk::main_loop(context, gs)
}
```

现在创建一个名为 `resources` 的新文件夹。RLTK 需要一些文件来运行，这就是我们放置它们的地方。下载 [resources.zip](./resources.zip)，并将其解压到这个文件夹中。请确保你有 `resources/backing.fs`（等等），而不是 `resources/resources/backing.fs`。

保存并返回到终端。输入 `cargo run`，你将看到一个控制台窗口显示 `Hello Rust`。

![截图](./c1-s1.png)

如果你刚接触 Rust，你可能会想知道 `Hello Rust` 代码到底做了什么，以及它为什么会在这里 - 所以我们将花一点时间来解释它。

1. 第一行相当于 C++ 的 `#include` 或 C# 的 `using`。它只是告诉编译器我们将需要来自 `rltk` 命名空间的 `Rltk` 和 `GameState` 类型。你曾经需要在这里添加额外的 `extern crate` 行，但最新版本的 Rust 现在可以自己弄清楚。
2. 使用 `struct State{}`，我们创建了一个新的 `结构`。结构类似于 Pascal 的 Records，或许多其他语言中的 Classes：你可以在它们内部存储一堆数据，并且还可以为它们附加“方法”（函数）。在这种情况下，我们实际上不需要任何数据 - 我们只需要一个放置代码的地方。如果你想了解更多关于结构的信息，[这是 Rust 书籍中关于该主题的章节](https://doc.rust-lang.org/book/ch05-00-structs.html)。
3. `impl GameState for State` 是相当口的！我们告诉 Rust 我们的 `State` 结构 *实现了* *特征* `GameState`。特征类似于其他语言中的接口或基类：它们为你设置了一个结构，你可以在自己的代码中实现它，然后让它与提供它的库交互 - 而无需该库了解你的代码的任何其他信息。在这种情况下，`GameState` 是 RLTK 提供的一个特征。RLTK 要求你有一个 - 它用它来在每一帧调用你的程序。你可以了解特征[在 Rust 书籍的这一章中](https://doc.rust-lang.org/book/ch10-02-traits.html)。
4. `fn tick(&mut self, ctx : &mut Rltk)` 是一个 *函数* 定义。我们在特征实现范围内，所以我们是为特征实现函数 - 所以它 *必须* 匹配特征要求的类型。函数是 Rust 的基本构建块，我推荐[关于该主题的 Rust 书籍章节](https://doc.rust-lang.org/book/ch03-03-how-functions-work.html)。
    1. 在这种情况下，`fn tick` 意味着“创建一个名为 tick 的函数”（它被称为 “tick”是因为它随着渲染的每一帧而 “滴答作响”；在游戏编程中，将每次迭代称为一次 tick 是很常见的）。
    2. 它不以 `-> type` 结尾，所以它相当于 C 中的 `void` 函数 - 它在调用后不返回任何数据。参数也可以从一点解释中受益。
    3. `&mut self` 意味着“此函数需要访问父结构，并可能更改它”（`mut` 是“可变”的缩写 - 意味着它可以更改结构内的变量 - “状态”）。你也可以在结构中有只有 `&self` 的函数 - 意味着，我们可以 *看到* 结构的内容，但不能更改它。如果你完全省略 `&self`，函数根本看不到结构 - 但可以像结构是 *命名空间* 一样调用它（你经常看到带有 `new` 的函数这样做 - 它们为你创建结构的新副本）。
    4. `ctx: &mut Rltk` 意味着“传入一个名为 `ctx` 的变量”（`ctx` 是“上下文”的缩写）。冒号表示我们正在指定变量必须是哪种 *类型*。
    5. `&` 意味着“传递引用” - 这是现有变量副本的 *指针*。变量没有被复制，你正在处理传入的版本；如果你进行更改，你将更改原始版本。[Rust 书籍比我能更好地解释这一点](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html)。
    6. 再次，`mut` 表示这是一个“可变”引用：你允许对上下文进行更改。
    7. 最后，`Rltk` 是你接收的变量的 *类型*。在这种情况下，它是 `RLTK` 库中定义的一个 `struct`，它提供了你可以对屏幕执行的多种操作。
5. `ctx.cls();` 表示“调用变量 `ctx` 提供的 `cls` 函数”。`cls` 是“清屏”的常见缩写 - 我们告诉我们的 *上下文* 它应该清除虚拟终端。在每一帧的开始时这样做是个好主意，除非你特别不想这样做。
6. `ctx.print(1, 1, "Hello Rust World");` 是请求 *上下文* 在位置 (1,1) *打印* “Hello Rust World”。
7. 现在我们来到了 `fn main()`。*每个* 程序都有一个 `main` 函数：它告诉操作系统程序从哪里开始。
8. ```rust
   use rltk::RltkBuilder;
    let context = RltkBuilder::simple80x50()
        .with_title("Roguelike 教程")
        .build()?;
   ```     
   是从 `struct` 内部调用 *函数* 的示例 - 其中该结构不接收 “self” 函数。在其他语言中，这被称为 *构造函数*。我们调用 `simple80x50` 函数（这是 RLTK 提供的一个构建器，用于创建一个宽 80 个字符、高 50 个字符的终端。窗口标题是“Roguelike 教程”。
9. `let gs = State{ };` 是一个 *变量* 赋值的示例（参见 [Rust 书籍](https://doc.rust-lang.org/book/ch03-01-variables-and-mutability.html)）。我们正在创建一个名为 `gs`（“游戏状态”的缩写）的新变量，并将其设置为上面定义的 `State` 结构的副本。
10. `rltk::main_loop(context, gs)` 调用 `rltk` 命名空间，激活一个名为 `main_loop` 的函数。它需要我们之前制作的 `context` 和 `GameState` - 所以我们传递它们。RLTK 试图消除运行 GUI/游戏应用程序的一些复杂性，并提供了这个包装器。该函数现在接管了程序的控制权，并且每次程序“滴答作响”（即完成一个循环并移动到下一个循环）时，都会调用你的 `tick` 函数（参见上文）。这可以每秒发生 60 次或更多次！

希望这有点道理！

## 使用教程

你可能想要使用教程代码而不必全部输入！好消息是，它已经在 GitHub 上供你参考了。你需要安装 `git`（RustUp 应该已经帮助你完成了这个）。选择你想要放置教程的位置，并打开一个终端：
```
cd <教程的路径>
git clone https://github.com/thebracket/rustrogueliketutorial .
```
过一段时间，这将会下载完整的教程（包括本书的源代码！）。它的布局如下（这并不完整！）：
```
───book
├───chapter-01-hellorust
├───chapter-02-helloecs
├───chapter-03-walkmap
├───chapter-04-newmap
├───chapter-05-fov
├───resources
├───src
```
这里有什么？

* `book` 文件夹包含本书的源代码。你可以忽略它，除非你想纠正我的拼写错误！
* 每个章节的示例代码都包含在 `chapter-xy-name` 文件夹中；例如，`chapter-01-hellorust`。
* `src` 文件夹包含一个简单的脚本，提醒你在运行任何东西之前切换到章节文件夹。
* `resources` 有你为这个示例下载的 ZIP 文件的内容。所有章节文件夹都预先配置为使用这个。
* `Cargo.toml` 设置为包含所有教程作为 "工作区条目" - 它们共享依赖项，所以它不会在每次使用时重新下载所有内容。

要运行一个示例，打开你的终端并：
```
cd <你放置教程的位置>
cd chapter-01-hellorust
cargo run
```
如果你使用的是 *Visual Studio Code*，你可以使用 *文件 -> 打开文件夹* 来打开你检查出的整个目录。使用内置终端，你可以简单地 `cd` 到每个示例并 `cargo run` 它。

## 访问教程源代码

你可以通过 [https://github.com/thebracket/rustrogueliketutorial](https://github.com/thebracket/rustrogueliketutorial) 访问所有教程的源代码。

## 更新教程

我经常更新这个教程 - 添加章节、修复问题等。你会定期想要打开教程目录，并输入 `git pull`。这告诉 `git`（源代码控制管理器）去 `Github` 仓库并寻找新内容。然后它会下载所有更改的内容，你再次拥有最新的教程。

## 更新你的项目

你可能会发现 `rltk_rs` 或另一个包已经更新，你想要最新版本。从你的项目文件夹开始，你可以输入 `cargo update` 来更新 *一切*。你可以输入 `cargo update --dryrun` 来查看它会更新什么，并且不改变任何内容（人们经常更新他们的 crates - 所以这可能会是一个大列表！）。

## 更新 Rust 本身

我不建议在 `Visual Studio Code` 或其他 IDE 内部运行这个，但如果你想确保你有 `Rust`（及相关工具）的最新版本，你可以输入 `rustup self update`。这更新了 Rust 更新工具（我知道这听起来相当递归）。然后你可以输入 `rustup update` 并安装所有工具的最新版本。

## 获取帮助

有多种方式可以获取帮助：

* 如果有任何问题、改进建议或希望我添加的内容，请随时联系我（我是 Twitter 上的 `@herberticus`）。
* Reddit 上的 [/r/rust](https://www.reddit.com/r/rust/) 社区成员对 Rust 语言问题非常有帮助。
* Reddit 上的 [/r/roguelikedev](https://www.reddit.com/r/roguelikedev/) 社区成员在 Rogueike 问题方面非常有帮助。他们的 Discord 社区也很活跃。

[在浏览器中运行本章示例，使用 WebAssembly（需要 WebGL2）](https://bfnightly.bracketproductions.com/rustbook/wasm/chapter-01-hellorust/)

---

版权所有 (C) 2019, Herbert Wolverson。

---
