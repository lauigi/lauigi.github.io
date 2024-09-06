---
layout: post
title:  "Build a Minesweeper Game with Rust and Bevy Engine"
date:   2021-03-14 17:05:00 +0800
categories: rust
---
You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

Jekyll requires blog post files to be named according to the following format:

`YEAR-MONTH-DAY-title.MARKUP`

Where `YEAR` is a four-digit number, `MONTH` and `DAY` are both two-digit numbers, and `MARKUP` is the file extension representing the format used in the file. After that, include the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets:

{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/

# Rust + Bevy Engine 游戏开发初学体验 （《扫雷》）

## 背景

背景就是没有游戏开发背景，甚至没有静态语言背景。15年Rust 1.0的时代被王院长安利过这个语言，学习曲线很陡，当时简单的hello world一下就扔掉了。后来逐渐听说Rust和wasm相关的前端方向的应用，这阵子又开始关注起来。

这些年的前端从业过程中的一个切身体会是最好要拓宽自己的能力面，不要自限于Vue.js、React这些关键词所编织的日常中（一如后台的CURD boys的日常困扰），所以利用春节假期看了一遍trpl和rustlings。又因为只看书和做题感觉很不牢靠，所以选择用剩下的几天假期实际做一个小游戏试试。

引擎的选择很简单，github trending上找到比较靠前的bevy就开始了。通过bevy，又了解到游戏开发中常见的ECS的架构模式。bevy正是以这种架构模式搭建的。

Rust的各种设计都有一种站在前人的肩膀上的感觉，整个体验过程很多这个感觉的验证，语言设计、包管理、lint和测试等等。

整个项目的源码位于[https://github.com/lauigi/bevy_mine_sweeper](https://github.com/lauigi/bevy_mine_sweeper)

![mine](/assets/mine.gif)

## 准备

### 语言方面

#### 安装Rust

参考官方指南。rustup很方便了，在mac和windows上都试过。win上的安装稍微多留意一下“Microsoft C++ 生成工具”。

### 引擎方面

如常在cargo.toml文件中加入bevy这个依赖后，cargo run或者test的时候就会自动安装了。我这里在实际引入bevy后编译的时候，一开始都报一段包含“d3d12.lib”字样的乱码。我猜测是没找到这个名字的库，于是辗转搜索到，directX sdk已经不再单独提供，需要安装windows sdk，而windows sdk又是通过前述的“Microsoft C++ 生成工具”来安装的，所以重新安装并记得勾选对应的选项就解决了。

## 体验过程

至少对我个人来说，边学边写的过程最好每个最小的进展都可以有明确快速的自我反馈。起步阶段除了频繁的使用println!宏外，不得不夸一下Rust单测功能的方便。Rust支持开发者在每个.rs文件里面直接放置测试代码，并且不需要额外配置，直接`cargo test`，cargo就会找到你的所有代码文件里面的test代码块来运行测试。到开始写引擎相关的部分后，则是通过画面、飘在窗口的一些debug文本来直观地审视实现效果。

### 基本的结构

扫雷的游戏机制很简单，比如在8 x 8大小的64个方块中，随机挑选10个方块视作地雷，其余的方块会标注出自身周围8个方块中地雷的个数。通过打开所有非地雷的方块达成胜利。

开始的开始，需要考虑游戏的地图数据如何表达。这里我设计成由包含每个地雷块的二维数组来实现。另外，地图也许直接保存下自身的宽高会方便一些，也许不会，总之也先放在一起：
```rust
struct MinePlayground {
    width: usize,
    height: usize,
    map: Vec<Vec<MineBlock>>,
}
```
Vec就是Rust中的动态数组，可以类比js的Array类型。MineBlock是还没实现的地雷块的结构体。

接下来分析每个地雷块的特征，一方面每个块可能是“地雷”、“数字”、“空格”这几种自身的类型（空格其实可以视作数字0，但初期不太能纠结出是否合并的必要，所以先这样处理了），另一方面又可能处于“未点击”、“标注为雷”、”问号“、”已打开“这几种状态，所以设计了两个枚举类型来表征这两个数据：
```rust
#[derive(Debug)]
enum BlockType {
    Mine,
    Space,
    Tip(usize),
}
#[derive(Debug)]
enum BlockStatus {
    Shown,
    Hidden,
    QuestionMarked,
    Flaged,
}
```
这里额外解释下`#[derive(Debug)]`这行，是为了使对应的结构体或枚举有方便程序员的格式化输出，可以参考trpl的附录C。所以最初的MineBlock结构体就是这样：
```rust
struct MineBlock {
    btype: BlockType,
    bstatus: BlockStatus,
}
```
数据准备好后首先尝试实现生成地图的功能。这个生成功能应该是位于MinePlayground这个层级的，所以我将其作为MinePlayground的方法来实现：
```rust
impl MinePlayground {
    pub fn init(&width: &usize, &height: &usize, &mine_count: &usize) -> Result<MinePlayground, String> {
        if !SIZE_RANGE.contains(&width) || !SIZE_RANGE.contains(&height) || !MINE_COUNT_RANGE.contains(&mine_count) {
            return Err(String::from("Parameters not in specific range!"));
        }
        // other code here
    }
}
```
函数签名的设计上，因为这个方法并不需要诸如宽高、地雷数量这些数据的所有权，所以应该使用引用就足够了。返回值上，因为会有诸如参数非法的情况（实际并不会有，只是为了巩固Rust学习成果），所以使用了枚举`Result<T, E>`作为返回值。成功时返回一个MinePlayground，失败则是一段错误文本。初期就预估到这是个对外的方法，所以直接加了pub关键字。"SIZE_RANGE"等常量，是我设定的合法的取值范围：
```rust
static SIZE_RANGE: std::ops::Range<usize> = 5..200;
```
当然，后面实际也没什么用。
如果要为这个方法写单测的话，想了下面这段：
```rust
#[cfg(test)]
mod tests{
    use super::*;
    #[test]
    fn test_init_map() {
        assert!(MinePlayground::init(&0, &0, &0).is_err());
        assert!(MinePlayground::init(&8, &8, &10).is_ok());
    }
}
```
回到正题。接下来面临的是如何把指定个数的地雷随机排布到地图上。这里查到的办法是利用Rust的rand库的shuttle方法，这个方法可以随机打乱一个序列的顺序：
```rust
let seeds_length = width * height;
let mut mine_seeds: Vec<bool> = Vec::with_capacity(seeds_length.into());
for i in 0..seeds_length {
    if i < mine_count { mine_seeds.push(true); }
    else { mine_seeds.push(false); }
}
let mut rng = rand::thread_rng();
mine_seeds.shuffle(&mut rng);
```
也就是生成一个长度等于块总数的动态数组，先把雷放在前面，标为true，然后用shuttle打乱顺序。一个热知识，通过with_capacity这类方法生成的实际上没有填充内容的动态数组，是没法走Rust的迭代器的，所以这里用了for in。
接下来是把这个一维的动态数组内容拆到每行有width个元素的“二维”动态数组中。后来反思了下，如果写好坐标转换的方法，似乎就没必要这要拆一下？这一步操作也顺路直接填了MineBlock结构体进去：
```rust
let mut mine_map: Vec<Vec<MineBlock>> = vec![];
for i in 0..height {
    mine_map.push(mine_seeds[i * width..i * width + width].iter().map(|&is_mine_block| {
        MineBlock {
            btype: if is_mine_block { BlockType::Mine } else { BlockType::Space },
            ..Default::default()
        }
    }).collect());
}
```
使用Default这个trait的原因是，后续假如扩展字段又不需要在这里特别定义的话，可以不用再修改这里，很方便。向地图中填充好地雷的block之后，接下来需要为其他方块生成提示数据，这里要做一些准备公所。首先是用到了一个获取某个block周围的block的坐标的方法（暴力了些，不知道是否有更合适的写法）
```rust
fn get_surroundings(&x: &usize, &y: &usize, &max_width: &usize, &max_height: &usize) -> Vec<(usize, usize)> {
    let max_x = max_width - 1;
    let max_y = max_height - 1;
    let mut r = vec![];
    if x > 0 { r.push((x - 1, y)); }
    if x < max_x { r.push((x + 1, y)); }
    if y > 0 {
        r.push((x, y - 1));
        if x > 0 { r.push((x - 1, y - 1)); }
        if x < max_x { r.push((x + 1, y - 1)); }
    }
    if y < max_y {
        r.push((x, y + 1));
        if x > 0 { r.push((x - 1, y + 1)); }
        if x < max_x { r.push((x + 1, y + 1)); }
    }
    r
}
```
结尾不带分号的这一行在Rust中等价于`return r;`。另一项准备工作是给BlockType实现一个自增的方法：
```rust
impl BlockType {
    fn increase (&mut self) {
        *self = match *self {
            Self::Tip(val) => Self::Tip(val + 1),
            Self::Space => Self::Tip(1),
            Self::Mine => Self::Mine,
        }
    }
}
impl MineBlock {
    fn add_tip(&mut self) {
        self.btype.increase();
    }
}
```
这样子在填数字的遍历中的代码可以简单一些：
```rust
for y in 0..mine_map.len() {
    let row = &mine_map[y];
    for x in 0..row.len() {
        if let BlockType::Space = mine_map[y][x].btype {
            let surroundings = get_surroundings(&x, &y, &width, &height);
            for (cur_x, cur_y) in surroundings.iter() {
                if let BlockType::Mine = mine_map[*cur_y][*cur_x].btype {
                    mine_map[y][x].add_tip();
                }
            }
        }
    }
}
```
最后把MinePlayground组装好，包在Ok中返回，就算暂时完成了init方法：
```rust
Ok(MinePlayground {
    width,
    height,
    map: mine_map,
})
```

### 从数据到渲染出初始画面

这部分开始就要引入bevy引擎了。
```rust
pub fn game_app(config: GameConfig) {
    App::build()
        .add_resource(WindowDescriptor {
            vsync: false,
            width: cmp::max(config.width * BLOCK_WIDTH, MIN_WIDTH) as f32,
            height: cmp::max(config.height * BLOCK_WIDTH + Y_MARGIN, MIN_HEIGHT) as f32,
            title: String::from("Mine Sweeper"),
            resizable: false,
            ..Default::default()
        })
        .add_resource(config)
        .add_plugins(DefaultPlugins)
        .add_plugin(GamePlugin)
        .run();
}
```
首先通过将WindowDescriptor这个结构体加入资源来实现对窗口参数的一些定制。config中是我们预先取得的地图宽高。参考其他扫雷游戏实现，这里决定禁用窗口缩放功能。另外，我也把config加入到资源中，以便后面取用。DefaultPlugins是bevy官方提供的一组默认插件，具体可以参阅文档。一般认为没有特殊需要的话直接引入就可以了。GamePlugin则是这个游戏本身的逻辑代码。

在ECS的体系内，实体（Entity）与组件（Component）都只是数据，所有动作、表达相关的功能，都应该放到系统（System）中实现，系统也会相应地读取、修改实体或组件。

```rust
struct GamePlugin;
impl Plugin for GamePlugin {
    fn build(&self, app: &mut AppBuilder) {
        app.add_startup_system(setup.system());
    }
}
```
add_startup_system是确保只会调用一次的system，而普通的add_system会在每一个tick中执行。

先来考虑一些初始化的设置：
```rust
fn setup(
    commands: &mut Commands,
    asset_server: Res<AssetServer>,
    mut texture_atlases: ResMut<Assets<TextureAtlas>>,
) {
    let font = asset_server.load("fonts/pointfree.ttf");
    let window = windows.get_primary().unwrap();
    commands
        .spawn(CameraUiBundle::default())
        .spawn(Camera2dBundle::default());
    commands.spawn(TextBundle {
        style: Style {
            align_self: AlignSelf::FlexEnd,
            position_type: PositionType::Absolute,
            position: Rect {
                top: Val::Px(5.0),
                left: Val::Px(5.0),
                ..Default::default()
            },
            ..Default::default()
        },
        text: Text {
            value: "debug text here".to_string(),
            font: font.clone(),
            style: TextStyle {
                font_size: 18.0,
                color: Color::rgba(0.0, 0.5, 0.5, 0.5),
                alignment: TextAlignment::default(),
            },
        },
        ..Default::default()
    }).with(DebugText);
    let texture_handle = asset_server.load("textures/block.png");
    let texture_atlas = TextureAtlas::from_grid(texture_handle, Vec2::new(SPRITE_SIZE, SPRITE_SIZE), 4, 1);
    let texture_atlas_handle = texture_atlases.add(texture_atlas);
    commands.insert_resource(texture_atlas_handle);
```
起手开始写的时候，一度纠结bevy中的system可以接收哪些参数的这个问题。结论是可以接受若干个特定类型的参数，如Commands、Res、Query等。Commands可以用来增加删除Entity、向Entity中增加删除组件、向app中插入资源等。一个需要注意的点是，Commands的生效不是同步的，一般在下一帧或者当前stage结束时。

CameraUiBundle / Camera2dBundle是一系列camera相关的实体。TextBundle是预先准备的用来展示debug信息的文本框。这里感觉到布局写法还是很像css的，还是说其实所有语言、UI的布局写法都差不多？结尾部分读取了用于每个地雷块展示的图片，并解析为具有4个格子的sprite素材，作为资源插入app中。

![image-20210303110048974](/assets/image-20210303110048974.png)
上图是我准备的游戏素材图，也就是前文中的block.png。接下来的步骤是实际开发（学习）体验中耗时最久的一步，也就是如何把地图从数据绘制成图像。也是为了尽快看到效果，所以暂时没考虑后续如何更新的事情：
```rust
struct WindowOffset {
    x: f32,
    y: f32,
}
struct RenderBlock {
    pos: Position,
}

fn setup (/* other code */) {
    // other code
    commands.insert_resource(WindowOffset {
        x: window.width() as f32 / 2.0 - BLOCK_WIDTH as f32 / 2.0,
        y: window.height() as f32 / 2.0 - BLOCK_WIDTH as f32 / 2.0,
    });
}

fn init_map_render(
    commands: &mut Commands,
    texture_atlases: Res<Assets<TextureAtlas>>,
    atlas_handle: Res<Handle<TextureAtlas>>,
    window_offset: Res<WindowOffset>,
    config: Res<GameConfig>,
) {
    for y in 0..config.height {
        for x in 0..config.width {
            let texture_atlas = texture_atlases.get_handle(atlas_handle.clone());
            commands
                .spawn(SpriteSheetBundle {
                    transform: Transform {
                        translation: Vec3::new(
                            (x * BLOCK_WIDTH) as f32 - window_offset.x,
                            (y * BLOCK_WIDTH) as f32 - window_offset.y,
                            0.0
                        ),
                        scale: Vec3::splat(0.5),
                        ..Default::default()
                    },
                    texture_atlas,
                    sprite: TextureAtlasSprite::new(HIDDEN_INDEX as u32),
                    ..Default::default()
                })
                .with(RenderBlock { pos: Position { x, y } });
        }
    }
}
```
总体来说就是为每一个MineBlock都在画布上绘制一个方块，HIDDEN_INDEX是前面block.png中代表隐藏状态的方块的序号。这里同时也向实体中放了RenderBlock这个component，是为考虑后面做query取出时可能方便一些。因为初始的绘制坐标是画布的中点，所以这里还需要用到一个由窗口和块的尺寸计算出的偏移值。

在开始下一节之前，应该是时候向app中注入单局游戏的地图数据了：
```rust
struct MapData {
    map_entity: Entity,
}

fn new_map(
    commands: &mut Commands,
    config: Res<GameConfig>,
) {
    commands.spawn((MinePlayground::init(&config.width, &config.height, &config.mine_count).unwrap(), ));
    commands.insert_resource(MapData {
        map_entity: commands.current_entity().unwrap(),
    });
}
```
写这段文字的时候突然想起来，这里把MinePlayground额外包在MapData结构里的操作是不是有些多余？有时间我会修改试试（立flag）。

### 数据响应点击

接下来我希望呈现的效果是鼠标点击后能够更新对应的“雷块”的图片。拆解一下，应该先实现捕获鼠标点击事件并修改数据源。鼠标位置的获取方式很轻易地就在stackoverflow上找到了写法，修改后大概是下面的样子：
```rust
struct CursorLocation(Vec2);

impl Plugin for GamePlugin {
    fn build(&self, app: &mut AppBuilder) {
        // other code
            .add_resource(CursorLocation(Vec2::new(0.0, 0.0)))
            .add_system(handle_movement.system());
    }
}

fn handle_movement(
    mut cursor_pos: ResMut<CursorLocation>,
    cursor_moved_events: Res<Events<CursorMoved>>,
    mut evr_cursor: Local<EventReader<CursorMoved>>,
) {
    for ev in evr_cursor.iter(&cursor_moved_events) {
        cursor_pos.0 = ev.position;
    }
}
```
总体来讲，是把鼠标位置视作资源插入到app中，并在后面监听鼠标事件、更新资源。在开始写点击事件处理之前，还有一些功能需要准备。一个是判定鼠标坐标是否位于可点击区域：
```rust
fn get_block_index_by_cursor_pos(pos: Vec2, config: GameConfig) -> Option<(usize, usize)> {
    let x = (pos.x / BLOCK_WIDTH as f32).floor() as usize;
    let y = (pos.y / BLOCK_WIDTH as f32).floor() as usize;
    if (0..config.height).contains(&y) && (0..config.width).contains(&x) {
        return Some((x, y));
    }
    None
}
```
另外就是游戏核心部分对左击和右击的处理：
```rust
pub enum ClickResult {
    Wasted,
    NothingHappened,
    Win,
}
impl MinePlayground {
    // ...
    pub fn click(&mut self, x: &usize, y: &usize) -> ClickResult {
        // code here
    }
    pub fn right_click(&mut self, x: &usize, y: &usize) {
        // code here
    }
}
```
在扫雷的设计中，左击是有可能对游戏状态产生影响的，这里定义了一个ClickResult枚举来表征这种影响：挂了/获胜/什么事都没发生。点击在一个空白块，也就是0块上的操作稍微复杂些，因为等于要对这个块周围的块也做一次点击操作。右击的处理简单些，只是在“未打开”/“插旗”/“标问号”这几个状态中循环。

### 画面响应数据

核心的点击响应方法实现后，需要在游戏流程这里响应引擎的点击输入事件，并调用对应的响应方法。bevy是把input也处理成资源的：
```rust
fn handle_click(
    btns: Res<Input<MouseButton>>,
    cursor_pos: Res<CursorLocation>,
    config: Res<GameConfig>,
    mut mquery: Query<&mut MinePlayground>,
    map_data: Res<MapData>,
) {
    if btns.just_released(MouseButton::Left) {
        if let Some((x, y)) = get_block_index_by_cursor_pos(cursor_pos.0, *config) {
            let mut mp: Mut<MinePlayground> = mquery.get_component_mut(map_data.map_entity).unwrap();
            let click_result = mp.click(&x, &y);
            match click_result {
                ClickResult::Wasted => {
                    // code here
                },
                ClickResult::Win => {
                    // code here
                }
                _ => {}
            }
        }
    }
    if btns.just_released(MouseButton::Right) {
        if let Some((x, y)) = get_block_index_by_cursor_pos(cursor_pos.0, *config) {
            let mut mp: Mut<MinePlayground> = mquery.get_component_mut(map_data.map_entity).unwrap();
            mp.right_click(&x, &y);
        }
    }
}
```
这里的`Query<&mut MinePlayground>`就是用来取出之前加入到app中的地图实体。确保点击发生在某个块的范围后，就把对应的坐标数据传入对应的点击响应方法中，最后再对不同的click_result做处理。具体的处理后面再加，现在想搞定的是把数据的变化反映在图像上。

bevy这里提供了如`Added<T>`、`Mutated<T>`、`Changed<T>`这一系列查询过滤器（在rust范畴应该叫智能指针）来实现只监听发生过变化的实体的功能，所以可以做到在地图数据有变化的时候，也就是点击后再更新画面：
```rust
fn render_map (
    query: Query<
        &MinePlayground,
        Changed<MinePlayground>, 
    >,
    mut sprites: Query<(&mut TextureAtlasSprite, &RenderBlock)>,
) {
    // code here
}
```
接下来不得不考虑如何展示每个块上的提示数字。一开始的想法是在ui的体系内给每个块绘制文本，但这里的问题是bevy的ui和2dcamera处于不同的体系内，想做到对齐感觉上需要很多工作量。所以我还是选择了把数字也绘制到精灵图中，最后block.png扩展成了这个样子：![block](/assets/block.png)

槽位多了以后需要准备一个对应格子状态与图像的方法：
```rust
impl MineBlock {
    fn get_sprite_index(&self) -> usize {
        match self.bstatus {
            BlockStatus::Flaged => 12,
            BlockStatus::QuestionMarked => 11,
            BlockStatus::Shown => {
                match self.btype {
                    BlockType::Mine => 9,
                    BlockType::Tip(val) => val,
                    BlockType::Space => 0,
                }
            },
            BlockStatus::Hidden => HIDDEN_INDEX,
        }
    }
}
```
以上，renader_map的内容就可以填充了：
```rust
for mp in query.iter() {
    for (mut sprite, rb) in sprites.iter_mut() {
        sprite.index = mp.map[rb.pos.y][rb.pos.x].get_sprite_index() as u32;
    }
}
```

### 整理游戏stage

前面的开发已经实现了游戏的基本能力，但现在还没法重新开始每一局，也就是需要整理游戏的stage并加以实现。bevy作为游戏引擎，提供了把某些system定义到某个stage的某个阶段来执行的能力。这里首先定义用以描述各个阶段的枚举：

```rust
#[derive(Debug, Clone, PartialEq)]
enum GameState {
    Prepare,
    Ready,
    Running,
    Over,
}
```
整理过后的build过程是这样子的：
```rust
impl Plugin for GamePlugin {
    fn build(&self, app: &mut AppBuilder) {
        app.init_resource::<ButtonMaterials>()
            .add_resource(CursorLocation(Vec2::new(0.0, 0.0)))
            .add_plugin(FrameTimeDiagnosticsPlugin)
            .add_resource(State::new(GameState::Prepare))
            .add_startup_system(setup.system())
            .add_system(fps_update.system())
            .add_system(debug_text_update.system())
            .add_system(restart_button_system.system())
            .add_startup_system(new_map.system())
            .add_system(handle_movement.system())
            .add_system(handle_click.system())
            .add_system(render_map.system())
            .add_stage_after(stage::UPDATE, STAGE, StateStage::<GameState>::default())
            .on_state_enter(STAGE, GameState::Prepare, init_map_render.system())
            .on_state_enter(STAGE, GameState::Ready, new_map.system());
    }
}
```
当然也需要在处理点击事件等方法里面也要加上有关stage控制的语句，另外也需要添加一个用来重新开始游戏的按钮。具体的代码不在本文展开了。

### 体验优化

一个常见的扫雷游戏体验优化是，如果监测到玩家第一次点击就踩了地雷，就在结算前偷偷把这个雷移走。在现有的架构上拓展这个功能还是比较方便的。也就是为MinePlayground结构体实现一个fix方法，内容是把传入坐标的那一块改为不是地雷，取到周围的块，将周围的块的提示数减1，当前块的提示数根据周围的雷块数量逐个加1。

减少的这个地雷从地图的一角开始遍历找到第一个非雷的空格（注意排除掉当前点击的这一格），然后再修正周围的提示数字就ok了。

### TODOs

后面懒得实现的扫雷必备功能。

#### 计时

计时功能参考cheatbook应该很容易就实现了，使用time资源，并且与游戏的stage做相关的绑定

#### 计数被标记的块

为MinePlayground结构体再加一个字段，然后在点击的回调里对这个数字进行处理。

## 总结

bevy还在发展阶段，book很不完整，非官方文档（cheatbook）内容组织也很散，知识获取上对初学者来讲需要更多耗时与精力。rust的实际编码体验就是和编译器战斗，至少在我这个初学阶段是这样的。整个项目完成下来大概用了三四天时间。（也不是完全投入的，因为中间还穿插着家务、icode理论考试等等事情）

对于rust/wasm在真·前端领域的应用（就是除了deno），除了重计算的业务（比如某个网盘上传前计算文件特征/播放器也许也算）、代码安全要求高的业务（一大坨wasm总比安慰式压缩混淆的js好吧），还看不到能够挤掉js的部分。最近看到yew的更新是加了类似react的hooks api，就感觉到大佬们是不是也没有明确的发展方向？也希望听听公司的前端老哥们的想法。

好了，就是这些。