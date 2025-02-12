# Missiles and Ranged Attacks

---

***About this tutorial***

*This tutorial is free and open source, and all code uses the MIT license - so you are free to do with it as you like. My hope is that you will enjoy the tutorial, and make great games!*

*If you enjoy this and would like me to keep writing, please consider supporting [my Patreon](https://www.patreon.com/blackfuture).*

[![Hands-On Rust](./beta-webBanner.jpg)](https://pragprog.com/titles/hwrust/hands-on-rust/)

---

当你阅读涉及暗夜精灵的科幻小说时，他们通常会偷偷地从黑暗中发射导弹武器。这实际上是本教程书包含暗夜精灵的原因：它们为我们提供了一个很好的机会来深入研究远程战斗的世界。我们已经有了一些远程战斗的元素：法术效果可以在远程发生，但目标选择系统有点笨拙——而且一点也不适合弓箭决斗。所以在本章中，我们将引入远程武器，并使暗夜精灵变得更加可怕。我们还想尝试改进导弹的粒子效果，以便玩家能清楚地看到发生了什么。

## 介绍远程武器

我们会稍微作弊，不考虑弹药；有些游戏会计算每一支箭，对于一个远程战斗角色来说，保持箭筒满是非常重要的。我们将专注于远程武器的方面，并假设弹药充足；这不是最现实的选择，但它使事情更易于管理！

### 定义短弓

让我们开始打开 `spawns.json` 并为短弓创建一个条目：
```json
{
    "name" : "Shortbow",
    "renderable": {
        "glyph" : ")",
        "fg" : "#FFAAAA",
        "bg" : "#000000",
        "order" : 2
    },
    "weapon" : {
        "range" : "4",
        "attribute" : "Quickness",
        "base_damage" : "1d4",
        "hit_bonus" : 0
    },
    "weight_lbs" : 2.0,
    "base_value" : 5.0,
    "initiative_penalty" : 1,
    "vendor_category" : "weapon"
},
```
你会注意到这与匕首条目非常相似；实际上，我是复制粘贴的，然后将“range”从“近战”改为“4”！我还暂时移除了模板魔法部分，以保持简单。现在我们打开 `components.rs`，并查看 `MeleeWeapon`——目的是制作远程武器。不幸的是，我们发现了一个设计错误！伤害都在武器内部，所以如果我们制作一个通用的 `RangedWeapon` 组件，我们会重复自己。通常来说，不要重复输入同一件事是一个好主意，所以我们将 `MeleeWeapon` 的名称更改为 `Weapon`——并添加了一个 `range` 字段。如果没有范围（它是一个 `Option`），那么它就是近战武器：
```rust
#[derive(Component, Serialize, Deserialize, Clone)]
pub struct Weapon {
    pub range : Option<i32>,
    pub attribute : WeaponAttribute,
    pub damage_n_dice : i32,
    pub damage_die_type : i32,
    pub damage_bonus : i32,
    pub hit_bonus : i32,
    pub proc_chance : Option<f32>,
    pub proc_target : Option<String>,
}
```
你需要打开 `main.rs`，`saveload_system.rs` 并将 `MeleeWeapon` 更改为 `Weapon`。还有一些其他代码也出现了问题。在 `melee_combat_system.rs` 中，只需将所有 `MeleeWeapon` 的实例替换为 `Weapon`。你还需要为处理自然攻击的虚拟武器添加 `range`：
```rust
let mut weapon_info = Weapon{
    range: None,
    attribute : WeaponAttribute::Might,
    hit_bonus : 0,
    damage_n_dice : 1,
    damage_die_type : 4,
    damage_bonus : 0,
    proc_chance : None,
    proc_target : None
};
```
为了使其像以前一样编译和运行，你可以更改 `raws/rawmaster.rs` 中的一个部分：
```rust
let mut wpn = Weapon{
    range : None,
    attribute : WeaponAttribute::Might,
    damage_n_dice : n_dice,
    damage_die_type : die_type,
    damage_bonus : bonus,
    hit_bonus : weapon.hit_bonus,
    proc_chance : weapon.proc_chance,
    proc_target : weapon.proc_target.clone()
};
```
这样足以使旧代码再次运行，并且有一个重要的优点：我们保持了武器代码的基本相同，因此所有的“特性”和“魔法模板”系统仍然有效。然而，有一个重要的限制：短弓仍然是近战武器！

我们可以打开 `raws/rawmaster.rs` 并更改相同的代码段以实例化一个 `range`（如果有的话）。这是一个好的开始——至少游戏有选项知道它是一个远程武器！
```rust
let mut wpn = Weapon{
    range : if weapon.range == "melee" { None } else { Some(weapon.range.parse::<i32>().expect("Not a number")) },
    attribute : WeaponAttribute::Might,
    damage_n_dice : n_dice,
    damage_die_type : die_type,
    damage_bonus : bonus,
    hit_bonus : weapon.hit_bonus,
    proc_chance : weapon.proc_chance,
    proc_target : weapon.proc_target.clone()
};
```

## 让玩家射击

现在我们知道武器*是*远程武器，这是一个很好的开始。让我们进入`spawner.rs`，并给玩家一把短弓。我们可能不会一直保留它，但它为我们提供了一个很好的基础来构建：
```rust
spawn_named_entity(&RAWS.lock().unwrap(), ecs, "Rusty Longsword", SpawnType::Equipped{by : player});
spawn_named_entity(&RAWS.lock().unwrap(), ecs, "Dried Sausage", SpawnType::Carried{by : player} );
spawn_named_entity(&RAWS.lock().unwrap(), ecs, "Beer", SpawnType::Carried{by : player});
spawn_named_entity(&RAWS.lock().unwrap(), ecs, "Stained Tunic", SpawnType::Equipped{by : player});
spawn_named_entity(&RAWS.lock().unwrap(), ecs, "Torn Trousers", SpawnType::Equipped{by : player});
spawn_named_entity(&RAWS.lock().unwrap(), ecs, "Old Boots", SpawnType::Equipped{by : player});
spawn_named_entity(&RAWS.lock().unwrap(), ecs, "Shortbow", SpawnType::Carried{by : player});
```
我们从背包中开始，所以玩家仍然需要做出有意识的决定来切换到使用远程武器（我们已经做了足够的近战工作，射击不应该成为默认！）- 但这节省了我们不得不在测试系统时四处寻找。继续`cargo run`快速测试你能否装备你的新弓。你还不能射击任何东西，但至少你可以装备它（并确信我们在组件更改中没有破坏太多）。

远程武器最困难的部分是它有一个*目标*：你射击的东西。我们希望目标选择简单，以免玩家无法弄清楚如何射击！让我们从向玩家显示他们装备的武器信息开始 - 如果它有射程，我们会包括在内。在`gui.rs`中，找到我们遍历装备物品并显示它们的代码部分（在我的版本中大约在第162行）。我们将扩展它一点：
```rust
// 装备
let mut y = 13;
let entities = ecs.entities();
let equipped = ecs.read_storage::<Equipped>();
let weapon = ecs.read_storage::<Weapon>();
for (entity, equipped_by) in (&entities, &equipped).join() {
    if equipped_by.owner == *player_entity {
        let name = get_item_display_name(ecs, entity);
        ctx.print_color(50, y, get_item_color(ecs, entity), black, &name);
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
                weapon_info += &format!(" (射程: {}, F键射击)", range);
            }
            weapon_info += " ├";
            ctx.print_color(3, 45, yellow, black, &weapon_info);
        }
    }
}
```
这是一个很好的开始，因为我们现在告诉用户他们有一把远程武器（通常显示武器升级的即时结果是个好主意！）：

![截图](./c70-s1.jpg)

所以，现在让玩家轻松地瞄准敌人！我们将从创建一个`Target`组件开始。在`components.rs`中（和通常一样，在`main.rs`和`saveload_system.rs`中注册）：
```rust
#[derive(Component, Debug, Serialize, Deserialize, Clone)]
pub struct Target {}
```
这个想法很简单：我们将把`Target`附加到我们当前瞄准的任何人。我们应该在地图上突出显示目标；所以我们去`camera.rs`并在实体渲染代码中添加以下内容：

```rust
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
                        ctx.set(entity_screen_x + 1, entity_screen_y + 1, render.fg, render.bg, render.glyph);
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
                ctx.set(entity_screen_x + 1, entity_screen_y + 1, render.fg, render.bg, render.glyph);
            }
        }
    }

    if targets.get(*entity).is_some() {
        let entity_screen_x = pos.x - min_x;
        let entity_screen_y = pos.y - min_y;
        ctx.set(entity_screen_x , entity_screen_y + 1, rltk::RGB::named(rltk::RED), rltk::RGB::named(rltk::YELLOW), rltk::to_cp437('['));
        ctx.set(entity_screen_x +2, entity_screen_y + 1, rltk::RGB::named(rltk::RED), rltk::RGB::named(rltk::YELLOW), rltk::to_cp437(']'));
    }
}
```

这段代码检查我们渲染的每个实体是否被瞄准，如果是，则在其周围渲染明亮的括号。我们还应提供一些关于如何使用瞄准系统的提示，所以在`gui.rs`中，我们按照以下方式修改我们的远程武器代码：
```rust
if let Some(weapon) = weapon.get(entity) {
    let mut weapon_info = if weapon.damage_bonus < 0 {
        format!("┤ {} ({}d{}{})", &name, weapon.damage_n_dice, weapon.damage_die_type, weapon.damage_bonus)
    } else if weapon.damage_bonus == 0 {
        format!("┤ {} ({}d{})", &name, weapon.damage_n_dice, weapon.damage_die_type)
    } else {
        format!("┤ {} ({}d{}+{})", &name, weapon.damage_n_dice, weapon.damage_die_type, weapon.damage_bonus)
    };

    if let Some(range) = weapon.range {
        weapon_info += &format!(" (射程: {}, F键射击, V键切换目标)", range);
    }
    weapon_info += " ├";
    ctx.print_color(3, 45, yellow, black, &weapon_info);
}
```
我们告诉用户按`V`键切换目标，所以我们需要实现这个功能！在此之前，我们需要想出一个默认的瞄准方案。由于我们关心的是*玩家的*目标，我们将前往`player.rs`并添加一些新函数。第一个函数确定哪些实体有资格成为目标：

```rust
fn get_player_target_list(ecs : &mut World) -> Vec<(f32,Entity)> {
    let mut possible_targets : Vec<(f32,Entity)> = Vec::new();
    let viewsheds = ecs.read_storage::<Viewshed>();
    let player_entity = ecs.fetch::<Entity>();
    let equipped = ecs.read_storage::<Equipped>();
    let weapon = ecs.read_storage::<Weapon>();
    let map = ecs.fetch::<Map>();
    let positions = ecs.read_storage::<Position>();
    let factions = ecs.read_storage::<Faction>();
    for (equipped, weapon) in (&equipped, &weapon).join() {
        if equipped.owner == *player_entity && weapon.range.is_some() {
            let range = weapon.range.unwrap();

            if let Some(vs) = viewsheds.get(*player_entity) {
                let player_pos = positions.get(*player_entity).unwrap();
                for tile_point in vs.visible_tiles.iter() {
                    let tile_idx = map.xy_idx(tile_point.x, tile_point.y);
                    let distance_to_target = rltk::DistanceAlg::Pythagoras.distance2d(*tile_point, rltk::Point::new(player_pos.x, player_pos.y));
                    if distance_to_target < range as f32 {
                        crate::spatial::for_each_tile_content(tile_idx, |possible_target| {
                            if possible_target != *player_entity && factions.get(possible_target).is_some() {
                                possible_targets.push((distance_to_target, possible_target));
                            }
                        });
                    }
                }
            }
        }
    }

    possible_targets.sort_by(|a,b| a.0.partial_cmp(&b.0).unwrap());
    possible_targets
}
```

这段函数有点复杂，让我们逐步解析：

1. 我们创建一个空的结果列表，包含可攻击的实体和它们与玩家的距离。
2. 我们遍历已装备的武器，查看玩家是否有远程武器。
3. 如果有，我们记录下其射程。
4. 然后我们查看玩家的视野，并检查每个瓦片是否在武器的射程内。
5. 如果在射程内，我们通过`tile_content`系统查看该瓦片中的实体。如果实体确实是有效的目标（它们有`Faction`成员资格），我们将它们添加到可能的目标列表中。
6. 我们按距离对可能的目标列表进行排序。

现在我们需要在玩家移动时选择一个新的目标。我们将选择最近的，因为更有可能直接攻击即时威胁。以下函数实现了这一点：
```rust
pub fn end_turn_targeting(ecs: &mut World) {
    let possible_targets = get_player_target_list(ecs);
    let mut targets = ecs.write_storage::<Target>();
    targets.clear();

    if !possible_targets.is_empty() {
        targets.insert(possible_targets[0].1, Target{}).expect("插入失败");
    }
}
```
我们希望在新回合的开始调用这个函数。因此我们进入`main.rs`，并修改游戏循环以捕获新回合的开始并调用这个函数：
```rust
RunState::Ticking => {
    let mut should_change_target = false;
    while newrunstate == RunState::Ticking {
        self.run_systems();
        self.ecs.maintain();
        match *self.ecs.fetch::<RunState>() {
            RunState::AwaitingInput => {
                newrunstate = RunState::AwaitingInput;
                should_change_target = true;
            }
            RunState::MagicMapReveal{ .. } => newrunstate = RunState::MagicMapReveal{ row: 0 },
            RunState::TownPortal => newrunstate = RunState::TownPortal,
            RunState::TeleportingToOtherLevel{ x, y, depth } => newrunstate = RunState::TeleportingToOtherLevel{ x, y, depth },
            RunState::ShowRemoveCurse => newrunstate = RunState::ShowRemoveCurse,
            RunState::ShowIdentify => newrunstate = RunState::ShowIdentify,
            _ => newrunstate = RunState::Ticking
        }
    }
    if should_change_target {
        player::end_turn_targeting(&mut self.ecs);
    }
}
```
现在我们回到`player.rs`并添加另一个函数来循环目标：
```rust
fn cycle_target(ecs: &mut World) {
    let possible_targets = get_player_target_list(ecs);
    let mut targets = ecs.write_storage::<Target>();
    let entities = ecs.entities();
    let mut current_target : Option<Entity> = None;

    for (e,_t) in (&entities, &targets).join() {
        current_target = Some(e);
    }

    targets.clear();
    if let Some(current_target) = current_target {
        if !possible_targets.len() > 1 {
            let mut index = 0;
            for (i, target) in possible_targets.iter().enumerate() {
                if target.1 == current_target {
                    index = i;
                }
            }

            if index > possible_targets.len()-2 {
                targets.insert(possible_targets[0].1, Target{});
            } else {
                targets.insert(possible_targets[index+1].1, Target{});
            }
        }
    }
}
```

这段函数虽然很长，但我故意保持它的长度以便于理解。它找到当前目标在当前目标列表中的索引。如果有多个目标，它会选择列表中的下一个目标。如果它已经到达列表的末尾，它会回到开头。现在我们需要捕获按下“V”键并调用这个函数。在`player_input`函数中，我们将添加一个新的部分：
```rust
// 远程
VirtualKeyCode::V => {
    cycle_target(&mut gs.ecs);
    return RunState::AwaitingInput;
}
```
如果你现在运行`cargo run`，你可以装备你的弓并开始选择目标：

![截图](./c70-s2.jpg)

### 射击物体

我们已经建立了一个用于战斗的模式：用`WantsToMelee`组件标记动作，然后在`MeleeCombatSystem`中处理。我们已经使用类似的方法来处理想要接近、使用技能或物品的情况——所以对于想要射击的情况，我们也这样做是有道理的。在`components.rs`中（并在`main.rs`和`saveload_system.rs`中注册），我们将添加以下内容：
```rust
#[derive(Component, Debug, ConvertSaveload, Clone)]
pub struct WantsToShoot {
    pub target : Entity
}
```
我们还想要制作一个新的系统，并将其存储在`ranged_combat_system.rs`中。它基本上是`melee_combat_system`的复制粘贴，但寻找的是`WantsToShoot`：

```rust
use specs::prelude::*;
use super::{Attributes, Skills, WantsToShoot, Name, gamelog::GameLog,
    HungerClock, HungerState, Pools, skill_bonus,
    Skill, Equipped, Weapon, EquipmentSlot, WeaponAttribute, Wearable, NaturalAttackDefense,
    effects::*, Map, Position};
use rltk::{to_cp437, RGB, Point};

pub struct RangedCombatSystem {}

impl<'a> System<'a> for RangedCombatSystem {
    #[allow(clippy::type_complexity)]
    type SystemData = ( Entities<'a>,
                        WriteExpect<'a, GameLog>,
                        WriteStorage<'a, WantsToShoot>,
                        ReadStorage<'a, Name>,
                        ReadStorage<'a, Attributes>,
                        ReadStorage<'a, Skills>,
                        ReadStorage<'a, HungerClock>,
                        ReadStorage<'a, Pools>,
                        WriteExpect<'a, rltk::RandomNumberGenerator>,
                        ReadStorage<'a, Equipped>,
                        ReadStorage<'a, Weapon>,
                        ReadStorage<'a, Wearable>,
                        ReadStorage<'a, NaturalAttackDefense>,
                        ReadStorage<'a, Position>,
                        ReadExpect<'a, Map>
                      );

    fn run(&mut self, data : Self::SystemData) {
        let (entities, mut log, mut wants_shoot, names, attributes, skills,
            hunger_clock, pools, mut rng, equipped_items, weapon, wearables, natural,
            positions, map) = data;

        for (entity, wants_shoot, name, attacker_attributes, attacker_skills, attacker_pools) in (&entities, &wants_shoot, &names, &attributes, &skills, &pools).join() {
            // 攻击者和防御者都活着吗？只有在这种情况下才进行攻击
            let target_pools = pools.get(wants_shoot.target).unwrap();
            let target_attributes = attributes.get(wants_shoot.target).unwrap();
            let target_skills = skills.get(wants_shoot.target).unwrap();
            if attacker_pools.hit_points.current > 0 && target_pools.hit_points.current > 0 {
                let target_name = names.get(wants_shoot.target).unwrap();

                // 发射效果
                let apos = positions.get(entity).unwrap();
                let dpos = positions.get(wants_shoot.target).unwrap();
                add_effect(
                    None, 
                    EffectType::ParticleProjectile{ 
                        glyph: to_cp437('*'),
                        fg : RGB::named(rltk::CYAN), 
                        bg : RGB::named(rltk::BLACK), 
                        lifespan : 300.0, 
                        speed: 50.0, 
                        path: rltk::line2d(
                            rltk::LineAlg::Bresenham, 
                            Point::new(apos.x, apos.y), 
                            Point::new(dpos.x, dpos.y)
                        )
                     }, 
                    Targets::Tile{tile_idx : map.xy_idx(apos.x, apos.y) as i32}
                );

                // 定义基本的 unarmed 攻击 - 如果装备了武器，会被下面的 wielding 检查覆盖
                let mut weapon_info = Weapon{
                    range: None,
                    attribute : WeaponAttribute::Might,
                    hit_bonus : 0,
                    damage_n_dice : 1,
                    damage_die_type : 4,
                    damage_bonus : 0,
                    proc_chance : None,
                    proc_target : None
                };

                if let Some(nat) = natural.get(entity) {
                    if !nat.attacks.is_empty() {
                        let attack_index = if nat.attacks.len()==1 { 0 } else { rng.roll_dice(1, nat.attacks.len() as i32) as usize -1 };
                        weapon_info.hit_bonus = nat.attacks[attack_index].hit_bonus;
                        weapon_info.damage_n_dice = nat.attacks[attack_index].damage_n_dice;
                        weapon_info.damage_die_type = nat.attacks[attack_index].damage_die_type;
                        weapon_info.damage_bonus = nat.attacks[attack_index].damage_bonus;
                    }
                }

                let mut weapon_entity : Option<Entity> = None;
                for (weaponentity,wielded,melee) in (&entities, &equipped_items, &weapon).join() {
                    if wielded.owner == entity && wielded.slot == EquipmentSlot::Melee {
                        weapon_info = melee.clone();
                        weapon_entity = Some(weaponentity);
                    }
                }

                let natural_roll = rng.roll_dice(1, 20);
                let attribute_hit_bonus = if weapon_info.attribute == WeaponAttribute::Might
                    { attacker_attributes.might.bonus }
                    else { attacker_attributes.quickness.bonus};
                let skill_hit_bonus = skill_bonus(Skill::Melee, &*attacker_skills);
                let weapon_hit_bonus = weapon_info.hit_bonus;
                let mut status_hit_bonus = 0;
                if let Some(hc) = hunger_clock.get(entity) { // Well-Fed 提供 +1 命中加值
                    if hc.state == HungerState::WellFed {
                        status_hit_bonus += 1;
                    }
                }
                let modified_hit_roll = natural_roll + attribute_hit_bonus + skill_hit_bonus
                    + weapon_hit_bonus + status_hit_bonus;
                //println!("Natural roll: {}", natural_roll);
                //println!("Modified hit roll: {}", modified_hit_roll);

                let mut armor_item_bonus_f = 0.0;
                for (wielded,armor) in (&equipped_items, &wearables).join() {
                    if wielded.owner == wants_shoot.target {
                        armor_item_bonus_f += armor.armor_class;
                    }
                }
                let base_armor_class = match natural.get(wants_shoot.target) {
                    None => 10,
                    Some(nat) => nat.armor_class.unwrap_or(10)
                };
                let armor_quickness_bonus = target_attributes.quickness.bonus;
                let armor_skill_bonus = skill_bonus(Skill::Defense, &*target_skills);
                let armor_item_bonus = armor_item_bonus_f as i32;
                let armor_class = base_armor_class + armor_quickness_bonus + armor_skill_bonus
                    + armor_item_bonus;

                //println!("Armor class: {}", armor_class);
                if natural_roll != 1 && (natural_roll == 20 || modified_hit_roll > armor_class) {
                    // 命中目标！目前我们只支持 1d4 伤害
                    let base_damage = rng.roll_dice(weapon_info.damage_n_dice, weapon_info.damage_die_type);
                    let attr_damage_bonus = attacker_attributes.might.bonus;
                    let skill_damage_bonus = skill_bonus(Skill::Melee, &*attacker_skills);
                    let weapon_damage_bonus = weapon_info.damage_bonus;

                    let damage = i32::max(0, base_damage + attr_damage_bonus + 
                        skill_damage_bonus + weapon_damage_bonus);

                    /*println!("Damage: {} + {}attr + {}skill + {}weapon = {}",
                        base_damage, attr_damage_bonus, skill_damage_bonus,
                        weapon_damage_bonus, damage
                    );*/
                    add_effect(
                        Some(entity),
                        EffectType::Damage{ amount: damage },
                        Targets::Single{ target: wants_shoot.target }
                    );
                    log.entries.push(format!("{} hits {}, for {} hp.", &name.name, &target_name.name, damage));

                    // Proc 效果
                    if let Some(chance) = &weapon_info.proc_chance {
                        let roll = rng.roll_dice(1, 100);
                        //println!("Roll {}, Chance {}", roll, chance);
                        if roll <= (chance * 100.0) as i32 {
                            //println!("Proc!");
                            let effect_target = if weapon_info.proc_target.unwrap() == "Self" {
                                Targets::Single{ target: entity }
                            } else {
                                Targets::Single { target : wants_shoot.target }
                            };
                            add_effect(
                                Some(entity),
                                EffectType::ItemUse{ item: weapon_entity.unwrap() },
                                effect_target
                            )
                        }
                    }

                } else  if natural_roll == 1 {
                    // 自然 1 未命中
                    log.entries.push(format!("{} considers attacking {}, but misjudges the timing.", name.name, target_name.name));
                    add_effect(
                        None,
                        EffectType::Particle{ glyph: rltk::to_cp437('‼'), fg: rltk::RGB::named(rltk::BLUE), bg : rltk::RGB::named(rltk::BLACK), lifespan: 200.0 },
                        Targets::Single{ target: wants_shoot.target }
                    );
                } else {
                    // 未命中
                    log.entries.push(format!("{} attacks {}, but can't connect.", name.name, target_name.name));
                    add_effect(
                        None,
                        EffectType::Particle{ glyph: rltk::to_cp437('‼'), fg: rltk::RGB::named(rltk::CYAN), bg : rltk::RGB::named(rltk::BLACK), lifespan: 200.0 },
                        Targets::Single{ target: wants_shoot.target }
                    );
                }
            }
        }

        wants_shoot.clear();
    }
}
```

大部分内容都直接来自之前的系统。你还需要将其添加到`main.rs`的`run_systems`中；在近战之后是一个好地方：
```rust
let mut ranged = RangedCombatSystem{};
ranged.run_now(&self.ecs);
```
眼尖的读者可能已经注意到我们还悄悄地加入了一个额外的`add_effect`调用，这次调用了`EffectType::ParticleProjectile`。这不是必需的，但在远程战斗中显示飞行的投射物确实能增强游戏的氛围。到目前为止，我们的粒子一直是静止的，所以让我们为它们添加一些“活力”！

在`components.rs`中，我们将更新`ParticleLifetime`组件以包含一个可选的动画：
```rust
#[derive(Serialize, Deserialize, Clone)]
pub struct ParticleAnimation {
    pub step_time : f32,
    pub path : Vec<Point>,
    pub current_step : usize,
    pub timer : f32
}

#[derive(Component, Serialize, Deserialize, Clone)]
pub struct ParticleLifetime {
    pub lifetime_ms : f32,
    pub animation : Option<ParticleAnimation>
}
```

这增加了`step_time` - 粒子应在每个步骤上停留多长时间。一个`path` - 一个`Point`的向量，列出了沿路的每个步骤。`current_step`和`timer`将用于跟踪投射物的进度。

你需要进入`particle_system.rs`并修改粒子生成，以默认包含`None`：

```rust
particles.insert(p, ParticleLifetime{ lifetime_ms: new_particle.lifetime, animation: None }).expect("Unable to insert lifetime");
```

我们在这里，将剔除函数（`cull_dead_particles`）重命名为`update_particles` - 更好地反映了它的功能。我们还将添加一些逻辑来检查是否有动画，并使其沿着动画轨道更新其位置：
```rust
pub fn update_particles(ecs : &mut World, ctx : &Rltk) {
    let mut dead_particles : Vec<Entity> = Vec::new();
    {
        // Aging out particles
        let mut particles = ecs.write_storage::<ParticleLifetime>();
        let entities = ecs.entities();
        let map = ecs.fetch::<Map>();
        for (entity, mut particle) in (&entities, &mut particles).join() {
            if let Some(animation) = &mut particle.animation {
                animation.timer += ctx.frame_time_ms;
                if animation.timer > animation.step_time && animation.current_step < animation.path.len()-2 {
                    animation.current_step += 1;

                    if let Some(pos) = ecs.write_storage::<Position>().get_mut(entity) {
                        pos.x = animation.path[animation.current_step].x;
                        pos.y = animation.path[animation.current_step].y;
                    }
                }
            }

            particle.lifetime_ms -= ctx.frame_time_ms;
            if particle.lifetime_ms < 0.0 {
                dead_particles.push(entity);
            }
        }
    }
    for dead in dead_particles.iter() {
        ecs.delete_entity(*dead).expect("Particle will not die");
    }
}
```

再次打开`main.rs`，搜索`cull_dead_particles`并将其替换为`update_particles`。

这样足以实际动画化粒子并在完成后使它们消失，但我们需要更新`Effects`系统以生成新类型的粒子。在`effects/mod.rs`中，我们将扩展`EffectType`枚举以包含新的类型：

```rust
#[derive(Debug)]
pub enum EffectType { 
    ...
    ParticleProjectile { glyph: rltk::FontCharType, fg : rltk::RGB, bg: rltk::RGB, lifespan: f32, speed: f32, path: Vec<Point> },
    ...
```

我们还必须更新同一文件中的`affect_tile`：

```rust
fn affect_tile(ecs: &mut World, effect: &mut EffectSpawner, tile_idx : i32) {
    if tile_effect_hits_entities(&effect.effect_type) {
        let content = ecs.fetch::<Map>().tile_content[tile_idx as usize].clone();
        content.iter().for_each(|entity| affect_entity(ecs, effect, *entity));
    }

    match &effect.effect_type {
        EffectType::Bloodstain => damage::bloodstain(ecs, tile_idx),
        EffectType::Particle{..} => particles::particle_to_tile(ecs, tile_idx, &effect),
        EffectType::ParticleProjectile{..} => particles::projectile(ecs, tile_idx, &effect),
        _ => {}
    }
}
```

这将调用`particles::projectile`，所以打开`effects/particles.rs`并添加函数：

```rust
pub fn projectile(ecs: &mut World, tile_idx : i32, effect: &EffectSpawner) {
    if let EffectType::ParticleProjectile{ glyph, fg, bg, 
        lifespan, speed, path } = &effect.effect_type 
    {
        let map = ecs.fetch::<Map>();
        let x = tile_idx % map.width;
        let y = tile_idx / map.width;
        std::mem::drop(map);
        ecs.create_entity()
            .with(Position{ x, y })
            .with(Renderable{ fg: *fg, bg: *bg, glyph: *glyph, render_order: 0 })
            .with(ParticleLifetime{
                lifetime_ms: path.len() as f32 * speed,
                animation: Some(ParticleAnimation{
                    step_time: *speed,
                    path: path.to_vec(),
                    current_step: 0,
                    timer: 0.0
                })
            })
            .build();
    }
}
```

如果你现在运行`cargo run`项目，你可以选择目标并射击 - 享受一点动画效果：

![截图](./c70-pewpew.gif)

## 让怪物进行反击

只有玩家拥有弓是非常不公平的。这也使游戏失去了很多挑战性：你可以射击接近你的东西，但他们不能反击。让我们添加一个新的怪物，*Bandit Archer*。它主要是*Bandit*的副本，但它们有短弓而不是匕首。在`spawns.json`中：

```json
{ "name" : "Bandit Archer", "weight" : 9, "min_depth" : 2, "max_depth" : 3 },
...
{
    "name" : "Bandit Archer",
    "renderable": {
        "glyph" : "☻",
        "fg" : "#FF5500",
        "bg" : "#000000",
        "order" : 1
    },
    "blocks_tile" : true,
    "vision_range" : 6,
    "movement" : "random_waypoint",
    "quips" : [ "站住，交出财物!", "好吧，把它交出来" ],
    "attributes" : {},
    "equipped" : [ "Shortbow", "Shield", "Leather Armor", "Leather Boots" ],
    "light" : {
        "range" : 6,
        "color" : "#FFFF55"
    },
    "faction" : "Bandits",
    "gold" : "1d6"
},
```

我们稍微改变了它们的颜色，并给它们的装备列表中添加了`Shortbow`。我们已经支持装备生成，所以这应该足以让弓出现在它们的装备中 - 但它们不知道如何使用它。我们已经在`ai/visible_ai_systems.rs`中处理了施法（以及类似龙息的东西） - 所以这是考虑添加射击的合理位置。我们可以简单地添加：检查是否有装备远程武器，如果有 - 检查范围并生成`WantsToShoot`。我们将修改反应`Attack`：

```rust
Reaction::Attack => {
    let range = rltk::DistanceAlg::Pythagoras.distance2d(
        rltk::Point::new(pos.x, pos.y),
        rltk::Point::new(reaction.0 as i32 % map.width, reaction.0 as i32 / map.width)
    );
    if let Some(abilities) = abilities.get(entity) {
        for ability in abilities.abilities.iter() {
            if range >= ability.min_range && range <= ability.range &&
                rng.roll_dice(1,100) <= (ability.chance * 100.0) as i32
            {
                use crate::raws::find_spell_entity_by_name;
                casting.insert(
                    entity,
                    WantsToCastSpell{
                        spell : find_spell_entity_by_name(&ability.spell, &names, &spells, &entities).unwrap(),
                        target : Some(rltk::Point::new(reaction.0 as i32 % map.width, reaction.0 as i32 / map.width))}
                ).expect("Unable to insert");
                done = true;
            }
        }
    }

    if !done {
        for (weapon, equip) in (&weapons, &equipped).join() {
            if let Some(wrange) = weapon.range {
                if equip.owner == entity {
                    rltk::console::log(format!("找到所有者。范围：{}/{}", wrange, range));
                    if wrange >= range as i32 {
                        rltk::console::log("插入射击");
                        wants_shoot.insert(entity, WantsToShoot{ target: reaction.2 }).expect("插入失败");
                        done = true;
                    }
                }
            }
        }
    }
    ...
```

如果你现在运行`cargo run`，土匪会进行反击！

## 制作魔法弓

将短弓添加到你的生成列表中：

```json
{ "name" : "Shortbow", "weight" : 2, "min_depth" : 3, "max_depth" : 100 },
```
你也可以为它添加魔法模板：

```json
{
    "name" : "Shortbow",
    "renderable": {
        "glyph" : ")",
        "fg" : "#FFAAAA",
        "bg" : "#000000",
        "order" : 2
    },
    "weapon" : {
        "range" : "4",
        "attribute" : "Quickness",
        "base_damage" : "1d4",
        "hit_bonus" : 0
    },
    "weight_lbs" : 2.0,
    "base_value" : 5.0,
    "initiative_penalty" : 1,
    "vendor_category" : "weapon",
    "template_magic" : {
        "unidentified_name" : "Unidentified Shortbow",
        "bonus_min" : 1,
        "bonus_max" : 5,
        "include_cursed" : true
    }
},
```

## 使暗夜精灵更加可怕

现在我们可以引入一些哥布林弓箭手，使洞穴变得更加可怕。我们不会在龙/蜥蜴等级中引入任何远程武器，以保持游戏平衡（游戏变得更简单了！）。我们可以像对土匪那样复制粘贴一个哥布林：

```json
{
    "name" : "Goblin Archer",
    "renderable": {
        "glyph" : "g",
        "fg" : "#FFFF00",
        "bg" : "#000000",
        "order" : 1
    },
    "blocks_tile" : true,
    "vision_range" : 8,
    "movement" : "static",
    "attributes" : {},
    "faction" : "Cave Goblins",
    "gold" : "1d6",
    "equipped" : [ "Shortbow", "Leather Armor", "Leather Boots" ],
},
```

这就达到了我们本章开始时的目标。我们想要给暗夜精灵手弩。我们首先在`spawns.json`中生成新的武器类型：

```json
{
    "name" : "Hand Crossbow",
    "renderable": {
        "glyph" : ")",
        "fg" : "#FFAAAA",
        "bg" : "#000000",
        "order" : 2
    },
    "weapon" : {
        "range" : "6",
        "attribute" : "Quickness",
        "base_damage" : "1d6",
        "hit_bonus" : 0
    },
    "weight_lbs" : 2.0,
    "base_value" : 5.0,
    "initiative_penalty" : 1,
    "vendor_category" : "weapon",
    "template_magic" : {
        "unidentified_name" : "Unidentified Hand Crossbow",
        "bonus_min" : 1,
        "bonus_max" : 5,
        "include_cursed" : true
    }
},
```

我们还应该将其添加到生成表中，但仅限于暗夜精灵等级：

```json
{ "name" : "Hand Crossbow", "weight" : 2, "min_depth" : 9, "max_depth" : 11 }
```

最后，我们将其交给暗夜精灵：

```json
{
    "name" : "Dark Elf",
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
    "faction" : "DarkElf",
    "gold" : "3d6",
    "level" : 6
},
```

就这样！当你到达守卫暗夜精灵城市入口的暗夜精灵时——他们现在可以射击你。我们将在下一章中详细介绍暗夜精灵的城市。

...

**The source code for this chapter may be found [here](https://github.com/thebracket/rustrogueliketutorial/tree/master/chapter-70-missiles)**


[Run this chapter's example with web assembly, in your browser (WebGL2 required)](https://bfnightly.bracketproductions.com/rustbook/wasm/chapter-70-missiles)
---

Copyright (C) 2019, Herbert Wolverson.

---