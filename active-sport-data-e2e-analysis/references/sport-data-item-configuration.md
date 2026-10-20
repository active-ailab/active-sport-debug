# 运动数据项配置

## 配置来源与生效链路

Zepp 工程从 `framework/engine/sportEngine/common/` 追踪：

```text
sports.xlsx
  -> sport_gen.py 及其调用的生成模块
  -> sport_group_auto.* / sport_rt_attr_config/sport_rt_attr_<项目>.h
  -> sport_rt_attr_auto.c 按项目宏选择属性表
  -> sportRt_getAttr(运动分组)
  -> 运行时数据读取、页面配置和其他消费者
```

按当前分支确认每个步骤，不能只看 Excel 或 Markdown 导出表。生成文件与生成器可能存在不同步或手动维护的部分。

## 工作表用途

| 工作表 | 主要用途 |
|---|---|
| `Sports` | 固件/App 运动类型、项目支持和运动分组等映射 |
| `sport_group` | 运动分组名称与顺序 |
| `code_attr_product` | 参与属性生成的工作表清单 |
| `code_attr_*` | 属性 ID、名称、外设标记、项目支持列和运动分组支持列 |
| `sport sensor` | 运动传感器配置 |
| `distance fusion` | 距离数据源与融合配置 |

上述名称取自已检查工程；实际任务先盘点工作表，按目标分支调整。

## 核对步骤

1. 从实际项目宏确认 `sport_rt_attr_auto.c` 选中的项目属性头文件。
2. 在 Excel 中查找包含该项目的 `code_attr_*` 工作表，不能按工程目录名猜工作表名。
3. 从 `Sports` 和 `sport_group_auto.c` 确认目标运动类型对应的分组；多个运动可共用分组，同一运动的运行模式也可能影响上层行为。
4. 从 `sport_gen.py` 确认项目列和分组列如何共同决定属性，不自行假设 Yes/No/空值的解释。
5. 按属性 ID/英文标识对照生成头文件，检查 Excel、生成代码和 `sports_md/` 导出表是否一致。
6. 对争议属性继续追踪计算函数、传感器前提、UI 数据页和保存函数。

当前检查过的生成器在 `generate_sport_rt_attr_maps()` 中要求属性对应的项目单元格和运动分组单元格同时满足条件，再调用 `append_rt_attr()`；具体值如何转成布尔属性还要检查该函数。不要把非空字符串直接解释成业务“支持”。

## 只读检查 Excel

使用 `openpyxl`，不修改工作簿：

```python
from openpyxl import load_workbook

path = 'framework/engine/sportEngine/common/sports.xlsx'
book = load_workbook(path, read_only=True, data_only=True)
for sheet in book:
    print(sheet.title, sheet.max_row, sheet.max_column)
book.close()
```

通过 `product_start/product_end`、属性标识和运动列标题定位单元格，报告保留工作表名与坐标。不要硬编码行列号。`data_only=True` 返回公式缓存值；公式缓存缺失或可疑时，用 `data_only=False` 检查公式，并对照生成代码。

`sport_gen.py` 可能切换到源码目录并覆盖多个文件。只读分析不要直接执行生成器；确需验证生成结果时在隔离副本中执行，再审查差异。

## 四种状态分别确认

| 状态 | 判断依据 |
|---|---|
| 配置支持 | 项目属性表、运动分组映射和运行时属性查询 |
| 有效出值 | 算法、传感器/外设、运动阶段、有效样本等条件 |
| 页面展示 | 默认页、自定义页、阶段切页和 UI 过滤逻辑 |
| 记录保存 | summary/detail 保存代码、协议 tag、宏和上传字段 |

实时属性可选不等于默认显示，也不等于记录落盘。外设字段还需区分“可选但未连接”“连接但无有效数据”和“已有有效值”。不能根据跑步/力量等阶段名称推断数据自动清零或不输出。

## 跨项目与分段影响表

比较时以属性 ID/英文标识对齐，保留大分类；同一分组在多个项目的支持集合取并集后，再检查每项是否受业务变更影响。

| 分类 | 数据项/标识 | 支持项目 | 出值条件 | 计算范围/公式 | 分段处理 | 记录字段/条件 |
|---|---|---|---|---|---|---|
| 时间、距离、配速、心率等 | ... | ... | 外设/样本/模式 | 整场/瞬时/窗口/本段/上段 | 边界修正/无需修正/缺口 | 保存函数、tag 或未保存 |

对本段/上段项检查基线与边界修正；对整场、瞬时和滑动窗口项分别确认原有口径。记录保存清单从保存代码独立建立，不能把实时项表直接当成落盘清单。
