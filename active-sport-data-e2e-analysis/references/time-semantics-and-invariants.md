# 时间语义与不变量

## 一、常见时间类型

| 时间 | 常见来源 | 正确语义 | 常见误用 |
|---|---|---|---|
| sensor/sample TS | 传感器或采样线程 | 原始样本产生时间 | 当成算法确认边界 |
| result TS | 算法输出 | 当前结果对应的数据时间 | 当成回调到达时间 |
| station/lap start TS | 算法或业务状态机 | 业务段开始边界 | 用记录节点时间替代 |
| station/lap end TS | 算法或业务状态机 | 业务段结束边界 | 用收到切站通知时间替代 |
| callback/dispatch TS | Engine/Service | 消息到达或派发时间 | 用作运动数据归属边界 |
| record `relative_time` | 记录保存模块 | 节点相对运动开始的提交/写入时刻，除非协议另有定义 | 直接命名为 `station_start_ts` |
| upload/parse TS | APP/云端 | 消费动作发生时间 | 用作原始业务时间 |

任何字段只有在代码、协议或文档明确证明后，才能承担另一种时间语义。

## 二、边界与归属

若业务约定 `E = station_end_ts`：

- `ts <= E` 属于旧段，`ts > E` 属于新段，或按协议明确的开闭区间执行。
- 延迟到达不改变数据原始归属。尾包应按自身原始 TS 或算法给出的边界归段。
- 自动识别延迟只影响“何时知道边界”，不应把确认时刻变成边界。
- 边界修正应重算上一段和当前段，不应减少整场累计量。

## 三、时间公式

常用对照：

```text
真实段时长       = E - S
记录构建滞后     = T_record - E
APP起点偏差      = T_app_start - S
APP终点偏差      = T_app_end - E
动作绝对开始时间 = S + rep_start_ms
动作绝对结束时间 = S + rep_end_ms
```

若 APP 采用 `T_record - lap_time` 作为起点：

```text
APP起点偏差 = (T_record - lap_time) - S
            = (T_record - E) - [lap_time - (E - S)]
```

这个分解可以区分“记录节点构建滞后”和“lap_time 自身口径差异”。

## 四、暂停与净时长

先确认每个量采用哪种时钟：

- wall time：暂停期间继续增加。
- active/net duration：暂停期间不增加。
- algorithm TS：取决于输入数据时间轴和算法约定。
- record relative time：可能按运动开始的 wall time，也可能按净时长。

验证暂停场景时记录 `pause_start`、`resume`、边界 S/E、LAP 时长和节点写入时间。不能仅凭“暂停了 N 秒”推断所有字段都应增加或扣除 N 秒。

### 总用时、运动用时与最后暂停

以下是已核对的Zepp RT引擎口径，分析其他分支、旧版本或特殊运动时仍需确认对应实现：

| 暂停情况 | 总用时 `duration_total` | 运动用时 `duration_net` |
|---|---|---|
| 暂停后恢复 | 包含这段暂停间隔 | 扣除这段暂停间隔 |
| 暂停后直接结束，未恢复 | 不包含最后暂停到结束保存的等待时间 | 同样不包含该等待时间 |

用相同单位验证：

```text
P_closed = 所有已恢复暂停间隔之和
P_open   = 最后仍未恢复的暂停间隔；没有则为0
total    = 计算时刻 - 运动起点 - P_open
net      = max(total - P_closed, 0)
```

对应源码追踪点：

- `sport_rt_engine_core.c` 的暂停/恢复处理：设置 `pause_timestamp`；恢复时累加 `pause_duration_total` 并清除当前暂停起点。
- `core/sport_result_calc.c::_duration_total_ms()`：计算总用时并扣除尚未恢复的暂停间隔。
- `core/sport_engine_result.c::sport_rt_result_update_duration()`：由总用时扣除累计暂停间隔得到净用时。
- `packages/services/sport/src/sport_record_save.c::_fillTimes()`、`_fillTimeMs()`：分别保存total/net以及两者之差；检查秒级截断和毫秒字段的具体精度。

例：10:00开始，10:10暂停、10:12恢复，10:20再暂停，10:25结束保存。总用时20分钟，运动用时18分钟；中途暂停2分钟计入total但不计入net，最后等待5分钟均不计入。

因此不能笼统写“每次暂停都从总用时扣除”或“暂停后一直冻结到保存”。恢复后会继续计算；需逐段区分已恢复暂停与最后未恢复暂停。

APP终点还需单独核对：如果采用起点加total，上例为10:20；如果采用起点加net，则为10:18。二者都不是10:25的保存时刻，也不能未经确认互相替代。实时算法是否收到暂停、是否继续累计，由其独立控制链决定。

## 五、异步与尾包

- `station_change` 是状态通知，不一定携带完整最终数据。
- UPDATE/AUTO_UPDATE 后可能继续收到旧站缓存和尾包。
- Service/UI 切站时刻、算法最终边界和记录节点写入时刻可能不同。
- 重复通知必须幂等；超时兜底不能抢在有效 final 前重复结算。
- 停止运动时还要区分站点 final、整场 final、保存放行三个状态。

## 六、证据强度

从强到弱：

1. 同一样本协议原值与两端解析值逐字段一致。
2. 原始日志数值可由源码公式复算。
3. 单端日志与代码路径吻合。
4. 截图或人工描述。

跨样本证据只能证明机制可能相同，不能宣称当前样本逐毫秒一致。
