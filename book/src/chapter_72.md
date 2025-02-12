# Text Layers

---

***About this tutorial***

*This tutorial is free and open source, and all code uses the MIT license - so you are free to do with it as you like. My hope is that you will enjoy the tutorial, and make great games!*

*If you enjoy this and would like me to keep writing, please consider supporting [my Patreon](https://www.patreon.com/blackfuture).*

[![Hands-On Rust](./beta-webBanner.jpg)](https://pragprog.com/titles/hwrust/hands-on-rust/)

---

默认的8x8字体在处理大量文本时可能会变得难以阅读，尤其是在结合后处理效果时。RLTK的图形控制台模式（基本上除了`curses`之外的所有模式）支持在同一屏幕上显示多个控制台，可以选择不同的字体。RLTK附带了一个VGA字体（8x16），这要容易阅读得多。我们将使用它，*但仅用于日志*。

使用VGA字体的第二层进行初始化很简单（参见RLTK示例2的详细信息）。扩展`main.rs`中的构建器代码：
```rust
let mut context = RltkBuilder::simple(80, 60)
    .with_title("Roguelike 教程")
    .with_font("vga8x16.png", 8, 16)
    .with_sparse_console(80, 30, "vga8x16.png")
    .build()?;
```
主循环中的“清除屏幕”需要扩展以清除两层。在`main.rs`（`tick`函数中），我们有一段代码已经70章没有碰过了 - 在帧开始时清除屏幕。现在我们想要清除两个控制台：
```rust
ctx.set_active_console(1);
ctx.cls();
ctx.set_active_console(0);
ctx.cls();
```
我在使用`TextBlock`组件和多个控制台时遇到了一些问题，所以写了一个替代方案。在`src/gamelog/logstore.rs`中，我们移除了`display_log`函数并添加了一个替代方案：
```rust
pub fn print_log(console: &mut Box<dyn Console>, pos: Point) {
    let mut y = pos.y;
    let mut x = pos.x;
    LOG.lock().unwrap().iter().rev().take(6).for_each(|log| {
        log.iter().for_each(|frag| {
            console.print_color(x, y, frag.color, RGB::named(rltk::BLACK), &frag.text);
            x += frag.text.len() as i32;
            x += 1;
        });
        y += 1;
        x = pos.x;
    });
}
```
并在`src/gamelog/mod.rs`中更正导出：
```rust
pub use logstore::{clear_log, clone_log, restore_log, print_log};
```
由于新代码处理渲染，绘制日志文件非常容易！更改`gui.rs`中的日志渲染：
```rust
// 绘制日志
gamelog::print_log(&mut rltk::BACKEND_INTERNAL.lock().consoles[1].console, Point::new(1, 23));
```
如果你现在运行`cargo run`，你会看到一个更容易阅读的日志部分：

![c72-s1.jpg](c72-s1.jpg)

## 让我们清理GUI代码

既然我们正在处理GUI，现在是清理它的好时机。添加一些鼠标支持也是不错的。我们将从将`gui.rs`转换为多文件模块开始。它非常大，所以拆分它本身就是一个胜利！创建一个新的文件夹`src/gui`并将`gui.rs`文件*移动*到其中。然后将该文件重命名为`mod.rs`。游戏将像以前一样工作。

然后我们进行一些调整：

* 创建一个新文件`gui/item_render.rs`。在`gui/mod.rs`中添加`mod item_render; pub use item_render::*;`，并将函数`get_item_color`和`get_item_display_name`移动到其中。
* RLTK现在支持绘制空心框，所以我们可以删除`draw_hollow_box`函数。将`draw_hollow_box(ctx, ...)`的调用替换为`ctx.draw_hollow_box(...)`。
* 创建一个新文件`gui/hud.rs`。在`gui/mod.rs`中添加`mod hud; pub use hud::*;`，并将以下函数移动到其中：`draw_attribute`，`draw_ui`。
* 创建一个新文件`gui/tooltips.rs`。在`gui/mod.rs`中添加`mod tooltips; pub use tooltips::*;`，并将`Tooltip`结构体和实现移动到其中，以及函数`draw_tooltips`。你需要将这个函数设为`pub`。
* 创建一个新文件`gui/inventory_menu.rs`。在`gui/mod.rs`中添加`mod inventory_menu; pub use inventory_menu::*;`，并将物品菜单代码移动到其中。
* 对于物品丢弃也是一样的。创建`gui/drop_item_menu.rs`，在`mod.rs`中添加`mod drop_item_menu; pub use drop_item_menu::*;`，并将物品丢弃菜单移动过去。
* 重复上述步骤，创建`gui/remove_item_menu.rs`和移动物品代码。
* 再次重复，创建`gui/remove_curse_menu.rs`。
* 再次 - 这次是`gui/identify_menu.rs`，`gui/ranged_target.rs`，`gui/main_menu.rs`，`gui/game_over_menu.rs`，`gui/cheat_menu.rs`和`gui/vendor_menu.rs`。

还有很多导入清理工作。如果你不确定需要什么，我建议参考[源代码](https://github.com/thebracket/rustrogueliketutorial/tree/master/chapter-72-textlayers)。一旦完成所有这些，`gui/mod.rs`将不包含任何功能：只是指向各个文件的指针。

游戏应该像以前一样运行：但是你的编译时间有所改善（特别是在增量构建中）！

## 在我们清理的同时 - 相机

有几章我一直觉得`camera.rs`不在`map`模块中是个问题。让我们把它移过去。将文件移动到`map`文件夹中。在`map/mod.rs`中添加一行`pub mod camera;`。这留下了一些需要清理的引用：

* 从`main.rs`中删除`pub mod camera;`。
* 在`map/camera.rs`中将`use super::`更改为`use crate::`。

## 批处理渲染

RLTK最近获得了新的渲染功能：能够批处理渲染。这使得渲染与系统兼容（你不能将RLTK作为资源添加，它有很多线程不安全的功能）。我们不会在本章中处理系统，但我们将切换到新的渲染路径。它稍微快一些，整体上更干净。好消息是你可以大量混合使用这两种风格，在切换过程中。

从启用系统开始。在`main.rs`的`tick`末尾添加一行：
```rust
rltk::render_draw_buffer(ctx);
```
这告诉RLTK将任何累积的绘图缓冲区提交到屏幕。通过首先添加这一行，我们确保我们切换的任何内容都将被渲染。

### 批处理相机

打开`map/camera.rs`文件。将`use rltk::`替换为`use rltk::prelude::*;`。现在RLTK支持预导入模块，我们应该使用它！然后，在`render_camera`函数的第一行添加以下内容：
```rust
let mut draw_batch = DrawBatch::new();
```
这将请求RLTK创建一个新的“绘制批次”。这些是高性能的、池化的对象，可以收集绘图指令并一次性提交。这非常有利于缓存，通常可以显著提高性能。

将第一个`set`命令替换为`draw_batch.set`：

```rust
// FROM
ctx.set(x as i32+1, y as i32+1, fg, bg, glyph);
// TO
draw_batch.set(
    Point::new(x+1, y+1),
    ColorPair::new(fg, bg),
    glyph
);
```

你需要逐步进行，并对所有的绘图调用进行相同的更改。在最后添加一行新的代码：
```rust
draw_batch.submit(0);
```
这将地图渲染作为一个批次提交。完成后的函数如下所示：

```rust
pub fn render_camera(ecs: &World, ctx : &mut Rltk) {
    let mut draw_batch = DrawBatch::new();
    let map = ecs.fetch::<Map>();
    let (min_x, max_x, min_y, max_y) = get_screen_bounds(ecs, ctx);

    // Render the Map

    let map_width = map.width-1;
    let map_height = map.height-1;

    for (y,ty) in (min_y .. max_y).enumerate() {
        for (x,tx) in (min_x .. max_x).enumerate() {
            if tx > 0 && tx < map_width && ty > 0 && ty < map_height {
                let idx = map.xy_idx(tx, ty);
                if map.revealed_tiles[idx] {
                    let (glyph, fg, bg) = tile_glyph(idx, &*map);
                    draw_batch.set(
                        Point::new(x+1, y+1),
                        ColorPair::new(fg, bg),
                        glyph
                    );
                }
            } else if SHOW_BOUNDARIES {
                draw_batch.set(
                    Point::new(x+1, y+1),
                    ColorPair::new(RGB::named(rltk::GRAY), RGB::named(rltk::BLACK)),
                    to_cp437('·')
                );
            }
        }
    }

    // Render entities
    let positions = ecs.read_storage::<Position>();
    let renderables = ecs.read_storage::<Renderable>();
    let hidden = ecs.read_storage::<Hidden>();
    let map = ecs.fetch::<Map>();
    let sizes = ecs.read_storage::<TileSize>();
    let entities = ecs.entities();
    let targets = ecs.read_storage::<Target>();

    let mut data = (&positions, &renderables, &entities, !&hidden).join().collect::<Vec<_>>();
    data.sort_by(|&a, &b| b.1.render_order.cmp(&a.1.render_order) );
    for (pos, render, entity, _hidden) in data.iter() {
        if let Some(size) = sizes.get(*entity) {
            for cy in 0 .. size.y {
                for cx in 0 .. size.x {
                    let tile_x = cx + pos.x;
                    let tile_y = cy + pos.y;
                    let idx = map.xy_idx(tile_x, tile_y);
                    if map.visible_tiles[idx] {
                        let entity_screen_x = (cx + pos.x) - min_x;
                        let entity_screen_y = (cy + pos.y) - min_y;
                        if entity_screen_x > 0 && entity_screen_x < map_width && entity_screen_y > 0 && entity_screen_y < map_height {
                            draw_batch.set(
                                Point::new(entity_screen_x + 1, entity_screen_y + 1),
                                ColorPair::new(render.fg, render.bg),
                                render.glyph
                            );
                        }
                    }
                }
            }
        } else {
            let idx = map.xy_idx(pos.x, pos.y);
            if map.visible_tiles[idx] {
                let entity_screen_x = pos.x - min_x;
                let entity_screen_y = pos.y - min_y;
                if entity_screen_x > 0 && entity_screen_x < map_width && entity_screen_y > 0 && entity_screen_y < map_height {
                    draw_batch.set(
                        Point::new(entity_screen_x + 1, entity_screen_y + 1),
                        ColorPair::new(render.fg, render.bg),
                        render.glyph
                    );
                }
            }
        }

        if targets.get(*entity).is_some() {
            let entity_screen_x = pos.x - min_x;
            let entity_screen_y = pos.y - min_y;
            draw_batch.set(
                Point::new(entity_screen_x , entity_screen_y + 1),
                ColorPair::new(RGB::named(rltk::RED), RGB::named(rltk::YELLOW)),
                to_cp437('[')
            );
            draw_batch.set(
                Point::new(entity_screen_x +2, entity_screen_y + 1),
                ColorPair::new(RGB::named(rltk::RED), RGB::named(rltk::YELLOW)),
                to_cp437(']')
            );
        }
    }

    draw_batch.submit(0);
}
```

如果你现在运行`cargo run`，它大部分与之前相同：但通常出现在地图上方的工具提示是不可见的（它们在下面，因为我们最后提交了）。

## 批处理GUI

我们将从`gui/hud.rs`开始，因为这是最混乱的部分！在文件开头添加`let mut draw_batch = DrawBatch::new();`，并在结尾添加`draw_batch.submit(5000);`。为什么是5000？地图上有80x60（4800）个可能的瓦片。提供的数字充当排序：所以我们保证在地图之后绘制GUI。然后，将所有的`ctx`调用转换为等效的批处理调用。同时，这也是将庞大的`draw_gui`函数拆分成更小的部分的好时机。完全重构的`gui/hud.rs`看起来像这样：

```rust
use rltk::prelude::*;
use specs::prelude::*;
use crate::{Pools, Map, Name, InBackpack,
    Equipped, HungerClock, HungerState, Attributes, Attribute, Consumable,
    StatusEffect, Duration, KnownSpells, Weapon, gamelog };
use super::{draw_tooltips, get_item_display_name, get_item_color};

fn draw_attribute(name : &str, attribute : &Attribute, y : i32, draw_batch: &mut DrawBatch) {
    let black = RGB::named(rltk::BLACK);
    let attr_gray : RGB = RGB::from_hex("#CCCCCC").expect("Oops");
    draw_batch.print_color(Point::new(50, y), name, ColorPair::new(attr_gray, black));
    let color : RGB =
        if attribute.modifiers < 0 { RGB::from_f32(1.0, 0.0, 0.0) }
        else if attribute.modifiers == 0 { RGB::named(rltk::WHITE) }
        else { RGB::from_f32(0.0, 1.0, 0.0) };
    draw_batch.print_color(Point::new(67, y), &format!("{}", attribute.base + attribute.modifiers), ColorPair::new(color, black));
    draw_batch.print_color(Point::new(73, y), &format!("{}", attribute.bonus), ColorPair::new(color, black));
    if attribute.bonus > 0 { 
        draw_batch.set(Point::new(72, y), ColorPair::new(color, black), to_cp437('+')); 
    }
}

fn box_framework(draw_batch : &mut DrawBatch) {
    let box_gray : RGB = RGB::from_hex("#999999").expect("Oops");
    let black = RGB::named(rltk::BLACK);

    draw_batch.draw_hollow_box(Rect::with_size(0, 0, 79, 59), ColorPair::new(box_gray, black)); // Overall box
    draw_batch.draw_hollow_box(Rect::with_size(0, 0, 49, 45), ColorPair::new(box_gray, black)); // Map box
    draw_batch.draw_hollow_box(Rect::with_size(0, 45, 79, 14), ColorPair::new(box_gray, black)); // Log box
    draw_batch.draw_hollow_box(Rect::with_size(49, 0, 30, 8), ColorPair::new(box_gray, black)); // Top-right panel

    // Draw box connectors
    draw_batch.set(Point::new(0, 45), ColorPair::new(box_gray, black), to_cp437('├'));
    draw_batch.set(Point::new(49, 8), ColorPair::new(box_gray, black), to_cp437('├'));
    draw_batch.set(Point::new(49, 0), ColorPair::new(box_gray, black), to_cp437('┬'));
    draw_batch.set(Point::new(49, 45), ColorPair::new(box_gray, black), to_cp437('┴'));
    draw_batch.set(Point::new(79, 8), ColorPair::new(box_gray, black), to_cp437('┤'));
    draw_batch.set(Point::new(79, 45), ColorPair::new(box_gray, black), to_cp437('┤'));
}

pub fn map_label(ecs: &World, draw_batch: &mut DrawBatch) {
    let box_gray : RGB = RGB::from_hex("#999999").expect("Oops");
    let black = RGB::named(rltk::BLACK);
    let white = RGB::named(rltk::WHITE);

    let map = ecs.fetch::<Map>();
    let name_length = map.name.len() + 2;
    let x_pos = (22 - (name_length / 2)) as i32;
    draw_batch.set(Point::new(x_pos, 0), ColorPair::new(box_gray, black), to_cp437('┤'));
    draw_batch.set(Point::new(x_pos + name_length as i32 - 1, 0), ColorPair::new(box_gray, black), to_cp437('├'));
    draw_batch.print_color(Point::new(x_pos+1, 0), &map.name, ColorPair::new(white, black));
}

fn draw_stats(ecs: &World, draw_batch: &mut DrawBatch, player_entity: &Entity) {
    let black = RGB::named(rltk::BLACK);
    let white = RGB::named(rltk::WHITE);
    let pools = ecs.read_storage::<Pools>();
    let player_pools = pools.get(*player_entity).unwrap();
    let health = format!("Health: {}/{}", player_pools.hit_points.current, player_pools.hit_points.max);
    let mana =   format!("Mana:   {}/{}", player_pools.mana.current, player_pools.mana.max);
    let xp =     format!("Level:  {}", player_pools.level);
    draw_batch.print_color(Point::new(50, 1), &health, ColorPair::new(white, black));
    draw_batch.print_color(Point::new(50, 2), &mana, ColorPair::new(white, black));
    draw_batch.print_color(Point::new(50, 3), &xp, ColorPair::new(white, black));
    draw_batch.bar_horizontal(
        Point::new(64, 1), 
        14, 
        player_pools.hit_points.current, 
        player_pools.hit_points.max, 
        ColorPair::new(RGB::named(rltk::RED), RGB::named(rltk::BLACK))
    );
    draw_batch.bar_horizontal(
        Point::new(64, 2), 
        14, 
        player_pools.mana.current, 
        player_pools.mana.max, 
        ColorPair::new(RGB::named(rltk::BLUE), RGB::named(rltk::BLACK))
    );
    let xp_level_start = (player_pools.level-1) * 1000;
    draw_batch.bar_horizontal(
        Point::new(64, 3), 
        14, 
        player_pools.xp - xp_level_start, 
        1000, 
        ColorPair::new(RGB::named(rltk::GOLD), RGB::named(rltk::BLACK))
    );
}

fn draw_attributes(ecs: &World, draw_batch: &mut DrawBatch, player_entity: &Entity) {
    let attributes = ecs.read_storage::<Attributes>();
    let attr = attributes.get(*player_entity).unwrap();
    draw_attribute("Might:", &attr.might, 4, draw_batch);
    draw_attribute("Quickness:", &attr.quickness, 5, draw_batch);
    draw_attribute("Fitness:", &attr.fitness, 6, draw_batch);
    draw_attribute("Intelligence:", &attr.intelligence, 7, draw_batch);
}

fn initiative_weight(ecs: &World, draw_batch: &mut DrawBatch, player_entity: &Entity) {
    let attributes = ecs.read_storage::<Attributes>();
    let attr = attributes.get(*player_entity).unwrap();
    let black = RGB::named(rltk::BLACK);
    let white = RGB::named(rltk::WHITE);
    let pools = ecs.read_storage::<Pools>();
    let player_pools = pools.get(*player_entity).unwrap();
    draw_batch.print_color(
        Point::new(50, 9),
        &format!("{:.0} lbs ({} lbs max)",
            player_pools.total_weight,
            (attr.might.base + attr.might.modifiers) * 15
        ),
        ColorPair::new(white, black)
    );
    draw_batch.print_color(
        Point::new(50,10), 
        &format!("Initiative Penalty: {:.0}", player_pools.total_initiative_penalty),
        ColorPair::new(white, black)
    );
    draw_batch.print_color(
        Point::new(50,11), 
        &format!("Gold: {:.1}", player_pools.gold),
        ColorPair::new(RGB::named(rltk::GOLD), black)
    );
}

fn equipped(ecs: &World, draw_batch: &mut DrawBatch, player_entity: &Entity) -> i32 {
    let black = RGB::named(rltk::BLACK);
    let yellow = RGB::named(rltk::YELLOW);
    let mut y = 13;
    let entities = ecs.entities();
    let equipped = ecs.read_storage::<Equipped>();
    let weapon = ecs.read_storage::<Weapon>();
    for (entity, equipped_by) in (&entities, &equipped).join() {
        if equipped_by.owner == *player_entity {
            let name = get_item_display_name(ecs, entity);
            draw_batch.print_color(
                Point::new(50, y), 
                &name,
                ColorPair::new(get_item_color(ecs, entity), black));
            y += 1;

            if let Some(weapon) = weapon.get(entity) {
                let mut weapon_info = if weapon.damage_bonus < 0 {
                    format!("┤ {} ({}d{}{})", &name, weapon.damage_n_dice, weapon.damage_die_type, weapon.damage_bonus)
                } else if weapon.damage_bonus == 0 {
                    format!("┤ {} ({}d{})", &name, weapon.damage_n_dice, weapon.damage_die_type)
                } else {
                    format!("┤ {} ({}d{}+{})", &name, weapon.damage_n_dice, weapon.damage_die_type, weapon.damage_bonus)
                };

                if let Some(range) = weapon.range {
                    weapon_info += &format!(" (range: {}, F to fire, V cycle targets)", range);
                }
                weapon_info += " ├";
                draw_batch.print_color(
                    Point::new(3, 45),
                    &weapon_info,
                    ColorPair::new(yellow, black));
            }
        }
    }
    y
}

fn consumables(ecs: &World, draw_batch: &mut DrawBatch, player_entity: &Entity, mut y : i32) -> i32 {
    y += 1;
    let black = RGB::named(rltk::BLACK);
    let yellow = RGB::named(rltk::YELLOW);
    let entities = ecs.entities();
    let consumables = ecs.read_storage::<Consumable>();
    let backpack = ecs.read_storage::<InBackpack>();
    let mut index = 1;
    for (entity, carried_by, _consumable) in (&entities, &backpack, &consumables).join() {
        if carried_by.owner == *player_entity && index < 10 {
            draw_batch.print_color(
                Point::new(50, y), 
                &format!("↑{}", index),
                ColorPair::new(yellow, black)
            );
            draw_batch.print_color(
                Point::new(53, y), 
                &get_item_display_name(ecs, entity),
                ColorPair::new(get_item_color(ecs, entity), black)
            );
            y += 1;
            index += 1;
        }
    }
    y
}

fn spells(ecs: &World, draw_batch: &mut DrawBatch, player_entity: &Entity, mut y : i32) -> i32 {
    y += 1;
    let black = RGB::named(rltk::BLACK);
    let blue = RGB::named(rltk::CYAN);
    let known_spells_storage = ecs.read_storage::<KnownSpells>();
    let known_spells = &known_spells_storage.get(*player_entity).unwrap().spells;
    let mut index = 1;
    for spell in known_spells.iter() {
        draw_batch.print_color(
            Point::new(50, y),
            &format!("^{}", index),
            ColorPair::new(blue, black)
        );
        draw_batch.print_color(
            Point::new(53, y),
            &format!("{} ({})", &spell.display_name, spell.mana_cost),
            ColorPair::new(blue, black)
        );
        index += 1;
        y += 1;
    }
    y
}

fn status(ecs: &World, draw_batch: &mut DrawBatch, player_entity: &Entity) {
    let mut y = 44;
    let hunger = ecs.read_storage::<HungerClock>();
    let hc = hunger.get(*player_entity).unwrap();
    match hc.state {
        HungerState::WellFed => {
            draw_batch.print_color(
                Point::new(50, y), 
                "Well Fed",
                ColorPair::new(RGB::named(rltk::GREEN), RGB::named(rltk::BLACK))
            );
            y -= 1;
        }
        HungerState::Normal => {}
        HungerState::Hungry => {
            draw_batch.print_color(
                Point::new(50, y),
                "Hungry",
                ColorPair::new(RGB::named(rltk::ORANGE), RGB::named(rltk::BLACK))
            );
            y -= 1;
        }
        HungerState::Starving => {
            draw_batch.print_color(
                Point::new(50, y),
                "Starving",
                ColorPair::new(RGB::named(rltk::RED), RGB::named(rltk::BLACK))
            );
            y -= 1;
        }
    }
    let statuses = ecs.read_storage::<StatusEffect>();
    let durations = ecs.read_storage::<Duration>();
    let names = ecs.read_storage::<Name>();
    for (status, duration, name) in (&statuses, &durations, &names).join() {
        if status.target == *player_entity {
            draw_batch.print_color(
                Point::new(50, y),
                &format!("{} ({})", name.name, duration.turns),
                ColorPair::new(RGB::named(rltk::RED), RGB::named(rltk::BLACK)),
            );
            y -= 1;
        }
    }
}

pub fn draw_ui(ecs: &World, ctx : &mut Rltk) {
    let mut draw_batch = DrawBatch::new();
    let player_entity = ecs.fetch::<Entity>();

    box_framework(&mut draw_batch);
    map_label(ecs, &mut draw_batch);
    draw_stats(ecs, &mut draw_batch, &player_entity);
    draw_attributes(ecs, &mut draw_batch, &player_entity);
    initiative_weight(ecs, &mut draw_batch, &player_entity);
    let mut y = equipped(ecs, &mut draw_batch, &player_entity);
    y += consumables(ecs, &mut draw_batch, &player_entity, y);
    spells(ecs, &mut draw_batch, &player_entity, y);
    status(ecs, &mut draw_batch, &player_entity);
    gamelog::print_log(&mut rltk::BACKEND_INTERNAL.lock().consoles[1].console, Point::new(1, 23));
    draw_tooltips(ecs, ctx);

    draw_batch.submit(5000);
}
```

## 批处理菜单

我们的各种菜单之间有很多共享的功能，可以合并成辅助函数。考虑到批处理，我们首先构建一个新的模块`gui/menus.rs`来持有这些通用功能：

```rust
use rltk::prelude::*;

pub fn menu_box<T: ToString>(draw_batch: &mut DrawBatch, x: i32, y: i32, width: i32, title: T) {
    draw_batch.draw_box(
        Rect::with_size(x, y-2, 31, width), 
        ColorPair::new(RGB::named(rltk::WHITE), RGB::named(rltk::BLACK))
    );
    draw_batch.print_color(
        Point::new(18, y-2), 
        &title.to_string(),
        ColorPair::new(RGB::named(rltk::MAGENTA), RGB::named(rltk::BLACK))
    );
}

pub fn menu_option<T:ToString>(draw_batch: &mut DrawBatch, x: i32, y: i32, hotkey: rltk::FontCharType, text: T) {
    draw_batch.set(
        Point::new(x, y), 
        ColorPair::new(RGB::named(rltk::WHITE), RGB::named(rltk::BLACK)),
        rltk::to_cp437('(')
    );
    draw_batch.set(
        Point::new(x+1, y), 
        ColorPair::new(RGB::named(rltk::YELLOW), RGB::named(rltk::BLACK)),
        hotkey
    );
    draw_batch.set(
        Point::new(x+2, y), 
        ColorPair::new(RGB::named(rltk::WHITE), RGB::named(rltk::BLACK)), 
        rltk::to_cp437(')')
    );
    draw_batch.print_color(
        Point::new(x+5, y), 
        &text.to_string(),
        ColorPair::new(RGB::named(rltk::YELLOW), RGB::named(rltk::BLACK))
    );
}
```

不要忘记修改 `gui/mod.rs` 以暴露这些功能：
```rust
mod menus;
pub use menus::*;
```
### 作弊菜单

有了新的辅助函数，`gui/cheat_menur.rs` 文件的重构变得非常简单：

```rust
use rltk::prelude::*;
use crate::State;
use super::{menu_option, menu_box};

#[derive(PartialEq, Copy, Clone)]
pub enum CheatMenuResult { NoResponse, Cancel, TeleportToExit, Heal, Reveal, GodMode }

pub fn show_cheat_mode(_gs : &mut State, ctx : &mut Rltk) -> CheatMenuResult {
    let mut draw_batch = DrawBatch::new();
    let count = 4;
    let mut y = (25 - (count / 2)) as i32;
    menu_box(&mut draw_batch, 15, y, (count+3) as i32, "Cheating!");
    draw_batch.print_color(
        Point::new(18, y+count as i32+1),
        "ESCAPE to cancel",
        ColorPair::new(RGB::named(rltk::YELLOW), RGB::named(rltk::BLACK))
    );

    menu_option(&mut draw_batch, 17, y, rltk::to_cp437('T'), "Teleport to next level");
    y += 1;
    menu_option(&mut draw_batch, 17, y, rltk::to_cp437('H'), "Heal all wounds");
    y += 1;
    menu_option(&mut draw_batch, 17, y, rltk::to_cp437('R'), "Reveal the map");
    y += 1;
    menu_option(&mut draw_batch, 17, y, rltk::to_cp437('G'), "God Mode (No Death)");

    draw_batch.submit(6000);

    match ctx.key {
        None => CheatMenuResult::NoResponse,
        Some(key) => {
            match key {
                VirtualKeyCode::T => CheatMenuResult::TeleportToExit,
                VirtualKeyCode::H => CheatMenuResult::Heal,
                VirtualKeyCode::R => CheatMenuResult::Reveal,
                VirtualKeyCode::G => CheatMenuResult::GodMode,
                VirtualKeyCode::Escape => CheatMenuResult::Cancel,
                _ => CheatMenuResult::NoResponse
            }
        }
    }
}
```

### 丢弃物品菜单

对于各种物品菜单，另一个辅助函数有助于减少重复的代码。在`gui/menus.rs`中添加以下内容：

```rust
pub fn item_result_menu<S: ToString>(
    draw_batch: &mut DrawBatch,
    title: S,
    count: usize,
    items: &[(Entity, String)],
    key: Option<VirtualKeyCode>
) -> (ItemMenuResult, Option<Entity>) {

    let mut y = (25 - (count / 2)) as i32;
    draw_batch.draw_box(
        Rect::with_size(15, y-2, 31, (count+3) as i32), 
        ColorPair::new(RGB::named(rltk::WHITE), RGB::named(rltk::BLACK))
    );
    draw_batch.print_color(
        Point::new(18, y-2), 
        &title.to_string(),
        ColorPair::new(RGB::named(rltk::YELLOW), RGB::named(rltk::BLACK))
    );
    draw_batch.print_color(
        Point::new(18, y+count as i32+1),
        "ESCAPE to cancel",
        ColorPair::new(RGB::named(rltk::YELLOW), RGB::named(rltk::BLACK))
    );

    let mut item_list : Vec<Entity> = Vec::new();
    let mut j = 0;
    for item in items {
        menu_option(draw_batch, 17, y, 97+j as rltk::FontCharType, &item.1);
        item_list.push(item.0);
        y += 1;
        j += 1;
    }

    match key {
        None => (ItemMenuResult::NoResponse, None),
        Some(key) => {
            match key {
                VirtualKeyCode::Escape => { (ItemMenuResult::Cancel, None) }
                _ => {
                    let selection = rltk::letter_to_option(key);
                    if selection > -1 && selection < count as i32 {
                        return (ItemMenuResult::Selected, Some(item_list[selection as usize]));
                    }
                    (ItemMenuResult::NoResponse, None)
                }
            }
        }
    }
}
```

这基本上是我们其他返回 `ItemMenuResult` 的菜单的通用版本。我们可以用它来显著简化 `gui/drop_item_menu.rs`：

```rust
use rltk::prelude::*;
use specs::prelude::*;
use crate::{State, InBackpack};
use super::{get_item_display_name, ItemMenuResult, item_result_menu};

pub fn drop_item_menu(gs : &mut State, ctx : &mut Rltk) -> (ItemMenuResult, Option<Entity>) {
    let mut draw_batch = DrawBatch::new();

    let player_entity = gs.ecs.fetch::<Entity>();
    let backpack = gs.ecs.read_storage::<InBackpack>();
    let entities = gs.ecs.entities();

    let mut items : Vec<(Entity, String)> = Vec::new();
    (&entities, &backpack).join()
        .filter(|item| item.1.owner == *player_entity )
        .for_each(|item| {
            items.push((item.0, get_item_display_name(&gs.ecs, item.0)))
        });

    let result = item_result_menu(
        &mut draw_batch,
        "Drop which item?",
        items.len(),
        &items,
        ctx.key
    );
    draw_batch.submit(6000);
    result
}
```

### 删除菜单项

这基本上是我们其他返回 `ItemMenuResult` 的菜单的通用版本。我们可以用它来显著简化 `gui/drop_item_menu.rs`：

```rust
use rltk::prelude::*;
use specs::prelude::*;
use crate::{State, Equipped };
use super::{get_item_display_name, ItemMenuResult, item_result_menu};

pub fn remove_item_menu(gs : &mut State, ctx : &mut Rltk) -> (ItemMenuResult, Option<Entity>) {
    let mut draw_batch = DrawBatch::new();

    let player_entity = gs.ecs.fetch::<Entity>();
    let backpack = gs.ecs.read_storage::<Equipped>();
    let entities = gs.ecs.entities();

    let mut items : Vec<(Entity, String)> = Vec::new();
    (&entities, &backpack).join()
        .filter(|item| item.1.owner == *player_entity )
        .for_each(|item| {
            items.push((item.0, get_item_display_name(&gs.ecs, item.0)))
        });

    let result = item_result_menu(
        &mut draw_batch,
        "Remove which item?",
        items.len(),
        &items,
        ctx.key
    );
    draw_batch.submit(6000);
    result
}
```

### 库存菜单

再次，我们的辅助函数极大地简化了库存菜单。我们可以替换`gui/inventory_menu.rs`：

```rust
use rltk::prelude::*;
use specs::prelude::*;
use crate::{State, InBackpack };
use super::{get_item_display_name, item_result_menu};

#[derive(PartialEq, Copy, Clone)]
pub enum ItemMenuResult { Cancel, NoResponse, Selected }

pub fn show_inventory(gs : &mut State, ctx : &mut Rltk) -> (ItemMenuResult, Option<Entity>) {
    let player_entity = gs.ecs.fetch::<Entity>();
    let backpack = gs.ecs.read_storage::<InBackpack>();
    let entities = gs.ecs.entities();

    let mut draw_batch = DrawBatch::new();

    let mut items : Vec<(Entity, String)> = Vec::new();
    (&entities, &backpack).join()
        .filter(|item| item.1.owner == *player_entity )
        .for_each(|item| {
            items.push((item.0, get_item_display_name(&gs.ecs, item.0)))
        });

    let result = item_result_menu(
        &mut draw_batch,
        "Inventory",
        items.len(),
        &items,
        ctx.key
    );
    draw_batch.submit(6000);
    result
}
```

### 识别菜单

辅助函数在简化 `gui/identify_menu.rs` 文件方面也很有用 - 但复杂的过滤器仍然相当长。用以下内容替换文件内容：

```rust
use rltk::prelude::*;
use specs::prelude::*;
use crate::{Name, State, InBackpack, Equipped, MasterDungeonMap, Item, ObfuscatedName };
use super::{get_item_display_name, item_result_menu, ItemMenuResult};

pub fn identify_menu(gs : &mut State, ctx : &mut Rltk) -> (ItemMenuResult, Option<Entity>) {
    let mut draw_batch = DrawBatch::new();

    let player_entity = gs.ecs.fetch::<Entity>();
    let equipped = gs.ecs.read_storage::<Equipped>();
    let backpack = gs.ecs.read_storage::<InBackpack>();
    let entities = gs.ecs.entities();
    let item_components = gs.ecs.read_storage::<Item>();
    let names = gs.ecs.read_storage::<Name>();
    let dm = gs.ecs.fetch::<MasterDungeonMap>();
    let obfuscated = gs.ecs.read_storage::<ObfuscatedName>();

    let mut items : Vec<(Entity, String)> = Vec::new();
    (&entities, &item_components).join()
        .filter(|(item_entity,_item)| {
            let mut keep = false;
            if let Some(bp) = backpack.get(*item_entity) {
                if bp.owner == *player_entity {
                    if let Some(name) = names.get(*item_entity) {
                        if obfuscated.get(*item_entity).is_some() && !dm.identified_items.contains(&name.name) {
                            keep = true;
                        }
                    }
                }
            }
            // It's equipped, so we know it's cursed
            if let Some(equip) = equipped.get(*item_entity) {
                if equip.owner == *player_entity {
                    if let Some(name) = names.get(*item_entity) {
                        if obfuscated.get(*item_entity).is_some() && !dm.identified_items.contains(&name.name) {
                            keep = true;
                        }
                    }
                }
            }
            keep
        })
        .for_each(|item| {
            items.push((item.0, get_item_display_name(&gs.ecs, item.0)))
        });

    let result = item_result_menu(
        &mut draw_batch,
        "Inventory",
        items.len(),
        &items,
        ctx.key
    );
    draw_batch.submit(6000);
    result
}
```

### 解除诅咒菜单

解除诅咒菜单与识别菜单非常相似，因此可以应用相同的原则。替换`gui/remove_curse_menu.rs`文件内容如下：

```rust
use rltk::prelude::*;
use specs::prelude::*;
use crate::{Name, State, InBackpack, Equipped, MasterDungeonMap, CursedItem, Item };
use super::{get_item_display_name, item_result_menu, ItemMenuResult};

pub fn remove_curse_menu(gs : &mut State, ctx : &mut Rltk) -> (ItemMenuResult, Option<Entity>) {
    let player_entity = gs.ecs.fetch::<Entity>();
    let equipped = gs.ecs.read_storage::<Equipped>();
    let backpack = gs.ecs.read_storage::<InBackpack>();
    let entities = gs.ecs.entities();
    let item_components = gs.ecs.read_storage::<Item>();
    let cursed = gs.ecs.read_storage::<CursedItem>();
    let names = gs.ecs.read_storage::<Name>();
    let dm = gs.ecs.fetch::<MasterDungeonMap>();

    let mut draw_batch = DrawBatch::new();

    let mut items : Vec<(Entity, String)> = Vec::new();
    (&entities, &item_components, &cursed).join()
        .filter(|(item_entity,_item,_cursed)| {
            let mut keep = false;
            if let Some(bp) = backpack.get(*item_entity) {
                if bp.owner == *player_entity {
                    if let Some(name) = names.get(*item_entity) {
                        if dm.identified_items.contains(&name.name) {
                            keep = true;
                        }
                    }
                }
            }
            // It's equipped, so we know it's cursed
            if let Some(equip) = equipped.get(*item_entity) {
                if equip.owner == *player_entity {
                    keep = true;
                }
            }
            keep
        })
        .for_each(|item| {
            items.push((item.0, get_item_display_name(&gs.ecs, item.0)))
        });

    let result = item_result_menu(
        &mut draw_batch,
        "Inventory",
        items.len(),
        &items,
        ctx.key
    );
    draw_batch.submit(6000);
    result
}
```

### 游戏结束菜单

游戏结束菜单是一个简单的从`ctx`到`DrawBatch`的转换。在`gui/game_over_menu.rs`文件中：

```rust
use rltk::prelude::*;

#[derive(PartialEq, Copy, Clone)]
pub enum GameOverResult { NoSelection, QuitToMenu }

pub fn game_over(ctx : &mut Rltk) -> GameOverResult {
    let mut draw_batch = DrawBatch::new();
    draw_batch.print_color_centered(
        15, 
        "Your journey has ended!",
        ColorPair::new(RGB::named(rltk::YELLOW), RGB::named(rltk::BLACK))
    );
    draw_batch.print_color_centered(
        17, 
        "One day, we'll tell you all about how you did.",
        ColorPair::new(RGB::named(rltk::WHITE), RGB::named(rltk::BLACK))
    );
    draw_batch.print_color_centered(
        18, 
        "That day, sadly, is not in this chapter..",
        ColorPair::new(RGB::named(rltk::WHITE), RGB::named(rltk::BLACK))
    );

    draw_batch.print_color_centered(
        19,
        &format!("You lived for {} turns.", crate::gamelog::get_event_count("Turn")),
        ColorPair::new(RGB::named(rltk::WHITE), RGB::named(rltk::BLACK))
    );
    draw_batch.print_color_centered(
        20,
        &format!("You suffered {} points of damage.", crate::gamelog::get_event_count("Damage Taken")),
        ColorPair::new(RGB::named(rltk::RED), RGB::named(rltk::BLACK))
    );
    draw_batch.print_color_centered(
        21,
        &format!("You inflicted {} points of damage.", crate::gamelog::get_event_count("Damage Inflicted")),
        ColorPair::new(RGB::named(rltk::RED), RGB::named(rltk::BLACK)));

    draw_batch.print_color_centered(
        23,
        "Press any key to return to the menu.",
        ColorPair::new(RGB::named(rltk::MAGENTA), RGB::named(rltk::BLACK))
    );

    draw_batch.submit(6000);

    match ctx.key {
        None => GameOverResult::NoSelection,
        Some(_) => GameOverResult::QuitToMenu
    }
}
```

### 远程目标菜单

`gui/ranged_target.rs` 是另一个简单的转换：

```rust
use rltk::prelude::*;
use specs::prelude::*;
use crate::{State, camera, Viewshed };
use super::ItemMenuResult;

pub fn ranged_target(gs : &mut State, ctx : &mut Rltk, range : i32) -> (ItemMenuResult, Option<Point>) {
    let (min_x, max_x, min_y, max_y) = camera::get_screen_bounds(&gs.ecs, ctx);
    let player_entity = gs.ecs.fetch::<Entity>();
    let player_pos = gs.ecs.fetch::<Point>();
    let viewsheds = gs.ecs.read_storage::<Viewshed>();

    let mut draw_batch = DrawBatch::new();

    draw_batch.print_color(
        Point::new(5, 0), 
        "Select Target:",
        ColorPair::new(RGB::named(rltk::YELLOW), RGB::named(rltk::BLACK))
    );

    // Highlight available target cells
    let mut available_cells = Vec::new();
    let visible = viewsheds.get(*player_entity);
    if let Some(visible) = visible {
        // We have a viewshed
        for idx in visible.visible_tiles.iter() {
            let distance = rltk::DistanceAlg::Pythagoras.distance2d(*player_pos, *idx);
            if distance <= range as f32 {
                let screen_x = idx.x - min_x;
                let screen_y = idx.y - min_y;
                if screen_x > 1 && screen_x < (max_x - min_x)-1 && screen_y > 1 && screen_y < (max_y - min_y)-1 {
                    draw_batch.set_bg(Point::new(screen_x, screen_y), RGB::named(rltk::BLUE));
                    available_cells.push(idx);
                }
            }
        }
    } else {
        return (ItemMenuResult::Cancel, None);
    }

    // Draw mouse cursor
    let mouse_pos = ctx.mouse_pos();
    let mut mouse_map_pos = mouse_pos;
    mouse_map_pos.0 += min_x - 1;
    mouse_map_pos.1 += min_y - 1;
    let mut valid_target = false;
    for idx in available_cells.iter() { if idx.x == mouse_map_pos.0 && idx.y == mouse_map_pos.1 { valid_target = true; } }
    if valid_target {
        draw_batch.set_bg(Point::new(mouse_pos.0, mouse_pos.1), RGB::named(rltk::CYAN));
        if ctx.left_click {
            return (ItemMenuResult::Selected, Some(Point::new(mouse_map_pos.0, mouse_map_pos.1)));
        }
    } else {
        draw_batch.set_bg(Point::new(mouse_pos.0, mouse_pos.1), RGB::named(rltk::RED));
        if ctx.left_click {
            return (ItemMenuResult::Cancel, None);
        }
    }

    draw_batch.submit(5000);

    (ItemMenuResult::NoResponse, None)
}
```

### 主菜单

`gui/main_menu.rs` 文件是另一个简单的转换：

```rust
use rltk::prelude::*;
use crate::{State, RunState, rex_assets::RexAssets };

#[derive(PartialEq, Copy, Clone)]
pub enum MainMenuSelection { NewGame, LoadGame, Quit }

#[derive(PartialEq, Copy, Clone)]
pub enum MainMenuResult { NoSelection{ selected : MainMenuSelection }, Selected{ selected: MainMenuSelection } }

pub fn main_menu(gs : &mut State, ctx : &mut Rltk) -> MainMenuResult {
    let mut draw_batch = DrawBatch::new();
    let save_exists = crate::saveload_system::does_save_exist();
    let runstate = gs.ecs.fetch::<RunState>();
    let assets = gs.ecs.fetch::<RexAssets>();
    ctx.render_xp_sprite(&assets.menu, 0, 0);

    draw_batch.draw_double_box(Rect::with_size(24, 18, 31, 10), ColorPair::new(RGB::named(rltk::WHEAT), RGB::named(rltk::BLACK)));

    draw_batch.print_color_centered(20, "Rust Roguelike Tutorial", ColorPair::new(RGB::named(rltk::YELLOW), RGB::named(rltk::BLACK)));
    draw_batch.print_color_centered(21, "by Herbert Wolverson", ColorPair::new(RGB::named(rltk::CYAN), RGB::named(rltk::BLACK)));
    draw_batch.print_color_centered(22, "Use Up/Down Arrows and Enter", ColorPair::new(RGB::named(rltk::GRAY), RGB::named(rltk::BLACK)));

    let mut y = 24;
    if let RunState::MainMenu{ menu_selection : selection } = *runstate {
        if selection == MainMenuSelection::NewGame {
            draw_batch.print_color_centered(y, "Begin New Game", ColorPair::new(RGB::named(rltk::MAGENTA), RGB::named(rltk::BLACK)));
        } else {
            draw_batch.print_color_centered(y, "Begin New Game", ColorPair::new(RGB::named(rltk::WHITE), RGB::named(rltk::BLACK)));
        }
        y += 1;

        if save_exists {
            if selection == MainMenuSelection::LoadGame {
                draw_batch.print_color_centered(y, "Load Game", ColorPair::new(RGB::named(rltk::MAGENTA), RGB::named(rltk::BLACK)));
            } else {
                draw_batch.print_color_centered(y, "Load Game", ColorPair::new(RGB::named(rltk::WHITE), RGB::named(rltk::BLACK)));
            }
            y += 1;
        }

        if selection == MainMenuSelection::Quit {
            draw_batch.print_color_centered(y, "Quit", ColorPair::new(RGB::named(rltk::MAGENTA), RGB::named(rltk::BLACK)));
        } else {
            draw_batch.print_color_centered(y, "Quit", ColorPair::new(RGB::named(rltk::WHITE), RGB::named(rltk::BLACK)));
        }

        draw_batch.submit(6000);

        match ctx.key {
            None => return MainMenuResult::NoSelection{ selected: selection },
            Some(key) => {
                match key {
                    VirtualKeyCode::Escape => { return MainMenuResult::NoSelection{ selected: MainMenuSelection::Quit } }
                    VirtualKeyCode::Up => {
                        let mut newselection;
                        match selection {
                            MainMenuSelection::NewGame => newselection = MainMenuSelection::Quit,
                            MainMenuSelection::LoadGame => newselection = MainMenuSelection::NewGame,
                            MainMenuSelection::Quit => newselection = MainMenuSelection::LoadGame
                        }
                        if newselection == MainMenuSelection::LoadGame && !save_exists {
                            newselection = MainMenuSelection::NewGame;
                        }
                        return MainMenuResult::NoSelection{ selected: newselection }
                    }
                    VirtualKeyCode::Down => {
                        let mut newselection;
                        match selection {
                            MainMenuSelection::NewGame => newselection = MainMenuSelection::LoadGame,
                            MainMenuSelection::LoadGame => newselection = MainMenuSelection::Quit,
                            MainMenuSelection::Quit => newselection = MainMenuSelection::NewGame
                        }
                        if newselection == MainMenuSelection::LoadGame && !save_exists {
                            newselection = MainMenuSelection::Quit;
                        }
                        return MainMenuResult::NoSelection{ selected: newselection }
                    }
                    VirtualKeyCode::Return => return MainMenuResult::Selected{ selected : selection },
                    _ => return MainMenuResult::NoSelection{ selected: selection }
                }
            }
        }
    }

    MainMenuResult::NoSelection { selected: MainMenuSelection::NewGame }
}
```

### 商家菜单

商家菜单系统需要做一些额外的工作，但不会太多。我们的辅助函数在这里不太有用：

```rust
use rltk::prelude::*;
use specs::prelude::*;
use crate::{Name, State, InBackpack, VendorMode, Vendor, Item };
use super::{get_item_display_name, get_item_color, menu_box};

#[derive(PartialEq, Copy, Clone)]
pub enum VendorResult { NoResponse, Cancel, Sell, BuyMode, SellMode, Buy }

fn vendor_sell_menu(gs : &mut State, ctx : &mut Rltk, _vendor : Entity, _mode : VendorMode) -> (VendorResult, Option<Entity>, Option<String>, Option<f32>) {
    let mut draw_batch = DrawBatch::new();
    let player_entity = gs.ecs.fetch::<Entity>();
    let names = gs.ecs.read_storage::<Name>();
    let backpack = gs.ecs.read_storage::<InBackpack>();
    let items = gs.ecs.read_storage::<Item>();
    let entities = gs.ecs.entities();

    let inventory = (&backpack, &names).join().filter(|item| item.0.owner == *player_entity );
    let count = inventory.count();

    let mut y = (25 - (count / 2)) as i32;
    menu_box(&mut draw_batch, 15, y, (count+3) as i32, "Sell Which Item? (space to switch to buy mode)");
    draw_batch.print_color(
        Point::new(18, y+count as i32+1),
        "ESCAPE to cancel",
        ColorPair::new(RGB::named(rltk::YELLOW), RGB::named(rltk::BLACK))
    );

    let mut equippable : Vec<Entity> = Vec::new();
    let mut j = 0;
    for (entity, _pack, item) in (&entities, &backpack, &items).join().filter(|item| item.1.owner == *player_entity ) {
        draw_batch.set(Point::new(17, y), ColorPair::new(RGB::named(rltk::WHITE), RGB::named(rltk::BLACK)), rltk::to_cp437('('));
        draw_batch.set(Point::new(18, y), ColorPair::new(RGB::named(rltk::YELLOW), RGB::named(rltk::BLACK)), 97+j as rltk::FontCharType);
        draw_batch.set(Point::new(19, y), ColorPair::new(RGB::named(rltk::WHITE), RGB::named(rltk::BLACK)), rltk::to_cp437(')'));

        draw_batch.print_color(
            Point::new(21, y), 
            &get_item_display_name(&gs.ecs, entity),
            ColorPair::new(get_item_color(&gs.ecs, entity), RGB::from_f32(0.0, 0.0, 0.0))
        );
        draw_batch.print(Point::new(50, y), &format!("{:.1} gp", item.base_value * 0.8));
        equippable.push(entity);
        y += 1;
        j += 1;
    }

    draw_batch.submit(6000);

    match ctx.key {
        None => (VendorResult::NoResponse, None, None, None),
        Some(key) => {
            match key {
                VirtualKeyCode::Space => { (VendorResult::BuyMode, None, None, None) }
                VirtualKeyCode::Escape => { (VendorResult::Cancel, None, None, None) }
                _ => {
                    let selection = rltk::letter_to_option(key);
                    if selection > -1 && selection < count as i32 {
                        return (VendorResult::Sell, Some(equippable[selection as usize]), None, None);
                    }
                    (VendorResult::NoResponse, None, None, None)
                }
            }
        }
    }
}

fn vendor_buy_menu(gs : &mut State, ctx : &mut Rltk, vendor : Entity, _mode : VendorMode) -> (VendorResult, Option<Entity>, Option<String>, Option<f32>) {
    use crate::raws::*;
    let mut draw_batch = DrawBatch::new();

    let vendors = gs.ecs.read_storage::<Vendor>();

    let inventory = crate::raws::get_vendor_items(&vendors.get(vendor).unwrap().categories, &RAWS.lock().unwrap());
    let count = inventory.len();

    let mut y = (25 - (count / 2)) as i32;
    menu_box(&mut draw_batch, 15, y, (count+3) as i32, "Buy Which Item? (space to switch to sell mode)");
    draw_batch.print_color(
        Point::new(18, y+count as i32+1),
        "ESCAPE to cancel",
        ColorPair::new(RGB::named(rltk::YELLOW), RGB::named(rltk::BLACK))
    );

    for (j,sale) in inventory.iter().enumerate() {
        draw_batch.set(Point::new(17, y), ColorPair::new(RGB::named(rltk::WHITE), RGB::named(rltk::BLACK)), rltk::to_cp437('('));
        draw_batch.set(Point::new(18, y), ColorPair::new(RGB::named(rltk::YELLOW), RGB::named(rltk::BLACK)), 97+j as rltk::FontCharType);
        draw_batch.set(Point::new(19, y), ColorPair::new(RGB::named(rltk::WHITE), RGB::named(rltk::BLACK)), rltk::to_cp437(')'));

        draw_batch.print(Point::new(21, y), &sale.0);
        draw_batch.print(Point::new(50, y), &format!("{:.1} gp", sale.1 * 1.2));
        y += 1;
    }

    draw_batch.submit(6000);

    match ctx.key {
        None => (VendorResult::NoResponse, None, None, None),
        Some(key) => {
            match key {
                VirtualKeyCode::Space => { (VendorResult::SellMode, None, None, None) }
                VirtualKeyCode::Escape => { (VendorResult::Cancel, None, None, None) }
                _ => {
                    let selection = rltk::letter_to_option(key);
                    if selection > -1 && selection < count as i32 {
                        return (VendorResult::Buy, None, Some(inventory[selection as usize].0.clone()), Some(inventory[selection as usize].1));
                    }
                    (VendorResult::NoResponse, None, None, None)
                }
            }
        }
    }
}

pub fn show_vendor_menu(gs : &mut State, ctx : &mut Rltk, vendor : Entity, mode : VendorMode) -> (VendorResult, Option<Entity>, Option<String>, Option<f32>) {
    match mode {
        VendorMode::Buy => vendor_buy_menu(gs, ctx, vendor, mode),
        VendorMode::Sell => vendor_sell_menu(gs, ctx, vendor, mode)
    }
}
```

### 工具提示

`gui/tooltip.rs` 也相对简单：

```rust
use rltk::prelude::*;
use specs::prelude::*;
use crate::{Pools, Map, Name, Hidden, camera, Attributes, StatusEffect, Duration };
use super::get_item_display_name;

struct Tooltip {
    lines : Vec<String>
}

impl Tooltip {
    fn new() -> Tooltip {
        Tooltip { lines : Vec::new() }
    }

    fn add<S:ToString>(&mut self, line : S) {
        self.lines.push(line.to_string());
    }

    fn width(&self) -> i32 {
        let mut max = 0;
        for s in self.lines.iter() {
            if s.len() > max {
                max = s.len();
            }
        }
        max as i32 + 2i32
    }

    fn height(&self) -> i32 { self.lines.len() as i32 + 2i32 }

    fn render(&self, draw_batch : &mut DrawBatch, x : i32, y : i32) {
        let box_gray : RGB = RGB::from_hex("#999999").expect("Oops");
        let light_gray : RGB = RGB::from_hex("#DDDDDD").expect("Oops");
        let white = RGB::named(rltk::WHITE);
        let black = RGB::named(rltk::BLACK);
        draw_batch.draw_box(Rect::with_size(x, y, self.width()-1, self.height()-1), ColorPair::new(white, box_gray));
        for (i,s) in self.lines.iter().enumerate() {
            let col = if i == 0 { white } else { light_gray };
            draw_batch.print_color(Point::new(x+1, y+i as i32+1), &s, ColorPair::new(col, black));
        }
    }
}

pub fn draw_tooltips(ecs: &World, ctx : &mut Rltk) {
    let mut draw_batch = DrawBatch::new();

    let (min_x, _max_x, min_y, _max_y) = camera::get_screen_bounds(ecs, ctx);
    let map = ecs.fetch::<Map>();
    let hidden = ecs.read_storage::<Hidden>();
    let attributes = ecs.read_storage::<Attributes>();
    let pools = ecs.read_storage::<Pools>();

    let mouse_pos = ctx.mouse_pos();
    let mut mouse_map_pos = mouse_pos;
    mouse_map_pos.0 += min_x - 1;
    mouse_map_pos.1 += min_y - 1;
    if mouse_pos.0 < 1 || mouse_pos.0 > 49 || mouse_pos.1 < 1 || mouse_pos.1 > 40 {
        return;
    }
    if mouse_map_pos.0 >= map.width-1 || mouse_map_pos.1 >= map.height-1 || mouse_map_pos.0 < 1 || mouse_map_pos.1 < 1
    {
        return;
    }
    if !map.in_bounds(rltk::Point::new(mouse_map_pos.0, mouse_map_pos.1)) { return; }
    let mouse_idx = map.xy_idx(mouse_map_pos.0, mouse_map_pos.1);
    if !map.visible_tiles[mouse_idx] { return; }

    let mut tip_boxes : Vec<Tooltip> = Vec::new();
    for entity in map.tile_content[mouse_idx].iter().filter(|e| hidden.get(**e).is_none()) {
        let mut tip = Tooltip::new();
        tip.add(get_item_display_name(ecs, *entity));

        // Comment on attributes
        let attr = attributes.get(*entity);
        if let Some(attr) = attr {
            let mut s = "".to_string();
            if attr.might.bonus < 0 { s += "Weak. " };
            if attr.might.bonus > 0 { s += "Strong. " };
            if attr.quickness.bonus < 0 { s += "Clumsy. " };
            if attr.quickness.bonus > 0 { s += "Agile. " };
            if attr.fitness.bonus < 0 { s += "Unheathy. " };
            if attr.fitness.bonus > 0 { s += "Healthy." };
            if attr.intelligence.bonus < 0 { s += "Unintelligent. "};
            if attr.intelligence.bonus > 0 { s += "Smart. "};
            if s.is_empty() {
                s = "Quite Average".to_string();
            }
            tip.add(s);
        }

        // Comment on pools
        let stat = pools.get(*entity);
        if let Some(stat) = stat {
            tip.add(format!("Level: {}", stat.level));
        }

        // Status effects
        let statuses = ecs.read_storage::<StatusEffect>();
        let durations = ecs.read_storage::<Duration>();
        let names = ecs.read_storage::<Name>();
        for (status, duration, name) in (&statuses, &durations, &names).join() {
            if status.target == *entity {
                tip.add(format!("{} ({})", name.name, duration.turns));
            }
        }

        tip_boxes.push(tip);
    }

    if tip_boxes.is_empty() { return; }

    let box_gray : RGB = RGB::from_hex("#999999").expect("Oops");
    let white = RGB::named(rltk::WHITE);

    let arrow;
    let arrow_x;
    let arrow_y = mouse_pos.1;
    if mouse_pos.0 < 40 {
        // Render to the left
        arrow = to_cp437('→');
        arrow_x = mouse_pos.0 - 1;
    } else {
        // Render to the right
        arrow = to_cp437('←');
        arrow_x = mouse_pos.0 + 1;
    }
    draw_batch.set(Point::new(arrow_x, arrow_y), ColorPair::new(white, box_gray), arrow);

    let mut total_height = 0;
    for tt in tip_boxes.iter() {
        total_height += tt.height();
    }

    let mut y = mouse_pos.1 - (total_height / 2);
    while y + (total_height/2) > 50 {
        y -= 1;
    }

    for tt in tip_boxes.iter() {
        let x = if mouse_pos.0 < 40 {
            mouse_pos.0 - (1 + tt.width())
        } else {
            mouse_pos.0 + (1 + tt.width())
        };
        tt.render(&mut draw_batch, x, y);
        y += tt.height();
    }

    draw_batch.submit(7000);
}
```

## 总结

这一章节有点痛苦，但我们已经使用新的批处理系统进行了渲染 - 并且完美地渲染了更大的日志文本。在未来的章节中，我们将在此基础上构建，当我们处理系统（和并发）时。

---

**The source code for this chapter may be found [here](https://github.com/thebracket/rustrogueliketutorial/tree/master/chapter-72-textlayers)**


[Run this chapter's example with web assembly, in your browser (WebGL2 required)](https://bfnightly.bracketproductions.com/rustbook/wasm/chapter-72-textlayers)
---

Copyright (C) 2019, Herbert Wolverson.

---