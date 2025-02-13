# 第4章 - 一个更有趣的地图

---

***About this tutorial***

*This tutorial is free and open source, and all code uses the MIT license - so you are free to do with it as you like. My hope is that you will enjoy the tutorial, and make great games!*

*If you enjoy this and would like me to keep writing, please consider supporting [my Patreon](https://www.patreon.com/blackfuture).*

[![Hands-On Rust](./beta-webBanner.jpg)](https://pragprog.com/titles/hwrust/hands-on-rust/)

---

在这一章中，我们将创建一个更有趣的地图。它将基于房间，并且看起来有点像许多早期的 rogue-like 游戏，如 Moria - 但复杂性较低。它还将为放置怪物提供一个很好的起点！

## 清理代码

我们将首先稍微清理一下代码，并使用单独的文件。随着项目的复杂性和大小的增加，将它们作为一组干净的文件/模块来保存是一个好主意，这样我们可以快速找到我们需要的东西（有时还可以提高编译时间）。

如果你查看[本章的源代码](https://github.com/thebracket/rustrogueliketutorial/tree/master/chapter-04-newmap)，你会发现我们将很多功能分解成了单独的文件。当你在 Rust 中创建一个新文件时，它自动成为一个 *模块*。然后你必须告诉 Rust 使用这些模块，所以 `main.rs` 增加了一些 `mod map` 和类似的语句，后面跟着 `pub use map::*`。这意味着“导入 map 模块，然后使用 - 并使其他模块可以访问 - 它的公共内容”。

我们还把很多 `struct` 变成了 `pub struct`，并为它们的成员添加了 `pub`。如果你 *不* 这样做，那么结构将仅限于该模块内部 - 你不能在代码的其他部分使用它。这相当于在 C++ 类定义中放置一个 `public:` 行，并在头文件中导出类型。Rust 使其更加干净，不需要写两次！

## 创建更有趣的地图

我们将首先将 `new_map`（现在在 `map.rs` 中）重命名为 `new_map_test`。我们将停止使用它，但保留一段时间 - 这是测试我们地图代码的一个不错的方式！我们还将使用 Rust 的文档标签来发布这个函数的作用，以防我们忘记：

```rust
/// 创建一个有 solid 边界和 400 个随机放置的墙的地图。不能保证它不会看起来很糟糕。
pub fn new_map_test() -> Vec<TileType> {
    ...
}
```

在规范的 Rust 中，如果你用 `///` 开头的注释前缀一个函数，它就会变成一个 *函数注释*。当你在函数头上悬停鼠标时，你的 IDE 将会显示你的注释文本，你可以使用 [Cargo 的文档功能](https://doc.rust-lang.org/cargo/commands/cargo-doc.html) 为你编写的系统制作漂亮的文档页面。如果你打算分享你的代码或与他人合作，这会很有用 - 但它也很好！

所以现在，在遵循[原始 libtcod 教程](http://rogueliketutorials.com/tutorials/tcod/part-3/)的精神下，我们将开始制作地图。我们的目标是随机放置房间，并用走廊将它们连接起来。

## 创建几个矩形房间

我们将从一个新函数开始：

```rust
pub fn new_map_rooms_and_corridors() -> Vec<TileType> {
    let mut map = vec![TileType::Wall; 80*50];

    map
}
```

这将创建一个 solid 80x50 的地图，所有瓦片都是墙壁 - 你不能移动！我们保留了函数签名，所以要在 `main.rs` 中更改我们想要使用的地图，只需将 `gs.ecs.insert(new_map_test());` 更改为 `gs.ecs.insert(new_map_rooms_and_corridors());`。我们再次使用 `vec!` 宏来简化我们的工作 - 有关其工作原理的讨论，请参见上一章。

由于此算法大量使用矩形和 `Rect` 类型 - 我们将首先在 `rect.rs` 中创建一个。我们还将包括一些在本章后面有用的实用函数：

```rust
pub struct Rect {
    pub x1 : i32,
    pub x2 : i32,
    pub y1 : i32,
    pub y2 : i32
}

impl Rect {
    pub fn new(x:i32, y: i32, w:i32, h:i32) -> Rect {
        Rect{x1:x, y1:y, x2:x+w, y2:y+h}
    }

    // 如果与此矩形重叠，则返回 true
    pub fn intersect(&self, other:&Rect) -> bool {
        self.x1 <= other.x2 && self.x2 >= other.x1 && self.y1 <= other.y2 && self.y2 >= other.y1
    }

    pub fn center(&self) -> (i32, i32) {
        ((self.x1 + self.x2)/2, (self.y1 + self.y2)/2)
    }
}
```

这里并没有什么真正的新内容，但让我们稍微分析一下：

1. 我们定义了一个名为 `Rect` 的 `struct`。我们添加了 `pub` 标签使其 *公开* - 它可以在模块外部使用（通过将其放入一个新文件，我们自动创建了一个代码模块；这是 Rust 内置的一种将代码分块的方式）。在 `main.rs` 中，我们可以添加 `pub mod Rect` 来表示“我们使用 `Rect`，并且由于我们在前面加了 `pub`，任何东西都可以从我们这里以 `super::rect::Rect` 的形式获取 `Rect`”。这不是很方便输入，所以第二行 `use rect::Rect` 将其缩短为 `super::Rect`。
2. 我们制作了一个新的 *构造函数*，名为 `new`。它使用返回简写并返回一个基于我们传入的 `x`、`y`、`width` 和 `height` 的矩形。
3. 我们定义了一个 *成员* 方法，`intersect`。它有一个 `&self`，意味着它可以查看它附加的 `Rect` - 但不能修改它（它是一个“纯”函数）。它返回一个布尔值：如果两个矩形重叠，则为 `true`，否则为 `false`。
4. 我们定义了 `center`，也是一个纯成员方法。它简单地返回矩形的中心坐标，作为一个 `x` 和 `y` 的 *元组*，分别在 `val.0` 和 `val.1` 中。

我们还将制作一个新的函数来将房间应用到地图上：

```rust
fn apply_room_to_map(room : &Rect, map: &mut [TileType]) {
    for y in room.y1 +1 ..= room.y2 {
        for x in room.x1 + 1 ..= room.x2 {
            map[xy_idx(x, y)] = TileType::Floor;
        }
    }
}
```

注意我们使用的是 `for y in room.y1 +1 ..= room.y2` - 这是一个 *包含范围*。我们想要一直到达 `y2` 的值，而不是 `y2-1`！否则，它相对直接：使用两个 for 循环来访问房间矩形内的每个瓦片，并将该瓦片设置为 `Floor`。

有了这两段代码，我们可以使用 `Rect::new(x, y, width, height)` 在任何地方创建一个新的矩形。我们可以使用 `apply_room_to_map(rect, map)` 将其作为地板添加到地图上。这足以添加两个测试房间。我们的地图函数现在看起来像这样：

```rust
pub fn new_map_rooms_and_corridors() -> Vec<TileType> {
    let mut map = vec![TileType::Wall; 80*50];

    let room1 = Rect::new(20, 15, 10, 15);
    let room2 = Rect::new(35, 15, 10, 15);

    apply_room_to_map(&room1, &mut map);
    apply_room_to_map(&room2, &mut map);

    map
}
```

如果您 `cargo run` 您的项目，您会看到我们现在有两个房间 - 但它们没有连接在一起。

## 创建走廊

两个未连接的房间并不好玩，所以让我们在它们之间添加一条走廊。我们需要一些比较函数，因此我们必须告诉Rust导入它们（在`map.rs`的顶部）：`use std::cmp::{max, min};`。`min`和`max`的作用如其名：它们返回两个值中的最小值或最大值。你可以使用`if`语句来做同样的事情，但一些计算机可能会将其优化为一个简单的（快速）调用；我们让Rust来决定！ 

然后我们创建两个函数，一个用于水平隧道，一个用于垂直隧道：

```rust
fn apply_horizontal_tunnel(map: &mut [TileType], x1:i32, x2:i32, y:i32) {
    for x in min(x1,x2) ..= max(x1,x2) {
        let idx = xy_idx(x, y);
        if idx > 0 && idx < 80*50 {
            map[idx as usize] = TileType::Floor;
        }
    }
}

fn apply_vertical_tunnel(map: &mut [TileType], y1:i32, y2:i32, x:i32) {
    for y in min(y1,y2) ..= max(y1,y2) {
        let idx = xy_idx(x, y);
        if idx > 0 && idx < 80*50 {
            map[idx as usize] = TileType::Floor;
        }
    }
}
```

然后我们在地图制作函数中添加一个调用，`apply_horizontal_tunnel(&mut map, 25, 40, 23);`，于是我们就在两个房间之间有了一个隧道！如果你运行（`cargo run`）项目，你可以在两个房间之间行走 - 而不是走进墙壁。所以我们的旧代码仍然有效，但现在它看起来更像一个rogue-like游戏。

## 创建一个简单的地牢

现在我们可以使用它来创建一个随机地牢。我们将修改我们的函数如下：

```rust
pub fn new_map_rooms_and_corridors() -> Vec<TileType> {
    let mut map = vec![TileType::Wall; 80*50];

    let mut rooms : Vec<Rect> = Vec::new();
    const MAX_ROOMS : i32 = 30;
    const MIN_SIZE : i32 = 6;
    const MAX_SIZE : i32 = 10;

    let mut rng = RandomNumberGenerator::new();

    for _ in 0..MAX_ROOMS {
        let w = rng.range(MIN_SIZE, MAX_SIZE);
        let h = rng.range(MIN_SIZE, MAX_SIZE);
        let x = rng.roll_dice(1, 80 - w - 1) - 1;
        let y = rng.roll_dice(1, 50 - h - 1) - 1;
        let new_room = Rect::new(x, y, w, h);
        let mut ok = true;
        for other_room in rooms.iter() {
            if new_room.intersect(other_room) { ok = false }
        }
        if ok {
            apply_room_to_map(&new_room, &mut map);        
            rooms.push(new_room);            
        }
    }

    map
}
```

有很多变化：

* 我们为要创建的最大房间数以及房间的最小和最大尺寸添加了`const`常量。这是我们第一次遇到`const`：它只是说“在开始时设置这个值，并且它永远不会改变”。这是在Rust中拥有全局变量的唯一简单方法；由于它们永远不会改变，它们通常甚至不存在，并且被烘焙到你使用的函数中。如果它们*确实*存在，因为它们不能改变，所以在多线程访问它们时没有问题。设置命名常量通常比使用“魔法数字”更干净 - 那是一个没有真正线索的硬编码值，你为什么选择那个值。
* 我们从RLTK获取了一个`RandomNumberGenerator`（这需要我们在`map.rs`顶部的`use`语句中添加）
* 我们正在随机构建宽度和高度。
* 然后我们随机放置房间，使`x`和`y`大于0且小于最大地图尺寸减一。
* 我们遍历现有的房间，如果新房间与我们已经放置的房间重叠，则拒绝新房间。
* 如果可以，我们将其应用到房间中。
* 我们正在保持房间在一个向量中，尽管我们还没有使用它。

在这一点上运行项目（`cargo run`）将给你一系列随机房间，它们之间没有走廊。

## 连接房间

现在我们需要通过走廊连接房间。我们将在地图生成器的 `if ok` 部分添加代码：

```rust
if ok {
    apply_room_to_map(&new_room, &mut map);

    if !rooms.is_empty() {
        let (new_x, new_y) = new_room.center();
        let (prev_x, prev_y) = rooms[rooms.len()-1].center();
        if rng.range(0,2) == 1 {
            apply_horizontal_tunnel(&mut map, prev_x, new_x, prev_y);
            apply_vertical_tunnel(&mut map, prev_y, new_y, new_x);
        } else {
            apply_vertical_tunnel(&mut map, prev_y, new_y, prev_x);
            apply_horizontal_tunnel(&mut map, prev_x, new_x, new_y);
        }
    }

    rooms.push(new_room);
}
```

1. 这段代码首先检查 `rooms` 列表是否为空。如果是，则没有之前的房间可以连接 - 因此我们忽略它。
2. 它获取房间的中心，并将其存储为 `new_x` 和 `new_y`。
3. 它获取向量中前一个房间的中心，并将其存储为 `prev_x` 和 `prev_y`。
4. 它掷骰子，一半时间绘制水平然后垂直隧道 - 另一半时间则相反。

现在尝试 `cargo run`。它看起来真的像一个Roguelike游戏！

## 放置玩家

目前，玩家总是从地图中心开始 - 使用新的生成器，这可能不是一个有效的起点！我们可以简单地将玩家移动到第一个房间的中心，但很可能我们的生成器需要知道所有房间的位置 - 这样我们可以在其中放置物品 - 而不仅仅是玩家的位置。因此，我们将修改 `new_map_rooms_and_corridors` 函数以同时返回房间列表。因此，我们将方法签名更改为：`pub fn new_map_rooms_and_corridors() -> (Vec<Rect>, Vec<TileType>) {`，并将返回语句更改为 `(rooms, map)`。

我们的 `main.rs` 文件也需要进行调整，以接受新的格式。我们在 `main.rs` 中将 `main` 函数更改为：

```rust
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
    gs.ecs.register::<Player>();

    let (rooms, map) = new_map_rooms_and_corridors();
    gs.ecs.insert(map);
    let (player_x, player_y) = rooms[0].center();

    gs.ecs
        .create_entity()
        .with(Position { x: player_x, y: player_y })
        .with(Renderable {
            glyph: rltk::to_cp437('@'),
            fg:::named(rltk::YELLOW),
            bg: RGB::named(rltk::BLACK),
        })
        .with(Player{})
        .build();

    rltk::main_loop(context, gs)
}
```

这大部分是相同的，但我们从 `new_map_rooms_and_corridors` 接收了 *房间列表* 和地图。然后我们将玩家放置在第一个房间的中心。

## 总结 - 支持数字键盘和 Vi 键

现在你有了一个看起来像Roguelike游戏的地图，将玩家放置在第一个房间，并允许你使用光标键进行探索。不是每台键盘都有容易访问的光标键（一些笔记本电脑需要有趣的键组合才能使用它们）。许多玩家喜欢使用数字键盘进行操作，但不是每台键盘都有。因此，我们还支持文本编辑器 `vi` 的方向键。这使硬核 UNIX 用户和普通玩家都感到满意。

我们暂时不考虑对角移动。在 `player.rs` 中，我们将 `player_input` 更改为如下所示：

```rust
pub fn player_input(gs: &mut State, ctx: &mut Rltk) {
    // 玩家移动
    match ctx.key {
        None => {} // 没有发生任何事情
        Some(key) => match key {
            VirtualKeyCode::Left |
            VirtualKeyCode::Numpad4 |
            VirtualKeyCode::H => try_move_player(-1, 0, &mut gs.ecs),

            VirtualKeyCode::Right |
            VirtualKeyCode::Numpad6 |
            VirtualKeyCode::L => try_move_player(1, 0, &mut gs.ecs),

            VirtualKeyCode::Up |
            VirtualKeyCode::Numpad8 |
            VirtualKeyCode::K => try_move_player(0, -1, &mut gs.ecs),

            VirtualKeyCode::Down |
            VirtualKeyCode::Numpad2 |
            VirtualKeyCode::J => try_move_player(0, 1, &mut gs.ecs),

            _ => {}
        },
    }
}
```

当你 `cargo run` 你的项目时，你应该会得到这样的结果：

![Screenshot](./c4-s1.gif)

**The source code for this chapter may be found [here](https://github.com/thebracket/rustrogueliketutorial/tree/master/chapter-04-newmap)**

[Run this chapter's example with web assembly, in your browser (WebGL2 required)](https://bfnightly.bracketproductions.com/rustbook/wasm/chapter-04-newmap/)

---

Copyright (C) 2019, Herbert Wolverson.

---
