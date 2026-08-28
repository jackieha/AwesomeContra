# 数据格式规范

本文定义 AwesomeContra 所有外部数据文件的格式。每一节都给出**可直接复制使用的完整示例**——示例即规范，字段列表只是对示例的注解。

通用规则：

- 编码 UTF-8，无 BOM，LF 换行。
- **JSON 不支持注释**。本文示例中的 `//` 注释仅供阅读，真实文件里必须删除。需要说明时用 `_note` 字段（所有加载器忽略下划线开头的键）。
- 所有格式带 `version` 整数字段。加载器遇到不认识的 `version` 必须**硬失败**，不允许猜测兼容。
- 校验失败一律硬失败（`throw`），不允许静默使用默认值。理由见 `design.md` §7。
- 坐标单位一律是**逻辑像素**（256×224 坐标系），不是 tile、不是屏幕像素。tile 索引另说明。
- 所有资源路径相对于仓库根，用 `/` 分隔。

---

## 1. Tilemap JSON

关卡地形、碰撞与实体放置。一个文件一关。位置：`assets/levels/<id>.json`。

### 1.1 设计要点

- **图层与碰撞分离**：`layers` 是画什么（视觉），`collision` 是撞什么（物理）。二者尺寸相同但内容独立——一块看起来是草丛的 tile 可以是 `solid`，一块看起来实心的墙可以是 `empty`（用于关底 Boss 的假墙）。混在一起会让美术改图意外改掉手感。
- **tile 索引从 1 开始，`0` 表示空**。这样稀疏图层里大量的 0 一眼可辨。
- 图层数据是**行优先的一维数组**，长度必须恰好等于 `width * height`。用一维数组而非二维是为了加载时零拆包、直接 `Int16Array` 化。

### 1.2 碰撞类型

| 值 | 名称 | 语义 |
|---|---|---|
| `0` | `empty` | 无碰撞 |
| `1` | `solid` | 四面实体，标准阻挡 |
| `2` | `oneway` | 单向平台：只在**从上方下落**时阻挡；上升与横向穿过；`下+跳` 可穿透下落 |
| `3` | `water` | 水域体积，触发玩家 `WATER` 状态（`design.md` §6.6）；不阻挡移动 |
| `4` | `hazard` | 立即致死（尖刺、深渊底） |
| `5` | `destructible` | 可破坏实体，被子弹命中后变 `empty`（桥面用） |

### 1.3 完整示例

`assets/levels/jungle.json` —— 为可读性裁成 24×14 tile 的一个片段；真实关卡 `width` 数千。

```jsonc
{
  "version": 1,
  "id": "jungle",
  "name": "Jungle",
  "tileSize": 16,              // 必须等于 design.md 的 TILE，加载器断言
  "width": 24,                 // 单位 tile，不是像素
  "height": 14,                // 14 * 16 = 224 = LOGICAL_H，第 1 关不做垂直滚动
  "atlas": "assets/atlas/jungle.json",

  // 卷轴与镜头
  "scroll": {
    "minX": 0,                 // 逻辑像素；摄像机左边界
    "maxX": 128,               // 摄像机左上角 x 的最大值 = width*16 - LOGICAL_W = 384-256
    "autoScroll": 0            // px/s，0 = 由玩家推进；>0 = 强制卷轴段
  },

  // 背景：不参与碰撞，按 parallax 系数偏移绘制
  "background": {
    "color": "sky",            // 调色板语义名，不是 hex——换调色板时不用改关卡
    "layers": [
      { "atlasFrame": "bg_treeline", "parallax": 0.25, "y": 96, "repeatX": true },
      { "atlasFrame": "bg_bush",     "parallax": 0.5,  "y": 160, "repeatX": true }
    ]
  },

  // 视觉图层，绘制顺序即数组顺序（后者压前者）
  "layers": [
    {
      "name": "terrain",
      "data": [
        0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
        0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
        0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
        0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
        0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
        0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
        0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
        0,0,0,0, 7,7,7,7, 0,0,0,0, 7,7,7,7,7, 0,0,0,0,0,0,0,   // y=7 两段木平台（顶边 112，距地面 48px，可达）
        0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
        0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
        1,1,1,1,1,1,1,1,1,1, 0,0,0,0, 1,1,1,1,1,1,1,1,1,1,   // y=10 地表，中间是河道
        2,2,2,2,2,2,2,2,2,2, 9,9,9,9, 2,2,2,2,2,2,2,2,2,2,   // y=11 土层 / 水面
        3,3,3,3,3,3,3,3,3,3, 9,9,9,9, 3,3,3,3,3,3,3,3,3,3,   // y=12 深土 / 水体
        3,3,3,3,3,3,3,3,3,3, 9,9,9,9, 3,3,3,3,3,3,3,3,3,3    // y=13
      ]
    },
    {
      "name": "decor",         // 纯装饰，永不碰撞
      "data": [
        0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
        0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
        0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
        0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
        0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
        0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
        0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
        0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
        0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
        0,0, 12,0,0,0,0,0, 12,0,0,0,0,0,0,0, 12,0,0,0,0,0,0,0,  // y=9 草丛
        0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
        0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
        0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
        0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0
      ]
    }
  ],

  // 碰撞层：单层，值取 1.2 的表。长度同样是 width*height。
  "collision": [
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0, 2,2,2,2, 0,0,0,0, 2,2,2,2,2, 0,0,0,0,0,0,0,   // 木平台 = oneway，顶边 112
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    1,1,1,1,1,1,1,1,1,1, 0,0,0,0, 1,1,1,1,1,1,1,1,1,1,   // 地表 = solid
    1,1,1,1,1,1,1,1,1,1, 3,3,3,3, 1,1,1,1,1,1,1,1,1,1,   // 水面 = water
    1,1,1,1,1,1,1,1,1,1, 3,3,3,3, 1,1,1,1,1,1,1,1,1,1,
    1,1,1,1,1,1,1,1,1,1, 3,3,3,3, 1,1,1,1,1,1,1,1,1,1
  ],

  // 实体放置点。x/y 为逻辑像素，指实体的「脚底中心」锚点。
  "entities": [
    { "type": "player_spawn", "x": 32,  "y": 160 },

    // 敌人 spawner：摄像机右边缘越过 triggerX 时生成（design.md §6.7）
    { "type": "spawn", "x": 200, "y": 160, "enemy": "runner",  "triggerX": 120, "facing": -1 },
    { "type": "spawn", "x": 232, "y": 160, "enemy": "cover",   "triggerX": 140, "facing": -1 },
    { "type": "spawn", "x": 176, "y": 176, "enemy": "frogman", "triggerX": 100, "facing": -1,
      "params": { "surfaceY": 176 } },        // 每个敌人类型自己解释 params

    // 道具载体
    { "type": "capsule", "x": 96, "y": 64, "triggerX": 40,
      "params": { "weapon": "S", "speed": 60, "dir": 1 } },
    { "type": "pod",     "x": 288, "y": 160, "triggerX": 180,
      "params": { "weapon": "M" } },

    // 段落标记：用于 Boss 触发、镜头锁定、BGM 切换
    { "type": "checkpoint", "x": 0,   "y": 160, "params": { "id": "start" } },
    { "type": "lock_camera","x": 336, "y": 0,   "params": { "until": "boss_dead" } },
    { "type": "boss",       "x": 352, "y": 96,  "triggerX": 336,
      "params": { "id": "wall_boss", "bgm": "audio/boss.mp3" } }
  ],

  "audio": { "bgm": "audio/jungle.mp3" }
}
```

### 1.4 字段表

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `version` | int | ✓ | 当前 `1` |
| `id` | string | ✓ | 唯一，须等于文件名（不含扩展名） |
| `name` | string | ✓ | 显示名 |
| `tileSize` | int | ✓ | 必须为 16 |
| `width` / `height` | int | ✓ | 单位 tile；`height` 第 1 关固定 14 |
| `atlas` | path | ✓ | 图集 JSON 路径 |
| `scroll.minX` / `maxX` | int | ✓ | 摄像机左上角 x 的钳制范围（逻辑像素） |
| `scroll.autoScroll` | number | | px/s；默认 0 |
| `background.color` | palette name | ✓ | **语义名**，不是 hex |
| `background.layers[]` | object[] | | `atlasFrame` / `parallax`(0–1) / `y` / `repeatX` |
| `layers[]` | object[] | ✓ | 至少一层；`name` 唯一；`data.length === width*height` |
| `collision` | int[] | ✓ | 长度 `width*height`，值 0–5 |
| `entities[]` | object[] | ✓ | 必须恰好含 1 个 `player_spawn` |
| `audio.bgm` | path | | 缺省则沿用上一段 BGM |

### 1.5 `tools/validate-level.js` 必须检查的项

1. `tileSize === 16`、`height === 14`（第 1 关）。
2. 每个 `layers[].data` 与 `collision` 长度 `=== width * height`。
3. `collision` 每个值 ∈ 0–5；每个 tile 索引在图集 frame 数范围内。
4. 恰好 1 个 `player_spawn`；其位置下方 ≤ 48px 内存在 `solid` 或 `oneway`。
5. `scroll.maxX === width*16 - 256`（除非 `autoScroll > 0`）。
6. 所有 `entities[].triggerX ≤ entities[].x`——否则实体会在摄像机已经越过它之后才生成，表现为"敌人凭空出现在身后"。
7. 所有引用的 `atlasFrame` / `enemy` id / `weapon` key 在对应表里存在。
8. `background.color` 与任何调色板引用是 `docs/palette.json` 里的合法语义名。
9. **可通关性**：从 `player_spawn` 出发，用 `JUMP_VELOCITY`/`GRAVITY`/`RUN_SPEED` 推出的最大跳跃跨度（≈ 3.5 tile 高、约 4 tile 远），逐段检查是否存在到关卡右端的落脚点链。这一项是防止"关卡数据改坏导致卡死"的最后一道闸。

---

## 2. 图集 JSON

一个 PNG + 一个 JSON。由 `tools/pack-atlas.js` 从 `assets/src/` 的散图生成——**JSON 是生成物，不手改**；要改就改源图和打包脚本的配置。

### 2.1 设计要点

- **`anchor` 是本格式的核心**。所有 frame 的锚点定义为"脚底中心"（`anchor.x = w/2`、`anchor.y = h`），使得实体只需要一个"站立点"坐标就能正确绘制，且换用不同尺寸的帧（站姿 16×24 → 卧倒 24×12）时脚不会飘。空中特效等无脚概念的用几何中心，需显式写出。
- 动画帧时长以**帧数**计，不是毫秒。定步长模拟下毫秒是有害的抽象——它会引入浮点累积误差，破坏确定性回放。
- `tiles` 段与 `frames` 段分开：tile 是规则网格，用索引直接算 uv，不需要逐个记 xywh；frames 是不规则精灵。

### 2.2 完整示例

`assets/atlas/jungle.json`

> 示例为节选。真实图集还需包含 §3 / §4 引用到的其余帧与动画（`bullet_f`、`bullet_s`、`bullet_r`、`grenadier`、`frogman`、`boom_big`、`splash` 等）——§1.5 与 §2.4 的引用完整性校验会强制这一点。

```jsonc
{
  "version": 1,
  "image": "assets/atlas/jungle.png",
  "imageSize": { "w": 256, "h": 256 },
  "palette": "docs/palette.json",     // lint-palette.js 用它校验 PNG 像素

  // 规则 tile 网格：索引 1..N 按行优先映射到这个矩形区域内的格子。
  // 索引 0 恒为空，不占格子。
  "tiles": {
    "size": 16,
    "origin": { "x": 0, "y": 0 },     // 网格左上角在图集中的位置
    "cols": 16,
    "count": 48                        // 有效 tile 数；索引 1..48
  },

  // 不规则精灵帧
  "frames": {
    // 玩家：16×24，锚点脚底中心 (8, 24)
    "p1_stand":     { "x": 0,  "y": 64, "w": 16, "h": 24, "anchor": { "x": 8, "y": 24 } },
    "p1_run_0":     { "x": 16, "y": 64, "w": 16, "h": 24, "anchor": { "x": 8, "y": 24 } },
    "p1_run_1":     { "x": 32, "y": 64, "w": 16, "h": 24, "anchor": { "x": 8, "y": 24 } },
    "p1_run_2":     { "x": 48, "y": 64, "w": 16, "h": 24, "anchor": { "x": 8, "y": 24 } },
    "p1_jump":      { "x": 64, "y": 64, "w": 16, "h": 24, "anchor": { "x": 8, "y": 24 } },
    "p1_aim_up":    { "x": 80, "y": 64, "w": 16, "h": 24, "anchor": { "x": 8, "y": 24 } },
    "p1_aim_upfw":  { "x": 96, "y": 64, "w": 16, "h": 24, "anchor": { "x": 8, "y": 24 } },
    // 卧倒帧尺寸不同，但锚点仍是脚底中心 → 站立点坐标不变，脚不飘
    "p1_prone":     { "x": 0,  "y": 88, "w": 24, "h": 12, "anchor": { "x": 12, "y": 12 } },
    // 水中半身帧
    "p1_water":     { "x": 24, "y": 88, "w": 16, "h": 12, "anchor": { "x": 8, "y": 12 } },

    // 敌人
    "runner_0":     { "x": 0,  "y": 112, "w": 16, "h": 24, "anchor": { "x": 8, "y": 24 } },
    "runner_1":     { "x": 16, "y": 112, "w": 16, "h": 24, "anchor": { "x": 8, "y": 24 } },
    "cover_0":      { "x": 32, "y": 112, "w": 16, "h": 16, "anchor": { "x": 8, "y": 16 } },
    "turret_0":     { "x": 48, "y": 112, "w": 24, "h": 24, "anchor": { "x": 12, "y": 24 } },

    // 子弹与特效：无脚概念 → 锚点为几何中心，必须显式写出
    "bullet_n":     { "x": 0,  "y": 144, "w": 4,  "h": 4,  "anchor": { "x": 2, "y": 2 } },
    "bullet_laser": { "x": 8,  "y": 144, "w": 16, "h": 4,  "anchor": { "x": 8, "y": 2 } },
    "boom_0":       { "x": 0,  "y": 160, "w": 16, "h": 16, "anchor": { "x": 8, "y": 8 } },
    "boom_1":       { "x": 16, "y": 160, "w": 16, "h": 16, "anchor": { "x": 8, "y": 8 } },
    "boom_2":       { "x": 32, "y": 160, "w": 16, "h": 16, "anchor": { "x": 8, "y": 8 } },

    // 道具
    "capsule":      { "x": 0,  "y": 184, "w": 16, "h": 12, "anchor": { "x": 8, "y": 12 } },
    "item_S":       { "x": 16, "y": 184, "w": 8,  "h": 8,  "anchor": { "x": 4, "y": 8 } },

    // 背景（parallax 层引用）
    "bg_treeline":  { "x": 0,  "y": 200, "w": 128, "h": 48, "anchor": { "x": 0, "y": 0 } },
    "bg_bush":      { "x": 128,"y": 200, "w": 128, "h": 32, "anchor": { "x": 0, "y": 0 } }
  },

  // 动画：frames 引用 + 每帧持续帧数
  "anims": {
    "p1_run":   { "frames": ["p1_run_0", "p1_run_1", "p1_run_2", "p1_run_1"],
                  "durations": [4, 4, 4, 4], "loop": true },
    // durations 可以是单个数字，表示所有帧同时长
    "runner":   { "frames": ["runner_0", "runner_1"], "durations": 6, "loop": true },
    // 爆炸：不循环，末帧结束后实体自毁
    "boom":     { "frames": ["boom_0", "boom_1", "boom_2"],
                  "durations": [3, 4, 5], "loop": false },
    // 道具字母闪烁：durations 不等长演示节奏控制
    "item_S_blink": { "frames": ["item_S", "item_S"], "durations": [8, 4], "loop": true,
                      "_note": "第二帧绘制时整体替换为 flame_hi，由代码处理，不占额外 frame" }
  }
}
```

### 2.3 字段表

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `version` | int | ✓ | 当前 `1` |
| `image` | path | ✓ | PNG 路径 |
| `imageSize.w/h` | int | ✓ | 必须与 PNG 实际尺寸一致，加载后断言 |
| `palette` | path | ✓ | 供 lint 使用 |
| `tiles.size` | int | ✓ | 16 |
| `tiles.origin.x/y` | int | ✓ | tile 网格左上角 |
| `tiles.cols` | int | ✓ | 每行 tile 数；tile 索引 `i` 的位置 = `origin + ((i-1)%cols, floor((i-1)/cols)) * size` |
| `tiles.count` | int | ✓ | 有效 tile 数 |
| `frames.<name>` | object | ✓ | `x`/`y`/`w`/`h`/`anchor{x,y}`，全部必填，单位为图集像素 |
| `anims.<name>.frames` | string[] | ✓ | 引用 `frames` 的键；长度 ≥ 1 |
| `anims.<name>.durations` | int \| int[] | ✓ | **帧数**；数组时长度必须等于 `frames.length` |
| `anims.<name>.loop` | bool | ✓ | `false` 时末帧停住，播完触发 `onAnimEnd` |

### 2.4 校验项

1. `imageSize` 与 PNG 实际尺寸一致。
2. 每个 frame 的 `x+w ≤ imageSize.w`、`y+h ≤ imageSize.h`。
3. 每个 frame 的 `anchor` 在 `[0,w] × [0,h]` 内。
4. `anims[].frames` 的每个名字在 `frames` 里存在。
5. `durations` 为数组时长度匹配；每个值 ≥ 1（0 帧动画会死循环）。
6. PNG 全部像素落在 `docs/palette.json` 的 24 色内，且 alpha ∈ {0, 255}。
7. tile 网格区域与任何 frame 矩形**不重叠**（打包脚本的正确性自检）。

---

## 3. 武器表

位置：`game/data/weapons.json`。数值来源是 `design.md` §6.5，本表是其机器可读形式。

```jsonc
{
  "version": 1,
  "default": "N",
  "weapons": {
    "N": {
      "name": "Rifle",
      "mode": "single",          // single | auto | spread | laser | homing | barrier
      "cooldown": 8,             // 帧
      "maxOnScreen": 5,
      "damage": 1,
      "bullet": {
        "speed": 300,
        "frame": "bullet_n",
        "box": { "w": 4, "h": 4 },
        "pierce": false,         // 命中后是否存活
        "gravity": 0
      },
      "sfx": "audio/sfx/shot_n.mp3"
    },
    "M": {
      "name": "Machine Gun", "mode": "auto", "cooldown": 5, "maxOnScreen": 8, "damage": 1,
      "bullet": { "speed": 300, "frame": "bullet_n", "box": { "w": 4, "h": 4 },
                  "pierce": false, "gravity": 0 },
      "sfx": "audio/sfx/shot_m.mp3"
    },
    "F": {
      "name": "Fire", "mode": "single", "cooldown": 12, "maxOnScreen": 4, "damage": 1,
      "bullet": { "speed": 200, "frame": "bullet_f", "box": { "w": 8, "h": 8 },
                  "pierce": false, "gravity": 0,
                  // spiral 只对 mode/bullet 组合有意义的字段，缺省即不启用
                  "spiral": { "radius": 8, "period": 20 } },
      "sfx": "audio/sfx/shot_f.mp3"
    },
    "S": {
      "name": "Spread", "mode": "spread", "cooldown": 14, "maxOnScreen": 15, "damage": 1,
      "spread": { "count": 5, "stepDeg": 12 },   // ±24° = 中心 ±(5-1)/2*12
      "bullet": { "speed": 280, "frame": "bullet_s", "box": { "w": 4, "h": 4 },
                  "pierce": false, "gravity": 0 },
      "sfx": "audio/sfx/shot_s.mp3"
    },
    "L": {
      "name": "Laser", "mode": "laser", "cooldown": 20, "maxOnScreen": 2, "damage": 3,
      "bullet": { "speed": 480, "frame": "bullet_laser", "box": { "w": 16, "h": 4 },
                  "pierce": true, "gravity": 0 },
      "sfx": "audio/sfx/shot_l.mp3"
    },
    "R": {
      "name": "Homing", "mode": "homing", "cooldown": 10, "maxOnScreen": 6, "damage": 1,
      "bullet": { "speed": 240, "frame": "bullet_r", "box": { "w": 6, "h": 6 },
                  "pierce": false, "gravity": 0,
                  "homing": { "turnRateDeg": 180 } },   // 度/秒
      "sfx": "audio/sfx/shot_r.mp3"
    },
    "B": {
      "name": "Barrier", "mode": "barrier",
      "cooldown": 0, "maxOnScreen": 0, "damage": 0,
      "barrier": { "frames": 600 },       // 10s 无敌；不改变当前武器
      "sfx": "audio/sfx/barrier.mp3"
    }
  }
}
```

字段规则：

- `mode` 是**唯一**决定射击行为分支的字段。加载器遇到未知 `mode` 硬失败。
- `bullet.gravity` 单位 px/s²，`0` 表示直线飞行。投掷类（榴弹）用非 0。
- `spiral` / `homing` / `spread` / `barrier` 是模式相关的可选子对象；缺省即该行为不启用。**不允许**用 `null` 占位。
- `B` 没有 `bullet`——加载器必须容忍 `mode: "barrier"` 时 `bullet` 缺失，其他 mode 时 `bullet` 必填。

---

## 4. 敌人表

位置：`game/data/enemies.json`。数值来源是 `design.md` §6.7。

```jsonc
{
  "version": 1,
  "enemies": {
    "runner": {
      "name": "Runner",
      "hp": 1,
      "speed": 60,
      "score": 100,
      "box":     { "w": 12, "h": 22 },   // 碰撞盒
      "hurtbox": { "w": 12, "h": 22 },   // 受击盒；敌人不缩小，只有玩家缩（design.md §6.2）
      "anim": "runner",
      "gravity": true,                   // 是否受重力
      "touchDamage": true,               // 触碰玩家是否致死
      "ai": "chase",                     // chase | cover | lob | sweep | submerge | retreat
      "params": {},
      "death": { "fx": "boom", "sfx": "audio/sfx/die_grunt.mp3" }
    },
    "cover": {
      "name": "Cover Soldier", "hp": 1, "speed": 0, "score": 200,
      "box": { "w": 12, "h": 16 }, "hurtbox": { "w": 12, "h": 16 },
      "anim": "cover", "gravity": true, "touchDamage": true,
      "ai": "cover",
      "params": { "fireInterval": 90, "bulletSpeed": 180, "aimAtPlayer": true },
      "death": { "fx": "boom", "sfx": "audio/sfx/die_grunt.mp3" }
    },
    "grenadier": {
      "name": "Grenadier", "hp": 1, "speed": 30, "score": 300,
      "box": { "w": 12, "h": 22 }, "hurtbox": { "w": 12, "h": 22 },
      "anim": "grenadier", "gravity": true, "touchDamage": true,
      "ai": "lob",
      "params": { "fireInterval": 120, "lobVx": 120, "lobVy": -240,
                  "blastRadius": 20, "bulletGravity": 900 },
      "death": { "fx": "boom", "sfx": "audio/sfx/die_grunt.mp3" }
    },
    "turret": {
      "name": "Turret", "hp": 3, "speed": 0, "score": 500,
      "box": { "w": 20, "h": 20 }, "hurtbox": { "w": 20, "h": 20 },
      "anim": "turret", "gravity": false, "touchDamage": false,
      "ai": "sweep",
      "params": { "fireInterval": 45, "dirs": 8, "bulletSpeed": 150,
                  "hitFlashFrames": 4 },
      "death": { "fx": "boom_big", "sfx": "audio/sfx/die_metal.mp3" }
    },
    "frogman": {
      "name": "Frogman", "hp": 1, "speed": 40, "score": 200,
      "box": { "w": 12, "h": 12 }, "hurtbox": { "w": 12, "h": 12 },
      "anim": "frogman", "gravity": false, "touchDamage": true,
      "ai": "submerge",
      "params": { "cycle": 100, "surfaceFrames": 30, "bulletSpeed": 180,
                  "surfaceY": 0 },       // 0 = 由关卡 entities[].params 覆盖
      "death": { "fx": "splash", "sfx": "audio/sfx/die_water.mp3" }
    },
    "bridge_guard": {
      "name": "Bridge Guard", "hp": 1, "speed": 45, "score": 150,
      "box": { "w": 12, "h": 22 }, "hurtbox": { "w": 12, "h": 22 },
      "anim": "runner", "gravity": true, "touchDamage": true,
      "ai": "retreat",
      "params": { "fireInterval": 100, "bulletSpeed": 180, "fallsWithBridge": true },
      "death": { "fx": "boom", "sfx": "audio/sfx/die_grunt.mp3" }
    }
  }
}
```

字段规则：

- `ai` 与武器的 `mode` 同理：唯一的行为分支键，未知值硬失败。
- `params` 的键由 `ai` 决定。关卡 `entities[].params` 会**浅合并覆盖**本表的 `params`（如 `frogman.surfaceY`）。
- 所有以帧为单位的字段（`fireInterval`、`cycle`、`hitFlashFrames`）必须是整数。
- Boss 不在本表——其部件结构与阶段逻辑写在 `game/entities/boss.js` 与关卡的 `boss` 实体 `params` 里，因为它不是可复用模板。

---

## 5. 输入回放格式

确定性回归测试的载体（`design.md` §9.2）。位置：`tests/replay/spec/*.json` 与 `tests/replay/session/*.json`。

### 5.1 输入位掩码

一个 16 位整数，每帧一个值：

| bit | 值 | 含义 |
|---|---|---|
| 0 | 1 | 左 |
| 1 | 2 | 右 |
| 2 | 4 | 上 |
| 3 | 8 | 下 |
| 4 | 16 | 射击 |
| 5 | 32 | 跳跃 |
| 6 | 64 | 暂停 |
| 7–15 | — | 保留，必须为 0 |

左右同时按下 = 视为无水平输入（不是"后按优先"）。上下同时 = 视为无垂直输入。这两条是确定性所需的明确规则，不能留给实现自由发挥。

### 5.2 稀疏帧编码

`frames` 是 `[frameIndex, inputBitmask]` 对的数组，**只记录输入发生变化的帧**。`frameIndex` 严格递增，第一项的 `frameIndex` 必须是 0。第 `f` 帧的实际输入 = 最后一个 `frameIndex ≤ f` 的项的掩码。

稀疏编码的原因：一次 30 秒的通关是 1800 帧，但真实操作的变化点只有一两百个。稀疏后录像文件小到可以直接进 git 并在 PR 里被人读懂。

### 5.3 完整示例

`tests/replay/spec/jump-onto-platform.json` —— 一个机制断言用例：

```jsonc
{
  "version": 1,
  "id": "jump-onto-platform",
  "description": "从地面 (y=160) 起跑右移并起跳，落在 row 7 的 oneway 平台上（顶边 y=112，高差 48px）",
  "level": "assets/levels/test-platforms.json",
  "seed": 12345,                 // engine/rng.js 的播种值；必须显式给出
  "totalFrames": 120,

  // [frameIndex, inputBitmask]；只记变化点
  "frames": [
    [0,   0],        // 静止
    [10,  2],        // 按右 → 开始跑
    [26,  34],       // 右 + 跳 (2|32)
    [28,  2],        // 松跳，继续按右
    [80,  0],        // 全松
    [119, 0]         // 末帧显式给出，便于阅读；非必须
  ],

  // 末态校验和：回放到 totalFrames 后计算，必须逐位相等
  "checksum": {
    "algo": "fnv1a32",
    "value": "0x8f3a21c7",
    // 参与哈希的字段，顺序即哈希输入顺序。改这个列表就是改校验和，需同步重录。
    "fields": ["player.x", "player.y", "player.vx", "player.vy", "player.state",
               "lives", "score", "aliveEnemies", "aliveBullets", "rng.state"]
  },

  // 可选：中途断言，比末态校验和更好定位问题。
  // frame 取值应留出余量，不要卡在预测的落地帧上——落地帧由录制结果确定，
  // 手算的连续解与逐帧欧拉积分会有 1~2 帧偏差。
  "asserts": [
    { "frame": 70, "expr": "player.onGround", "equals": true },
    { "frame": 70, "expr": "player.state",    "equals": "STAND" },
    { "frame": 70, "expr": "player.y",        "equals": 112 }   // row 7 顶边 = 7*16
  ]
}
```

**这个用例的几何必须自洽，写关卡与用例时先算一遍**：地面在 row 10，顶边 `10*16 = 160`，玩家脚底 y=160。最大跳高 `JUMP_VELOCITY² / (2*GRAVITY) = 320² / 1800 ≈ 56.9px`。因此可达的落脚点高差必须 **< 56.9px**：

- row 7 平台顶边 `7*16 = 112`，高差 `160 - 112 = 48px` < 56.9 → **可达** ✓
- row 6 平台顶边 `6*16 = 96`，高差 `160 - 96 = 64px` > 56.9 → **从地面不可达**，只能由 row 7 平台二次起跳抵达（高差 16px）

`tools/validate-level.js` 的可通关性检查（§1.5 第 9 项）就是在自动化这个算术。

`tests/replay/session/full-jungle-clear.json` —— 完整通关录像，结构相同但 `frames` 长得多、只留末态校验和：

```jsonc
{
  "version": 1,
  "id": "full-jungle-clear",
  "description": "第 1 关从出生点通关到 Boss 击杀，不死亡",
  "level": "assets/levels/jungle.json",
  "seed": 20260828,
  "totalFrames": 5400,
  "frames": [[0,0],[12,2],[18,34],[20,2],[41,18],[/* …约 200 项… */],[5399,0]],
  "checksum": {
    "algo": "fnv1a32",
    "value": "0x1d4e90ab",
    "fields": ["player.x", "player.y", "lives", "score", "levelCleared"]
  }
}
```

### 5.4 字段表

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `version` | int | ✓ | 当前 `1` |
| `id` | string | ✓ | 唯一，等于文件名 |
| `description` | string | ✓ | 人读的意图。**必填**——一个只有数字的回放用例失败时无人知道它原本想验证什么 |
| `level` | path | ✓ | 关卡文件 |
| `seed` | int | ✓ | PRNG 播种值 |
| `totalFrames` | int | ✓ | 回放长度 |
| `frames` | [int,int][] | ✓ | 稀疏输入；`frameIndex` 严格递增且首项为 0；掩码高 9 位为 0 |
| `checksum.algo` | string | ✓ | 当前只支持 `fnv1a32` |
| `checksum.value` | hex string | ✓ | |
| `checksum.fields` | string[] | ✓ | 参与哈希的字段路径，**顺序有意义** |
| `asserts[]` | object[] | | `frame` / `expr` / `equals`；建议 spec 用例都写 |

### 5.5 校验和计算

```
h = 0x811c9dc5
for each field in checksum.fields (按数组顺序):
    v = 读取该路径的值
    s = 规范化(v)          // 见下
    for each byte b of UTF-8(s):
        h = ((h XOR b) * 0x01000193) mod 2^32
```

**规范化规则**（浮点是确定性的最大陷阱，必须写死）：

- 整数 → 十进制字符串。
- 浮点 → **先乘 65536 取整**（即 16.16 定点），再转十进制字符串。理由：`0.1 + 0.2` 在不同 JS 引擎版本上可以有不同的最后一位，直接哈希浮点会让校验和在换 Node 版本时莫名失效。16.16 定点在 px 单位下的精度是 1/65536 px，远超需要。
- 布尔 → `"1"` / `"0"`。
- 字符串（如 `player.state`）→ 原样。
- 数组长度类字段（`aliveEnemies`）→ 整数处理。

### 5.6 录制与更新流程

1. web 宿主开 `?record=1`，正常游玩；退出时下载 JSON（自动稀疏编码 + 计算校验和）。
2. 放进 `tests/replay/`，跑一次确认通过。
3. **当一次改动确实要改变游戏行为时**，重跑 `npm run replay:rebless` 重算校验和，并在 PR 描述里说明"哪些用例的校验和变了、为什么应该变"。
   无说明的校验和变更等同于未解释的手感回归，**PR 应被拒绝**。这条纪律是整个测试策略成立的关键——校验和一旦可以随手重刷，这层保护就归零了。
