    # Act29Side 音频切换机制分析

> 原始数据来源:`D:\Dev\ArknightsGameData-master\zh_CN\gamedata`
> 相关活动:act29side(活动地图位于 `levels/activities/act29side/`)

---

## 1. 机制概述

Act29Side 是一套**地图级音乐状态机**:地图按时间轴在三种音频状态间自动切换——

- **激昂 (enthusiastic)**:激昂 BGM
- **低沉 (depressed)**:低沉 BGM
- **静默 (none)**:静默(无音乐 / 环境音)

时间轴**完全由数据驱动**。状态切换本身不需要任何代码改动,只需要一张图上挂一个 `env_system_new` rune。单位侧的阶段效果(BGM 切换、buff)则由宿主 buff 读取同一个 env 系统 key 实现。

---

## 2. 分层架构

```
┌─ 第 3 层:时间轴状态机(地图级,纯数据)────────────────────┐
│  env_system_new rune                                    │
│  └─ blackboard.key = "env_008_act29side_audio_type_switcher"│
│     └─ blackboard 中的 {音频类型: 持续秒数} 有序序列    │
└──────────────────────────────────────────────────────────┘
                           │ 引擎注册的 env 系统驱动
                           ▼
┌─ 第 2 层:宿主 buff(挂载在单位/陷阱上)──────────────────┐
│  enemy_act29side[buff_creator]      轮询当前类型→上lite buff│
│  enemy_act29side[buff_creator_bb]   生成 激昂fx/低沉fx     │
│  enemy_act29side[buff_enthu_lite]   激昂态自检            │
│  enemy_act29side[buff_depressed_lite] 低沉态自检          │
│  enemy_act29side[active]            激活态 buff           │
│  enemy_ltniak[switch_audiotype]     boss 登场切换          │
│  enemy_ltniak[end_audio_when_dead]  boss 死亡静默          │
└──────────────────────────────────────────────────────────┘
                           │ Act29SideCheckCurrentAudioType
                           ▼
┌─ 第 1 层:引擎实现 ──────────────────────────────────────┐
│  音频资源(BGM 文件)+ UI 插件(ui_plugin_name)            │
└──────────────────────────────────────────────────────────┘
```

**核心认知:时间轴状态机是纯数据驱动的。** 只要二进制里注册了 `env_008_act29side_audio_type_switcher`(它随原版游戏一并分发,已存在),在任意地图写这个 rune 时间轴就会跑。但单位效果、BGM 文件、UI 插件不包含在 rune 里,需要另行挂载。

---

## 3. 地图入口:`env_system_new` rune

标准结构(取自 `level_act29side_02.json`):

```json
"runes": [{
  "difficultyMask": "ALL",
  "key": "env_system_new",
  "professionMask": 1023,
  "buildableMask": "ALL",
  "blackboard": [
    { "key": "key", "value": 0.0, "valueStr": "env_008_act29side_audio_type_switcher" },
    { "key": "duration_portal_active", "value": 30.0 },
    { "key": "none", "value": 75.0 },
    { "key": "enthusiastic", "value": 100.0 },
    { "key": "duration_audio_buff", "value": 100.0 },
    { "key": "ui_plugin_name", "value": 0.0, "valueStr": "act29side_battle_ui_plugin" }
  ]
}],
"globalBuffs": null
```

### rune 字段说明

| 字段 | 说明 |
|---|---|
| `difficultyMask` | 生效难度,`ALL` = 全部 |
| `key` | 外层通用 rune 类型,固定 `env_system_new` |
| `professionMask` | 职业掩码,1023 = 全职业 |
| `buildableMask` | 部署位掩码,`ALL` = 全部 |
| `blackboard` | 具体配置(见下) |

### blackboard key 速查

| key | 类型 | 作用 |
|---|---|---|
| `key` | valueStr | **必填**。选择 env 系统:`env_008_act29side_audio_type_switcher` |
| `none` | float(秒) | 静默段时长 |
| `depressed` | float(秒) | 低沉段时长(可重复出现 = 多段低沉) |
| `enthusiastic` | float(秒) | 激昂段时长(可重复出现 = 多段激昂) |
| `duration_audio_buff` | float(秒) | 音频状态 buff 的刷新周期,单位侧靠它感知当前阶段 |
| `duration_portal_active` | float(秒) | 传送门激活时长,`trap_portlexi[sp_ctrl]` 用 `durationKey` 读它 |
| `duration_boss_audio_buff` | float(秒) | boss 阶段音频 buff 时长 |
| `extra_load_bgm` | valueStr | boss 战 BGM 轨道 key(需对应音频资产) |
| `ui_plugin_name` | valueStr | 加载的 UI 插件(如 `act29side_battle_ui_plugin`) |

---

## 4. 时间轴规律(纯数据)

**blackboard 数组里 `none` / `depressed` / `enthusiastic` 的出现顺序 = 时间轴顺序;值 = 该阶段持续秒数。** 想循环,就把状态按序重复排。

各图时间轴对比(原始数据):

| 地图 | 时间轴 |
|---|---|
| sub-1-3 | none:10 → depressed:60 → enthusiastic:60 → depressed:60 → enthusiastic:60;+duration_boss_audio_buff:40 +extra_load_bgm "battle.ON_CUSTOM_TRIGGER.bat_witchking_p2_..." |
| ex01 | none:26 → enthusiastic:999 |
| ex02 | none:75 → enthusiastic:100 |
| ex04 | none:30 → enthusiastic:300 |
| ex05 | none:65 → depressed:180 |
| ex09 | none:30 → depressed:40 → depressed:240(两段低沉连续) |

`globalBuffs` 在所有 act29side 关卡中均为 null(不依赖关卡全局 buff)。

---

## 5. 宿主 buff 链(单位侧感知与生效)

### 5.1 轮询创建器 `enemy_act29side[buff_creator]`

`buff_template_data.json` L112980。ON_BUFF_TRIGGER 每 0.2s:
- `Act29SideCheckCurrentAudioType`(`_evnSysKey: "env_008_act29side_audio_type_switcher"`)查当前类型
- 按类型创建 `enemy_act29side[buff_enthu_lite]` / `buff_depressed_lite`

### 5.2 黑板创建器 `enemy_act29side[buff_creator_bb]`

L113202。ON_BUFF_TRIGGER 每 0.2s:
- 查当前类型是否为 `None`
- 若否,再查 `IsBlackboardEqualWithString`(`_var: "type"`, `_compareValue: "depressed"` / "enthusiastic")
- 对应创建 `enemy_act29side[depressed_fx]` / `[enthu_fx]`(fx 特效 buff,`templateKey: "empty"`)
- 用 `CheckContainsBuff` 避免重复上 fx

### 5.3 阶段自检 lite buff

- `enemy_act29side[buff_enthu_lite]`(L112850):每 0.2s 自检,当前类型不再是激昂 → FinishBuff
- `enemy_act29side[buff_depressed_lite]`(L112915):同上,低沉

### 5.4 激活态

`enemy_act29side[active]` 作为 buffKey 在 L113506、L113748 被创建(由上述创建器发出),标记单位处于音频机制激活态。

---

## 6. Boss 相关 buff(敌 ltniak)

| buff | 触发 | 行为 |
|---|---|---|
| `enemy_ltniak[switch_audiotype]` (L108592) | ON_BUFF_START | `Act29SideSwitchCurretnAudioType`,`_switchToOposite: true`,`_isTriggeredByBoss: true`(boss 登场 → 切到相反类型) |
| `enemy_ltniak[end_audio_when_dead]` (L107562) | ON_OWNER_FINISH | Switch 节点,`_switchToOposite: false`,`_muteAudio: true`(boss 死亡 → 静默) |
| Switch 节点 (L108890) | 初始化 | `_isFirstTime: true`(记录初始类型) |

---

## 7. 陷阱相关

| 陷阱 | 说明 |
|---|---|
| `trap_condtr` (L117217) | Switch 节点 `_switchToOposite: true` + `LogExtraBattleInfo "audio_type_changed"` |
| `trap_portlexi[sp_ctrl]` (L116060) | `durationKey: "duration_portal_active"`(L116133),从 env 系统黑板读取传送门激活时长 |

---

## 8. 两个核心行为节点

| 节点 | 作用 |
|---|---|
| `Act29SideCheckCurrentAudioType` | 读 `_evnSysKey` 指向的 env 系统,返回当前音频类型(用作 IfElse 条件) |
| `Act29SideSwitchCurretnAudioType` | 切换当前类型;`_switchToOposite` 取反、`_muteAudio` 静默、`_isTriggeredByBoss` boss 触发的强化切换、`_isFirstTime` 初始化 |

注意节点名里 `Curretn` 是原版拼写错误(应为 Current),照抄即可。

---

## 9. 复刻到另一张地图需要什么

| 机制部分 | 由谁提供 | 复刻方式 |
|---|---|---|
| 时间轴状态机 | `env_system_new` rune + blackboard | ✅ 只写 rune,时间轴自动跑 |
| 阶段效果 buff | `enemy_act29side[buff_creator]` 等宿主 buff | 把模板挂到单位/陷阱上(模板已在全局数据,无需重写) |
| BGM 实际切换 | 音频资源 + `extra_load_bgm` / boss 段 | 需要对应音频文件,光填数据不会出声 |
| UI 提示条 | `ui_plugin_name: act29side_battle_ui_plugin` | 需要对应 UI 插件 |

**结论:时间轴是纯数据的,`env_system_new` rune 即可复刻。完整机制还需挂宿主 buff + 音频资产 + UI 插件。**

---

## 10. 与 ArkChess aceffect 实现对比

本工程 `effectBuffInfoDataDict/*.json` 与原版 rune 是**同一套模式**:

```json
// 本工程 aceffect_enemy_1.json
[{
  "blackboard": [
    { "key": "key", "value": 0, "valueStr": "enemy_move_speed_mul" },
    { "key": "move_speed", "value": 0.15 },
    { "key": "enemy_exclude", "value": 0, "valueStr": "enemy_9012_acloon" }
  ],
  "countType": "NONE",
  "key": "env_gbuff_new"
}]
```

| 原版 | 本工程 |
|---|---|
| rune `key: env_system_new` | 数组项 `key: env_gbuff_new` / `env_gbuff_new_with_verify` / `auto_chess_change_map` |
| blackboard 内层 `key: valueStr` 选 env 系统 | blackboard 首项 `key: "key"` + `valueStr` 选子系统 |

**要在 ArkChess 复刻:** 照 `aceffect_band_33.json` 的方式加一个数组项,`key` 写 `env_system_new`,blackboard 首项 `valueStr` 写 `env_008_act29side_audio_type_switcher`,后续按 §3 填时间轴即可。

---

## 11. 待办 / 可继续深挖

- [ ] 捋清 `enemy_act29side[active]` 两处创建方(113506 / 113748)的完整挂载链
- [ ] 确认本工程引擎是否已注册 `env_008_act29side_audio_type_switcher`(随原版分发,大概率已注册)
- [ ] 若要在 ArkChess 加音乐机制,确认 `extra_load_bgm` 对应的音频资源能否接入
