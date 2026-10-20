# 大小核配置与跨核链路

## 1. 先确认本次构建的核分工

对目标项目分别确认大核和小核的 defconfig、实际生成配置、源码版本与固件版本。defconfig 描述配置意图；存在配置覆盖或旧构建产物时，需继续检查实际配置、编译记录或 map 文件。

区分以下角色：

| 角色 | 要确认的问题 |
|---|---|
| 功能支持 | FEATURE/SUPPORT 宏是否让公共类型、适配器和接口可用 |
| 本地实现 | BUILD 宏是否选入 core、算法、管理器等实现 |
| 代理调用 | 当前核编译 local 还是 remote Agent |
| 业务消费 | Service、应用、UI 和保存模块在哪个核运行 |

不能由某个功能宏开启直接推断整个模块已编译；还要追踪模块依赖和源码选择。

## 2. 逐层验证编译入口

按以下顺序检查：

1. 项目大小核 defconfig 和实际生成配置。
2. `Config.in` 的依赖、默认值和 select。
3. `module.mk`、CMake 或其他构建清单的模块选择和 SOURCES 条件。
4. local/remote Agent 的源码选择及实现。
5. 需要进一步确认时，检查编译命令、目标文件或对应核的 map/ELF。

Zepp 工程可使用以下入口搜索，再按真实路径调整：

```bash
rg -n 'HMI_CONFIG_BUILD_ENGINE_SPORT_RT|HMI_CONFIG_BUILD_ENGINE_HYBRID_TRAINING|APP_SENSORHUB' configs/
rg -n 'SOURCES|COMPONENTS|local|remote|BUILD_ENGINE' framework/engine/sportEngine/module.mk
rg -n 'SOURCES|COMPONENTS|hyrox_sim' packages/services/sport/module.mk
rg --files framework/engine | rg 'agent.*(local|remote)|module\.mk$'
```

`HMI_CONFIG_BUILD_ENGINE_SPORT_RT`、`HMI_CONFIG_BUILD_ENGINE_HYBRID_TRAINING` 是不同模块的本地实现选择条件，必须分别核对。其他项目可能采用不同核分工。

## 3. 公共层和兼容层单独判断

- `common/`、`compat/`、设置接口或头文件可能在两个核同时参与编译。
- 同一源文件在两核编译时，内部宏分支可能不同。
- 静态全局变量通常是各核各自的实例；不能默认共享状态。
- remote API 的返回可能只代表命令已发送，需查实现确认是否等待实际完成。
- Service 是否编译，应从模块选入和依赖判断；小核支持某项运动不等于编译大核 Service。
- 不能根据目录含有 `sport_rt_engine` 或文件名含有 `service` 就断定执行核。

## 4. 为争议数据画出往返链路

至少追踪一条控制链和一条结果链，填写实际核归属：

```text
调用者 -> remote Agent -> 消息打包 -> 跨核传输
       -> local Agent -> 本地 Engine -> 实际处理

本地结果 -> 结果打包 -> 跨核传输 -> remote 结果处理
         -> listener/业务消费 -> UI 或记录保存
```

对每一跳记录函数、核、消息类型、有效载荷长度、执行线程或队列，以及完成条件。若数据经共享内存或其他通路传递，按实际实现补充。

## 5. 跨核参数核对

分别检查发送侧与接收侧：

- 命令 ID、magic/version、长度和结构定义是否匹配。
- 字段顺序、类型宽度、对齐、packing 和序列化规则。
- 宏是否改变结构布局或接收分支；若会改变，两端配置必须满足同一通信契约。
- 是否传递进程/核内指针，缓冲区是否在异步消费前失效。
- 新载荷被旧分支解释时，哪个字段会被读成时长、类型或索引。
- 错误或未知长度是否拒绝处理，还是意外回退到旧结构。

数值异常时，用相同字节按发送结构和接收结构分别解释，确认错误读法能否复现异常。不能仅凭异常数值推测解析错误。

## 6. 合并大小核日志的时间轴

- 先标注日志来自哪个核，区分日志到达时间和载荷内部时间。
- 确认两核 UTC、单调时钟和 tick 的单位及同步关系。
- 使用命令 ID、序号、站点索引、数据时间戳等关联事件；只有墙钟接近时，保持推断标记。
- 分别记录发送、接收、执行、结果返回、业务消费时刻，才能判断队列积压发生在哪一段。
- 缺少对端日志时，只能确认本端动作，不能把发送成功视为对端执行成功。

## 7. 核归属记录表

| 模块/函数 | 编译核 | 执行核 | 编译条件 | 代理/消息方向 | 证据 |
|---|---|---|---|---|---|
| 争议数据的生产、控制、回调和保存位置 | 待确认 | 待确认 | 宏与构建规则 | local/remote/无 | 配置、源码、map 或日志 |

只填写与本次问题相关的模块。配置不确定时保留待确认项，并指出最小补证材料。
