# AutoChess Blackboard 手册

本文只整理 `buff_template_data.json` 中自走棋相关模板使用的 Blackboard 节点。

## 1. Blackboard 的几种作用域

| 作用域 | 典型节点 | 含义 |
|---|---|---|
| 当前 Buff BB | `AssignValueToBB`、`ModifyBlackboard` | 当前 buff 自己携带的临时变量。 |
| Ability BB | `AddAbilityBlackboard`、`ModifyAbilityBlackboard` | 技能/能力实例的变量。 |
| Character Shared BB | `AddCharacterSharedBlackboard`、`FilterByCharacterSharedBlackboard` | 干员跨 buff 共享的变量。 |
| Global BB | `AddGlobalBlackboard`、`FilterByGlobalBlackboard` | 玩家或战斗全局变量；自走棋通道为 `AUTOCHESS`。 |
| Buff BB（其他 Buff） | `AssignBuffBlackboardFromOthers`、`AddBuffBlackboard` | 读取或修改指定 buff 的 Blackboard。 |

普通 `blackboard` 字段中的：

```json
{ "key": "atk", "value": 0.2, "valueStr": null }
```

表示数值 BB；`valueStr` 非空时表示字符串 BB。

## 2. 赋值类节点

### `AssignValueToBB`

直接给当前 Buff BB 赋值。

```json
{
  "$type": "...+AssignValueToBB, Assembly-CSharp",
  "_blackboardKey": "add_count",
  "_value": 0,
  "_copyFromKey": "consume_count",
  "_assignString": false
}
```

逻辑：

```text
_copyFromKey 有值 → BB[add_count] = BB[consume_count]
否则             → BB[add_count] = _value
```

`_assignString = true` 时用于字符串值。

### `EnsureBlackboardDefaultValue`

仅在指定 BB 不存在时补默认值。

```json
{
  "$type": "...+EnsureBlackboardDefaultValue, Assembly-CSharp",
  "_defaultSettings": [
    { "key": "sp", "val": 0, "valStr": null, "overrideIfExists": false }
  ]
}
```

`overrideIfExists = true` 时，即使已有值也会覆盖。

### `AssignBuffBlackboardFromOthers`

读取另一个 buff 的 BB，写入当前 buff BB。

```json
{
  "_targetType": "BUFF_OWNER",
  "_blackboardKey": "ammo_accu",
  "_valueKey": "consumed_cnt",
  "_buffKey": "act1autochess_gar_event_consume_ammo",
  "_sourceType": "TARGET"
}
```

逻辑：

```text
当前 BB[ammo_accu] = 指定 buff BB[consumed_cnt]
```

## 3. 加减乘除与数值计算

### `BlackboardAdd`

对当前 Buff BB 做加法。

```json
{
  "$type": "...+BlackboardAdd, Assembly-CSharp",
  "_blackboardKey": "triggered_cnt",
  "_addition": 1,
  "_additionKey": null,
  "_isFloat": false
}
```

逻辑：

```text
BB[triggered_cnt] += 1
```

如果 `_additionKey` 有值，则使用另一个 BB 的值作为加数：

```text
BB[target] += BB[_additionKey]
```

### `AddBuffBlackboard`

修改目标 buff 的 BB，支持加法、减法和上限。

```json
{
  "$type": "...+AddBuffBlackboard, Assembly-CSharp",
  "_targetType": "BUFF_OWNER",
  "_blackboardKey": "consumed_cnt",
  "_buffKey": "act1autochess_gar_event_consume_ammo",
  "_addition": 1,
  "_additionKey": null,
  "_maxValueKey": null,
  "_isMinus": false
}
```

逻辑：

```text
目标 buff BB[consumed_cnt] += 1
```

关键字段：

| 字段 | 作用 |
|---|---|
| `_addition` | 固定加数。 |
| `_additionKey` | 从当前 BB 读取加数。 |
| `_isMinus` | 改为减法。 |
| `_maxValueKey` | 用指定 BB 作为上限。 |
| `_useCurBuffBBWhenDoAddition` | 使用当前 buff 的 BB 参与计算。 |
| `_checkBuffSource` | 限制目标 buff 的来源。 |

### `ModifyBlackboard`

把一个或多个 BB 的值复制/重算到另一个 BB。

```json
{
  "$type": "...+ModifyBlackboard, Assembly-CSharp",
  "_blackboardKeys": "atk_scale",
  "_fromBlackboardKeys": "atk_scale_1",
  "_value": 0,
  "_addBasedOriginValue": false,
  "_checkFromBlackboardValue": false
}
```

基本逻辑：

```text
BB[atk_scale] = BB[atk_scale_1]
```

多个 key 使用逗号分隔，并按位置对应。

### `CalculateBlackboardValueViaParams`

自走棋模板中最主要的通用计算节点，支持加、减、乘、除、取余、最小值、最大值和取整。

```json
{
  "$type": "...+CalculateBlackboardValueViaParams, Assembly-CSharp",
  "_inputKey": "bond_stack_cnt",
  "_outputKey": "real_attack_speed",
  "_multiplyParamKey": "attack_speed_per_stack",
  "_dividedParamKey": null,
  "_useRemainder": false,
  "_addParamKey": "base_attack_speed",
  "_minusParamKey": null,
  "_minValueKey": null,
  "_maxValueKey": null,
  "_finalAbs": true,
  "_finalCeil": false,
  "_finalFloor": false,
  "_finalRound": false
}
```

通常可还原为：

```text
result = BB[inputKey]
result *= BB[multiplyParamKey]   // 若有
result /= BB[dividedParamKey]    // 若有
result += BB[addParamKey]        // 若有
result -= BB[minusParamKey]      // 若有
result = abs(result)             // finalAbs
result = ceil/floor/round(result) // 对应开关
result = min/max(result, BB[key]) // 对应上下限
BB[outputKey] = result
```

字段意义：

| 字段 | 运算 |
|---|---|
| `_inputKey` | 输入 BB。 |
| `_outputKey` | 输出 BB。可与输入相同，表示原地修改。 |
| `_multiplyParamKey` | 乘以指定 BB。 |
| `_dividedParamKey` | 除以指定 BB。 |
| `_useRemainder` | 使用除法余数。 |
| `_addParamKey` | 加上指定 BB。 |
| `_minusParamKey` | 减去指定 BB。 |
| `_minValueKey` | 结果下限。 |
| `_maxValueKey` | 结果上限。 |
| `_finalAbs` | 取绝对值。 |
| `_finalCeil` | 向上取整。 |
| `_finalFloor` | 向下取整。 |
| `_finalRound` | 四舍五入。 |

例如：

```json
{
  "_inputKey": "bond_stack_cnt",
  "_outputKey": "real_attack_speed",
  "_multiplyParamKey": "attack_speed_per_stack",
  "_addParamKey": "base_attack_speed"
}
```

等价于：

```text
real_attack_speed = bond_stack_cnt × attack_speed_per_stack
                    + base_attack_speed
```

## 4. AutoChess 专用赋值节点

这些节点从自走棋状态读取数据，并写入指定 BB。

| 节点 | 主要字段 | 写入内容 |
|---|---|---|
| `AutoChessAssignChessLevelToBlackboard` | `_targetType`、`_blackboardKey` | 目标干员等阶。 |
| `AutoChessAssignBondStackCntToBB` | `_bondBlackboardKey`、`_bondId`、`_keyToStoreCnt` | 指定盟约层数。 |
| `AutoChessAssignBondCharCntToBB` | `_bondId`、`_keyToStoreCnt` | 指定盟约干员人数。 |
| `AutoChessAssignSameBondCharacterCntToBB` | `_blackboardKey` | 与目标同盟约的干员人数。 |
| `AutoChessAssignActiveGarrisonBondCntToBlackboard` | `_blackboardKey` | 当前激活卫戍盟约数量/层数信息。 |
| `AutoChessAssignChessCntToBB` | `_assignKey` | 场上或相关范围内的干员数量。 |
| `AutochessAssignEquipCntToBlackboard` | `_blackboardKey` | 装备数量。 |
| `AutoChessAssignBattleLayersToBlackboard` | `_blackboardKeys` | 战斗层数信息。 |
| `AutoChessAssignUseMagicCntInThisRound` | `_blackboardKeys` | 本回合使用魔法/道具次数。 |
| `AutoChessAssignGainedCharacterCountLastTime` | `_blackboardKey` | 上次获得的干员数量。 |
| `AutoChessAssignTargetPlayerSideToBlackboard` | `_blackboardKey` | 目标所属阵营字符串。 |

### 典型例子：按干员等阶传给 Garrison

```json
{
  "$type": "...+AutoChessAssignChessLevelToBlackboard, Assembly-CSharp",
  "_targetType": "BUFF_OWNER",
  "_blackboardKey": "add_count"
}
```

结果：

```text
BB[add_count] = BUFF_OWNER.chessLevel
```

## 5. AutoChess 专用筛选/判断节点

这些节点本身通常不修改 BB，而是读取 BB 或根据自走棋数据让后续节点继续/停止。

### `FilterByBlackboardValue`

比较数值 BB。

```json
{
  "$type": "...+FilterByBlackboardValue, Assembly-CSharp",
  "_targetType": "BUFF_OWNER",
  "_blackboardKey": "respawn_cnt",
  "_valueToCompare": 0,
  "_anotherKeyToCompare": "max_respawn_cnt",
  "_condType": "LT"
}
```

含义：

```text
BB[respawn_cnt] < BB[max_respawn_cnt]
```

`_anotherKeyToCompare` 为空时，与 `_valueToCompare` 比较。

常见 `_condType`：

```text
EQUALS   等于
GT       大于
GE       大于等于
LT       小于
LE       小于等于
NE       不等于（若版本支持）
```

### `FilterByBlackboardStrIsValue`

比较字符串 BB。

```json
{
  "_valueKey": "target_side",
  "_valueKeyToCompare": "source_side",
  "_checkIsEqual": true,
  "_useOrdinalIgnoreCase": true
}
```

含义：比较 `BB[target_side]` 与 `BB[source_side]` 是否相等。

### `IsBlackboardZero`

判断数值 BB 是否为 0。

```json
{
  "$type": "...+IsBlackboardZero, Assembly-CSharp",
  "_var": "triggered",
  "_noVarShowWarning": true
}
```

含义：

```text
BB[triggered] == 0
```

常用于一次性触发锁：

```text
IsBlackboardZero(triggered)
→ 执行效果
→ BlackboardAdd(triggered, 1)
```

### `IsBlackboardEqualWithString`

将字符串 BB 与常量或另一个 BB 比较。

```json
{
  "_var": "player_side",
  "_compareValue": "SIDE_A",
  "_compareBBKey": null,
  "_useBuffBlackboard": false
}
```

### `FilterByCharacterSharedBlackboard`

读取目标干员的共享 BB 并比较。

```json
{
  "_target": "BUFF_OWNER",
  "_blackboardKey": "reborn_mask",
  "_valueToCompare": 0,
  "_condType": "EQUALS"
}
```

### `FilterByGlobalBlackboard`

读取自走棋全局 BB 并比较。

```json
{
  "_blackboardKey": "band28_buff_respawn_cnt",
  "_channel": "AUTOCHESS",
  "_valueToCompare": 0,
  "_condType": "GT"
}
```

## 6. 共享 BB 与全局 BB 的修改

### `AddCharacterSharedBlackboard`

修改干员共享 BB。

```json
{
  "_target": "BUFF_OWNER",
  "_isStringBB": false,
  "_blackboardKey": "reborn_mask",
  "_value": 1,
  "_isOverwrite": true
}
```

`_isOverwrite = true`：直接覆盖；否则通常为累加/写入共享值。

### `AddGlobalBlackboard`

修改自走棋全局 BB。

```json
{
  "_blackboardKey": "band28_buff_respawn_cnt",
  "_channel": "AUTOCHESS",
  "_value": 1,
  "_valueBlackboardKey": "",
  "_overwrite": false
}
```

逻辑：

```text
GlobalBB[AUTOCHESS][band28_buff_respawn_cnt] += 1
```

### `AssignGlobalBlackboardToBlackboard`

把全局 BB 复制到当前 Buff BB。

```json
{
  "_globalblackboardKey": "band28_buff_respawn_cnt",
  "_blackboardKey": "respawn_cnt",
  "_channel": "AUTOCHESS",
  "_assignString": false
}
```

## 7. 逻辑组合

### `IfElse`

不是 BB 节点，但负责组合 BB 判断：

```json
{
  "$type": "...+IfElse, Assembly-CSharp",
  "_conditionNode": { "...判断节点..." },
  "_succeedNodes": [ "条件成立时执行" ],
  "_failNodes": [ "条件不成立时执行" ]
}
```

### `IfNot`

反转前一个判断节点的结果。

### 常见组合

```text
FilterByBlackboardValue
→ IfElse
   ├─ succeedNodes：执行效果
   └─ failNodes：跳过或结束 buff
```

```text
IsBlackboardZero(lock)
→ Dice(prob)
→ AutoChessTriggerGarrisonAbility
→ AddBuffBlackboard(lock, 1)
```

## 8. 与 Garrison 节点的连接

`AutoChessTriggerGarrisonAbility` 自身只负责把当前目标和 Blackboard 交给卫戍执行器；前置节点负责准备 Blackboard。

```text
AutoChessAssignChessLevelToBlackboard
  → BB[add_count] = 干员等阶

AutoChessTriggerGarrisonAbility
  → 按 BB[key]、BB[bond_type]、BB[bond_add_type]
    执行卫戍效果
```

典型的“崇高牺牲”就是：

```text
FilterCharacterBondIds(egirShip)
→ AssignChessLevelToBlackboard(add_count)
→ TriggerGarrisonAbility
```
