# 城市中的一夜

---

***关于本教程***

*本教程是免费和开源的，所有代码都使用MIT许可证 - 因此您可以自由地使用它。我希望您会喜欢这个教程，并制作出伟大的游戏！*

*如果您喜欢这个教程并希望我继续写作，请考虑支持[我的Patreon](https://www.patreon.com/blackfuture)。*

![Hands-On Rust](./beta-webBanner.jpg)](https://pragprog.com/titles/hwrust/hands-on-rust/)

---

游戏的下一个关卡是一个黑暗精灵城市。设计文档对细节描述不多，但我们知道以下几点：

* 它最终会通往深渊的传送门。
* 黑暗精灵是喜欢内斗、背后捅刀子的疯子，应该表现出这样的行为。
* 黑暗精灵城市令人惊讶地像城市一样，只是位于地下深处。
* 灯光将会很重要。

## 生成基础城市

`map_builders/mod.rs`文件中的`level_builder`函数控制了给定层级调用的地图算法。为新的地图类型添加一个占位符条目：
```rust
pub fn level_builder(new_depth: i32, width: i32, height: i32) -> BuilderChain {
    rltk::console::log(format!("Depth: {}", new_depth));
    match new_depth {
        1 => town_builder(new_depth, width, height),
        2 => forest_builder(new_depth, width, height),
        3 => limestone_cavern_builder(new_depth, width, height),
        4 => limestone_deep_cavern_builder(new_depth, width, height),
        5 => limestone_transition_builder(new_depth, width, height),
        6 => dwarf_fort_builder(new_depth, width, height),
        7 => mushroom_entrance(new_depth, width, height),
        8 => mushroom_builder(new_depth, width, height),
        9 => mushroom_exit(new_depth, width, height),
        10 => dark_elf_city(new_depth, width, height),
        _ => random_builder(new_depth, width, height)
    }
}
```
在同一个文件的顶部，为新的构建器模块添加导入：
```rust
mod dark_elves;
use dark_elves::*;
```
并在新的`map_builders/dark_elves.rs`文件中创建一个占位符构建器：
```rust
use super::{BuilderChain, XStart, YStart, AreaStartingPosition, 
    CullUnreachable, VoronoiSpawning,
    AreaEndingPosition, XEnd, YEnd, BspInteriorBuilder };

pub fn dark_elf_city(new_depth: i32, width: i32, height: i32) -> BuilderChain {
    println!("Dark elf builder");
    let mut chain = BuilderChain::new(new_depth, width, height, "Dark Elven City");
    chain.start_with(BspInteriorBuilder::new());
    chain.with(AreaStartingPosition::new(XStart::CENTER, YStart::CENTER));
    chain.with(CullUnreachable::new());
    chain.with(AreaStartingPosition::new(XStart::RIGHT, YStart::CENTER));
    chain.with(AreaEndingPosition::new(XEnd::LEFT, YEnd::CENTER));
    chain.with(VoronoiSpawning::new());
    chain
}
```
这生成的是一个不太像城市的地图（只是一个BSP内部地图）- 但这是一个很好的开始。我选择这个作为基础构建器，因为它不会浪费任何空间。我喜欢想象这个城市是一个大型互联的房间群，穷人的精灵住房在危险的位置（在顶部）。所以我们将用相对“正常”的黑暗精灵和他们的奴隶来填充这个层级。

## 添加一些黑暗精灵

如果我们只想到处放置黑暗精灵，只需在`spawns.json`的`spawn_table`部分添加一行即可：
```json
{ "name" : "Dark Elf", "weight": 10, "min_depth": 10, "max_depth": 11 }
```
那太无聊了，所以我们不要这样做。我们的黑暗精灵分为*阿巴特部落*、*巴博部落*和*奇罗部落*（A，B，C，明白了吗？）。由于亚拉之护身符的深渊影响，他们陷入了可怕的内部争斗和战争中！我们稍后会担心如何区分这些部落，现在让我们做一些条目来提供三个彼此仇恨的黑暗精灵群体。

在`spawns.json`的`factions`部分，创建三个新的派系：
```json
{ "name" : "DarkElfA", "responses" : { "Default" : "attack", "DarkElfA" : "ignore", "DarkElfB" : "attack", "DarkElfC" : "attack" } },
{ "name" : "DarkElfB", "responses" : { "Default" : "attack", "DarkElfB" : "ignore", "DarkElfA" : "attack", "DarkElfC" : "attack" } },
{ "name" : "DarkElfC", "responses" : { "Default" : "attack", "DarkElfC" : "ignore", "DarkElfA" : "attack", "DarkElfB" : "attack" } }
```
注意他们如何忽略自己的部落，并攻击其他部落。这是制造战争的关键！我们的派系系统已经支持交战的群体 - 只是我们还没有广泛使用它。现在找到`mobs`部分，并将“Dark Elf”复制三次 - 每个派系一次：
```json
{
    "name" : "Arbat Dark Elf",
    "renderable": {
        "glyph" : "e",
        "fg" : "#FF0000",
        "bg" : "#000000",
        "order" : 1
    },
    "blocks_tile" : true,
    "vision_range" : 8,
    "movement" : "random_waypoint",
    "attributes" : {},
    "equipped" : [ "Hand Crossbow", "Scimitar", "Buckler", "Drow Chain", "Drow Leggings", "Drow Boots" ],
    "faction" : "DarkElfA",
    "gold" : "3d6",
    "level" : 6
},

{
    "name" : "Barbo Dark Elf",
    "renderable": {
        "glyph" : "e",
        "fg" : "#FF0000",
        "bg" : "#000000",
        "order" : 1
    },
    "blocks_tile" : true,
    "vision_range" : 8,
    "movement" : "random_waypoint",
    "attributes" : {},
    "equipped" : [ "Hand Crossbow", "Scimitar", "Buckler", "Drow Chain", "Drow Leggings", "Drow Boots" ],
    "faction" : "DarkElfB",
    "gold" : "3d6",
    "level" : 6
},

{
    "name" : "Cirro Dark Elf",
    "renderable": {
        "glyph" : "e",
        "fg" : "#FF0000",
        "bg" : "#000000",
        "order" : 1
    },
    "blocks_tile" : true,
    "vision_range" : 8,
    "movement" : "random_waypoint",
    "attributes" : {},
    "equipped" : [ "Hand Crossbow", "Scimitar", "Buckler", "Drow Chain", "Drow Leggings", "Drow Boots" ],
    "faction" : "DarkElfC",
    "gold" : "3d6",
    "level" : 6
},
```
在生成表中，我们希望它们出现在第10层：
```json
{ "name" : "Arbat Dark Elf", "weight": 10, "min_depth": 10, "max_depth": 11 },
{ "name" : "Barbo Dark Elf", "weight": 10, "min_depth": 10, "max_depth": 11 },
{ "name" : "Cirro Dark Elf", "weight": 10, "min_depth": 10, "max_depth": 11 }
```
如果你现在`cargo run`，并且作弊进入第10层（我建议使用上帝模式和传送） - 你会发现自己处于三个部落之间的战争地带。到处都是战斗，他们只是暂停相互残杀，以便谋杀玩家。有大量的混乱 - 混沌之神会感到骄傲。

## 部落区分

拥有完全相同的部落有点无聊。基本的“黑暗精灵”可以保持不变，但让我们添加一些风味，使部落*感觉*不同。

### 阿巴特部落

我们首先将阿巴特部落的颜色改为更浅的红色 - 一种粉红色。将他们的黑暗精灵的"fg"属性替换为`#FFAAAA`。我们还会剥夺他们的弩。他们是近战导向的部落。将`Scimitar`替换为`Scimitar +1`。修改后的`Arbat Dark Elf`如下所示：
```json
{
    "name" : "Arbat Dark Elf",
    "renderable": {
        "glyph" : "e",
        "fg" : "#FFAAAA",
        "bg" : "#000000",
        "order" : 1
    },
    "blocks_tile" : true,
    "vision_range" : 8,
    "movement" : "random_waypoint",
    "attributes" : {},
    "equipped" : [ "Scimitar +1", "Buckler", "Drow Chain", "Drow Leggings", "Drow Boots" ],
    "faction" : "DarkElfA",
    "gold" : "3d6",
    "level" : 6
},
```
让我们也给他们一些领导者 - 更强大的战士：
```json
{
    "name" : "Arbat Dark Elf Leader",
    "renderable": {
        "glyph" : "E",
        "fg" : "#FFAAAA",
        "bg" : "#000000",
        "order" : 1
    },
    "blocks_tile" : true,
    "vision_range" : 8,
    "movement" : "random_waypoint",
    "attributes" : {},
    "equipped" : [ "Scimitar +2", "Buckler +1", "Drow Chain", "Drow Leggings", "Drow Boots" ],
    "faction" : "DarkElfA",
    "gold" : "3d6",
    "level" : 7
},
```

他们也值得拥有一些兽人奴隶：
```json
{
    "name" : "Arbat Orc Slave",
    "renderable": {
        "glyph" : "o",
        "fg" : "#FFAAAA",
        "bg" : "#000000",
        "order" : 1
    },
    "blocks_tile" : true,
    "vision_range" : 8,
    "movement" : "static",
    "attributes" : {},
    "faction" : "DarkElfA",
    "gold" : "1d8"
},
```
最后，将它们放入生成表中：
```json
{ "name" : "Arbat Dark Elf", "weight": 10, "min_depth": 10, "max_depth": 11 },
{ "name" : "Arbat Dark Elf Leader", "weight": 7, "min_depth": 10, "max_depth": 11 },
{ "name" : "Arbat Orc Slave", "weight": 14, "min_depth": 10, "max_depth": 11 },
```
他们可能会为他们的近战焦点感到后悔，但我们并不太关心他们的健康！

### 巴博部落

相反，我们将使巴博部落更加偏向于远程战斗 - 并且更加稀有，因为这是非常危险的。我们还会给他们一把匕首而不是弯刀，并将他们的颜色改为橙色：

```json
{
    "name" : "Barbo Dark Elf",
    "renderable": {
        "glyph" : "e",
        "fg" : "#FF9900",
        "bg" : "#000000",
        "order" : 1
    },
    "blocks_tile" : true,
    "vision_range" : 8,
    "movement" : "random_waypoint",
    "attributes" : {},
    "equipped" : [ "Hand Crossbow +1", "Dagger", "Buckler", "Drow Chain", "Drow Leggings", "Drow Boots" ],
    "faction" : "DarkElfB",
    "gold" : "3d6",
    "level" : 6
},
```

他们还有一些奴隶 - 这次是带有远程武器的哥布林：
```json
{
    "name" : "Barbo Goblin Archer",
    "renderable": {
        "glyph" : "g",
        "fg" : "#FF9900",
        "bg" : "#000000",
        "order" : 1
    },
    "blocks_tile" : true,
    "vision_range" : 8,
    "movement" : "static",
    "attributes" : {},
    "faction" : "Cave Goblins",
    "gold" : "1d6",
    "equipped" : [ "Shortbow", "Leather Armor", "Leather Boots" ]
},
```
最后，更新生成表以包括他们：
```json
{ "name" : "Barbo Dark Elf", "weight": 9, "min_depth": 10, "max_depth": 11 },
{ "name" : "Barbo Goblin Archer", "weight": 13, "min_depth": 10, "max_depth": 11 },
```

### Clan 部落

我们将使Cirro部落强大且稀有。基本的Cirro Dark Elf如下所示：
```json
{
    "name" : "Cirro Dark Elf",
    "renderable": {
        "glyph" : "e",
        "fg" : "#FF00FF",
        "bg" : "#000000",
        "order" : 1
    },
    "blocks_tile" : true,
    "vision_range" : 8,
    "movement" : "random_waypoint",
    "attributes" : {},
    "equipped" : [ "Hand Crossbow", "Scimitar", "Buckler", "Drow Chain", "Drow Leggings", "Drow Boots" ],
    "faction" : "DarkElfC",
    "gold" : "3d6",
    "level" : 7
},
```
我们还会给他们领导者 - 可以网住你的女祭司：
```json
{
        "name" : "Cirro Dark Priestess",
        "renderable": {
            "glyph" : "E",
            "fg" : "#FF00FF",
            "bg" : "#000000",
            "order" : 1
        },
        "blocks_tile" : true,
        "vision_range" : 8,
        "movement" : "random_waypoint",
        "attributes" : {},
        "equipped" : [ "Hand Crossbow", "Scimitar", "Buckler", "Drow Chain", "Drow Leggings", "Drow Boots" ],
        "faction" : "DarkElfC",
        "gold" : "3d6",
        "level" : 8,
        "abilities" : [
            { "spell" : "", "chance" : 0.2, "range" : 6.0, "min_range" : 3.0 }
        ]
    },
```
不是奴隶，我们将给他们蜘蛛：
```json
{
    "name" : "Cirro Spider",
    "level" : 3,
    "attributes" : {},
    "renderable": {
        "glyph" : "s",
        "fg" : "#FF00FF",
        "bg" : "#000000",
        "order" : 1
    },
    "blocks_tile" : true,
    "vision_range" : 6,
    "movement" : "static",
    "natural" : {
        "armor_class" : 12,
        "attacks" : [
            { "name" : "bite", "hit_bonus" : 1, "damage" : "1d12" }
        ]
    },
    "abilities" : [
        { "spell" : "Web", "chance" : 0.2, "range" : 6.0, "min_range" : 3.0 }
    ],
    "faction" : "DarkElfC"
},
```
这也需要更新生成表：
```json
{ "name" : "Cirro Dark Elf", "weight": 7, "min_depth": 10, "max_depth": 11 },
{ "name" : "Cirro Dark Priestess", "weight": 6, "min_depth": 10, "max_depth": 11 },
{ "name" : "Cirro Spider", "weight": 10, "min_depth": 10, "max_depth": 11 }
```
如果你现在`cargo run`项目，你会发现黑暗精灵们相互残杀 - 并且存在很好的多样性。

## 总结

这是一章简短的章节：因为大多数先决条件已经写好了。这对整个引擎来说是个好兆头：我们现在可以构建一个非常不同风格的级别，而不需要太多新代码。在下一章中，我们将进一步进入黑暗精灵城市 - 试图制作一个更加开放的城市级别。混乱将继续！

---

**本章的源代码可以在这里找到 [here](https://github.com/thebracket/rustrogueliketutorial/tree/master/chapter-74-darkcity)**


[在浏览器中用web assembly运行本章的示例（需要WebGL2）](https://bfnightly.bracketproductions.com/rustbook/wasm/chapter-74-darkcity)
---

版权所有 (C) 2019, Herbert Wolverson。

---
