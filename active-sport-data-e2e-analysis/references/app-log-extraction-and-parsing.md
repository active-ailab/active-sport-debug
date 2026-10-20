# APP 日志提取与解析

## 目标

从 Jira 附件或质量平台日志中取得可检索的 APP 原始日志，锁定一笔运动，并分别证明以下五个观察点：

1. 记录协议被 APP 解析成什么值。
2. 数据以什么模型和 trackId 落地。
3. 业务算法实际收到什么输入。
4. 业务算法输出是否成功。
5. 详情页或上传链路最终消费什么值。

这五点可能属于不同代码路径。某一条链路通过，不能自动证明其他链路通过。

## 一、创建隔离工作目录

附件下载命令可能把文件写入当前目录，必须先进入当前问题专用目录：

```bash
ISSUE=ITS-00000
WORK=/tmp/sport-data-e2e/$ISSUE
mkdir -p "$WORK/raw" "$WORK/unpacked" "$WORK/decoded"
cd "$WORK/raw"
```

不要在源码仓库根目录直接下载或解密。分析期间不要使用 `--clean`，避免删除原始 Logan 文件。

## 二、获取日志

### Jira 已有附件

```bash
zepp jira attachment list "$ISSUE"
zepp jira attachment download "$ISSUE"
```

下载后记录附件名、大小和校验值：

```bash
find "$WORK/raw" -type f -print0 | sort -z | xargs -0 sha256sum > "$WORK/SHA256SUMS"
```

### Jira 指向质量平台反馈

可直接下载并解密：

```bash
zepp feedback download "$ISSUE" \
  -o "$WORK/raw" --delog --jsonl --format json \
  > "$WORK/feedback-download.json"
```

如果 Jira 没有关联反馈链接，优先使用 Jira 已上传附件；不要假设 `feedback download` 一定能取得数据。

## 三、展开压缩包

每个压缩包展开到独立子目录，避免同名文件互相覆盖。根据环境使用 `unzip`、`7z` 或 `tar`；嵌套压缩包继续展开，但保留原包。

先盘点内容：

```bash
find "$WORK/raw" "$WORK/unpacked" -type f \
  -printf '%s\t%TY-%Tm-%Td %TH:%TM:%TS\t%p\n' | sort -nr \
  > "$WORK/file-inventory.tsv"
```

不要把图片、数据库、运动记录二进制和 APP 日志混为一类；分别登记用途。

## 四、解密 Logan 日志

对单个文件或包含 Logan 文件的目录执行：

```bash
zepp app delog "$WORK/unpacked" \
  -o "$WORK/decoded" --jsonl --format json \
  > "$WORK/delog-result.json"
```

若附件未压缩，也可直接对 `$WORK/raw` 执行。结果包括：

- `*-decode.txt`：完整文本，适合 `rg`、查看多行上下文和超长数组。
- `*-decode.jsonl`：每行一个结构化事件，通常包含 `ts/level/thread/tag/msg`，适合 `jq` 精确过滤。
- `delog-result.json`：解密成功、跳过和失败清单。

检查 `errors`，并确认目标测试时间段确实存在。`skipped` 可能是普通文本、图片或压缩包，不等同于失败。

缺少依赖时按 CLI 提示安装：

```bash
python -m pip install HMLogan==3.5.3 \
  --index-url https://internal-artifactory.zepp.top/simple/
python -m pip install pycryptodome
```

## 五、锁定唯一运动样本

证据优先级：

1. `trackId`。
2. session/运动开始 epoch 时间。
3. 运动类型和设备标识。
4. 用户描述的本地时间窗口。

先查身份锚点：

```bash
TRACK_ID=1790000000
rg -n -i "track.?id[=: ]*$TRACK_ID|$TRACK_ID" "$WORK/decoded"
```

再查运动插入、解析和页面加载事件：

```bash
rg -n -i \
  "SportDataObserver|INSERT|sportType|parse|detail|AlgoRunner|sports_batch" \
  "$WORK/decoded"
```

如果同一日志中存在多笔运动，不得只按“21:42”之类墙钟时间过滤。先用 trackId 建立 session，再用时间窗口收窄。

## 六、JSONL 精确过滤

先查看一条实际记录确认字段名：

```bash
find "$WORK/decoded" -name '*-decode.jsonl' -print -quit \
  | xargs -r head -n 3
```

标准 `zepp app delog --jsonl` 通常提供 `ts/level/thread/tag/msg`。按消息字段过滤：

```bash
jq -c --arg id "$TRACK_ID" '
  select(((.msg // "") | contains($id)) or
         ((.msg // "") | test("sports_batch|station_info|hyroxRepLap|station_start_ts|station_end_ts"; "i")))
' "$WORK"/decoded/*-decode.jsonl > "$WORK/app-evidence.jsonl"
```

若实际 JSONL 字段不同，先按首行结构调整表达式，不要套用后得到空结果还宣称日志无记录。

## 七、五个观察点的检索词

检索词只是入口，最终字段顺序和语义必须回到 APP 源码确认。

| 观察点 | 常见检索锚点 | 需要提取的内容 |
|---|---|---|
| 协议解析 | `hyroxRepLap`、record tag、parser 类名 | 原始字段、解析后字段、协议版本/长度 |
| 持久化/模型 | `SportDataObserver`、`INSERT`、`trackId` | trackId、sportType、数据库/模型对象 |
| 业务算法输入 | `sports_batch`、`station_info`、runner/data source | 完整输入数组、字段位置、调用次数 |
| 算法输出 | algo/insight/result 相关 tag | 成功/失败、结果窗口、错误码 |
| 详情页/上传 | detail、lap、station、upload/serialize | 最终开始/结束/时长、字段来源 |

HYROX/REP 问题可补充：

```bash
rg -n -i \
  "hyroxRepLap|station_start_ts|station_end_ts|rep_start|rep_end|sports_batch|station_info" \
  "$WORK/decoded"
```

查看文本上下文时保留行号：

```bash
rg -n -C 8 "sports_batch|station_info" "$WORK/decoded"/*-decode.txt
```

## 八、结构化解析规则

对每条关键日志生成证据记录：

| 字段 | 要求 |
|---|---|
| source | 文件名和原始行号 |
| log_ts | APP 日志时间及其时区 |
| track_id | 当前运动唯一标识 |
| stage | parser/model/algo_input/algo_output/detail/upload |
| entity | session/lap/station/rep |
| raw_value | 日志原值，不先改单位 |
| semantic | 源码确认后的业务含义 |
| consumer | 使用该值的类/函数/页面 |
| confidence | 已证实/推断/待验证 |

解析数组时必须先从 APP 生产代码确认下标。例如 `station_info` 的第六项是什么，不能仅凭旧报告沿用。记录完整数组和解析后的字段表，防止新增字段导致下标整体移动。

同一源 Logan 同时产生 TXT 和 JSONL 时，两者是同一证据的两种视图，不构成独立交叉验证。

## 九、时间归一化

- 保留原始 epoch ms，不先四舍五入。
- 明确日志文件名中的 `UTC+8.0`、日志 `ts` 和协议 epoch 是否同一时区表示。
- 人类可读时间只作为辅助列，计算始终使用原始毫秒值。
- APP 日志到达时间不等于协议字段所代表的业务时间。

需要转换时可用：

```bash
python3 - <<'PY'
from datetime import datetime, timezone, timedelta
ms = 1790000000000
tz = timezone(timedelta(hours=8))
print(datetime.fromtimestamp(ms / 1000, tz).isoformat(timespec='milliseconds'))
PY
```

## 十、APP 源码反向确认

从日志 tag、类名或字段名反向搜索 APP 源码：

```bash
rg -n "sports_batch|station_info|hyroxRepLap|stationStartTs|stationEndTs" /path/to/app/source
```

至少确认：

1. parser 如何处理新旧协议长度。
2. 模型字段是否改名或换单位。
3. 业务算法输入使用哪个字段及数组下标。
4. 详情页是否复用相同字段，还是仍走 LAP 时间线。
5. 上传序列化是否再次转换时间。

## 十一、APP 侧完成判据

只有满足以下条件，才能写“APP 消费链路通过”：

- 目标 trackId 在日志中唯一锁定。
- 协议解析值和上游记录值一致。
- 业务算法实际输入值已找到，而非只看到 parser 输出。
- 算法输出成功或失败原因已确认。
- 需要验收的 UI/上传链路已单独核对。

如果仅证明 `station_start_ts` 进入算法输入，应写“算法输入起点通过”；不能扩大为“APP 全链路通过”。
