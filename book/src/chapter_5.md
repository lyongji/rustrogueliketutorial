# 第五章 - 视野

---

***关于本教程***

*本教程是免费和开源的，所有代码均使用MIT许可证 - 因此您可以自由地使用它。我希望您会喜欢这个教程，并制作出伟大的游戏！*

*如果您喜欢这个教程并希望我继续写作，请考虑支持[我的Patreon](https://www.patreon.com/blackfuture)。*

[![实践Rust](./beta-webBanner.jpg)](https://pragprog.com/titles/hwrust/hands-on-rust/)

---

我们有一个漂亮绘制的地图，但它显示了整个地牢！这减少了探索的有用性 - 如果我们已经知道一切所在，为什么还要费力探索？本章将添加“视野”，并调整渲染以显示我们已经发现的地图部分。它还将重构地图为其自己的结构，而不是仅仅一个瓦片向量。

本章从第4章的代码开始。

# 地图重构

我们将地图相关的函数和数据放在一起，以保持我们在制作越来越复杂的游戏时的清晰度。这部分工作的重点是创建一个新的`Map`结构，并将我们的辅助函数移动到它的实现中。
```rust
use rltk::{ RGB, Rltk, RandomNumberGenerator };
use super::{Rect};
use std::cmp::{max, min};

#[derive(PartialEq, Copy, Clone)]
pub enum TileType {
    Wall, Floor
}

pub struct Map {
    pub tiles : Vec<TileType>,
    pub rooms : Vec<Rect>,
    pub width : i32,
    pub height : i32
}

impl Map {
    pub fn xy_idx(&self, x: i32, y: i32) -> usize {
        (y as usize * self.width as usize) + x as usize
    }

    fn apply_room_to_map(&mut self, room : &Rect) {
        for y in room.y1 +1 ..= room.y2 {
            for x in room.x1 + 1 ..= room.x2 {
                let idx = self.xy_idx(x, y);
                self.tiles[idx] = TileType::Floor;
            }
        }
    }

    fn apply_horizontal_tunnel(&mut self, x1:i32, x2:i32, y:i32) {
        for x in min(x1,x2) ..= max(x1,x2) {
            let idx = self.xy_idx(x, y);
            if idx > 0 && idx < self.width as usize * self.height as usize {
                self.tiles[idx as usize] = TileType::Floor;
            }
        }
    }

    fn apply_vertical_tunnel(&mut self, y1:i32, y2:i32, x:i32) {
        for y in min(y1,y2) ..= max(y1,y2) {
            let idx = self.xy_idx(x, y);
            if idx > 0 && idx < self.width as usize * self.height as usize {
                self.tiles[idx as usize] = TileType::Floor;
            }
        }
    }

    /// 使用来自http://rogueliketutorials.com/tutorials/tcod/part-3/的算法创建新地图
    /// 这会生成一些随机房间和连接它们的走廊。
    pub fn new_map_rooms_and_corridors() -> Map {
        let mut map = Map{
            tiles : vec![TileType::Wall; 80*50],
            rooms : Vec::new(),
            width : 80,
            height: 50
        };

        const MAX_ROOMS : i32 = 30;
        const MIN_SIZE : i32 = 6;
        const MAX_SIZE : i32 = 10;

        let mut rng = RandomNumberGenerator::new();

        for i in 0..MAX_ROOMS {
            let w = rng.range(MIN_SIZE, MAX_SIZE);
            let h = rng.range(MIN_SIZE, MAX_SIZE);
            let x = rng.roll_dice(1, map.width - w - 1) - 1;
            let y = rng.roll_dice(1, map.height - h - 1) - 1;
            let new_room = Rect::new(x, y, w, h);
            let mut ok = true;
            for other_room in map.rooms.iter() {
                if new_room.intersect(other_room) { ok = false }
            }
            if ok {
                map.apply_room_to_map(&new_room);

                if !map.rooms.is_empty() {
                    let (new_x, new_y) = new_room.center();
                    let (prev_x, prev_y) = map.rooms[map.rooms.len()-1].center();
                    if rng.range(0,2) == 1 {
                        map.apply_horizontal_tunnel(prev_x, new_x, prev_y);
                        map.apply_vertical_tunnel(prev_y, new_y, new_x);
                    } else {
                        map.apply_vertical_tunnel(prev_y, new_y, prev_x);
                        map.apply_horizontal_tunnel(prev_x, new_x, new_y);
                    }
                }

                map.rooms.push(new_room);
            }
        }

        map
    }
}
```

`main`和`player`也有一些变化 - 请参见示例源代码以获取所有详细信息。这大大清理了我们的代码 - 我们可以传递一个`Map`，而不是一个向量。如果我们想教`Map`做更多的事情 - 我们有了一个地方可以这样做。

# 视野组件

不仅玩家有视野限制！最终，我们也希望怪物能考虑它们能看到什么。因此，由于这是可重用的代码，我们将创建一个`Viewshed`组件。（我喜欢“视野”这个词；它来自地图学世界 - 字面上意思是“我从这里能看到什么？”- 完美地描述了我们的问题）。我们将给每个拥有*Viewshed*的实体一个它们能看到的瓦片索引列表。在`components.rs`中我们添加：
```rust
#[derive(Component)]
pub struct Viewshed {
    pub visible_tiles : Vec<rltk::Point>,
    pub range : i32
}
```
在`main.rs`中，我们告诉系统关于这个新组件：
```rust
gs.ecs.register::<Viewshed>();
```
最后，在`main.rs`中，我们也给`Player`一个`Viewshed`组件：
```rust
gs.ecs
    .create_entity()
    .with(Position { x: player_x, y: player_y })
    .with(Renderable {
        glyph: rltk::to_cp437('@'),
        fg: RGB::named(rltk::YELLOW),
        bg: RGB::named(rltk::BLACK),
    })
    .with(Player{})
    .with(Viewshed{ visible_tiles : Vec::new(), range : 8 })
    .build();
```
玩家现在变得越来越复杂了 - 这是好事，它显示了ECS的用途！

# 一个新系统：通用视野

我们将首先定义一个*系统*来为我们处理这个问题。我们希望这个系统是通用的，所以它适用于任何可以从知道它能看到什么中受益的东西。我们创建一个新的文件，`visibility_system.rs`：
```rust
use specs::prelude::*;
use super::{Viewshed, Position};

pub struct VisibilitySystem {}

impl<'a> System<'a> for VisibilitySystem {
    type SystemData = ( WriteStorage<'a, Viewshed>, 
                        WriteStorage<'a, Position>);

    fn run(&mut self, (mut viewshed, pos) : Self::SystemData) {
        for (viewshed,pos) in (&mut viewshed, &pos).join() {
        }
    }
}
```
现在我们必须调整`main.rs`中的`run_systems`来实际调用系统：
```rust
impl State {
    fn run_systems(&mut self) {
        let mut vis = VisibilitySystem{};
        vis.run_now(&self.ecs);
        self.ecs.maintain();
    }
}
```
我们还必须告诉`main.rs`使用新模块：
```rust
mod visibility_system;
use visibility_system::VisibilitySystem;
```
这实际上*什么也不做*，但我们已经将一个系统添加到调度器中，一旦我们充实实际绘制视野的代码，它将适用于每个同时具有*Viewshed*和*Position*组件的实体。

# 向RLTK请求视野：特质实现

RLTK的设计不关心你选择如何布局你的地图：我希望它对任何人都有用，不是每个人都按照本教程的方式制作地图。为了在我们的地图实现和RLTK之间搭建桥梁，它提供了一些*特质*供我们支持。在这个例子中，我们需要`BaseMap`和`Algorithm2D`。别担心，它们足够简单，容易实现。

在我们的`map.rs`文件中，我们添加了以下内容：
```rust
impl Algorithm2D for Map {
    fn dimensions(&self) -> Point {
        Point::new(self.width, self.height)
    }
}
```
RLTK能够从`dimensions`函数中推断出许多其他特质：点索引（及其逆运算）、边界检查等。我们返回我们已经使用的尺寸，`self.width`和`self.height`。

我们还需要支持`BaseMap`。我们目前不需要全部功能，所以我们将使用默认设置。在`map.rs`中：
```rust
impl BaseMap for Map {
    fn is_opaque(&self, idx:usize) -> bool {
        self.tiles[idx as usize] == TileType::Wall
    }
}
```
`is_opaque`简单地返回瓦片是否为墙，是则返回true，否则返回false。如果我们添加更多类型的瓦片，这将需要扩展，但目前这样可以工作。我们现在将默认留下特质的其他部分（因此不需要输入任何其他内容）。

# 向RLTK请求视野：系统

所以回到`visibility_system.rs`，我们现在有向RLTK请求视野所需的一切。我们扩展我们的`visibility_system.rs`文件，使其看起来像这样：

```rust
use specs::prelude::*;
use super::{Viewshed, Position, Map};
use rltk::{field_of_view, Point};

pub struct VisibilitySystem {}

impl<'a> System<'a> for VisibilitySystem {
    type SystemData = ( ReadExpect<'a, Map>,
                        WriteStorage<'a, Viewshed>, 
                        WriteStorage<'a, Position>);

    fn run(&mut self, data : Self::SystemData) {
        let (map, mut viewshed, pos) = data;

        for (viewshed,pos) in (&mut viewshed, &pos).join() {
            viewshed.visible_tiles.clear();
            viewshed.visible_tiles = field_of_view(Point::new(pos.x, pos.y), viewshed.range, &*map);
            viewshed.visible_tiles.retain(|p| p.x >= 0 && p.x < map.width && p.y >= 0 && p.y < map.height );
        }
    }
}
```

这里有很多内容，而视野实际上是其中最简单的部分：

* 我们添加了一个`ReadExpect<'a, Map>` - 这意味着系统应该传递我们的`Map`以供使用。我们使用`ReadExpect`，因为没有地图是一种失败。
* 在循环中，我们首先清除可见瓦片的列表。
* 然后我们调用RLTK的`field_of_view`函数，提供起始点（实体的位置，来自`pos`），范围（来自视野），以及一个稍微复杂的“解引用，然后获取引用”来从ECS中解包`Map`。
* 最后，我们使用向量的`retain`方法删除任何不符合我们指定条件的条目。这是一个*lambda*或*closure* - 它遍历向量，传递`p`作为参数。如果`p`在地图边界内，我们保留它。这防止其他函数尝试访问工作地图区域之外的瓦片。

这将现在每帧运行一次（这有些过度，稍后会更多关于这个）- 并存储一个可见瓦片的列表。

# 渲染视野 - 非常糟糕！

作为一个初步尝试，我们将更改我们的`draw_map`函数以检索地图和玩家的视野。它只会绘制视野中的瓦片：
```rust
pub fn draw_map(ecs: &World, ctx : &mut Rltk) {
    let mut viewsheds = ecs.write_storage::<Viewshed>();
    let mut players = ecs.write_storage::<Player>();
    let map = ecs.fetch::<Map>();

    for (_player, viewshed) in (&mut players, &mut viewsheds).join() {
        let mut y = 0;
        let mut x = 0;
        for tile in map.tiles.iter() {
            // 根据瓦片类型渲染瓦片
            let pt = Point::new(x,y);
            if viewshed.visible_tiles.contains(&pt) {
                match tile {
                    TileType::Floor => {
                        ctx.set(x, y, RGB::from_f32(0.5, 0.5, 0.5), RGB::from_f32(0., 0., 0.), rltk::to_cp437('.'));
                    }
                    TileType::Wall => {
                        ctx.set(x, y, RGB::from_f32(0.0, 1.0, 0.0), RGB::from_f32(0., 0., 0.), rltk::to_cp437('#'));
                    }
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
}
```
如果你现在运行示例（`cargo run`），它将向你展示玩家能看到什么。没有记忆，性能非常糟糕 - 但它就在那里，而且大致正确。

很明显，我们正在正确的轨道上，但我们需要一种更有效的方法来做这件事。如果玩家能记住他们看到的东西，那将是非常好的。

# 扩展地图以包含已揭示的瓦片

为了模拟地图记忆，我们将扩展我们的`Map`类以包含一个`revealed_tiles`结构。它只是地图上每个瓦片的一个`bool`值 - 如果为true，则我们知道那里有什么。我们的`Map`定义现在看起来像这样：

```rust
#[derive(Default)]
pub struct Map {
    pub tiles : Vec<TileType>,
    pub rooms : Vec<Rect>,
    pub width : i32,
    pub height : i32,
    pub revealed_tiles : Vec<bool>
}
```

我们还需要扩展填充地图的函数以包含新类型。在`new_rooms_and_corridors`中，我们将地图创建扩展为：
```rust
let mut map = Map{
    tiles : vec![TileType::Wall; 80*50],
    rooms : Vec::new(),
    width : 80,
    height: 50,
    revealed_tiles : vec![false; 80*50]
};
```
这将为每个瓦片添加一个`false`值。

我们修改了`draw_map`函数，使其查看这个值，而不是每次都迭代组件。现在，该函数看起来像这样：
```rust
pub fn draw_map(ecs: &World, ctx : &mut Rltk) {
    let map = ecs.fetch::<Map>();

    let mut y = 0;
    let mut x = 0;
    for (idx,tile) in map.tiles.iter().enumerate() {
        // 根据瓦片类型渲染瓦片
        if map.revealed_tiles[idx] {
            match tile {
                TileType::Floor => {
                    ctx.set(x, y, RGB::from_f32(0.5, 0.5, 0.5), RGB::from_f32(0., 0., 0.), rltk::to_cp437('.'));
                }
                TileType::Wall => {
                    ctx.set(x, y, RGB::from_f32(0.0, 1.0, 0.0), RGB::from_f32(0., 0., 0.), rltk::to_cp437('#'));
                }
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
```
这将渲染一个黑色屏幕，因为我们从未将任何瓦片设置为已揭示！所以现在我们扩展了`VisibilitySystem`，使其知道如何标记瓦片为已揭示。为此，它需要检查一个实体是否是玩家 - 如果是，它会更新地图的已揭示状态：

```rust
use specs::prelude::*;
use super::{Viewshed, Position, Map, Player};
use rltk::{field_of_view, Point};

pub struct VisibilitySystem {}

impl<'a> System<'a> for VisibilitySystem {
    type SystemData = ( WriteExpect<'a, Map>,
                        Entities<'a>,
                        WriteStorage<'a, Viewshed>, 
                        WriteStorage<'a, Position>,
                        ReadStorage<'a, Player>);

    fn run(&mut self, data : Self::SystemData) {
        let (mut map, entities, mut viewshed, pos, player) = data;

        for (ent,viewshed,pos) in (&entities, &mut viewshed, &pos).join() {
            viewshed.visible_tiles.clear();
            viewshed.visible_tiles = field_of_view(Point::new(pos.x, pos.y), viewshed.range, &*map);
            viewshed.visible_tiles.retain(|p| p.x >= 0 && p.x < map.width && p.y >= 0 && p.y < map.height );
                
            //如果这是玩家，显示他们可以看到什么
            let p : Option<&Player> = player.get(ent);
            if let Some(p) = p {
                for vis in viewshed.visible_tiles.iter() {
                    let idx = map.xy_idx(vis.x, vis.y);
                    map.revealed_tiles[idx] = true;
                }
            }
        }
    }
}
```

主要的改动是我们获取了实体列表以及组件，并获得了对玩家存储的只读访问。我们将这些添加到要迭代的列表中，并添加了`let p : Option<&Player> = player.get(ent);`来检查这是否是玩家。较为神秘的`if let Some(p) = p`只有在存在`Player`组件时才运行。然后我们计算索引，并标记为已揭示。

如果你现在运行（`cargo run`），它将比之前的版本快得多，并且能记住你已经去过的地方。

# 进一步加快速度 - 仅在需要时重新计算视野

但它仍然不是最有效的！让我们只在需要时更新视野。我们在`Viewshed`组件中添加一个`dirty`标志：
```rust
#[derive(Component)]
pub struct Viewshed {
    pub visible_tiles : Vec<rltk::Point>,
    pub range : i32,
    pub dirty : bool
}
```
我们还需要更新`main.rs`中的初始化，以表示视野实际上是脏的：`.with(Viewshed{ visible_tiles : Vec::new(), range: 8, dirty: true })`。

我们可以扩展系统来检查`dirty`标志是否为真，并且只有在为真时才重新计算 - 完成后，将`dirty`标志设置为false。现在我们需要在玩家移动时设置标志 - 因为他们能看到的东西已经改变了！我们在`player.rs`中更新`try_move_player`：
```rust
pub fn try_move_player(delta_x: i32, delta_y: i32, ecs: &mut World) {
    let mut positions = ecs.write_storage::<Position>();
    let mut players = ecs.write_storage::<Player>();
    let mut viewsheds = ecs.write_storage::<Viewshed>();
    let map = ecs.fetch::<Map>();

    for (_player, pos, viewshed) in (&mut players, &mut positions, &mut viewsheds).join() {
        let destination_idx = map.xy_idx(pos.x + delta_x, pos.y + delta_y);
        if map.tiles[destination_idx] != TileType::Wall {
            pos.x = min(79 , max(0, pos.x + delta_x));
            pos.y = min(49, max(0, pos.y + delta_y));

            viewshed.dirty = true;
        }
    }
}
```
这应该已经很熟悉了：我们添加了`viewsheds`来获取写存储，并将其包含在我们正在迭代的组件类型列表中。然后在一个调用中将标志设置为`true`。

如果你输入`cargo run`，游戏现在再次运行得非常快。

# 将我们记得但看不见的部分灰显

最后一个扩展：我们希望渲染我们知道的地图部分，但当前看不见的部分。所以我们在`Map`中添加了一个当前可见的瓦片列表：

```rust
#[derive(Default)]
pub struct Map {
    pub tiles : Vec<TileType>,
    pub rooms : Vec<Rect>,
    pub width : i32,
    pub height : i32,
    pub revealed_tiles : Vec<bool>,
    pub visible_tiles : Vec<bool>
}
```

我们的创建方法也需要像之前那样将所有值设置为false：`visible_tiles : vec![false; 80*50]`。接下来，在我们的`VisibilitySystem`中，我们在开始迭代之前清除可见瓦片的列表 - 并在我们找到它们时标记当前可见的瓦片。因此，当我们更新视野时运行的代码如下：
```rust
if viewshed.dirty {
    viewshed.dirty = false;
    viewshed.visible_tiles.clear();
    viewshed.visible_tiles = field_of_view(Point::new(pos.x, pos.y), viewshed.range, &*map);
    viewshed.visible_tiles.retain(|p| p.x >= 0 && p.x < map.width && p.y >= 0 && p.y < map.height );

    // 如果这是玩家，显示他们能看到什么
    let _p : Option<&Player> = player.get(ent);
    if let Some(_p) = _p {
        for t in map.visible_tiles.iter_mut() { *t = false };
        for vis in viewshed.visible_tiles.iter() {
            let idx = map.xy_idx(vis.x, vis.y);
            map.revealed_tiles[idx] = true;
            map.visible_tiles[idx] = true;
        }
    }
}
```
现在我们调整`draw_map`函数来不同地处理已揭示但当前不可见的瓦片。新的`draw_map`函数如下所示：
```rust
pub fn draw_map(ecs: &World, ctx : &mut Rltk) {
    let map = ecs.fetch::<Map>();

    let mut y = 0;
    let mut x = 0;
    for (idx,tile) in map.tiles.iter().enumerate() {
        // 根据瓦片类型渲染瓦片

        if map.revealed_tiles[idx] {
            let glyph;
            let mut fg;
            match tile {
                TileType::Floor => {
                    glyph = rltk::to_cp437('.');
                    fg = RGB::from_f32(0.0, 0.5, 0.5);
                }
                TileType::Wall => {
                    glyph = rltk::to_cp437('#');
                    fg = RGB::from_f32(0., 1.0, 0.);
                }
            }
            if !map.visible_tiles[idx] { fg = fg.to_greyscale() }
            ctx.set(x, y, fg, RGB::from_f32(0., 0., 0.), glyph);
        }

        // 移动坐标
        x += 1;
        if x > 79 {
            x = 0;
            y += 1;
        }
    }
}
```
如果您`cargo run`您的项目，您现在将拥有稍微呈青色的可见地板和绿色墙壁 - 以及移出视野时的灰色。性能应该很好！恭喜 - 您现在拥有一个不错的工作视野系统。

![Screenshot](./c5-s1.gif)

**The source code for this chapter may be found [here](https://github.com/thebracket/rustrogueliketutorial/tree/master/chapter-05-fov)**

[Run this chapter's example with web assembly, in your browser (WebGL2 required)](https://bfnightly.bracketproductions.com/rustbook/wasm/chapter-05-fov/)

---

Copyright (C) 2019, Herbert Wolverson.

---
