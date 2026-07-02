# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

这是一个 **AzerothCore WOTLK 3.3.5** 的服务端模组，实现了一套**扩展子职业系统 (Expanded Subclass System)**。玩家通过使用职业卷轴道具习得新的法术，从而为原有职业添加类似 D&D 子职业的玩法深度。

## 仓库结构

```
CSV/                  # DBC 客户端数据库导入文件
├── Spell.csv         # 自定义法术定义 (277条, ID 8675300+)
├── SkillLine.csv     # 子职业技能类别定义 (20条, ID 1000-1019)
├── SkillLineAbility.csv  # 法术-技能线关联 (320条)
├── SkillRaceClassInfo.csv # 技能种族/职业可用性 (20条, flags=1040)
└── SpellIcon.csv     # 法术图标引用 (20条, ID 8000-8019)

LUA/                  # Eluna Lua 引擎脚本 (20个, 每子职业一个)
└── *Scroll.lua       # 卷轴使用逻辑: 教授法术 + 消耗道具

SQL/                  # 数据库 SQL
└── ClassScrolls.sql  # item_template 表插入 (20条, entry 500500-500519)

Interface/Icons/      # BLP 格式的法术图标 (20个)
```

## 架构核心概念

### 法术ID命名空间

- 所有自定义法术 ID 从 **8675300** 开始，避免与暴雪默认 ID 冲突
- 每个子职业约12-16个自定义变体法术（每个技能 × 3-4种能量类型）
- 卷轴道具 ID 范围: **500500-500519**
- 技能线 ID 范围: **1000-1019**
- 图标 ID 范围: **8000-8019**

### 能量类型适配系统

| 能量类型 | 值 | 对应职业 |
|----------|----|---------|
| Mana | `0` | 德鲁伊(11), 猎人(3), 法师(8), 圣骑士(2), 牧师(5), 萨满(7), 术士(9) |
| Rage | `1` | 战士(1) |
| Energy | `3` | 潜行者(4) |
| Runic Power | `6` | 死亡骑士(6) |

### LUA 脚本模式

所有 20 个 LUA 脚本结构完全相同，差异仅在于数据：

1. 定义 `ITEM_ENTRY`（对应卷轴道具ID）
2. 定义 `SPELLS_BY_CLASS` 表 — 按职业ID(key)映射到法术列表
   - 战士(1) / 潜行者(4) / DK(6) 使用**自定义新法术ID** (8675300+)
   - Mana 职业(2,3,5,7,8,9,11) 通常使用**原始暴雪法术ID** 或自定义 Mana 版
3. `OnUse*Scroll(event, player, item, target)` — 检查职业 → 教学法术 → 消耗卷轴
4. `RegisterItemEvent(ITEM_ENTRY, 2, OnUse*Scroll)` — 注册道具使用事件

### CSV 数据关系

- `Spell.csv` 中每个自定义法术根据目标职业设置不同的 `PowerType`、`ManaCost` 和效果数值
- `SkillLine.csv` 中 `CategoryID=7`（次级技能类别），`SpellIconID` 指向 `SpellIcon.csv`
- `SkillLineAbility.csv` 中 `RaceMask=0, ClassMask=0`（不限制种族/职业，由Lua脚本运行时处理）
- `SkillRaceClassInfo.csv` 中 `Flags=1040`（技能在法术书中始终可见）

## 如何添加新子职业

1. **CSV/Spell.csv** — 为新子职业添加4个技能 × 3-4个能量变体的法术行，分配新 ID（递增使用）
2. **CSV/SkillLine.csv** — 添加一个新技能线行（递增 ID: 1020, 1021, ...），分配新图标ID
3. **CSV/SkillLineAbility.csv** — 为每个法术变体添加关联行（SkillLine=新技能线ID, Spell=对应的法术ID）
4. **CSV/SkillRaceClassInfo.csv** — 添加技能可用性行（SkillID=新技能线ID, Flags=1040）
5. **CSV/SpellIcon.csv** — 添加图标引用行
6. **LUA/*Scroll.lua** — 创建新卷轴脚本，映射职业ID到对应法术
7. **SQL/ClassScrolls.sql** — 添加 `item_template` INSERT 行（分配新 entry ID）
8. **Interface/Icons/*.blp** — 添加 BLP 格式图标

## 法术数据行关键字段

编辑 `Spell.csv` 新法术时需关注：
- `PowerType`: 0/Mana, 1/Rage, 3/Energy, 6/RunicPower
- `ManaCost`: 对应能量类型的消耗值
- `Attributes`: `589824`(直接伤害) / `268435472`(召唤)
- `Effect_1/2/3`: `2`(直接伤害), `6`(光环), `56`(召唤)
- `SpellIconID`: 指向 `SpellIcon.csv` 的图标ID
- `SchoolMask`: `16`(冰霜), `1`(物理), `4`(火焰), `32`(暗影) 等
- `DurationIndex`: `35`(8秒), `21`(15秒), `1`(瞬发/永久) 等
- 多语言字段 (`Name_Lang_*`, `Description_Lang_*`) 至少填充 `enUS`

## 部署到 AzerothCore

1. 将 CSV 文件内容导入对应的 DBC 表
2. 将 LUA 脚本放置于 `lua_scripts/` 目录（Eluna 引擎）
3. 在 `worldserver` 数据库中执行 `ClassScrolls.sql`
4. 将 BLP 图标放置于客户端的 `Interface/Icons/` 目录
5. 打补丁到客户端 MPQ 或使用自定义补丁

## 注意事项

- 法术 ID 起始值 `8675300` 是硬性约定，不可与暴雪ID冲突
- 虽然 CSV 中 `RaceMask=0, ClassMask=0` 写了全种族/职业可用，但实际的职业限制由 LUA 脚本在运行时强制执行
- 不同职业卷轴的提示消息文案略有不同（如 Aquamancer: "You have learned the secrets of the Aquamancer!" vs Necromancer: "You have mastered the forbidden arts of the Necromancer!"）
- 卷轴消耗后不可退还，若不满足职业要求或已学法术后不会消耗道具
