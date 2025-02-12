# 广场一夜

---

***关于本教程***

*本教程是免费和开源的，所有代码都使用MIT许可证 - 因此您可以自由地使用它。我希望您会喜欢这个教程，并制作出伟大的游戏！*

*如果您喜欢这个并希望我继续写作，请考虑支持[我的Patreon](https://www.patreon.com/blackfuture)。*

![Hands-On Rust](./beta-webBanner.jpg)](https://pragprog.com/titles/hwrust/hands-on-rust/)

---

城市级别故意弄得混乱：英雄正在穿越狭窄、蔓延的黑暗精灵地下城 - 面对不同贵族房子的军队，他们也在互相残杀。这导致了快速、紧张的战斗。城市的最后一部分是广场 - 它旨在提供对比。城市中的公园有一个通往深渊的传送门，只有最富裕/有影响力的黑暗精灵才能在这里建造。所以，尽管是在地下，但它更像是户外城市的感觉。

那么让我们思考一下广场级别由什么组成：

* 一个相当大的公园，由一些强大的坏人守卫。由于我们紧邻通往他们家园的传送门，我们可以在这里首次添加一些恶魔化的东西。
* 一些较大的建筑物。
* 雕像、喷泉和类似的小饰品。

继续思考黑暗精灵，他们并不真正以他们的城市规划而闻名。他们本质上是一种混沌物种。因此，我们希望避免给人一种他们真的计划了他们的城市，并精心建造以使其有意义的印象。事实上，不按常理出牌增加了超现实感。

## 生成广场

就像我们对其他级别构建器所做的那样，我们需要为第11级添加一个占位符构建器。打开 `map_builders/mod.rs` 并为第11级添加对 `dark_elf_plaza` 的调用：

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
        11 => dark_elf_plaza(new_depth, width, height),
        _ => random_builder(new_depth, width, height)
    }
}
```
现在打开 `map_builders/dark_elves.rs` 并创建一个新的地图生成器函数——`dark_elf_plaza`。我们将从生成一个BSP内部地图开始；这将会改变，但至少可以让一些东西编译通过：

```rust
pub fn dark_elf_plaza(new_depth: i32, width: i32, height: i32) -> BuilderChain {
    println!("Dark elf plaza builder");
    let mut chain = BuilderChain::new(new_depth, width, height, "Dark Elven Plaza");
    chain.start_with(BspInteriorBuilder::new());
    chain.with(AreaStartingPosition::new(XStart::LEFT, YStart::CENTER));
    chain.with(CullUnreachable::new());
    chain.with(AreaEndingPosition::new(XEnd::RIGHT, YEnd::CENTER));
    chain.with(VoronoiSpawning::new());
    chain
}
```

### 故意糟糕的城市规划

现在我们有了与前一级完全相同的地图，让我们构建一个生成器来创建广场。我们将从制作一个无聊的空地图开始 - 只是为了验证我们的地图构建器是否正常工作。在`dark_elves.rs`文件的末尾，粘贴以下内容：

```rust
// Plaza Builder
use super::{InitialMapBuilder, BuilderMap, TileType };

pub struct PlazaMapBuilder {}

impl InitialMapBuilder for PlazaMapBuilder {
    #[allow(dead_code)]
    fn build_map(&mut self, build_data : &mut BuilderMap) {
        self.empty_map(build_data);
    }
}

impl PlazaMapBuilder {
    #[allow(dead_code)]
    pub fn new() -> Box<PlazaMapBuilder> {
        Box::new(PlazaMapBuilder{})
    }

    fn empty_map(&mut self, build_data : &mut BuilderMap) {
        build_data.map.tiles.iter_mut().for_each(|t| *t = TileType::Floor);
    }
}
```
你还需要进入 `dark_elf_plaza` 函数，并将初始构建器更改为使用它：

```rust
pub fn dark_elf_plaza(new_depth: i32, width: i32, height: i32) -> BuilderChain {
    println!("Dark elf plaza builder");
    let mut chain = BuilderChain::new(new_depth, width, height, "Dark Elven Plaza");
    chain.start_with(PlazaMapBuilder::new());
    chain.with(AreaStartingPosition::new(XStart::LEFT, YStart::CENTER));
    chain.with(CullUnreachable::new());
    chain.with(AreaEndingPosition::new(XEnd::RIGHT, YEnd::CENTER));
    chain.with(VoronoiSpawning::new());
    chain
}
```

如果你现在运行游戏并传送到最后一个级别，"广场"会变成一个巨大的开放空间，充满了互相残杀的生物。我觉得这很有趣，但这并不是我们想要的结果。

![](./c75-emptymap.jpg)

广场需要被划分为包含广场内容的区域。这类似于我们之前为Voronoi地图所做的事情，但我们不打算创建细胞墙壁，而只是在其中放置内容的区域。让我们开始通过扩展地图构建器来调用一个名为`spawn_zones`的新函数：

```rust
impl InitialMapBuilder for PlazaMapBuilder {
    #[allow(dead_code)]
    fn build_map(&mut self, build_data : &mut BuilderMap) {
        self.empty_map(build_data);
        self.spawn_zones(build_data);
    }
}
```
我们将首先使用我们之前的 Voronoi 代码，使其始终具有 32 个种子并使用勾股定理计算距离：

```rust
fn spawn_zones(&mut self, build_data : &mut BuilderMap) {
    let mut voronoi_seeds : Vec<(usize, rltk::Point)> = Vec::new();

    while voronoi_seeds.len() < 32 {
        let vx = crate::rng::roll_dice(1, build_data.map.width-1);
        let vy = crate::rng::roll_dice(1, build_data.map.height-1);
        let vidx = build_data.map.xy_idx(vx, vy);
        let candidate = (vidx, rltk::Point::new(vx, vy));
        if !voronoi_seeds.contains(&candidate) {
            voronoi_seeds.push(candidate);
        }
    }

    let mut voronoi_distance = vec![(0, 0.0f32) ; 32];
    let mut voronoi_membership : Vec<i32> = vec![0 ; build_data.map.width as usize * build_data.map.height as usize];
    for (i, vid) in voronoi_membership.iter_mut().enumerate() {
        let x = i as i32 % build_data.map.width;
        let y = i as i32 / build_data.map.width;

        for (seed, pos) in voronoi_seeds.iter().enumerate() {
            let distance = rltk::DistanceAlg::PythagorasSquared.distance2d(
                rltk::Point::new(x, y),
                pos.1
            );
            voronoi_distance[seed] = (seed, distance);
        }

        voronoi_distance.sort_by(|a,b| a.1.partial_cmp(&b.1).unwrap());

        *vid = voronoi_distance[0].0 as i32;
    }

    // 这里将放置生成代码
}
```
在新的 `spawn_zones` 函数的末尾，我们有一个名为 `voronoi_membership` 的数组，它将每个图块分类为32个区域中的一个。这些区域保证是连续的。让我们编写一些快速代码来计算每个区域的大小，以验证我们的工作：

```rust
// Make a list of zone sizes and cull empty ones
let mut zone_sizes : Vec<(i32, usize)> = Vec::with_capacity(32);
for zone in 0..32 {
    let num_tiles = voronoi_membership.iter().filter(|z| **z == zone).count();
    if num_tiles > 0 {
        zone_sizes.push((zone, num_tiles));
    }
}
println!("{:?}", zone_sizes);
```
这将每次产生不同的结果，但可以让我们对创建的区域数量及其大小有一个大致的了解。下面是快速测试运行的结果：

```
[(0, 88), (1, 60), (2, 143), (3, 261), (4, 192), (5, 165), (6, 271), (7, 68), (8, 151), (9, 78), (10, 45), (11, 154), (12, 132), (13, 88), (14, 162), (15, 49), (16, 138), (17, 57), (18, 206), (19, 117), (20, 168), (21, 67), (22, 153), (23, 119), (24, 41), (25, 48), (26, 78), (27, 118), (28, 197), (29, 129), (30, 163), (31, 94)]
```

所以我们知道区域创建是有效的：有32个区域，没有一个区域过小 - 尽管有些区域相当大。让我们按大小降序排序这个列表：
```rust
zone_sizes.sort_by(|a,b| b.1.cmp(&a.1));
```
这会产生一个加权“重要性”地图：最大的区域排在前面，最小的区域排在最后。我们将根据重要性顺序在这些区域中生成内容。大的“传送门公园”保证是最大的区域。这是我们的创建系统的开始：
```rust
// 开始创建区域地形
zone_sizes.iter().enumerate().for_each(|(i, (zone, _))| {
    match i {
        0 => self.portal_park(build_data, &voronoi_membership, *zone),
        _ => {}
    }
});
```
`portal_park` 的占位符签名如下：
```rust
fn portal_park(&mut self, build_data : &mut BuilderMap, voronoi_membership: &[i32], zone: i32) {
}
```

我们将使用这种模式逐步填充广场。现在，我们将跳过传送门公园并首先添加一些其他特征。

### 坚固的岩石
我们从最简单的开始：将一些较小的区域转化为坚固的岩石。这些可能是精灵尚未开采的区域，或者更可能是他们留下这些区域来支撑洞穴。我们将使用一个之前未涉及的功能：“match guard”（匹配守卫）。你可以通过以下方式让`match`处理"大于"的情况：
```rust
// 开始创建区域地形
zone_sizes.iter().enumerate().for_each(|(i, (zone, _))| {
    match i {
        0 => self.portal_park(build_data, &voronoi_membership, *zone),
        i if i > 20 => self.fill_zone(build_data, &voronoi_membership, *zone, TileType::Wall),
        _ => {}
    }
});
```
实际的`fill_zone`函数非常简单：它找到区域内的所有瓦片并将其转化为墙壁：
```rust
fn fill_zone(&mut self, build_data : &mut BuilderMap, voronoi_membership: &[i32], zone: i32, tile_type: TileType) {
        voronoi_membership
            .iter()
            .enumerate()
            .filter(|(_, tile_zone)| **tile_zone == zone)
            .for_each(|(idx, _)| build_data.map.tiles[idx] = tile_type);
    }
```

这已经为我们的地图注入了一些生机：
![](./c75-solidrock.jpg)


### 水池

洞穴往往是阴冷潮湿的地方。黑暗精灵可能会喜欢一些水池——广场以宏伟的水池闻名！让我们扩展"默认"匹配，随机创建区域水池：
```rust
// 开始创建区域地形
zone_sizes.iter().enumerate().for_each(|(i, (zone, _))| {
    match i {
        0 => self.portal_park(build_data, &voronoi_membership, *zone),
        i if i > 20 => self.fill_zone(build_data, &voronoi_membership, *zone, TileType::Wall),
        _ => {
            let roll = crate::rng::roll_dice(1, 6);
            match roll {
                1 => self.fill_zone(build_data, &voronoi_membership, *zone, TileType::DeepWater),
                2 => self.fill_zone(build_data, &voronoi_membership, *zone, TileType::ShallowWater),
                _ => {}
            }
        }
    }
});```
注意当我们不匹配其他条件时，会掷骰子？如果结果是1或2，我们会添加不同深度的水池。实际添加水池与添加岩石类似——只是我们改为添加水。

添加一些水体特征后，区域更加生动：

![](./c75-pools.jpg)

### 钟乳石公园
钟乳石（以及它们的双胞胎石笋）是真实洞穴中的自然景观。它们是黑暗精灵公园的完美候选。我们希望在城市中增添一些色彩，所以用草地环绕它们。这些是精心培育的公园，为黑暗精灵的闲暇活动提供隐私空间（你不会想知道细节…）。

将其添加到"未知"区域选项中：
```rust
// 开始创建区域地形
zone_sizes.iter().enumerate().for_each(|(i, (zone, _))| {
    match i {
        0 => self.portal_park(build_data, &voronoi_membership, *zone),
        i if i > 20 => self.fill_zone(build_data, &voronoi_membership, *zone, TileType::Wall),
        _ => {
            let roll = crate::rng::roll_dice(1, 6);
            match roll {
                1 => self.fill_zone(build_data, &voronoi_membership, *zone, TileType::DeepWater),
                2 => self.fill_zone(build_data, &voronoi_membership, *zone, TileType::ShallowWater),
                3 => self.stalactite_display(build_data, &voronoi_membership, *zone),
                _ => {}
            }
        }
    }
});
```
使用与`fill_zone`类似的函数来用草地或钟乳石填充区域：
```rust
fn stalactite_display(&mut self, build_data : &mut BuilderMap, voronoi_membership: &[i32], zone: i32) {
        voronoi_membership
            .iter()
            .enumerate()
            .filter(|(_, tile_zone)| **tile_zone == zone)
            .for_each(|(idx, _)| {
                build_data.map.tiles[idx] = match crate::rng::roll_dice(1,10) {
                    1 => TileType::Stalactite,
                    2 => TileType::Stalagmite,
                    _ => TileType::Grass,
                };
            });
    }
```
### 公园与祭祀区

一些带有座椅的植被区域能增强公园氛围。这些应该是较大的区域——这是该区域的主题。我不认为黑暗精灵会坐着听音乐会，所以让我们在中间设置一个带有血渍的祭坛。注意我们如何在`match`中使用"或"语句来匹配第二和第三大的区域：
```rust
// 开始创建区域地形
zone_sizes.iter().enumerate().for_each(|(i, (zone, _))| {
    match i {
        0 => self.portal_park(build_data, &voronoi_membership, *zone),
        1 | 2 => self.park(build_data, &voronoi_membership, *zone),
        i if i > 20 => self.fill_zone(build_data, &voronoi_membership, *zone, TileType::Wall),
        _ => {
            let roll = crate::rng::roll_dice(1, 6);
            match roll {
                1 => self.fill_zone(build_data, &voronoi_membership, *zone, TileType::DeepWater),
                2 => self.fill_zone(build_data, &voronoi_membership, *zone, TileType::ShallowWater),
                3 => self.stalactite_display(build_data, &voronoi_membership, *zone),
                _ => {}
            }
        }
    }
});
```

实际创建公园的过程稍复杂：
```rust
fn park(&mut self, build_data : &mut BuilderMap, voronoi_membership: &[i32], zone: i32, seeds: &[(usize, rltk::Point)]) {
    let zone_tiles : Vec<usize> = voronoi_membership
        .iter()
        .enumerate()
        .filter(|(_, tile_zone)| **tile_zone == zone)
        .map(|(idx, _)| idx)
        .collect();

    // 初始全部设为草地
    zone_tiles.iter().for_each(|idx| build_data.map.tiles[*idx] = TileType::Grass);

    // 在中心添加石质区域
    let center = seeds[zone as usize].1;
    for y in center.y-2 ..= center.y+2 {
        for x in center.x-2 ..= center.x+2 {
            let idx = build_data.map.xy_idx(x, y);
            build_data.map.tiles[idx] = TileType::Road;
            if crate::rng::roll_dice(1,6) > 2 {
                build_data.map.bloodstains.insert(idx);
            }
        }
    }

    // 中心放置祭坛
    build_data.spawn_list.push((
        build_data.map.xy_idx(center.x, center.y),
        "Altar".to_string()
    ));

    // 为观众添加椅子
    zone_tiles.iter().for_each(|idx| {
        if build_data.map.tiles[*idx] == TileType::Grass && crate::rng::roll_dice(1, 6)==1 {
            build_data.spawn_list.push((
                *idx,
                "Chair".to_string()
            ));
        }
    });
}
```

我们首先收集可用瓦片列表，然后用草地覆盖。找到沃罗诺伊区域的中心点（即生成该区域的种子点），在该区域中心铺设道路。在中心生成祭坛、随机血渍和大量椅子。所有这些生成逻辑共同构成了一个（不太宜人的）主题公园。

这些公园区域看起来足够混乱：
![](./c75-altar.jpg)

### 添加通道

在这一点上，没有保证你实际上可以穿越地图。水和墙壁可能会以正好错误的方式巧合地阻塞你的前进。这不是一件好事！让我们使用我们在创建第一个Voronoi构建器时遇到的系统来识别Voronoi区域之间的边缘——并用道路替换边缘瓦片。这确保了区域之间有通道，同时也在地图上产生了漂亮的六边形效果。

首先，在 `spawn_zones` 的末尾添加一个调用道路构建器的调用：
```rust
// 清除路径
self.make_roads(build_data, &voronoi_membership);
```
现在我们实际上必须建造一些道路。大部分代码与Voronoi边缘检测相同。我们不是在区域内部放置地板，而是在边缘放置道路：

```rust
fn make_roads(&mut self, build_data : &mut BuilderMap, voronoi_membership: &[i32]) {
    for y in 1..build_data.map.height-1 {
        for x in 1..build_data.map.width-1 {
            let mut neighbors = 0;
            let my_idx = build_data.map.xy_idx(x, y);
            let my_seed = voronoi_membership[my_idx];
            if voronoi_membership[build_data.map.xy_idx(x-1, y)] != my_seed { neighbors += 1; }
            if voronoi_membership[build_data.map.xy_idx(x+1, y)] != my_seed { neighbors += 1; }
            if voronoi_membership[build_data.map.xy_idx(x, y-1)] != my_seed { neighbors += 1; }
            if voronoi_membership[build_data.map.xy_idx(x, y+1)] != my_seed { neighbors += 1; }

            if neighbors > 1 {
                build_data.map.tiles[my_idx] = TileType::Road;
            }
        }
    }
}
```
有了这个，地图就可以通行了。道路勾勒出边缘，而不显得过于方形：

![](./c75-edgeroads.jpg)

### 清理生成物

目前，地图非常混乱——而且很可能迅速杀死你。有大型开放区域，充满了坏蛋、陷阱（你为什么会在公园里建陷阱？）和四处散落的物品。混乱是好事，但有一种随机性是过多的。我们希望地图在随机中具有一定的意义。

让我们首先从构建器链中完全移除随机实体生成器：
```rust
pub fn dark_elf_plaza(new_depth: i32, width: i32, height: i32) -> BuilderChain {
    println!("黑暗精灵广场构建器");
    let mut chain = BuilderChain::new(new_depth, width, height, "黑暗精灵广场");
    chain.start_with(PlazaMapBuilder::new());
    chain.with(AreaStartingPosition::new(XStart::LEFT, YStart::CENTER));
    chain.with(CullUnreachable::new());
    chain.with(AreaEndingPosition::new(XEnd::RIGHT, YEnd::CENTER));
    chain
}
```
这将给你一个没有敌人的地图，尽管它仍然有一些椅子和祭坛。这是一个“主题公园”地图——所以我们将在给定区域中保留对生成物的控制。它对玩家的帮助很少——我们即将到达终点，所以他们可能已经准备好了！

让我们开始在公园/祭坛区域放置一些怪物。一个黑暗精灵家族或另一个家族在那里，导致敌人集群。现在找到`park`函数，我们将扩展“添加椅子”部分：
```rust
// 为观众添加椅子，以及观众自己
let available_enemies = match crate::rng::roll_dice(1, 3) {
    1 => vec![
        "Arbat黑暗精灵",
        "Arbat黑暗精灵领袖",
        "Arbat兽人奴隶",
    ],
    2 => vec![
        "Barbo黑暗精灵",
        "Barbo哥布林弓箭手",
    ],
    _ => vec![
        "Cirro黑暗精灵",
        "Cirro黑暗女祭司",
        "Cirro蜘蛛",
    ]
};

zone_tiles.iter().for_each(|idx| {
    if build_data.map.tiles[*idx] == TileType::Grass {
            match crate::rng::roll_dice(1, 10) {
            1 => build_data.spawn_list.push((
                    *idx,
                    "Chair".to_string()
                )),
            2 => {
                let to_spawn = crate::rng::range(0, available_enemies.len() as i32);
                build_data.spawn_list.push((
                    *idx,
                    available_enemies[to_spawn as usize].to_string()
                ));
            }
            _ => {}
        }
    }
});
```
我们在这里做了几件新事情。我们随机为公园分配一个所有者——A、B或C组的黑暗精灵。然后我们为每个组制作一个可用的生成列表，并在该公园中生成一些。这确保了公园*开始*时由一个派系拥有。由于他们经常可以看到对方，所以大屠杀即将开始——但至少这是有主题的大屠杀。

我们将把钟乳石画廊和水池留空，没有敌人。它们只是装饰，提供了一个安静的区域来隐藏/休息（看？我们并不是完全不公道！）。

### 传送门公园

现在我们已经完成了地图的基本形状，是时候专注于公园了。要做的第一件事是阻止出口随机生成。更改基本地图构建器以不包括出口放置：
```rust
pub fn dark_elf_plaza(new_depth: i32, width: i32, height: i32) -> BuilderChain {
    println!("黑暗精灵广场构建器");
    let mut chain = BuilderChain::new(new_depth, width, height, "黑暗精灵广场");
    chain.start_with(PlazaMapBuilder::new());
    chain.with(AreaStartingPosition::new(XStart::LEFT, YStart::CENTER));
    chain.with(CullUnreachable::new());
    chain
}
```
这将使你完全没有出口。我们希望将其放置在传送门公园的中心。让我们扩展函数签名以包括沃罗诺伊种子，并使用种子点放置出口——就像我们对其他公园所做的那样：
```rust
fn portal_park(&mut self, build_data : &mut BuilderMap, voronoi_membership: &[i32], zone: i32, seeds: &[(usize, rltk::Point)]) {
    let center = seeds[zone as usize].1;
    let idx = build_data.map.xy_idx(center.x, center.y);
    build_data.map.tiles[idx] = TileType::DownStairs;
}
```
现在，让我们通过在公园里覆盖砾石来使传送门公园更加突出：
```rust
fn portal_park(&mut self, build_data : &mut BuilderMap, voronoi_membership: &[i32], zone: i32, seeds: &[(usize, rltk::Point)]) {
    let zone_tiles : Vec<usize> = voronoi_membership
        .iter()
        .enumerate()
        .filter(|(_, tile_zone)| **tile_zone == zone)
        .map(|(idx, _)| idx)
        .collect();

    // 全部开始为砾石
    zone_tiles.iter().for_each(|idx| build_data.map.tiles[*idx] = TileType::Gravel);

    // 添加出口
    let center = seeds[zone as usize].1;
    let idx = build_data.map.xy_idx(center.x, center.y);
    build_data.map.tiles[idx] = TileType::DownStairs;
}
```
接下来，我们将在出口周围添加一些祭坛：
```rust
// 在出口周围添加一些祭坛
let altars = [
    build_data.map.xy_idx(center.x - 2, center.y),
    build_data.map.xy_idx(center.x + 2, center.y),
    build_data.map.xy_idx(center.x, center.y - 2),
    build_data.map.xy_idx(center.x, center.y + 2),
];
altars.iter().for_each(|idx| build_data.spawn_list.push((*idx, "Altar".to_string())));
```
这为通往深渊的出口提供了一个很好的开始。你将出口放在了正确的位置，周围有令人毛骨悚然的祭坛，并且有一个清晰的通道。它也没有风险（除了地图上到处都是精灵互相残杀之外）。

让我们在出口处添加一场Boss战斗，使出口更加具有挑战性。这是通往深渊之前的最后一项大挑战，所以这是一个自然的地点。我随机生成了一个恶魔名字，并决定将Boss命名为“Vokoth”。让我们在出口旁边的一个格子生成它：

```rust
let demon_spawn = build_data.map.xy_idx(center.x+1, center.y+1);
build_data.spawn_list.push((demon_spawn, "Vokoth".to_string()));
```

这段代码不会起任何作用，除非我们定义Vokoth！我们想要一个强大的反派角色。让我们回顾一下在`spawns.json`中如何定义黑龙的：
```rust
{
    "name" : "Black Dragon",
    "renderable": {
        "glyph" : "D",
        "fg" : "#FF0000",
        "bg" : "#000000",
        "order" : 1,
        "x_size" : 2,
        "y_size" : 2
    },
    "blocks_tile" : true,
    "vision_range" : 12,
    "movement" : "static",
    "attributes" : {
        "might" : 13,
        "fitness" : 13
    },
    "skills" : {
        "Melee" : 18,
        "Defense" : 16
    },
    "natural" : {
        "armor_class" : 17,
        "attacks" : [
            { "name" : "bite", "hit_bonus" : 4, "damage" : "1d10+2" },
            { "name" : "left_claw", "hit_bonus" : 2, "damage" : "1d10" },
            { "name" : "right_claw", "hit_bonus" : 2, "damage" : "1d10" }
        ]
    },
    "loot_table" : "Wyrms",
    "faction" : "Wyrm",
    "level" : 6,
    "gold" : "20d10",
    "abilities" : [
        { "spell" : "Acid Breath", "chance" : 0.2, "range" : 8.0, "min_range" : 2.0 }
    ]
},
```
这是一个非常强大的怪物，可以作为深渊恶魔的好模板。让我们克隆它（复制粘贴时间！）并为Vokoth构建一个条目：
```rust
{
    "name" : "Vokoth",
    "renderable": {
        "glyph" : "&",
        "fg" : "#FF0000",
        "bg" : "#000000",
        "order" : 1,
        "x_size" : 2,
        "y_size" : 2
    },
    "blocks_tile" : true,
    "vision_range" : 6,
    "movement" : "static",
    "attributes" : {
        "might" : 13,
        "fitness" : 13
    },
    "skills" : {
        "Melee" : 18,
        "Defense" : 16
    },
    "natural" : {
        "armor_class" : 17,
        "attacks" : [
            { "name" : "whip", "hit_bonus" : 4, "damage" : "1d10+2" }
        ]
    },
    "loot_table" : "Wyrms",
    "faction" : "Wyrm",
    "level" : 8,
    "gold" : "20d10",
    "abilities" : []
}
```
现在如果你玩游戏，你会在通往深渊的出口处遇到一个讨厌的恶魔怪物。

![](./c75-vokoth.jpg)

## 收尾工作

我们现在已经完成了倒数第二个部分！你可以战斗到底层到达黑暗精灵广场，并找到通往深渊的 gateway - 但只有在你能够躲避一个庞大的恶魔和一群精灵的情况下——几乎没有任何帮助提供。接下来，我们将开始构建深渊。

---

**本章的源代码可以在这里找到[这里](https://github.com/thebracket/rustrogueliketutorial/tree/master/chapter-75-darkplaza)**


[在浏览器中运行本章的示例，使用web assembly (需要WebGL2)](https://bfnightly.bracketproductions.com/rustbook/wasm/chapter-75-darkplaza)
---

版权所有 (C) 2019, Herbert Wolverson.

---
