# AwesomeContra

一款横版跑射（run-and-gun）像素动作游戏，主形态为**微信小游戏**，同时保留普通微信小程序宿主与 H5 宿主。

- 逻辑分辨率 `256 × 224`，16×16 tile，整数倍缩放，24 色调色板
- 自研零依赖 Canvas 2D 引擎，固定 1/60 步长，确定性输入回放
- MVP 范围：单人、第 1 关「丛林」完整可通关 + 关底 Boss

**当前状态：Stage 1 地基。** 设计文档与技术规范已落库，尚无可运行代码。

## 致敬声明

本项目是一款受经典横版跑射游戏启发的**致敬作**，**不是** Konami 的官方作品，与 Konami 及其任何关联方无任何关联，未获其授权或认可。「魂斗罗」/「Contra」是其各自权利人的商标。

- 本仓库**不包含**任何原版素材：无原版精灵图、背景、音乐、音效、字体、关卡数据，无 ROM 提取物或从 ROM 反推的数据。
- 全部像素素材与音频由本项目自制，统一约束在 [`docs/palette.json`](docs/palette.json) 的 24 色内。
- 全部关卡编排为原创，不逐段还原原版关卡。
- 借鉴的仅是通用游戏设计语汇（横版卷轴、8 向瞄准、字母武器道具、一击死）。

引入来源不明的图片或音频，是任何 PR 被直接拒绝的理由。详见 [`docs/design.md`](docs/design.md) §1。

## 目录结构

```
/engine       平台无关内核（主循环、渲染、输入、音频、资源、对象池、PRNG）
/game         玩法层（entities / systems / data / states）
/platform     宿主适配：wx-minigame（发布）、web（开发与 CI）、wx-miniprogram（备用）
/assets       图集 PNG+JSON、tilemap JSON、音频
/tools        图集打包、调色板与宿主 API lint、关卡校验、素材生成
/tests        unit（纯函数）、replay（确定性回放）、e2e（Playwright）
/docs         设计文档、格式规范、ADR
```

`engine/` 与 `game/` **不允许出现任何 `wx.` 前缀标识符**或直接触碰 `document` / `window`——所有宿主能力经 platform adapter 注入，由 `tools/lint-no-host-api.js` 在 CI 中强制。

## 文档

先读设计文档，它是全项目的单一规范来源；实现某个数据文件时再翻格式规范。

| 文档 | 内容 |
|---|---|
| [`docs/design.md`](docs/design.md) | **主规范**：合规前提、三宿主架构、显示规范、玩法机制细则、性能预算、测试策略、路线图、常量表 |
| [`docs/formats.md`](docs/formats.md) | 数据格式规范，每种格式带可直接复制的完整示例：tilemap、图集、武器表、敌人表、输入回放 |
| [`docs/palette.json`](docs/palette.json) | 24 色调色板（NES 风格子集），带语义名与明暗 ramp |
| [`docs/adr/0001-engine-choice.md`](docs/adr/0001-engine-choice.md) | 为什么自研引擎而不用 Cocos / LayaAir / Egret |
| [`docs/adr/0002-minigame-vs-miniprogram.md`](docs/adr/0002-minigame-vs-miniprogram.md) | 为什么主形态是小游戏，以及如何保留普通小程序宿主 |

任何实现与 `docs/design.md` 冲突时，**先改文档再改代码**。文档里的常量（`docs/design.md` §11）是单一真源，不允许在别处重复字面量。

## 后续开发入口

工作以严格串行的 Step 链推进——仓库早期所有改动都落在同一批文件上，并行只会制造冲突。

| Step | 内容 | 路线图阶段 |
|---|---|---|
| 1 | 设计文档与技术规范落库 ← **当前** | Stage 1 地基 |
| 2 | 工程脚手架、H5 与微信小游戏双宿主、测试基建 | Stage 1 地基 |
| 3 | 引擎内核：定步长循环、渲染/输入/资源抽象、对象池、确定性回放 | Stage 1 地基 |
| 4 | 物理与碰撞：重力、AABB、单向平台、水域 | Stage 2 可玩核心 |
| 5 | 玩家控制器、8 向瞄准、触屏操控、摄像机卷轴 → vertical slice | Stage 2 可玩核心 |

Step 5 交付 vertical slice 后，以实际手感为依据再细拆战斗系统、关卡内容与打磨阶段。完整路线图见 `docs/design.md` §10。

构建与运行说明将在 Step 2 随脚手架一并加入本节。

## 贡献约定

- 素材只用 `docs/palette.json` 的 24 色，alpha 只允许 0 或 255。
- `game/` 与 `engine/` 的每帧执行路径**零堆分配**：禁止 `new`、对象/数组字面量、闭包、`map`/`filter`/`forEach`、字符串拼接。启动与关卡加载阶段不受此限。
- 数据文件校验失败必须硬失败，不允许静默使用默认值。
- 改动确实要改变游戏行为时，重算回放校验和并在 PR 描述里说明**哪些用例变了、为什么应该变**。无说明的校验和变更等同于未解释的手感回归，PR 应被拒绝。
