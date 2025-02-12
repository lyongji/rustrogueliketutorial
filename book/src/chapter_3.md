# 第三章 - 地图漫步

---

***关于本教程***

*本教程是免费和开源的，所有代码均使用MIT许可证 - 因此您可以自由地使用它。我希望您会喜欢这个教程，并制作出伟大的游戏！*

*如果您喜欢这个教程并希望我继续写作，请考虑支持[我的Patreon](https://www.patreon.com/blackfuture)。*

[![实践Rust](./beta-webBanner.jpg)](https://pragprog.com/titles/hwrust/hands-on-rust/)

---

本教程的其余部分将致力于制作一个Roguelike游戏。[Rogue](https://en.wikipedia.org/wiki/Rogue_(video_game))出现在1980年，作为一个文本模式的地下城探险游戏。它催生了一个整个“Roguelike”游戏类型：程序化生成的地图，跨越多个级别的目标狩猎和“永久死亡”（死亡时重新开始）。定义的来源是许多在线争论的原因；我更愿意避免这种争论！

没有地图可以探索的Roguelike游戏是有点无意义的，所以在本章中我们将组装一个基本的地图，绘制它，并让玩家在地图上走动。我们从第二章的代码开始，但删除了红色的笑脸（以及它们的向左倾向）。

## 定义地图瓦片

我们首先允许两种瓦片类型：墙壁和地板。我们可以使用`enum`（要了解更多关于枚举的信息，[The Rust Book](https://doc.rust-lang.org/book/ch06-00-enums.html)有一个*很大的*关于它们的章节）来表示：

```rust
#[derive(PartialEq, Copy, Clone)]
enum TileType {
    Wall, Floor
}
```

注意，我们包含了一些派生特性（这次是Rust内置的派生宏）：`Copy` 和 `Clone`。`Clone` 为类型添加了一个 `.clone()` 方法，允许以编程方式复制对象。`Copy` 改变了赋值时的默认行为，从 *移动* 对象到制作一个副本 - 所以 `tile1 = tile2` 会留下两个有效的值，而不是处于“已移动”状态。

`PartialEq` 允许我们使用 `==` 来判断两个瓦片类型是否匹配。如果我们 *不* 派生这些特性，`if tile_type == TileType::Wall` 将无法编译！

## 构建一个简单的地图

现在我们将制作一个返回 `vec`（向量）的函数，代表一个简单的地图。我们将使用一个与整个地图大小相同的向量，这意味着我们需要一种方法来找出给定x/y位置对应的数组索引。所以首先，我们定义一个新的函数 `xy_idx`：

```rust
pub fn xy_idx(x: i32, y: i32) -> usize {
    (y as usize * 80) + x as usize
}
```

这很简单：它将 `y` 位置乘以地图宽度（80），然后加上 `x`。这保证了每个位置都有一个瓦片，并且有效地将它在内存中映射出来，以便从左到右阅读。

我们在这里使用了Rust函数的简写形式。注意到函数返回一个 `usize`（相当于C/C++中的 `size_t` - 无论平台使用的基本大小类型是什么）- 并且函数体末尾没有 `;`？任何以没有分号的语句结尾的函数都会将该行视为 `return` 语句。所以这与键入 `return (y as usize * 80) + x as usize` 是一样的。这来自Rust作者的其他最爱语言，`ML` - 它也使用相同的简写。这种风格被认为是“Rustacean”（规范的Rust；我总是想象一个有着可爱小爪子和壳的Rust怪物）所以我们在教程中采用了它。

然后我们编写一个*构造函数*来制作地图：
```rust
fn new_map() -> Vec<TileType> {
    let mut map = vec![TileType::Floor; 80*50];

    // 设置边界为墙壁
    for x in 0..80 {
        map[xy_idx(x, 0)] = TileType::Wall;
        map[xy_idx(x, 49)] = TileType::Wall;
    }
    for y in 0..50 {
        map[xy_idx(0, y)] = TileType::Wall;
        map[xy_idx(79, y)] = TileType::Wall;
    }

    // 现在我们将随机生成一些墙壁。这可能不太美观，但作为一个示例还是不错的。
    // 首先，获取线程本地的随机数生成器：
    let mut rng = rltk::RandomNumberGenerator::new();

    for _i in 0..400 {
        let x = rng.roll_dice(1, 79);
        let y = rng.roll_dice(1, 49);
        let idx = xy_idx(x, y);
        if idx != xy_idx(40, 25) {
            map[idx] = TileType::Wall;
        }
    }

    map
}
```

这里有一些我们之前没有遇到过的语法，让我们来逐步解析：

1. `fn new_map() -> Vec<TileType>` 定义了一个名为 `new_map` 的函数。它不接受任何参数，因此可以从任何地方调用。
2. 它 *返回* 一个 `Vec`。`Vec` 是Rust的 *向量*（如果你熟悉C++，它基本上与C++的 `std::vector` 相同）。向量类似于 *数组*（参见 [Rust by Example的这一章节](https://doc.rust-lang.org/rust-by-example/primitives/array.html)），它允许你将一堆数据放入列表中并访问每个元素。与 *数组* 不同的是，`Vec` 没有大小限制 - 并且在程序运行时大小可以改变。因此，你可以 `push`（添加）新项目，并在需要时 `remove` 它们。 [Rust by Example 有一个关于向量的很好的章节](https://doc.rust-lang.org/rust-by-example/std/vec.html)；学习它们是一个好主意 - 它们被 *到处* 使用。
3. `let mut map = vec![TileType::Floor; 80*50];` 是一个看起来令人困惑的语句！让我们来逐步解析：
    1. `let mut map` 表示“创建一个新变量”（`let`），“让我改变它”（`mut`）并命名为“map”。
    2. `vec!` 是 *宏*，是Rust标准库中的另一个宏。感叹号是Rust表示“这是一个过程宏”（与之前看到的派生宏相对）的方式。过程宏像函数一样运行 - 它们定义了一个 *过程*，它们只是大大减少了你的输入量。
    3. `vec!` 宏在方括号中接受其参数。
    4. 第一个参数是新向量的每个元素的 *值*。在这种情况下，我们将创建的每个条目设置为 `Floor`（来自 `TileType` 枚举）。
    5. 第二个参数是我们应该创建的瓦片数量。它们都将被设置为我们上面设置的值。在这种情况下，我们的地图是80x50个瓦片（4000个瓦片 - 但我们让编译器为我们计算！）。因此，我们需要创建4000个瓦片。
    6. 你可以用 `for _i in 0..4000 { map.push(TileType::Floor); }` 替换 `vec!` 调用。事实上，宏为你做的就是这件事 - 但使用宏 definitely 更少输入量！
4. `for x in 0..80 {` 是一个 `for` 循环（[参见这里](https://doc.rust-lang.org/rust-by-example/flow_control/for.html)），就像我们在之前的例子中使用的那样。在这种情况下，我们正在迭代 `x` 从0到79。
5. `map[xy_idx(x, 0)] = TileType::Wall;` 首先调用我们上面定义的 `xy_idx` 函数来获取 `x, 0` 的向量索引。然后 *索引* 向量，告诉它将向量中该位置的条目设置为墙壁。我们再次为 `x,49` 做同样的操作。
6. 我们做同样的操作，但是循环 `y` 从0到49 - 并在我们的地图上设置垂直墙壁。
7. `let mut rng = rltk::RandomNumberGenerator::new();` 调用 `RLTK` 的 `new` 函数中的 `RandomNumberGenerator` 类型，并将其赋值给名为 `rng` 的变量。我们要求RLTK给我们一个新的骰子 roller。
8. `for _i in 0..400 {` 与其他 `for` 循环相同，但注意 `i` 前面的 `_`。我们实际上并没有查看 `i` 的值 - 我们只是希望循环运行400次。如果你有一个未使用的变量，Rust会给你一个警告；在变量前添加下划线前缀告诉Rust，这是可以的，我们故意这样做。
9. `let x = rng.roll_dice(1, 79);` 调用我们在7中获取的 `rng`，并要求它给出1到79之间的随机数。RLTK不采用排他范围，因为它试图模仿旧的D&D规则中的骰子是 `1d20` 或类似。在这种情况下，我们应该感到高兴，因为计算机不关心发明一个79面的骰子的几何难度！我们还获得了1到49之间的 `y` 值。我们已经掷了想象中的骰子，并在地图上找到了一个随机位置。
10. 我们将变量 `idx`（简称“索引”）设置为我们在上面掷骰子得到的坐标的向量索引。
11. `if idx != xy_idx(40, 25) {` 检查 `idx` 是否不是正中间（我们将从这里开始，所以不想在墙里面开始！）。
12. 如果不是中间，我们将随机掷骰子的位置设置为墙壁。

这很简单：它在地图的外围放置了墙壁，然后在除了玩家起始点之外的任何地方随机添加了400个墙壁。

## 将地图展示给世界

Specs包含了一个“资源”的概念 - ECS可以使用的共享数据。因此，在我们的`main`函数中，我们将一个随机生成的地图添加到游戏世界中：
```rust
gs.ecs.insert(new_map());
```
现在地图可以在ECS可以访问的任何地方使用！现在在您的代码中，您可以使用相当繁琐的`let map = self.ecs.get_mut::<Vec<TileType>>();`来访问地图；以更简单的方式在系统中可用。实际上，有*几种*方法可以获取地图的值，包括`ecs.get`，`ecs.fetch`。`get_mut`获取一个“可变的”（您可以更改它）对地图的引用 - 包装在可选的（以防地图不存在）。`fetch`跳过了`Option`类型，直接给您一个地图。您可以在[Specs Book](https://specs.amethyst.rs/docs/tutorials/04_resources.html)中了解更多关于这个的信息。

## 绘制地图

现在我们有了一个可用的地图，我们应该将它显示在屏幕上！新`draw_map`函数的完整代码如下：

```rust
fn draw_map(map: &[TileType], ctx : &mut Rltk) {
    let mut y = 0;
    let mut x = 0;
    for tile in map.iter() {
        // Render a tile depending upon the tile type
        match tile {
            TileType::Floor => {
                ctx.set(x, y, RGB::from_f32(0.5, 0.5, 0.5), RGB::from_f32(0., 0., 0.), rltk::to_cp437('.'));
            }
            TileType::Wall => {
                ctx.set(x, y, RGB::from_f32(0.0, 1.0, 0.0), RGB::from_f32(0., 0., 0.), rltk::to_cp437('#'));
            }
        }

        // Move the coordinates
        x += 1;
        if x > 79 {
            x = 0;
            y += 1;
        }
    }
}
```

这部分代码主要是直接的，并使用了我们已经介绍过的概念。在声明中，我们将地图作为 `&[TileType]` 而不是 `&Vec<TileType>` 传递；这允许我们传递地图的“切片”（部分）。我们暂时不会这样做，但这可能在以后有用。这也是一种更为“正宗的”（即：符合Rust习惯的）做法，并且lint工具（`clippy`）会对此发出警告。[如果你对切片感兴趣，Rust Book可以教你相关知识](https://doc.rust-lang.org/rust-by-example/primitives/array.html)。

否则，它利用了我们存储地图的方式——行与行相邻，一个接一个。因此，它遍历整个地图结构，为每个瓦片增加1到 `x` 位置。如果到达地图宽度，它将 `x` 清零并增加1到 `y`。这样我们就不会反复读取整个数组——这可能会变慢。实际的渲染非常简单：我们根据瓦片类型进行 `match`，并为墙壁/地板绘制一个句点或井号。

我们还应该调用该函数！在我们的 `tick` 函数中，添加：
```rust
let map = self.ecs.fetch::<Vec<TileType>>();
draw_map(&map, ctx);
```
`fetch` 调用是新的（我们上面提到了它）。`fetch` 要求你承诺你知道你请求的资源确实存在——如果不存在，它将会崩溃。它并不完全返回一个引用——它是一个 `shred` 类型，大多数时候表现得像一个引用，但偶尔需要一点强制转换才能成为引用。我们将在需要跨越这座桥时担心这个问题，但现在请提前警告！

## 使墙壁变得坚实

所以现在如果你运行程序（`cargo run`），你将会有一个绿色和灰色的地图，上面有一个黄色的 `@` 可以四处走动。不幸的是，你很快就会发现玩家可以穿过墙壁！幸运的是，这很容易纠正。

为了实现这一点，我们修改 `try_move_player` 以读取地图并检查目的地是否开放：
```rust
fn try_move_player(delta_x: i32, delta_y: i32, ecs: &mut World) {
    let mut positions = ecs.write_storage::<Position>();
    let mut players = ecs.write_storage::<Player>();
    let map = ecs.fetch::<Vec<TileType>>();

    for (_player, pos) in (&mut players, &mut positions).join() {
        let destination_idx = xy_idx(pos.x + delta_x, pos.y + delta_y);
        if map[destination_idx] != TileType::Wall {
            pos.x = min(79 , max(0, pos.x + delta_x));
            pos.y = min(49, max(0, pos.y + delta_y));
        }
    }
}
```
新的部分是 `let map = ...` 部分，它使用 `fetch` 与主循环中相同的方式（这是将其存储在ECS中的优势——你可以在不试图强制Rust使用全局变量的情况下在到处访问它！）。我们使用 `let destination_idx = xy_idx(pos.x + delta_x, pos.y + delta_y);` 计算玩家目的地的单元格索引——如果不是墙壁，我们就正常移动。

现在运行程序（`cargo run`），你将在地图中拥有一个玩家——并且可以移动，被墙壁正确阻挡。

![截图](./c3-s1.gif)

完整的程序现在如下所示：

```rust
use rltk::{GameState, Rltk, RGB, VirtualKeyCode};
use specs::prelude::*;
use std::cmp::{max, min};
use specs_derive::*;



// 定义一个组件，表示位置
#[derive(Component)]
struct Position {
    x: i32,
    y: i32,
}

// 定义一个组件，表示可渲染的对象
#[derive(Component)]
struct Renderable {
    glyph: rltk::FontCharType,
    fg: RGB,
    bg: RGB,
}
 
// 定义一个组件，表示玩家
#[derive(Component, Debug)]
struct Player {}

// 定义一个枚举，表示地图上的瓦片类型
#[derive(PartialEq, Copy, Clone)]
enum TileType {
    Wall, Floor
}

// 定义游戏状态
struct State {
    ecs: World
}

// 将二维坐标转换为线性索引
pub fn xy_idx(x: i32, y: i32) -> usize {
    (y as usize * 80) + x as usize
}

// 生成新的地图
fn new_map() -> Vec<TileType> {
    let mut map = vec![TileType::Floor; 80*50];

    // 设置地图边界为墙壁
    for x in 0..80 {
        map[xy_idx(x, 0)] = TileType::Wall;
        map[xy_idx(x, 49)] = TileType::Wall;
    }
    for y in 0..50 {
        map[xy_idx(0, y)] = TileType::Wall;
        map[xy_idx(79, y)] = TileType::Wall;
    }

    // 现在我们将随机生成一些墙壁。这可能不太美观，但作为一个示例还是不错的。
    // 首先，获取线程本地的随机数生成器：
    let mut rng = rltk::RandomNumberGenerator::new();

    for _i in 0..400 {
        let x = rng.roll_dice(1, 79);
        let y = rng.roll_dice(1, 49);
        let idx = xy_idx(x, y);
        if idx != xy_idx(40, 25) {
            map[idx] = TileType::Wall;
        }
    }

    map
}

fn try_move_player(delta_x: i32, delta_y: i32, ecs: &mut World) {
    // 尝试移动玩家
    let mut positions = ecs.write_storage::<Position>();
    let mut players = ecs.write_storage::<Player>();
    let map = ecs.fetch::<Vec<TileType>>();

    for (_player, pos) in (&mut players, &mut positions).join() {
        let destination_idx = xy_idx(pos.x + delta_x, pos.y + delta_y);
        if map[destination_idx] != TileType::Wall {
            pos.x = min(79 , max(0, pos.x + delta_x));
            pos.y = min(49, max(0, pos.y + delta_y));
        }
    }
}

fn player_input(gs: &mut State, ctx: &mut Rltk) {
    // 处理玩家输入
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

fn draw_map(map: &[TileType], ctx : &mut Rltk) {
    // 绘制地图
    let mut y = 0;
    let mut x = 0;
    for tile in map.iter() {
        // 根据瓦片类型渲染瓦片
        match tile {
            TileType::Floor => {
                ctx.set(x, y, RGB::from_f32(0.5, 0.5, 0.5), RGB::from_f32(0., 0., 0.), rltk::to_cp437('.'));
            }
            TileType::Wall => {
                ctx.set(x, y, RGB::from_f32(0.0, 1.0, 0.0), RGB::from_f32(0., 0., 0.), rltk::to_cp437('#'));
            }
        }

        // 移动坐标
        x += 1;
        if x > 79 {
            x = 0;
            y += 1;
        }
    }
}

// 定义一个名为 State 的结构体，实现 GameState trait
impl GameState for State {
    // 定义 tick 方法，用于处理每一帧的逻辑
    fn tick(&mut self, ctx : &mut Rltk) {
        // 清除屏幕
        ctx.cls();

        // 处理玩家输入
        player_input(self, ctx);
        // 运行游戏系统
        self.run_systems();

        // 从 ECS 中获取地图数据
        let map = self.ecs.fetch::<Vec<TileType>>();
        // 绘制地图
        draw_map(&map, ctx);

        // 从 ECS 中读取 Position 和 Renderable 组件
        let positions = self.ecs.read_storage::<Position>();
        let renderables = self.ecs.read_storage::<Renderable>();

        // 遍历所有具有 Position 和 Renderable 组件的实体
        for (pos, render) in (&positions, &renderables).join() {
            // 在屏幕上绘制实体
            ctx.set(pos.x, pos.y, render.fg, render.bg, render.glyph);
        }
    }
}

// 为 State 结构体实现 run_systems 方法
impl State {
    // 运行游戏系统
    fn run_systems(&mut self) {
        // 维护 ECS 世界
        self.ecs.maintain();
    }
}

// 主函数
fn main() -> rltk::BError {
    // 使用 RltkBuilder 创建一个简单的 80x50 的窗口
    use rltk::RltkBuilder;
    let context = RltkBuilder::simple80x50()
        .with_title("Roguelike Tutorial")
        .build()?;
    // 创建游戏状态
    let mut gs = State {
        ecs: World::new()
    };
    // 在 ECS 中注册 Position、Renderable 和 Player 组件
    gs.ecs.register::<Position>();
    gs.ecs.register::<Renderable>();
    gs.ecs.register::<Player>();

    // 在 ECS 中插入新的地图
    gs.ecs.insert(new_map());

    // 在 ECS 中创建一个玩家实体
    gs.ecs
        .create_entity()
        .with(Position { x: 40, y: 25 }) // 设置玩家位置
        .with(Renderable { // 设置玩家渲染信息
            glyph: rltk::to_cp437('@'),
            fg: RGB::named(rltk::YELLOW),
            bg: RGB::named(rltk::BLACK),
        })
        .with(Player{}) // 标记为玩家实体
        .build();

    // 进入游戏主循环
    rltk::main_loop(context, gs)
}
```

**本章的源代码可以在[这里](https://github.com/thebracket/rustrogueliketutorial/tree/master/chapter-03-walkmap)找到**

[在浏览器中用Web汇编运行本章的示例（需要WebGL2）](https://bfnightly.bracketproductions.com/rustbook/wasm/chapter-03-walkmap/)

---

版权所有 (C) 2019, Herbert Wolverson。

---

