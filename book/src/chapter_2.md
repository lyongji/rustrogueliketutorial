# 第2章 - 实体与组件

---

***关于本教程***

*本教程是免费和开源的，所有代码均使用MIT许可证 - 因此你可以随意使用。我希望你会喜欢这个教程，并制作出伟大的游戏！*

*如果你喜欢这个教程并希望我继续写作，请考虑支持[我的Patreon](https://www.patreon.com/blackfuture)。*

![实战Rust](./beta-webBanner.jpg)

---

本章将介绍整个实体组件系统（ECS），它将构成本教程其余部分的基础。Rust有一个非常好的ECS，名为Specs - 本教程将向你展示如何使用它，并试图演示一些早期使用它的好处。

## 关于实体与组件

如果你以前从事过游戏开发，你可能会习惯于面向对象的设计（这在原始的Python `libtcod`教程中非常常见，该教程也是本教程的灵感来源）。面向对象（OOP）设计并没有什么真正的问题 - 但游戏开发者已经逐渐远离它，主要是因为当你开始将游戏扩展到原始设计理念之外时，它可能会变得相当混乱。

你可能见过像这样的简化类层次结构：
```
BaseEntity
    Monster
        MeleeMob
            OrcWarrior
        ArcherMob
            OrcArcher
```
你可能会有比这更复杂的东西，但它作为一个例子是可行的。`BaseEntity`会包含作为实体出现在地图上所需的代码/数据，`Monster`表示它是一个坏人，`MeleeMob`会包含寻找近战目标、接近并击杀它们的逻辑。同样，`ArcherMob`会尝试保持最佳距离，并从安全距离使用远程武器开火。这种分类的问题在于它可能具有限制性，而且在你意识到之前 - 你开始为更复杂的组合编写单独的类。例如，如果我们想出一个既能进行近战又能进行弓箭的兽人 - 并且如果你完成了*与绿皮交朋友*的任务，它可能会变得友好？你可能会从所有这些中组合逻辑到一个特殊情况类中。它有效 - 并且许多游戏正是这样发布的 - 但如果有一种更容易的方法呢？

基于实体的组件设计试图消除层次结构，而是实现一组“组件”来描述你想要的内容。一个“实体”是一个*东西* - 任何东西，真的。一个兽人，一只狼，一瓶药水，一个以太硬盘格式化幽灵 - 任何你想要的东西。它也非常简单：只是一个识别号码。实体能够拥有尽可能多的*组件*的魔力来自于你想要添加的组件。组件只是数据，按你想要给实体的任何属性进行分组。

例如，你可以构建具有以下组件的相同一组怪物：`Position`，`Renderable`，`Hostile`，`MeleeAI`，`RangedAI`，以及某种战斗统计组件（告诉你他们的武器、生命值等）。一个兽人战士需要一个位置，这样你就知道他们在哪里，一个可渲染的，这样你就知道如何绘制它们。它是敌对的，所以你将其标记为敌对。给它一个近战AI和一套游戏统计数据，你就有了让它接近玩家并尝试击打他们的所有内容。一个弓箭手可能是同样的东西，但用远程AI替换近战AI。一个混合体可以保留所有组件，但要么有两者要么有额外的如果想要自定义行为。如果你的兽人变得友好，你可以移除敌对组件 - 并添加一个友好的组件。

换句话说：组件就像你的继承树，但不是*继承*特性，而是通过添加组件直到它做你想要的事情。这通常被称为“组合”。

ECS中的“S”代表“系统”。一个*系统*是一段代码，它从实体/组件列表中收集数据并对它做些什么。实际上，它与继承模型非常相似，但在某些方面它是“反向”的。例如，在OOP系统中绘制通常是这样的：*对每个BaseEntity，调用该实体的Draw命令*。在一个ECS系统中，它将是*获取所有具有位置和可渲染组件的实体，并使用该数据来绘制它们*。

对于小型游戏，ECS通常感觉像是给代码添加了一些额外的输入。的确如此。你提前做额外的工作，为了以后让生活更轻松。

这有很多要消化的，所以我们将看一个简单的例子，说明ECS如何使你的生活变得更轻松。

重要的是要知道，ECS只是处理组合的一种方式。还有许多其他方式，实际上并没有正确的答案。通过一些搜索，你可以找到许多不同的方法来接近ECS。有许多面向对象的方法。有许多“自由函数”的方法。它们都有价值，并且可以为你工作。我在本书中选择了实体-组件方法，但也有*许多*其他的方式来剥猫。随着你获得经验，你会找到一个让你感到舒适的方法！我的建议是：如果有人告诉你某种方法是“正确”的，忽略他们 - 编程是制作出能工作的东西的艺术，而不是追求纯粹性的追求！

## 在项目中包含Specs

首先，我们需要告诉Cargo我们将使用Specs。打开你的`Cargo.toml`文件，并将`dependencies`部分更改为如下所示：
```toml
[dependencies]
rltk = { version = "0.8.0" }
specs = "0.16.1"
specs-derive = "0.4.1"
```
这很简单：我们告诉Rust我们仍然想使用RLTK，并且我们还要求使用Specs（版本号在撰写本文时是当前的；你可以通过输入`cargo search specs`来检查是否有新的版本）。我们还添加了`specs-derive` - 它提供了一些辅助代码，以减少你必须输入的样板代码量。

在`main.rs`的顶部，我们添加几行代码：
```rust
use rltk::{GameState, Rltk, RGB, VirtualKeyCode};
use specs::prelude::*;
use std::cmp::{max, min};
use specs_derive::Component;
```
`use rltk::`是简写；你*可以*每次想要控制台时输入`rltk::Console`；这告诉Rust我们只想输入`Console`。同样，`use specs::prelude::*`行在这里，这样我们就不需要不断地输入`specs::prelude::World`，而只是想要`World`。

> 旧的Rust需要一个看起来可怕的`macro_use`调用。你不再需要那个：你可以直接使用宏。

我们需要从Specs的派生组件中获取派生：所以我们添加`use specs_derive::Component;`。

## 定义位置组件

我们将构建一个小演示，使用ECS在屏幕上放置字符并将它们移动。这的基本部分是定义一个`位置` - 这样实体就知道它们在哪里。我们将保持简单：位置只是屏幕上的X和Y坐标。

所以，我们定义一个`struct`（这些类似于C中的struct，Pascal中的记录等。 - 一组数据存储在一起。参见[Rust书籍关于结构的章节](https://doc.rust-lang.org/book/ch05-00-structs.html)）：
```rust
struct Position {
    x: i32,
    y: i32,
}
```
非常简单！一个`位置`组件有一个x和y坐标，作为32位整数。我们的`位置`结构被称为`POD` - 简称“普通旧数据”。也就是说，它只是数据，没有任何自己的逻辑。这是“纯”ECS（实体组件系统）组件的一个常见主题：它们只是数据，没有关联的逻辑。逻辑将在其他地方实现。使用这种模型有两个原因：它使你所有的代码，*做某事*保持在“系统”（即跨组件和实体的代码）中，并且性能 - 它非常快，将所有的位置保存在内存中，没有重定向。

在这一点上，你可以使用`位置`，但几乎没有帮助你存储它们或分配给任何人 - 所以我们需要告诉Specs这是组件。Specs提供了*很多*选项，但我们想保持简单。没有`specs-derive`帮助的长格式将如下所示：
```rust
struct Position {
    x: i32,
    y: i32,
}

impl Component for Position {
    type Storage = VecStorage<Self>;
}
```
你可能会在游戏完成时有*很多*组件 - 所以有很多输入。不仅如此，而且一遍又一遍地输入相同的事情 - 有可能变得令人困惑。幸运的是，`specs-derive`提供了一种更容易的方法。你可以替换上面的代码：
```rust
#[derive(Component)]
struct Position {
    x: i32,
    y: i32,
}
```
这是什么意思？`#[derive(x)]`是一个*宏*，它说“从我的基本数据中，请派生出*x*所需的样板代码”；在这种情况下，*x*是一个`组件`。宏为你生成额外的代码，所以你不必为每个组件都输入它。这使得使用组件变得非常简单！之前的`#[macro_use] use specs_derive::Component;`在这里得到了利用；*派生宏*是一种特殊的宏，它为你的结构实现了额外的功能 - 节省了很多输入。

## 定义可渲染组件

将字符放在屏幕上的第二部分是*我们应该绘制什么字符，以及什么颜色？*为了处理这个问题，我们将创建第二个组件 - `Renderable`。它将包含前景色、背景色和字形（如`@`）来渲染。所以我们将创建第二个组件结构：
```rust
#[derive(Component)]
struct Renderable {
    glyph: rltk::FontCharType,
    fg: RGB,
    bg: RGB,
}
```
`RGB`来自RLTK，代表一种颜色。这就是为什么我们有`use rltk::{... RGB}`语句 - 否则，我们每次都会输入`rltk::RGB` - 节省了按键次数。再次，这是一个*普通旧数据*结构，我们使用*派生*宏来添加组件存储信息，而无需输入所有内容。

## 世界与注册

现在我们有两种组件类型，但没有地方放它们就不太有用！Specs要求你在启动时*注册*你的组件。你要注册什么？一个`World`！

`World`是由Rust crate `Specs`提供的ECS。如果你愿意，你可以有多个，但我们现在不会深入讨论。我们将扩展我们的`State`结构，以便有一个地方来存储世界：
```rust
struct State {
    ecs: World
}
```
现在在`main`中，当我们创建世界时 - 我们会在其中放入一个ECS：
```rust
let mut gs = State {
    ecs: World::new()
};
```
注意`World::new()`是另一个*构造函数* - 它是`World`类型中的一个方法，但没有引用`self`。所以它不适用于现有的`World`对象 - 它只能创建新的对象。这是Rust中到处使用的模式，所以熟悉它是个好主意。[Rust书籍有一个关于这个主题的部分](https://doc.rust-lang.org/book/ch05-03-method-syntax.html)。

下一步是告诉ECS我们创建的组件。我们是在创建世界后立即这样做：
```rust
gs.ecs.register::<Position>();
gs.ecs.register::<Renderable>();
```
这告诉我们的`World`查看我们给它的类型，并进行一些内部魔法为它们中的每一个创建存储系统。Specs已经简化了这一点；只要它实现了`Component`，你可以将任何你喜欢的东西作为组件放入！

## 创建实体

现在我们有了一个知道如何存储`Position`和`Renderable`组件的`World`。仅仅拥有这些组件并不帮助我们，除了提供结构的指示。为了*使用*它们，它们需要被附加到游戏中的某个东西上。在ECS世界中，那东西被称为*实体*。实体相当简单；它们不过是一个识别号码，告诉ECS一个实体存在。它们可以附加任何组合的组件。在这种情况下，我们将创建一个*实体*，它知道自己在屏幕上的位置，并且知道它应该在屏幕上如何表示。

我们可以像这样创建一个具有`Renderable`和`Position`组件的实体：
```rust
gs.ecs
    .create_entity()
    .with(Position { x: 40, y: 25 })
    .with(Renderable {
        glyph: rltk::to_cp437('@'),
        fg: RGB::named(rltk::YELLOW),
        bg: RGB::named(rltk::BLACK),
    })
    .build();
```
这告诉我们的`World`（`gs`中的`ecs` - 我们的游戏状态），我们想要一个新的实体。那个实体应该有一个位置（我们选择了控制台的中间），我们希望它能以黄色在黑色上的`@`符号进行渲染。这非常简单；我们甚至没有存储实体（如果我们想的话可以），我们只是在告诉世界它存在！

注意我们使用了一个有趣的布局：许多不以`;`结尾的函数来分隔语句的结束，而是用许多`.`调用另一个函数。这被称为*构建器模式*，在Rust中非常常见。以这种方式组合函数被称为*方法链*（*方法*是结构内的一个函数）。它的工作原理是每个函数都返回自身的副本 - 所以每个函数依次运行，将自身传递给链中的下一个方法。所以在本例中，我们从`create_entity`调用开始 - 它返回一个新的、空的实体。在该实体上，我们调用`with` - 将组件附加到它。这又返回了部分构建的实体 - 所以我们可以再次调用`with`来添加`Renderable`组件。最后，`.build()`取组装好的实体并完成困难的部分 - 实际上将所有分散的部分放入ECS的正确部分中。

你可以轻松地添加更多的实体，如果你愿意。让我们这样做：
```rust
for i in 0..10 {
    gs.ecs
    .create_entity()
    .with(Position { x: i * 7, y: 20 })
    .with(Renderable {
        glyph: rltk::to_cp437('☺'),
        fg: RGB::named(rltk::RED),
        bg: RGB::named(rltk::BLACK),
    })
    .build();
}
```
这是我们教程中第一次调用`for`循环！如果你使用过其他编程语言，概念将会熟悉：运行循环，将`i`设置为从0到9的每个值。等等 - 9，你说？Rust的范围是*排他的* - 它不包括范围中的最后一个数字！这是为了熟悉像C这样的语言，它们通常写`for (i=0; i<10; ++i)`。如果你实际上*想*走到范围的末尾（所以是0到10），你会写相当神秘的`for i in 0..=10`。[Rust书籍提供了一个关于理解Rust中控制流的优秀入门](https://doc.rust-lang.org/book/ch03-05-control-flow.html)。

你会注意到我们将它们放在不同的位置（每次7个字符，10次），并且我们将`@`改为了`☺` - 一个笑脸（`to_cp437`是RLTK提供的一个辅助函数，让你输入/粘贴Unicode并得到旧DOS/CP437字符集中等效的成员。你可以用`1`替换`to_cp437('☺')`得到相同的结果）。你可以在这里找到可用的字形[这里](http://dwarffortresswiki.org/index.php/Character_table)。

## 迭代实体 - 通用渲染系统

现在我们有11个实体，它们具有不同的渲染特性和位置。利用这些数据做一些事情是个好主意！在我们的`tick`函数中，我们用以下代码替换绘制"Hello Rust"的调用：
```rust
let positions = self.ecs.read_storage::<Position>();
let renderables = self.ecs.read_storage::<Renderable>();

for (pos, render) in (&positions, &renderables).join() {
    ctx.set(pos.x, pos.y, render.fg, render.bg, render.glyph);
}
```
这段代码做了什么？`let positions = self.ecs.read_storage::<Position>();`请求ECS提供用于存储`Position`组件的容器的读取访问权限。同样，我们请求对`Renderable`存储的读取访问权限。只有在这两者都存在的情况下绘制字符才有意义 - 你*需要*一个`Position`来知道在哪里绘制，以及`Renderable`来知道绘制什么！你可以在[Specs文档](https://specs.amethyst.rs/docs/tutorials/01_intro.html)中了解更多关于这些存储的信息。重要的是`read_storage` - 我们在请求对用于存储每种类型组件的结构体的只读访问。

幸运的是，Specs为我们提供了支持：
```rust
for (pos, render) in (&positions, &renderables).join() {
```
这行代码表示将位置和可渲染组件进行`join`操作；就像数据库的连接一样，它只返回同时具有两者的实体。然后它使用Rust的“解构”将每个结果（每个实体一个结果，具有两个组件）放置在变量中。所以对于`for`循环的每次迭代 - 你都会得到属于同一实体的两个组件。这足以绘制它！

`join`函数返回一个*迭代器*。[Rust文档有一节关于迭代器的介绍](https://doc.rust-lang.org/book/ch13-02-iterators.html)。在C++中，迭代器提供了`begin`、`next`和`end`函数 - 你可以使用它们在集合中移动元素。Rust扩展了相同的概念，只是更强大：几乎任何东西都可以成为迭代器，只要你愿意。迭代器与`for`循环配合得非常好 - 你可以将任何迭代器作为`for x in iterator`循环的目标。我们之前讨论的`0..10`实际上是一个*范围* - 它为Rust提供了一个*迭代器*来导航。

这里另一个有趣的地方是括号。在Rust中，当你用括号包裹变量时，你是在创建一个*元组*。这些只是组合在一起的变量集合 - 但不需要为此专门创建一个结构体。你可以通过数字访问（`mytuple.0`、`mytuple.1`等）单独访问它们，或者你可以*解构*它们。`(one, two) = (1, 2)`将变量`one`设置为`1`，变量`two`设置为`2`。这就是我们在这里所做的：`join`迭代器返回包含`Position`和`Renderable`组件的*元组*，分别作为`.0`和`.1`。由于输入这些内容既难看又不清，我们*解构*它们为命名变量`pos`和`render`。这可能会让人困惑，如果你感到困难，我建议阅读[Rust By Example中的元组部分](https://doc.rust-lang.org/rust-by-example/primitives/tuples.html)。
```rust
ctx.set(pos.x, pos.y, render.fg, render.bg, render.glyph);
```
我们为每个具有*两者*（`Position`和`Renderable`组件）的实体运行此代码。`join`方法传递给我们两者，保证它们属于同一个实体。任何只有一个或另一个的实体 - 但不是两者都有 - 简直不会包含在我们返回的数据中。

`ctx`是当`tick`运行时传递给我们的RLTK实例。它提供了一个名为`set`的函数，可以将单个终端字符设置为你选择的字形/颜色。所以我们传递给它来自`pos`的数据（该实体的`Position`组件），以及来自`render`的颜色/字形（该实体的`Renderable`组件）。

有了这个设置，任何具有`Position`和`Renderable`组件的实体都将被渲染到屏幕上！你可以添加尽可能多的实体，它们将被渲染。移除一个组件或另一个，它们将不会被渲染（例如，如果一个物品被捡起，你可能会移除它的`Position`组件 - 并添加另一个指示它在你背包中的组件；更多关于这个的内容将在后续教程中介绍）

## 渲染 - 完整代码

如果你正确地输入了所有这些代码，你的`main.rs`现在看起来像这样：
```rust
use rltk::{GameState, Rltk, RGB};
use specs::prelude::*;
use std::cmp::{max, min};
use specs_derive::Component;

#[derive(Component)]
struct Position {
    x: i32,
    y: i32,
}

#[derive(Component)]
struct Renderable {
    glyph: rltk::FontCharType,
    fg: RGB,
    bg: RGB,
}

struct State {
   : World
}

impl GameState for State {
    fn tick(&mut self, ctx : &mut Rltk) {
        ctx.cls();
        let positions = self.ecs.read_storage::<Position>();
        let renderables = self.ecs.read_storage::<Renderable>();

        for (pos, render) in (&positions, &renderables).join() {
            ctx.set(pos.x, pos.y, render.fg, render.bg, render.glyph);
        }
    }
}

fn main() -> rltk::BError {
    use rltk::RltkBuilder;
    let context = RltkBuilder::simple80x50()
        .with_title("Roguelike 教程")
        .build()?;
    let mut gs = State {
        ecs: World::new()
    };
    gs.ecs.register::<Position>();
    gs.ecs.register::<Renderable>();

    gs.ecs
        .create_entity()
        .with(Position { x: 40, y: 25 })
        .with(Renderable {
            glyph: rltk::to_cp437('@'),
            fg: RGB::named(rltk::YELLOW),
            bg: RGB::named(rltk::BLACK),
        })
        .build();

    for i in 0..10 {
        gs.ecs
        .create_entity()
        .with(Position { x: i * 7, y: 20 })
        .with(Renderable {
            glyph: rltk::to_cp437('☺'),
            fg: RGB::named(rltk::RED),
            bg: RGB::named(rltk::BLACK),
        })
        .build();
    }

    rltk::main_loop(context, gs)
}
```
运行它（使用`cargo run`）将给你以下结果：

![截图](./c2-s1.png)

## 示例系统 - 随机移动

这个示例展示了ECS如何渲染一组不同的实体。你可以尝试创建不同的实体，这里有很多可能性！不幸的是，这看起来相当无聊 - 没有东西在移动！让我们稍微调整一下，使其看起来像一个射击场。

首先，我们将创建一个名为`LeftMover`的新组件。具有此组件的实体表示它们真的喜欢向左移动。组件定义非常简单；像这样的没有数据的组件被称为“标签组件”。我们将其与其他组件定义放在一起：
```rust
#[derive(Component)]
struct LeftMover {}
```
现在我们必须告诉ECS使用该类型。在我们的其他`register`调用中，我们添加：
```
gs.ecs.register::<LeftMover>();
```
现在，让我们只让红色的笑脸成为左移者。因此，它们的定义扩展为：
```rust
for i in 0..10 {
    gs.ecs
    .create_entity()
    .with(Position { x: i * 7, y: 20 })
    .with(Renderable {
        glyph: rltk::to_cp437('☺'),
        fg: RGB::named(rltk::RED),
        bg: RGB::named(rltk::BLACK),
    })
    .with(LeftMover{})
    .build();
}
```
注意我们添加了一行：`.with(LeftMover{})` - 这就是为这些实体添加另一个组件所需的全部内容（而不是黄色的`@`）。

现在让我们真正让它们移动。我们将定义我们的第一个*系统*。系统是一种将实体/组件逻辑组合在一起并独立运行的方式。有很多复杂的灵活性可用，但我们将保持简单。以下是我们的`LeftWalker`系统所需的所有内容：
```rust
struct LeftWalker {}

impl<'a> System<'a> for LeftWalker {
    type SystemData = (ReadStorage<'a, LeftMover>, 
                        WriteStorage<'a, Position>);

    fn run(&mut self, (lefty, mut pos) : Self::SystemData) {
        for (_lefty,pos) in (&lefty, &mut pos).join() {
            pos.x -= 1;
            if pos.x < 0 { pos.x = 79; }
        }
    }
}
```
这并不像我想要的那样好/简单，但当你理解它时，它是有意义的。让我们分步骤来看：

* `struct LeftWalker {}` 只定义了一个空结构 - 用于附加逻辑。
* `impl<'a> System<'a> for LeftWalker` 意味着我们为我们的`LeftWalker`结构实现了Specs的`System`特性。`'a`是*生命周期*指定符：系统表示它使用的组件必须存在足够长的时间以运行系统。现在，不值得太担心这个。[如果你感兴趣，Rust书籍可以稍微澄清一下](https://doc.rust-lang.org/book/ch10-00-generics.html)。
* `type SystemData` 定义了一个类型，告诉Specs系统需要什么。在这种情况下，需要读取`LeftMover`组件，并写入（因为它们会更新）`Position`组件。你可以在这里混合和匹配你需要的内容，正如我们将在后面的章节中看到的。
* `fn run` 是`impl System`所要求的实际特性实现。它接受自身和我们定义的`SystemData`。
* for循环是系统的简写，与我们渲染系统中的迭代相同：它将为每个具有`LeftMover`和`Position`的实体运行一次。注意我们在`LeftMover`变量名前加了一个下划线：我们实际上从未使用它，我们只是要求实体*有*一个。下划线告诉Rust“我们知道我们没用它，这不是一个错误！”并且在每次编译时停止警告我们。
* 循环的核心非常简单：我们从位置组件中减去一，如果它小于零，我们就回到屏幕的右边。

注意这与我们编写渲染代码的方式非常相似 - 但不是调用*进入* ECS，ECS系统是调用*进入*我们的函数/系统。判断使用哪一个可以是一个艰难的判断。如果你的系统*只需要* ECS中的数据，那么系统是放置它的正确地方。如果它还需要访问程序的其他部分，那么最好在系统外部实现 - 调用*进入*。

现在我们已经*编写*了我们的系统，我们需要能够使用它。我们将在我们的`State`中添加一个`run_systems`函数：
```rust
impl State {
    fn run_systems(&mut self) {
        let mut lw = LeftWalker{};
        lw.run_now(&self.ecs);
        self.ecs.maintain();
    }
}
```
这相对简单：

1. `impl State` 意味我们想要为`State`实现功能。
2. `fn run_systems(&mut self)` 意味我们正在定义一个*函数*，它需要*可变*（即允许更改）的* self *访问；这意味着它可以访问其`State`实例中的数据，使用`self.`关键字。
3. `let mut lw = LeftWalker{}` 创建了一个新的（可变的）`LeftWalker`系统实例。
4. `lw.run_now(&self.ecs)` 告诉系统运行，并告诉它如何找到ECS。
5. `self.ecs.maintain()` 告诉Specs，如果系统排队了任何更改，它们现在应该应用于世界。

最后，我们实际上想要运行我们的系统。在`tick`函数中，我们添加：
```rust
self.run_systems();
```
好处是这将运行我们注册到调度器中的*所有*系统；所以当我们添加更多系统时，我们不必担心调用它们（甚至不必担心按正确的顺序调用它们）。有时你仍然需要比调度器更多的访问权限；我们的渲染器不是一个系统，因为它需要来自RLTK的`Context`（我们将在未来的章节中改进这一点）。

所以你的代码现在看起来像这样：
```rust
use rltk::{GameState, Rltk, RGB};
use specs::prelude::*;
use std::cmp::{max, min};
use specs_derive::Component;

#[derive(Component)]
struct Position {
    x: i32,
    y: i32,
}

#[derive(Component)]
struct Renderable {
    glyph: rltk::FontCharType,
    fg: RGB,
    bg: RGB,
}

#[derive(Component)]
struct LeftMover {}
 
struct State {
    ecs: World,
}

impl GameState for State {
    fn tick(&mut self, ctx : &mut Rltk) {
        ctx.cls();

        self.run_systems();

        let positions = self.ecs.read_storage::<Position>();
        let renderables = self.ecs.read_storage::<Renderable>();

        for (pos, render) in (&positions, &renderables).join() {
            ctx.set(pos.x, pos.y, render.fg, render.bg, render.glyph);
        }
    }
}

struct LeftWalker {}

impl<'a> System<'a> for LeftWalker {
    type SystemData = (ReadStorage<'a, LeftMover>, 
                        WriteStorage<'a, Position>);

    fn run(&mut self, (lefty, mut pos) : Self::SystemData) {
        for (_lefty,pos) in (&lefty, &mut pos).join() {
            pos.x -= 1;
            if pos.x < 0 { pos.x = 79; }
        }
    }
}

impl State {
    fn run_systems(&mut self) {
        let mut lw = LeftWalker{};
        lw.run_now(&self.ecs);
        self.ecs.maintain();
    }
}

fn main() -> rltk::BError {
    use rltk::RltkBuilder;
    let context = RltkBuilder::simple80x50()
        .with_title("Roguelike 教程")
        .build()?;
    let mut gs = State {
        ecs: World::new()
    };
    gs.ecs.register::<Position>();
    gs.ecs.register::<Renderable>();
    gs.ecs.register::<LeftMover>();

    gs.ecs
        .create_entity()
        .with(Position { x: 40, y: 25 })
        .with(Renderable {
            glyph: rltk::to_cp437('@'),
            fg: RGB::named(rltk::YELLOW),
            bg: RGB::named(rltk::BLACK),
        })
        .build();

    for i in 0..10 {
        gs.ecs
        .create_entity()
        .with(Position { x: i * 7, y: 20 })
        .with(Renderable {
            glyph: rltk::to_cp437('☺'),
            fg: RGB::named(rltk::RED),
            bg: RGB::named(rltk::BLACK),
        })
        .with(LeftMover{})
        .build();
    }

    rltk::main_loop(context, gs)
}
```
如果你运行它（使用`cargo run`），红色的笑脸将向左飞驰，而`@`则在一旁观看。

![截图](./c2-s2.gif)

## 移动玩家

最后，让我们使`@`符号能够通过键盘控制移动。为了知道哪个实体是玩家，我们将创建一个新的标记组件：
```rust
#[derive(Component, Debug)]
struct Player {}
```
我们将其添加到注册中：
```rust
gs.ecs.register::<Player>();
```
并且我们将其添加到玩家的实体中：
```rust
gs.ecs
    .create_entity()
    .with(Position { x: 40, y: 25 })
    .with(Renderable {
        glyph: rltk::to_cp437('@'),
        fg: RGB::named(rltk::YELLOW),
        bg: RGB::named(rltk::BLACK),
    })
    .with(Player{})
    .build();
```
现在我们实现一个新的函数，`try_move_player`：
```rust
fn try_move_player(delta_x: i32, delta_y: i32, ecs: &mut World) {
    let mut positions = ecs.write_storage::<Position>();
    let mut players = ecs.write_storage::<Player>();

    for (_player, pos) in (&mut players, &mut positions).join() {
        pos.x = min(79 , max(0, pos.x + delta_x));
        pos.y = min(49, max(0, pos.y + delta_y));
    }
}
```
借助我们之前的经验，我们可以看到这个函数获取了`Player`和`Position`的写访问权限。它然后将两者连接起来，确保它只作用于同时具有这两种组件类型的实体 - 在这种情况下，只有玩家。它然后将`delta_x`加到`x`上，`delta_y`加到`y`上 - 并进行一些检查以确保你没有试图离开屏幕。

我们将添加第二个函数来读取RLTK提供的键盘信息：
```rust
fn player_input(gs: &mut State, ctx: &mut Rltk) {
    // 玩家移动
    match ctx.key {
        None => {} // 没有发生任何事情
        Some(key) => match key {
            VirtualKeyCode::Left => try_move_player(-1, 0, &mut gs.ecs),
            VirtualKeyCode::Right => try_move_player(1, 0, &mut gs.ecs),
            VirtualKeyCode::Up => try_move_player(0, -1, &mut gs.ecs),
            VirtualKeyCode::Down => try_move_player(0, 1, &mut gs.ecs),
            _ => {}
        },
    }
}
```
这里有相当多的新功能！上下文提供了关于键的信息 - 但用户可能没有按任何键！Rust为此提供了一个特性，称为`Option`类型。`Option`类型有两个可能的值：`None`（没有数据），或`Some(x)` - 表示这里有数据，保存在里面。

上下文提供了一个`key`变量。它是一个*枚举* - 也就是说，一个变量可以保存一组预定义值中的一个（在这种情况下，键盘上的键）。Rust的枚举非常强大，实际上可以保存值 - 但我们还没有使用它。

因此，要从`Option`中提取数据，我们需要*拆包*它。有一个名为`unwrap`的函数 - 但如果你在没有数据的情况下调用它，你的程序将会崩溃！所以我们将使用Rust的`match`命令来窥视内部。匹配是Rust的一个强大优势，我强烈推荐[关于它的Rust书籍章节](https://doc.rust-lang.org/book/ch06-00-enums.html)，或者如果你更喜欢通过示例学习，可以看[Rust by Example部分](https://doc.rust-lang.org/rust-by-example/flow_control/match.html)。

所以我们调用`match ctx.key` - Rust期望我们提供一个可能的匹配列表。对于`ctx.key`，只有两个可能的值：`Some`或`None`。`None => {}`行表示“匹配`ctx.key`没有数据的情况” - 并运行一个空块。`Some(key)`是另一个选项；有*一些*数据 - 我们将要求Rust将其给我们作为一个名为`key`的变量（你可以随意命名）。

然后，我们再次`match`，这次是键。我们为每个我们想要处理的可能性写一行：`VirtualKeyCode::Left => try_move_player(-1, , &mut gs.ecs)`表示如果`key`等于`VirtualKeyCode::Left`（`VirtualKeyCode`是枚举类型的名称），我们应该调用我们的`try_move_player`函数，参数为(-1, 0)。我们为所有四个方向重复这一点。`_ => {}`看起来相当奇怪；`_`意味着*其他任何东西*。所以我们告诉Rust，任何其他键码可以在这里忽略。Rust相当挑剔：如果你没有指定每个可能的枚举，它将给出编译器错误！通过包含默认值，我们不必输入每个可能的按键。

这个函数接受当前的游戏状态和上下文，查看上下文中的`key`变量，并在按下相关的移动键时调用适当的移动命令。最后，我们将其添加到`tick`中：
```rust
player_input(self, ctx);
```
如果你运行你的程序（使用`cargo run`），你现在有一个键盘控制的`@`符号，而笑脸则向左飞驰！

![截图](./c2-s3.gif)

## 第2章的最终代码

完整示例的源代码可以在`chapter-02-helloecs`中找到，可以直接运行。代码如下所示：

```rust
use rltk::{GameState, Rltk, RGB, VirtualKeyCode};
use specs::prelude::*;
use std::cmp::{max, min};
use specs_derive::Component;



#[derive(Component)]
struct Position {
    x: i32,
    y: i32,
}

#[derive(Component)]
struct Renderable {
    glyph: rltk::FontCharType,
    fg: RGB,
    bg: RGB,
}

#[derive(Component)]
struct LeftMover {}
 
#[derive(Component, Debug)]
struct Player {}

struct State {
    ecs: World
}

fn try_move_player(delta_x: i32, delta_y: i32, ecs: &mut World) {
    let mut positions = ecs.write_storage::<Position>();
    let mut players = ecs.write_storage::<Player>();

    for (_player, pos) in (&mut players, &mut positions).join() {
        pos.x = min(79 , max(0, pos.x + delta_x));
        pos.y = min(49, max(0, pos.y + delta_y));
    }
}

fn player_input(gs: &mut State, ctx: &mut Rltk) {
    // 玩家移动
    match ctx.key {
        None => {} // 无事发生
        Some(key) => match key {
            VirtualKeyCode::Left => try_move_player(-1, 0, &mut gs.ecs),
            VirtualKeyCode::Right => try_move_player(1, 0, &mut gs.ecs),
            VirtualKeyCode::Up => try_move_player(0, -1, &mut gs.ecs),
            VirtualKeyCode::Down => try_move_player(0, 1, &mut gs.ecs),
            _ => {}
        },
    }
}

impl GameState for State {
    fn tick(&mut self, ctx : &mut Rltk) {
        ctx.cls();

        player_input(self, ctx);
        self.run_systems();

        let positions = self.ecs.read_storage::<Position>();
        let renderables = self.ecs.read_storage::<Renderable>();

        for (pos, render) in (&positions, &renderables).join() {
            ctx.set(pos.x, pos.y, render.fg, render.bg, render.glyph);
        }
    }
}

struct LeftWalker {}

impl<'a> System<'a> for LeftWalker {
    type SystemData = (ReadStorage<'a, LeftMover>, 
                        WriteStorage<'a, Position>);

    fn run(&mut self, (lefty, mut pos) : Self::SystemData) {
        for (_lefty,pos) in (&lefty, &mut pos).join() {
            pos.x -= 1;
            if pos.x < 0 { pos.x = 79; }
        }
    }
}

impl State {
    fn run_systems(&mut self) {
        let mut lw = LeftWalker{};
        lw.run_now(&self.ecs);
        self.ecs.maintain();
    }
}

fn main() -> rltk::BError {
    use rltk::RltkBuilder;
    let context = RltkBuilder::simple80x50()
        .with_title("Roguelike 教程")
        .build()?;
    let mut gs = State {
        ecs: World::new()
    };
    gs.ecs.register::<Position>();
    gs.ecs.register::<Renderable>();
    gs.ecs.register::<LeftMover>();
    gs.ecs.register::<Player>();

    gs.ecs
        .create_entity()
        .with(Position { x: 40, y: 25 })
        .with(Renderable {
            glyph: rltk::to_cp437('@'),
            fg: RGB::named(rltk::YELLOW),
            bg: RGB::named(rltk::BLACK),
        })
        .with(Player{})
        .build();

    for i in 0..10 {
        gs.ecs
        .create_entity()
        .with(Position { x: i * 7, y: 20 })
        .with(Renderable {
            glyph: rltk::to_cp437('☺'),
            fg: RGB::named(rltk::RED),
            bg: RGB::named(rltk::BLACK),
        })
        .with(LeftMover{})
        .build();
    }

    rltk::main_loop(context, gs)
}
```

这一章节内容很多，但提供了一个非常坚实的构建基础。最棒的是：你现在已经超越了许多有抱负的开发者！你已经在屏幕上放置了实体，并且可以使用键盘进行移动。

**本章的源代码可以在这里找到 [这里](https://github.com/thebracket/rustrogueliketutorial/tree/master/chapter-02-helloecs)**

[在浏览器中用WebAssembly运行本章示例（需要WebGL2）](https://bfnightly.bracketproductions.com/rustbook/wasm/chapter-02-helloecs/)

---

版权所有 (C) 2019, Herbert Wolverson。

---
