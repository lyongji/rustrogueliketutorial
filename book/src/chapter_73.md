# Scanning The Systems

---

***About this tutorial***

*This tutorial is free and open source, and all code uses the MIT license - so you are free to do with it as you like. My hope is that you will enjoy the tutorial, and make great games!*

*If you enjoy this and would like me to keep writing, please consider supporting [my Patreon](https://www.patreon.com/blackfuture).*

[![Hands-On Rust](./beta-webBanner.jpg)](https://pragprog.com/titles/hwrust/hands-on-rust/)

---

Specs提供了非常出色的调度系统：它可以自动使用并发性，使您的游戏运行得非常快。那么我们为什么不用它呢？因为Web Assembly！WASM不支持与其它平台相同的线程方式，因此在WASM上编译的Specs应用程序在尝试调度系统时会失败。这对桌面应用程序来说是不公平的。此外，当前的`run_systems`函数看起来也不太美观：
```rust
fn run_systems(&mut self) {
    let mut mapindex = MapIndexingSystem{};
    mapindex.run_now(&self.ecs);
    let mut vis = VisibilitySystem{};
    vis.run_now(&self.ecs);
    ... // 许多更多的系统

```因此，本章的目标是构建一个接口来检测WASM，并在WASM不存在时回退到单线程调度器。如果WASM不存在，我们希望使用Specs调度器。我们还希望有一个更简洁的系统接口，并且不需要多次指定系统。

## 开始构建系统模块

首先，我们将创建一个新的目录：`src/systems`。这将包含独立的*系统*设置，但现在我们将使用它来开始构建一个可以在原生使用和WASM中的单线程调用器之间切换的设置。在新的`src/systems`目录中，创建一个名为`mod.rs`的文件。您现在可以将其留空。

> 警告：这里有一些中等难度的宏和配置。您可以自由使用它，并在需要时了解其工作原理。我们已经进行了73章，所以希望我们已经准备好了！

创建另一个新目录：`src/systems/dispatcher`。在该文件夹中，放置另一个空的`mod.rs`文件。

现在转到`main.rs`并添加一行`mod systems;`：这是为了将其包含在编译中（我们稍后会担心整洁的使用）。修改`src/systems/mod.rs`以包含`mod dispatcher` - 这只是确保它被编译。

## 泛化调度

根据我们的规格/想法，我们知道我们希望有一种通用的方式来运行系统 - 并且不关心哪个底层设置是活动的（从编程的角度来看）。这听起来像是*trait*的工作 - trait是Rust对多态性、继承（有点像）和接口的回答之一。将以下内容添加到`src/systems/dispatcher/mod.rs`：
```rust
use specs::prelude::World;
use super::*;

pub trait UnifiedDispatcher {
    fn run_now(&mut self, ecs : *mut World);
}
```
这指定了我们的`UnifiedDispatcher` trait将提供一个名为`run_now`的方法，该方法接受自身（用于状态）和ECS作为可变参数。

## 单线程调度

我们从简单的案例开始。添加一个新文件，`src/systems/dispatcher/single_thread.rs`。在`dispatcher/mod.rs`中，添加`mod single_thread; pub use single_thread::*;`。

在`single_thread.rs`中，我们需要一些库支持 - 所以从以下导入开始：
```rust
use super::super::*;
use super::UnifiedDispatcher;
use specs::prelude::*;
```
接下来，我们需要一个存储系统的地方。我们将以Specs风格传递可运行目标（见下文），但对于单线程执行，我们的目标是按它们被传递的顺序运行它们。与之前的`run_now`不同，我们将提前创建系统并仅迭代/执行它们 - 而不是每次都重新创建它们。让我们从结构定义开始：
```rust
pub struct SingleThreadedDispatcher<'a> {
    pub systems : Vec<Box<dyn RunNow<'a>>>
}
```
这里有一些关于生命周期的额外复杂性（我们将Specs的`RunNow` trait放在与结构相同的生命周期上），但这很简单：每个使用Specs系统功能的系统都实现了`RunNow` trait。因此，我们只需存储一个boxed（由于它们的大小不同，我们必须使用指针间接）`RunNow` trait的向量。

实际上执行它们要困难一些。以下方法有效：
```rust
impl<'a> UnifiedDispatcher for SingleThreadedDispatcher<'a> {
    fn run_now(&mut self, ecs : *mut World) {
        unsafe {
            for sys in self.systems.iter_mut() {
                sys.run_now(&*ecs);
            }
            crate::effects::run_effects_queue(&mut *ecs);
        }
    }
}
```
可能有一种更好的写法，但我一直遇到生命周期问题。`World`和系统往往都是`'static`的 - 它们存在于程序的生命周期中。说服Rust相信这一点让我头疼了一整天，直到我最终决定只使用`unsafe`并相信自己做得对！

请注意，我们将`World`作为*可变指针*而不是常规的可变引用。解引用可变指针本质上是危险的：Rust无法确定您是否不会违反生命周期保证。因此，`unsafe`块允许我们这样做。由于我们添加系统并且从不删除它们，并且在没有工作的`World`调用系统将会失败 - 我们可以这样做。（如果有人能给我一个安全的实现，我会很高兴使用它！）。该函数简单地遍历所有系统并执行它们 - 并在最后运行效果队列。

所以这是相对简单的部分。困难的部分是我们希望以Specs风格的调度调用方式 - 转换为有用的系统数据。我们还想以将适用于*任何*调度类型的方式来做这件事，并且我们不想多次声明我们的系统。在挠头一段时间后，我想出了一个生成函数的*宏*：

```rust
macro_rules! construct_dispatcher {
    (
        $(
            (
                $type:ident,
                $name:expr,
                $deps:expr
            )
        ),*
    ) => {
        fn new_dispatch() -> Box<dyn UnifiedDispatcher + 'static> {
            let mut dispatch = SingleThreadedDispatcher{
                systems : Vec::new()
            };

            $(
                dispatch.systems.push( Box::new( $type {} ));
            )*

            return Box::new(dispatch);
        }
    };
}
```

宏总是很难教授；如果你不小心，它们开始看起来像Perl。它们并不那么多的生成*代码*，而是生成*语法* - 然后在编译时“烹饪”成代码。查看Specs如何构建系统，每个系统都有一行像这样的代码：
```rust
.with(HelloWorld, "hello_world", &[])
```
所以我们为每个系统指定了三个数据片段：系统*类型*，一个名称，和一个字符串数组，指定依赖关系。对于单线程使用，我们实际上会忽略最后两个（并信任用户以正确的顺序输入系统）。将此映射到宏的参数部分，我们有：
```rust
macro_rules! construct_dispatcher {
    (
        $(
            (
                $type:ident,
                $name:expr,
                $deps:expr
            )
        ),*
    ) => {
```
`$(...),*`表示“重复此块的内容，0..*n*次。然后三个参数在括号内 - 使它们成为一个*元组*。`$type`是系统的类型 - 是一个*标识符*（而不是纯类型）。`$name`和`$deps`只是表达式。

在宏的主体中：
```ruyst
) => {
    fn new_dispatch() -> Box<dyn UnifiedDispatcher + 'static> {
        let mut dispatch = SingleThreadedDispatcher{
            systems : Vec::new()
        };

        $(
            dispatch.systems.push( Box::new( $type {} ));
        )*

        return Box::new(dispatch);
    }
};
```
我们定义了一个名为`new_dispatch`的新函数。它返回一个被框起来的、动态的、`'static`的`UnifiedDispatcher`。（宏在运行时不会定义函数！）。它首先创建一个带有空系统向量的`SingleThreadedDispatcher`的新实例。然后它遍历每个*元组*，将一个空系统推入向量中。最后，它返回结构 - 用框包裹着。

我们实际上并没有*制作*函数 - 我们只是教会了Rust如何做。所以在`src/systems/dispatch/mod.rs`中，我们需要定义它，以及它需要使用的系统：
```rust
#[macro_use]
mod single_thread;
pub use single_thread::*;

construct_dispatcher!(
    (MapIndexingSystem, "map_index", &[]),
    (VisibilitySystem, "visibility", &[]),
    (EncumbranceSystem, "encumbrance", &[]),
    (InitiativeSystem, "initiative", &[]),
    (TurnStatusSystem, "turnstatus", &[]),
    (QuipSystem, "quips", &[]),
    (AdjacentAI, "adjacent", &[]),
    (VisibleAI, "visible", &[]),
    (ApproachAI, "approach", &[]),
    (FleeAI, "flee", &[]),
    (ChaseAI, "chase", &[]),
    (DefaultMoveAI, "default_move", &[]),
    (MovementSystem, "movement", &[]),
    (TriggerSystem, "triggers", &[]),
    (MeleeCombatSystem, "melee", &[]),
    (RangedCombatSystem, "ranged", &[]),
    (ItemCollectionSystem, "pickup", &[]),
    (ItemEquipOnUse, "equip", &[]),
    (ItemUseSystem, "use", &[]),
    (SpellUseSystem, "spells", &[]),
    (ItemIdentificationSystem, "itemid", &[]),
    (ItemDropSystem, "drop", &[]),
    (ItemRemoveSystem, "remove", &[]),
    (HungerSystem, "hunger", &[]),
    (ParticleSpawnSystem, "particle_spawn", &[]),
    (LightingSystem, "lighting", &[])
);

pub fn new() -> Box<dyn UnifiedDispatcher + 'static> {
    new_dispatch()
}
```
这定义了一个`new()`函数，简单地传递调用`new_dispatch`的结果。对`construct_dispatcher!`的宏调用*制作*了这个函数 - 带有所有系统定义（我包含了所有）。

## 移动我们的系统

为了方便访问，我已经将所有的系统（仅Specs系统）移动到了新的`systems`模块中。您可以在[源代码](https://github.com/thebracket/rustrogueliketutorial/tree/master/chapter-73-systems)中看到实现细节。移动它们实际上非常简单：

1. 将系统（或系统文件夹）移动到`systems`中。
2. 从`main.rs`中删除寻找它的`mod`和`use`语句。
3. 在系统中，将`use super::`替换为`use crate::`。
4. 调整`src/systems/mod.rs`以编译（`mod`）并使用系统。

完成的`src/systems/mod.rs`如下所示。请注意，我们添加了一个易于访问的`new`函数来获取新的系统调度器：
```rust
mod dispatcher;
pub use dispatcher::UnifiedDispatcher;

// 系统导入
mod map_indexing_system;
use map_indexing_system::MapIndexingSystem;
mod visibility_system;
use visibility_system::VisibilitySystem;
mod ai;
use ai::*;
mod movement_system;
use movement_system::MovementSystem;
mod trigger_system;
use trigger_system::TriggerSystem;
mod melee_combat_system;
use melee_combat_system::MeleeCombatSystem;
mod ranged_combat_system;
use ranged_combat_system::RangedCombatSystem;
mod inventory_system;
use inventory_system::*;
mod hunger_system;
use hunger_system::HungerSystem;
pub mod particle_system;
use particle_system::ParticleSpawnSystem;
mod lighting_system;
use lighting_system::LightingSystem;

pub fn build() -> Box<dyn UnifiedDispatcher + 'static> {
    dispatcher::new()
}
```
`particle_system`有点不同，因为它有*其他*在其他地方使用的函数。您需要找到这些函数，并将它们的路径调整为`crate::systems::particle_system::`。

现在打开`main.rs`，并将新的调度器添加到`State`中：
```rust
pub struct State {
    pub ecs: World,
    mapgen_next_state : Option<RunState>,
    mapgen_history : Vec<Map>,
    mapgen_index : usize,
    mapgen_timer : f32,
    dispatcher : Box<dyn systems::UnifiedDispatcher + 'static>
}
```
`State`的初始化器更改为（在`fn main()`中）：
```rust
let mut gs = State {
    ecs: World::new(),
    mapgen_next_state : Some(RunState::MainMenu{ menu_selection: gui::MainMenuSelection::NewGame }),
    mapgen_index : 0,
    mapgen_history: Vec::new(),
    mapgen_timer: 0.0,
    dispatcher: systems::build()
};
```
我们现在可以*大大*简化我们的`run_systems`函数：
```rust
impl State {
    fn run_systems(&mut self) {
        self.dispatcher.run_now(&mut self.ecs);
        self.ecs.maintain();
    }
}
```
如果您现在`cargo run`项目，它将像以前一样运行：但是系统的执行现在更加直接。它可能甚至更快，因为我们没有在每次执行时重新制作系统。

## 多线程调度

通过单线程调度器，我们获得了一些清晰度和组织性，但我们还没有完全释放Specs的力量！创建一个新的文件，`src/systems/dispatcher/multi_thread.rs`。 

我们将首先创建一个新的结构来保存Specs调度器，并包含一些引用：
```rust
use super::UnifiedDispatcher;
use specs::prelude::*;

pub struct MultiThreadedDispatcher {
    pub dispatcher: specs::Dispatcher<'static, 'static>
}

我们还需要实现`run_now`（使用我们创建的`UnifiedDispatcher` trait）：

impl<'a> UnifiedDispatcher for MultiThreadedDispatcher {
    fn run_now(&mut self, ecs : *mut World) {
        unsafe {
            self.dispatcher.dispatch(&mut *ecs);
            crate::effects::run_effects_queue(&mut *ecs);
        }
    }
}
```
这相当简单：它只是告诉Specs“调度”我们存储的调度器，然后执行效果队列。

再次，我们需要一个宏来处理输入：
```rust
macro_rules! construct_dispatcher {
    (
        $(
            (
                $type:ident,
                $name:expr,
                $deps:expr
            )
        ),*
    ) => {
        fn new_dispatch() -> Box<dyn UnifiedDispatcher + 'static> {
            use specs::DispatcherBuilder;

            let dispatcher = DispatcherBuilder::new()
                $(
                    .with($type{}, $name, $deps)
                )*
                .build();

            let dispatch = MultiThreadedDispatcher{
                dispatcher : dispatcher
            };

            return Box::new(dispatch);
        }
    };
}
```
这与单线程版本完全相同的输入。这是故意的：它们被设计为可互换的。它还制作了一个`new_dispatch`函数，具有相同的返回类型。如果调用Specs的`DispatchBuilder::new`，然后遍历宏参数为每组系统数据添加`.with(...)`行。最后，它调用`.build`并将其存储在`MultiThreadedDispatcher`结构中 - 并以框的形式返回自己。

这实际上相当简单，但留下了一个大问题：我们如何知道我们要使用哪一个？我们几乎只在WASM32中使用单线程版本，否则我们希望利用Specs的线程和效率。因此，我们修改`src/systems/dispatcher/mod.rs`以包含*条件编译*：

```rust
#[cfg(target_arch = "wasm32")]
#[macro_use]
mod single_thread;

#[cfg(not(target_arch = "wasm32"))]
#[macro_use]
mod multi_thread;

#[cfg(target_arch = "wasm32")]
pub use single_thread::*;

#[cfg(not(target_arch = "wasm32"))]
pub use multi_thread::*;

use specs::prelude::World;
use super::*;

pub trait UnifiedDispatcher {
    fn run_now(&mut self, ecs : *mut World);
}

construct_dispatcher!(
    (MapIndexingSystem, "map_index", &[]),
    (VisibilitySystem, "visibility", &[]),
    (EncumbranceSystem, "encumbrance", &[]),
    (InitiativeSystem, "initiative", &[]),
    (TurnStatusSystem, "turnstatus", &[]),
    (QuipSystem, "quips", &[]),
    (AdjacentAI, "adjacent", &[]),
    (VisibleAI, "visible", &[]),
    (ApproachAI, "approach", &[]),
    (FleeAI, "flee", &[]),
    (ChaseAI, "chase", &[]),
    (DefaultMoveAI, "default_move", &[]),
    (MovementSystem, "movement", &[]),
    (TriggerSystem, "triggers", &[]),
    (MeleeCombatSystem, "melee", &[]),
    (RangedCombatSystem, "ranged", &[]),
    (ItemCollectionSystem, "pickup", &[]),
    (ItemEquipOnUse, "equip", &[]),
    (ItemUseSystem, "use", &[]),
    (SpellUseSystem, "spells", &[]),
    (ItemIdentificationSystem, "itemid", &[]),
    (ItemDropSystem, "drop", &[]),
    (ItemRemoveSystem, "remove", &[]),
    (HungerSystem, "hunger", &[]),
    (ParticleSpawnSystem, "particle_spawn", &[]),
    (LightingSystem, "lighting", &[])
);

pub fn new() -> Box<dyn UnifiedDispatcher + 'static> {
    new_dispatch()
}
```

单线程导入由条件编译标记 precede：
```rust
#[cfg(target_arch = "wasm32")]
```
这表示“仅当目标架构为 `wasm32` 时才编译 accompanying 行”。

同样，多线程版本有相反的标记：
```rust
#[cfg(not(target_arch = "wasm32"))]
```
我们在调度器的 `mod` 和 `use` 语句中都重复了这些。所以如果你在运行WASM，你会得到 `#[macro_use] mod single_thread; use single_thread::*`。如果你在本地运行，你会得到 `#[macro_use] mod multi_thread; use multi_thread::*`。Rust不会编译没有 `mod` 语句包含的模块：所以我们只构建*一个*调度器策略。由于我们正在使用它，宏 `construct_dispatcher!` 被放置在我们的本地（`systems::dispatcher`）命名空间中 - 所以我们对宏的调用运行我们连接的无论哪个版本。

这是*编译时调度*，是一个非常强大的设置。RLTK在内部大量使用它来自定义各种硬件后端。

所以如果你现在 `cargo run` 你的项目 - 游戏将使用Specs调度器运行。如果你启动一个系统监视器，你可以看到它正在使用多个线程！

## 所以为什么当我们添加线程时，它没有爆炸？

有一个常见的习惯说法是“我有一个bug。我添加了8个线程，现在我有8个bug。” 这很可能，但Rust努力推广“无畏的并发”。Rust本身保护 against *数据竞争* - 不允许两个系统同时访问/更改相同的数据，这是某些其他语言中的常见bug来源。然而，它并不保护 against 逻辑问题 - 例如，一个系统需要来自前一个系统的信息，只有那个其他系统还没有运行。

Specs代表你进一步提高了安全性。这是我们地图索引系统的 `SystemData` 定义：

```rust
WriteExpect<'a, Map>,
ReadStorage<'a, Position>,
ReadStorage<'a, BlocksTile>,
ReadStorage<'a, TileSize>,
Entities<'a>
```

记得我们是如何精心指定我们想要对资源和组件进行`写入`还是`读取`访问的吗？Specs实际上使用这一点来进行调度。当它构建调度器时，它会寻找`写入`访问 - 并确保没有两个系统可以同时写入相同的数据。它还确保在写入被锁定时不会读取数据。*然而*，系统可以并发*读取*数据。因此，在这种情况下，Specs保证任何需要*读取*地图的系统都会等待`MapIndexingSystem`完成*写入*。

这有助于构建依赖链 - 并逻辑地排序系统。例如：

* `MapIndexingSystem` 写入地图，并读取 `Position`，`BlocksTile` 和 `TileSize`。
* `VisibilitySystem` 写入地图，`Viewshed`，`Hidden` 和 `RandomNumberGenerator`。它读取 `Position`，`Name` 和 `BlocksVisibility`。
* `EncumbranceSystem` 写入 `EquipmentChanged`，`Pools`，`Attributes` 并从 `Item`，`InBackpack`，`Equipped`，`Entity`，`AttributeBonus`，`StatusEffect` 和 `Slow` 读取。
* `InitiativeSystem` 写入 `Initiative`，`MyTurn`，`RandomNumberGenerator`，`RunState`，`Duration`，`EquipmentChanged` 并从 `Position`，`Attributes`，`Entity`，`Point`，`Pools`，`StatusEffect`，`DamageOverTime` 读取。
* `TurnStatusSystem` 写入 `MyTurn`，并从 `Confusion`，`RunState`，`StatusEffect` 读取。

我们可以继续列举所有系统，但这是一个很好的示例。从中我们可以确定：

1. `MapIndexingSystem` 锁定地图，因此无法与 `VisibilitySystem` 并发运行。由于 `MapIndexingSystem` 是首先定义的，它将首先运行。
2. `VisibilitySystem` 锁定地图，视野，隐藏和随机数生成器。因此，它必须等待视野系统完成。
3. `EncumbranceSystem` 锁定随机数生成器，因此必须等待视野系统完成。
4. `InitiativeSystem` 也锁定随机数生成器，因此必须等待负载系统完成。
5. `TurnStatusSystem` 锁定 `MyTurn` - 而 `InitiativeSystem` 也锁定它。因此，它必须等待该系统完成。

换句话说：我们还没有真正实现太多并行处理！通过使用Specs的调度器，我们从效率增益中受益 - 因此我们获得了一些好处（在我的本地计算机的调试模式下，它确实感觉更快了！）。

## 量化“感觉更快”

让我们在屏幕上添加一个帧率指示器，这样我们就*知道*我们所做的是否有帮助。我们将使其作为编译时标志（就像地图调试显示一样）可选。在`main.rs`中的地图标志旁边添加：

```rust
const SHOW_FPS : bool = true;
```

在 `tick` 的最后，提交渲染批次后 - 添加以下行：
```rust
if SHOW_FPS {
    ctx.print(1, 59, &format!("FPS: {}", ctx.fps));
}
```
如果你现在 `cargo run`，你将在屏幕底部看到一个FPS计数器。在我的系统上，它基本上总是显示 `60`。如果你想看看它*能*有多快，我们需要关闭 `vsync`。这是对 `main` 函数中的 `RLTK` 初始化的一个简单更改：
```rust
let mut context = RltkBuilder::simple(80, 60)
    .with_title("Roguelike 教程")
    .with_font("vga8x16.png", 8, 16)
    .with_sparse_console(80, 30, "vga8x16.png")
    .with_vsync(false)
    .build();
```
现在它显示我的帧率为200区域！

## 并行化 RNG

`RandomNumberGenerator` 作为可写资源是我们在系统中无法访问并发的最大原因。它被到处使用，系统必须等待另一个生成随机数。我们*可以*简单地在需要随机数时使用本地RNG - 但这样我们就失去了设置*随机种子*的能力（更多内容将在未来的章节中介绍！）。相反，我们将创建一个*全局*随机数生成器并用互斥锁保护它 - 这样程序可以从同一个源获取随机数。

让我们创建一个新的文件，`src/rng.rs`。在 `main.rs` 中添加 `pub mod rng;`，我们将使用 `lazy_static` 构建随机数生成器包装器。我们将使用 `Mutex` 来保护它，以便可以安全地从多个线程使用：

```rust
use std::sync::Mutex;
use rltk::prelude::*;

lazy_static! {
    static ref RNG: Mutex<RandomNumberGenerator> =
        Mutex::new(RandomNumberGenerator::new());
}

pub fn reseed(seed: u64) {
    *RNG.lock().unwrap() = RandomNumberGenerator::seeded(seed);
}

pub fn roll_dice(n:i32, die_type: i32) -> i32 {
    RNG.lock().unwrap().roll_dice(n, die_type)
}

pub fn range(min: i32, max: i32) -> i32
{
    RNG.lock().unwrap().range(min, max)
}
```

现在进入`main.rs`的`main`函数，并删除以下行：
```rust
gs.ecs.insert(rltk::RandomNumberGenerator::new());
```
现在，每当尝试访问RNG资源时，游戏都会崩溃！因此，我们需要搜索整个程序，找到所有使用RNG的地方，并将它们全部替换为`crate::rng::roll_dice`或`crate::rng::range`。否则，语法保持不变。这是一个*很大的*改变，主要是相同的代码。查看[源代码](https://github.com/thebracket/rustrogueliketutorial/tree/master/chapter-73-systems)以获取工作版本。（一个很好的副作用是，我们不再在地图构建器中到处传递`rng`；它们变得更加简洁！）

随着这个依赖关系的解决，我们现在能够以更多的并发性进行操作。您的FPS应该已经提高，如果您观察进程监视器，我们会更加多线程化。

## 总结

本章大大清理了我们的系统处理。它更快、更苗条、更好看 - 以牺牲了一些`unsafe`块（管理得当）和一个糟糕的宏为代价。我们还使`RNG`成为全局变量，但安全地将其包装在互斥锁中。结果？在我的系统上，游戏在发布模式下以1,300 FPS运行，并现在受益于Specs惊人的线程能力。即使在单线程模式下，它也能以可观的1,100 FPS运行（在我的系统上：一个core i7）。

---

**本章的源代码可以在这里找到**[这里](https://github.com/thebracket/rustrogueliketutorial/tree/master/chapter-73-systems)


[在浏览器中运行本章的示例，使用Web Assembly（需要WebGL2）](https://bfnightly.bracketproductions.com/rustbook/wasm/chapter-73-systems)
---

版权所有（C）2019，Herbert Wolverson。

---
