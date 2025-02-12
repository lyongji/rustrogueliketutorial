# Improved Logging and Counting Achievement

---

***About this tutorial***

*This tutorial is free and open source, and all code uses the MIT license - so you are free to do with it as you like. My hope is that you will enjoy the tutorial, and make great games!*

*If you enjoy this and would like me to keep writing, please consider supporting [my Patreon](https://www.patreon.com/blackfuture).*

[![Hands-On Rust](./beta-webBanner.jpg)](https://pragprog.com/titles/hwrust/hands-on-rust/)

---

大多数Roguelike游戏都非常重视游戏日志。日志会在游戏结束时被整合到*morgue file*中（详细描述了你的游戏过程），它用于显示游戏世界中的情况，对硬核玩家来说是无价之宝。我们一直在使用一个相当简单的日志设置（多亏了Mark McCaskey的辛勤工作，它不再令人痛苦地缓慢）。在本章中，我们将构建一个良好的日志系统 - 并将其作为成就和进度跟踪系统的基础。我们还将改善日志的GUI。

目前，我们通过直接调用数据结构来添加到游戏日志中。它看起来像这样：
```rust
log.entries.push(format!("{} hits {}, for {} hp.", &name.name, &target_name.name, damage));
```
这不是一种很好的方法：它要求你必须*直接*访问日志，不提供任何格式化功能，并且要求系统了解日志的内部工作原理。我们也没有将日志作为保存游戏的一部分进行序列化（以及在加载时进行反序列化）。最后，还有很多我们没有记录但可能需要记录的事情；这是因为将日志作为资源包含进来相当烦人。就像效果系统一样，它应该是无缝的、简单的，并且在多线程环境中是安全的（如果你不使用WASM的话）。

本章将纠正这些缺陷。

## 构建API

我们首先将创建一个新的目录，`src/gamelog`。我们将`src/gamelog.rs`的内容移动到其中，并将文件重命名为`mod.rs` - 换句话说，我们创建了一个新的模块。这应该继续正常工作 - 模块的名字并没有改变。

将以下内容追加到`mod.rs`：
```rust
pub struct LogFragment {
    pub color : RGB,
    pub text : String
}
```
新的`LogFragment`类型将存储日志条目的*片段*。每个片段可以包含一些文本和颜色，允许创建丰富、多彩的日志条目。一组片段可以组成一行日志。

接下来，我们将创建另一个新文件 - 这次命名为`src/gamelog/logstore.rs`。将以下内容粘贴到其中：

```rust
use std::sync::Mutex;
use super::LogFragment;
use rltk::prelude::*;

lazy_static! {
    static ref LOG : Mutex<Vec<Vec<LogFragment>>> = Mutex::new(Vec::new());
}

pub fn append_fragment(fragment : LogFragment) {
    LOG.lock().unwrap().push(vec![fragment]);
}

pub fn append_entry(fragments : Vec<LogFragment>) {
    LOG.lock().unwrap().push(fragments);
}

pub fn clear_log() {
    LOG.lock().unwrap().clear();
}

pub fn log_display() -> TextBuilder {
    let mut buf = TextBuilder::empty();

    LOG.lock().unwrap().iter().rev().take(12).for_each(|log| {
        log.iter().for_each(|frag| {
            buf.fg(frag.color);
            buf.line_wrap(&frag.text);
        });
        buf.ln();
    });

    buf
}
```

这里需要消化 quite a bit of information:

* 在核心部分，我们使用 `lazy_static` 来定义一个 *全局* 日志条目存储。它是一个向量 of vectors，这次组成 fragments。所以外层向量是日志的 *行*，内层向量构成了日志的 *片段*。它由 `Mutex` 保护，使其在多线程环境中使用安全。
* `append_fragment` 锁定日志，并将单个片段作为新行追加。
* `append_entry` 锁定日志，并追加一个片段向量（新行）。
* `clear_log`如其名所述：清空日志。
* `log_display` 构建一个 RLTK `TextBuilder` 对象，这是一种安全的构建大量文本以供渲染的方式，考虑了诸如 line wrapping 等因素。它取12个条目，因为那是我能显示的最大日志。

在 `mod.rs` 中，添加以下三行代码来处理使用模块和导出其部分：
```rust
mod logstore;
use logstore::*;
pub use logstore::{clear_log, log_display};
```
这使我们大大简化了显示日志的过程。打开 `gui.rs`，找到日志绘制代码（在示例中是第248行）。用以下代码替换日志绘制：

```rust
// Draw the log
let mut block = TextBlock::new(1, 46, 79, 58);
block.print(&gamelog::log_display());
block.render(&mut rltk::BACKEND_INTERNAL.lock().consoles[0].console);
```

这指定了日志文本块的确切位置，作为一个RLTK的`TextBlock`对象。然后将`log_display()`的结果打印到该块上，并在控制台零（正在使用的控制台）上进行渲染。

现在，我们需要一种方法向日志中添加文本。构建器模式是一个很自然的选择；在大多数情况下，我们是逐步构建日志条目的细节。创建另一个文件，命名为`src/gamelog/builder.rs`：

```rust
use rltk::prelude::*;
use super::{LogFragment, append_entry};

pub struct Logger {
    current_color : RGB,
    fragments : Vec<LogFragment>
}

impl Logger {
    pub fn new() -> Self {
        Logger{
            current_color : RGB::named(rltk::WHITE),
            fragments : Vec::new()
        }
    }

    pub fn color(mut self, color: (u8, u8, u8)) -> Self {
        self.current_color = RGB::named(color);
        self
    }

    pub fn append<T: ToString>(mut self, text : T) -> Self {
        self.fragments.push(
            LogFragment{
                color : self.current_color,
                text : text.to_string()
            }
        );
        self
    }

    pub fn log(self) {
        append_entry(self.fragments)
    }
}
```

这定义了一个新的类型 `Logger`。它跟踪当前的输出颜色和组成日志条目的片段列表。`new` 函数创建一个新的实例，而 `log` 函数将日志条目提交到全局互斥变量。你可以调用 `color` 来更改当前写入颜色，`append` 用于添加字符串（我们使用 `ToString`，因此不再需要到处使用麻烦的 `to_string()` 调用！）

在 `gamelog/mod.rs` 中，我们需要使用并导出这个模块：
```rust
mod builder;
pub use builder::*;
```
要看到它的效果，打开 `main.rs` 并找到我们向资源列表添加新日志文件的位置，以及 "Welcome to Rusty Roguelike" 这一行。现在，我们将保留原始内容 - 并使用新的设置来开始日志：
```rust
gs.ecs.insert(gamelog::GameLog{ entries : vec!["Welcome to Rusty Roguelike".to_string()] });
gamelog::clear_log();
gamelog::Logger::new()
    .append("Welcome to")
    .color(rltk::CYAN)
    .append("Rusty Roguelike")
    .log();
```
这很干净：无需获取资源，文本/颜色附加易于阅读！如果你现在 `cargo run`，你将看到一个彩色显示的单个日志条目：

![c71-s1.jpg](c71-s1.jpg)

## 强制使用API

现在是时候做一些破坏性的事情了。在`src/gamelog/mod.rs`中，**删除**以下内容：
```rust
pub struct GameLog {
    pub entries : Vec<String>
}
```
如果你使用的是IDE，你的项目现在可能都是红色的错误提示！我们刚刚删除了旧的日志方式 - 所以*每个*对旧日志的引用现在都是编译错误。这是可以接受的，因为我们要过渡到新的系统。

从`main.rs`开始，我们可以删除对旧日志的引用。删除新的日志行，以及之前添加的所有日志信息。找到`generate_world_map`函数，并将初始日志清除/设置移动到这里：
```rust
fn generate_world_map(&mut self, new_depth : i32, offset: i32) {
    self.mapgen_index = 0;
    self.mapgen_timer = 0.0;
    self.mapgen_history.clear();
    let map_building_info = map::level_transition(&mut self.ecs, new_depth, offset);
    if let Some(history) = map_building_info {
        self.mapgen_history = history;
    } else {
        map::thaw_level_entities(&mut self.ecs);
    }

    gamelog::clear_log();
    gamelog::Logger::new()
        .append("Welcome to")
        .color(rltk::CYAN)
        .append("Rusty Roguelike")
        .log();
}
```
如果你现在`cargo build`项目，你将会有很多错误。我们需要逐步更新*所有*日志引用以使用新系统。

## 使用API

打开`src/inventory_system/collection_system.rs`。在`use`语句中，删除对`gamelog::GameLog`的引用（它已经不存在了）。删除寻找游戏日志的`WriteExpect`（以及元组中的匹配`mut gamelog`）。将`gamelog.push`语句替换为：
```rust
crate::gamelog::Logger::new()
    .append("你捡起了")
    .color(rltk::CYAN)
    .append(
        super::obfuscate_name(pickup.item, &names, &magic_items, &obfuscated_names, &dm)
    )
    .log();
```
你需要对`src/inventory_system/drop_system.rs`进行基本相同的更改。删除导入和资源后，日志消息系统变为：
```rust
if entity == *player_entity {
    crate::gamelog::Logger::new()
        .append("你丢弃了")
        .color(rltk::CYAN)
        .append(
            super::obfuscate_name(to_drop.item, &names, &magic_items, &obfuscated_names, &dm)
        )
        .log();
}
```
同样，在`src/inventory_system/equip_use.rs`中，删除`gamelog`。同时删除`log_entries`变量和循环追加它的代码。有很多日志条目需要清理：

```rust
// Cursed item unequipping
crate::gamelog::Logger::new()
    .append("You cannot unequip")
    .color(rltk::CYAN)
    .append(&name.name)
    .color(rltk::WHITE)
    .append("- it is cursed!")
    .log();
can_equip = false;
...
// Unequipped item
crate::gamelog::Logger::new()
    .append("You unequip")
    .color(rltk::CYAN)
    .append(&name.name)
    .log();
...
// Wield
crate::gamelog::Logger::new()
    .append("You equip")
    .color(rltk::CYAN)
    .append(&names.get(useitem.item).unwrap().name)
    .log();
```

同样，`src/hunger_system.rs`文件也需要更新。再次移除`gamelog`，并用新系统的等效代码替换`log.push`行。
```rust
crate::gamelog::Logger::new()
    .color(rltk::ORANGE)
    .append("你不再吃得很好")
    .log();
...
crate::gamelog::Logger::new()
    .color(rltk::ORANGE)
    .append("你饿了")
    .log();
...
crate::gamelog::Logger::new()
    .color(rltk::RED)
    .append("你正在挨饿！")
    .log();
...
crate::gamelog::Logger::new()
    .color(rltk::RED)
    .append("你的饥饿感变得痛苦！你受到了1点伤害。")
    .log();
```
`src/trigger_system.rs`也需要同样的处理。再次移除`gamelog`，并用新系统的日志条目替换。

```rust
crate::gamelog::Logger::new()
    .color(rltk::RED)
    .append(&name.name)
    .color(rltk::WHITE)
    .append("triggers!")
    .log();
```

`src/ai/quipping.rs` 需要进行相同的处理。移除 `gamelog`，并用新的日志记录方式替换：
```rust
crate::gamelog::Logger::new()
    .color(rltk::YELLOW)
    .append(&name.name)
    .color(rltk::WHITE)
    .append("说")
    .color(rltk::CYAN)
    .append(&quip.available[quip_index])
    .log();
```
`src/ai/encumbrance_system.rs` 也有相同的更改。再次移除 `gamelog` - 并用新的日志记录方式替换：
```rust
crate::gamelog::Logger::new()
    .color(rltk::ORANGE)
    .append("你负载过重，受到了 Initiative 罚值。")
    .log();
```
`src/effects/damage.rs` 的日志记录方式略有不同，但我们可以现在统一机制。首先移除 `use crate::gamelog::GameLog;` 这一行。然后替换所有 `log_entries.push` 行为使用新的 `Logger` 接口的行：
```rust
crate::gamelog::Logger::new()
    .color(rltk::MAGENTA)
    .append("恭喜，你现在是等级")
    .append(format!("{}", player_stats.level))
    .log();
...
crate::gamelog::Logger::new().color(rltk::GREEN).append("你感觉更强壮了！").log();
...
crate::gamelog::Logger::new().color(rltk::GREEN).append("你感觉更健康了！").log();
...
crate::gamelog::Logger::new().color(rltk::GREEN).append("你感觉更敏捷了！").log();
...
crate::gamelog::Logger::new().color(rltk::GREEN).append("你感觉更聪明了！").log();
```
在 `src/effects/trigger.rs` 中也是相同的处理；移除 `GameLog`，并用新的日志记录代码替换：
```rust
crate::gamelog::Logger::new()
    .color(rltk::CYAN)
    .append(&ecs.read_storage::<Name>().get(item).unwrap().name)
    .color(rltk::WHITE)
    .append("没有剩余的充能了！")
    .log();
...
crate::gamelog::Logger::new()
    .append("你吃了")
    .color(rltk::CYAN)
    .append(&names.get(entity).unwrap().name)
    .log();
...
crate::gamelog::Logger::new().append("地图向你展示了！").log();
...
crate::gamelog::Logger::new().append("你已经在城镇里了，所以卷轴没有效果。").log();
...
crate::gamelog::Logger::new().append("你被传送回了城镇！").log();
...
```
再次，`src/player.rs` 也是相同的处理。移除 `GameLog`，并用新的构建器语法替换日志条目：
```rust
crate::gamelog::Logger::new()
    .append("你朝")
    .color(rltk::CYAN)
    .append(&name.name)
    .log();
...
crate::gamelog::Logger::new().append("这里没有下行的路。").log();
...
crate::gamelog::Logger::new().append("这里没有上行的路。").log();
...
None => crate::gamelog::Logger::new().append("这里没有可以捡起来的东西。").log(),
...
crate::gamelog::Logger::new().append("你没有足够的法力来施展那个技能！").log();
```
在`visibility_system.rs`中也是相同的处理。再次删除`GameLog`，并用新的日志记录方式替换：
```rust
crate::gamelog::Logger::new()
    .append("你发现了：")
    .color(rltk::RED)
    .append(&name.name)
    .log();
```
在`melee_combat_system.rs`中也需要进行相同的更改：不再使用`GameLog`，并更新文本输出以使用新的构建系统：
```rust
crate::gamelog::Logger::new()
    .color(rltk::YELLOW)
    .append(&name.name)
    .color(rltk::WHITE)
    .append("击中了")
    .color(rltk::YELLOW)
    .append(&target_name.name)
    .color(rltk::WHITE)
    .append("造成")
    .color(rltk::RED)
    .append(format!("{}", damage))
    .color(rltk::WHITE)
    .append("点伤害。")
    .log();
...
crate::gamelog::Logger::new()
    .color(rltk::CYAN)
    .append(&name.name)
    .color(rltk::WHITE)
    .append("考虑攻击")
    .color(rltk::CYAN)
    .append(&target_name.name)
    .color(rltk::WHITE)
    .append("但判断时机错误！")
    .log();
...
crate::gamelog::Logger::new()
    .color(rltk::CYAN)
    .append(&name.name)
    .color(rltk::WHITE)
    .append("攻击")
    .color(rltk::CYAN)
    .append(&target_name.name)
    .color(rltk::WHITE)
    .append("但没有击中。")
    .log();
```
现在你应该对所需的更改有相当好的理解了。如果你查看[源代码](https://github.com/thebracket/rustrogueliketutorial/tree/master/chapter-71-logging)，我已经对所有其他`gamelog`的实例进行了更改。

一旦你完成了所有的更改，你可以`cargo run`你的游戏 - 并看到一个明亮彩色的日志：

![c71-s2.jpg](c71-s2.jpg)

## 使日志记录任务更简单

在遍历代码时，更新日志条目 - 出现了很多共同点。最好强制执行一些样式一致性（并减少所需的输入量）。我们将在日志构建器中添加一些方法（在`src/gamelog/builder.rs`中）来帮助：
```rust
pub fn npc_name<T: ToString>(mut self, text : T) -> Self {
    self.fragments.push(
        LogFragment{
            color : RGB::named(rltk::YELLOW),
            text : text.to_string()
        }
    );
    self
}

pub fn item_name<T: ToString>(mut self, text : T) -> Self {
    self.fragments.push(
        LogFragment{
            color : RGB::named(rltk::CYAN),
            text : text.to_string()
        }
    );
    self
}

pub fn damage(mut self, damage: i32) -> Self {
    self.fragments.push(
        LogFragment{
            color : RGB::named(rltk::RED),
            text : format!("{}", damage).to_string()
        }
    );
    self
}
```
现在我们可以再次遍历一些日志条目代码，使用更简单的语法。例如，在`src\ai\quipping.rs`中，我们可以替换：
```rust
crate::gamelog::Logger::new()
    .color(rltk::YELLOW)
    .append(&name.name)
    .color(rltk::WHITE)
    .append("说")
    .color(rltk::CYAN)
    .append(&quip.available[quip_index])
    .log();
```
使用：
```rust
crate::gamelog::Logger::new()
    .npc_name(&name.name)
    .append("说")
    .npc_name(&quip.available[quip_index])
    .log();
```
或者，在`melee_combat_system.rs`中，可以大大缩短伤害公告：
```rust
crate::gamelog::Logger::new()
    .npc_name(&name.name)
    .append("击中了")
    .npc_name(&target_name.name)
    .append("造成")
    .damage(damage)
    .append("点伤害。")
    .log();
```
再次，我已经遍历了项目源代码并应用了这些增强功能。

## 保存和加载日志

为了使保存和加载日志更简单，我们将向`gamelog/logstore.rs`添加两个辅助函数：
```rust
pub fn clone_log() -> Vec<Vec<crate::gamelog::LogFragment>> {
    LOG.lock().unwrap().clone()
}

pub fn restore_log(log : &mut Vec<Vec<crate::gamelog::LogFragment>>) {
    LOG.lock().unwrap().clear();
    LOG.lock().unwrap().append(log);
}
```

第一个函数提供了一个日志的克隆副本。第二个函数清空日志，并追加一个新的日志。你需要打开 `gamelog/mod.rs` 并将这些函数添加到导出的函数列表中：
```rust
pub use logstore::{clear_log, log_display, clone_log, restore_log};
```
在这里，我们还需要为 `LogFragment` 结构体添加一些派生属性：
```rust
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize, Clone)]
pub struct LogFragment {
    pub color : RGB,
    pub text : String
}
```
现在打开 `components.rs`，并修改 `DMSerializationHelper` 结构体以包含日志：
```rust
#[derive(Component, Serialize, Deserialize, Clone)]
pub struct DMSerializationHelper {
    pub map : super::map::MasterDungeonMap,
    pub log : Vec<Vec<crate::gamelog::LogFragment>>
}
```
打开 `saveload_system.rs`，当序列化地图时，我们将包括日志：
```rust
let savehelper2 = ecs
    .create_entity()
    .with(DMSerializationHelper{ map : dungeon_master, log: crate::gamelog::clone_log() })
    .marked::<SimpleMarker<SerializeMe>>()
    .build();
```
当反序列化地图时，我们还将恢复日志：
```rust
for (e,h) in (&entities, &helper2).join() {
    let mut dungeonmaster = ecs.write_resource::<super::map::MasterDungeonMap>();
    *dungeonmaster = h.map.clone();
    deleteme2 = Some(e);
    crate::gamelog::restore_log(&mut h.log.clone());
}
```
这就是保存/加载日志的全部内容：它与 Serde 配合得很好（在完整的 JSON 上可能有点慢），但效果很好。

## 计数事件

作为实现成就的一步，我们需要能够计数相关事件。创建一个新文件，`src/gamelog/events.rs`，并粘贴以下内容：
```rust
use std::collections::HashMap;
use std::sync::Mutex;

lazy_static! {
    static ref EVENTS : Mutex<HashMap<String, i32>> = Mutex::new(HashMap::new());
}

pub fn clear_events() {
    EVENTS.lock().unwrap().clear();
}

pub fn record_event<T: ToString>(event: T, n : i32) {
    let event_name = event.to_string();
    let mut events_lock = EVENTS.lock();
    let mut events = events_lock.as_mut().unwrap();
    if let Some(e) = events.get_mut(&event_name) {
        *e += n;
    } else {
        events.insert(event_name, n);
    }
}

pub fn get_event_count<T: ToString>(event: T) -> i32 {
    let event_name = event.to_string();
    let events_lock = EVENTS.lock();
    let events = events_lock.unwrap();
    if let Some(e) = events.get(&event_name) {
        *e
    } else {
        0
    }
}
```
这与我们存储日志的方式类似：它是一个“懒静态”，有一个互斥锁安全包装。内部是一个`HashMap`，按事件名称索引并包含一个计数器。`record_event`将事件添加到运行总数（如果不存在则创建一个新的）。`get_event_count`返回0或命名计数器的总数。

在`main.rs`中，找到`RunState::AwaitingInput`的主循环处理程序 - 我们将其扩展为计数玩家存活的回合数：
```rust
RunState::AwaitingInput => {
    newrunstate = player_input(self, ctx);
    if newrunstate != RunState::AwaitingInput {
        crate::gamelog::record_event("Turn", 1);
    }
}
```
我们还应该在`generate_world_map`的末尾清除计数器状态：
```rust
fn generate_world_map(&mut self, new_depth : i32, offset: i32) {
    self.mapgen_index = 0;
    self.mapgen_timer = 0.0;
    self.mapgen_history.clear();
    let map_building_info = map::level_transition(&mut self.ecs, new_depth, offset);
    if let Some(history) = map_building_info {
        self.mapgen_history = history;
    } else {
        map::thaw_level_entities(&mut self.ecs);
    }

    gamelog::clear_log();
    gamelog::Logger::new()
        .append("欢迎来到")
        .color(rltk::CYAN)
        .append("Rusty Roguelike")
        .log();

    gamelog::clear_events();
}
```
为了演示它的工作原理，让我们在玩家的死亡屏幕上显示玩家存活的回合数。在`gui.rs`中，打开`game_over`函数并添加一个回合计数器：
```rust
pub fn game_over(ctx : &mut Rltk) -> GameOverResult {
    ctx.print_color_centered(15, RGB::named(rltk::YELLOW), RGB::named(rltk::BLACK), "你的旅程已经结束！");
    ctx.print_color_centered(17, RGB::named(rltk::WHITE), RGB::named(rltk::BLACK), "总有一天，我们会告诉你你是如何做到的。");
    ctx.print_color_centered(18, RGB::named(rltk::WHITE), RGB::named(rltk::BLACK), "遗憾的是，这一天不是在本章中...");

    ctx.print_color_centered(19, RGB::named(rltk::WHITE), RGB::named(rltk::BLACK), &format!("你存活了 {} 回合。", crate::gamelog::get_event_count("Turn")));

    ctx.print_color_centered(21, RGB::named(rltk::MAGENTA), RGB::named(rltk::BLACK), "按任意键返回菜单。");

    match ctx.key {
        None => GameOverResult::NoSelection,
        Some(_) => GameOverResult::QuitToMenu
    }
}
```
如果你现在`cargo run`，你的回合数将被计数。以下是我尝试被杀死的运行结果：

![c71-s3.jpg](c71-s3.jpg)

## Bracket 开始统计数量

这是一个非常灵活的系统：你可以从任何地方计数几乎所有你想要的东西！让我们记录玩家在整个游戏中受到的伤害。打开`src/effects/damage.rs`并修改`inflict_damage`函数：
```rust
pub fn inflict_damage(ecs: &mut World, damage: &EffectSpawner, target: Entity) {
    let mut pools = ecs.write_storage::<Pools>();
    let player_entity = ecs.fetch::<Entity>();
    if let Some(pool) = pools.get_mut(target) {
        if !pool.god_mode {
            if let Some(creator) = damage.creator {
                if creator == target { 
                    return; 
                }
            }
            if let EffectType::Damage{amount} = damage.effect_type {
                pool.hit_points.current -= amount;
                add_effect(None, EffectType::Bloodstain, Targets::Single{target});
                add_effect(None, 
                    EffectType::Particle{ 
                        glyph: rltk::to_cp437('‼'),
                        fg : rltk::RGB::named(rltk::ORANGE),
                        bg : rltk::RGB::named(rltk::BLACK),
                        lifespan: 200.0
                    }, 
                    Targets::Single{target}
                );
                if target == *player_entity {
                    crate::gamelog::record_event("Damage Taken", amount);
                }
                if damage.creator == *player_entity {
                    crate::gamelog::record_event("Damage Inflicted", amount);
                }

                if pool.hit_points.current < 1 {
                    add_effect(damage.creator, EffectType::EntityDeath, Targets::Single{target});
                }
            }
        }
    }
}
```
我们将再次修改`gui.rs`的`game_over`函数以显示受到的伤害：
```rust
pub fn inflict_damage(ecs: &mut World, damage: &EffectSpawner, target: Entity) {
    let mut pools = ecs.write_storage::<Pools>();
    let player_entity = ecs.fetch::<Entity>();
    if let Some(pool) = pools.get_mut(target) {
        if !pool.god_mode {
            if let Some(creator) = damage.creator {
                if creator == target { 
                    return; 
                }
            }
            if let EffectType::Damage{amount} = damage.effect_type {
                pool.hit_points.current -= amount;
                add_effect(None, EffectType::Bloodstain, Targets::Single{target});
                add_effect(None, 
                    EffectType::Particle{ 
                        glyph: rltk::to_cp437('‼'),
                        fg : rltk::RGB::named(rltk::ORANGE),
                        bg : rltk::RGB::named(rltk::BLACK),
                        lifespan: 200.0
                    }, 
                    Targets::Single{target}
                );
                if target == *player_entity {
                    crate::gamelog::record_event("Damage Taken", amount);
                }
                if let Some(creator) = damage.creator {
                    if creator == *player_entity {
                        crate::gamelog::record_event("Damage Inflicted", amount);
                    }
                }

                if pool.hit_points.current < 1 {
                    add_effect(damage.creator, EffectType::EntityDeath, Targets::Single{target});
                }
            }
        }
    }
}
```
现在死亡时会显示你在整个运行中遭受的伤害：

![c71-s4.jpg](c71-s4.jpg)

当然，你可以根据自己的需要扩展这个功能。几乎所有可量化的东西现在都可以被跟踪，如果你愿意的话。

### 保存和加载计数器

在`src/gamelog/events.rs`中添加两个函数：
```rust
pub fn clone_events() -> HashMap<String, i32> {
    EVENTS.lock().unwrap().clone()
}

pub fn load_events(events : HashMap<String, i32>) {
    EVENTS.lock().unwrap().clear();
    events.iter().for_each(|(k,v)| {
        EVENTS.lock().unwrap().insert(k.to_string(), *v);
    });
}
```
现在打开`components.rs`，并修改`DMSerializationHelper`：
```rust
#[derive(Component, Serialize, Deserialize, Clone)]
pub struct DMSerializationHelper {
    pub map : super::map::MasterDungeonMap,
    pub log : Vec<Vec<crate::gamelog::LogFragment>>,
    pub events : HashMap<String, i32>
}
```
然后在`saveload_system.rs`中，我们可以将克隆的事件包含在我们的序列化过程中：
```rust
let savehelper2 = ecs
    .create_entity()
    .with(DMSerializationHelper{ 
        map : dungeon_master, 
        log: crate::gamelog::clone_log(), 
        events : crate::gamelog::clone_events() 
    })
    .marked::<SimpleMarker<SerializeMe>>()
    .build();
```
当我们反序列化时，导入事件：
```rust
for (e,h) in (&entities, &helper2).join() {
    let mut dungeonmaster = ecs.write_resource::<super::map::MasterDungeonMap>();
    *dungeonmaster = h.map.clone();
    deleteme2 = Some(e);
    crate::gamelog::restore_log(&mut h.log.clone());
    crate::gamelog::load_events(h.events.clone());
}
```
## 总结

我们现在有了漂亮的彩色日志，以及玩家成就的计数器。这离Steam（或XBOX）风格的成就只差一步——我们将在下一章中介绍。
 

---

**The source code for this chapter may be found [here](https://github.com/thebracket/rustrogueliketutorial/tree/master/chapter-71-logging)**


[Run this chapter's example with web assembly, in your browser (WebGL2 required)](https://bfnightly.bracketproductions.com/rustbook/wasm/chapter-71-logging)
---

Copyright (C) 2019, Herbert Wolverson.

---