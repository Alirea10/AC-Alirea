# Garrison Blackboard 手册

本文只记录与 `AutoChessTriggerGarrisonAbility` 直接相关的 Blackboard 值。

## 1. 节点本身

```json
{
  "$type": "Torappu.Battle.Action.Nodes+AutoChessTriggerGarrisonAbility, Assembly-CSharp",
  "_target": "BUFF_OWNER"
}
```

节点没有自己的数值参数。它读取当前 buff 的 Blackboard，并把配置交给卫戍效果执行器。

## 2. Blackboard 字段

### `key`

类型：`string`

选择要执行的卫戍事件或效果模板。

```json
{ "key": "act1autochess_gar_event_useskill" }
```

含义：当前 Blackboard 用于“开启技能时”的卫戍效果。

### `garrison_key`

类型：`string`

标识具体的卫戍特质实例。

```json
{ "garrison_key": "garrison_95_a" }
```

主要用于区分具体特质，并可能参与每场战斗的触发次数或增加量限制。不是所有效果都有这个字段。

### `bond_type`

类型：`string`

决定盟约目标如何选择。

| 值 | 意义 | 示例 |
|---|---|---|
| `bond_self` | 目标干员自身所属的盟约 | 自身开技能时增加自身盟约 |
| `bond_by_id` | 使用 `bond_id` 指定的盟约 | 增加指定的阿戈尔盟约 |
| `bond_actived_maxstack` | 当前已激活且层数最高的盟约 | 增加当前最高盟约 |

### `bond_id`

类型：`string`

当 `bond_type` 为 `bond_by_id` 时，指定要增加的盟约。

```json
{
  "bond_type": "bond_by_id",
  "bond_id": "egirShip"
}
```

多个盟约可以用逗号分隔：

```json
{ "bond_id": "egirShip,arcaneShip" }
```

### `bond`

类型：`string`

旧式或通用盟约增加逻辑使用的盟约列表字段，作用类似 `bond_id`。

```json
{ "bond": "sargonShip,preciShip" }
```

在典型战斗 buff 中优先使用 `bond_id`；`bond` 主要属于另一套通用盟约效果配置。

### `bond_add_type`

类型：`string`

决定增加层数的计算方式。

| 值 | 意义 | 使用的数量 |
|---|---|---|
| `by_count` | 固定增加 | `bond_add_count` |
| `by_charlevel` | 按目标干员等阶增加 | 目标等阶或运行时写入的 `add_count` |
| `by_charcount_samerow` | 按目标所在行的干员数增加 | 行内人数 × `bond_add_count_multi` |

### `bond_add_count`

类型：`number`

固定增加的盟约层数，通常与 `bond_add_type = by_count` 搭配。

```json
{
  "bond_add_type": "by_count",
  "bond_add_count": 2
}
```

含义：每次触发增加 2 层。

### `bond_add_count_multi`

类型：`number`

按人数计算增加量时使用的倍率。

```json
{
  "bond_type": "bond_actived_maxstack",
  "bond_add_type": "by_charcount_samerow",
  "bond_add_count_multi": 1
}
```

含义：目标所在行每有 1 名干员，增加 1 层当前最高盟约。

### `add_count`

类型：`number`

运行时动态写入的增加数量。通常由前置节点生成，而不是静态写在 garrison 配置中。

```text
AutoChessAssignChessLevelToBlackboard
    → add_count = 目标干员等阶
AutoChessTriggerGarrisonAbility
    → 使用 add_count 增加盟约
```

### `max_add_count_per_battle`

类型：`number`

本场作战通过该卫戍特质最多增加的盟约层数。

```json
{
  "bond_add_count": 1,
  "max_add_count_per_battle": 7
}
```

含义：每次增加 1 层，但本场最多累计增加 7 层。

## 3. 完整示例：崇高牺牲

效果：阿戈尔干员被击倒时，提供等同于该干员等阶的阿戈尔盟约层数。

### Blackboard

```json
{
  "key": "act1autochess_band13_buff",
  "bond_type": "bond_by_id",
  "bond_id": "egirShip",
  "bond_add_type": "by_charlevel"
}
```

### 动作链

```text
ON_OWNER_KILLED
  → 检查 BUFF_OWNER 是否属于 egirShip
  → 将 BUFF_OWNER 的等阶写入 add_count
  → AutoChessTriggerGarrisonAbility(BUFF_OWNER)
```

### 等价逻辑

```csharp
if (owner.HasBond("egirShip"))
{
    int addCount = owner.ChessLevel;
    AddBondStack("egirShip", addCount);
}
```

这里的节点不是去查找死亡干员的另一个 `garrisonId`，而是执行当前 buff Blackboard 中配置的盟约效果。

## 4. 最小配置模板

### 固定增加指定盟约

```json
{
  "key": "act1autochess_gar_event_useskill",
  "bond_type": "bond_by_id",
  "bond_id": "sargonShip",
  "bond_add_type": "by_count",
  "bond_add_count": 1
}
```

### 增加自身盟约

```json
{
  "key": "act1autochess_gar_event_useskill",
  "bond_type": "bond_self",
  "bond_add_type": "by_count",
  "bond_add_count": 1
}
```

### 按目标等阶增加

```json
{
  "key": "act1autochess_band13_buff",
  "bond_type": "bond_by_id",
  "bond_id": "egirShip",
  "bond_add_type": "by_charlevel"
}
```

## 5. 字段组合规则

```text
bond_type = bond_self
  → 不需要 bond_id

bond_type = bond_by_id
  → 需要 bond_id

bond_add_type = by_count
  → 需要 bond_add_count

bond_add_type = by_charlevel
  → 通常需要前置节点写入 add_count

bond_add_type = by_charcount_samerow
  → 通常需要 bond_add_count_multi
```
