# 表具能效计费链路设计

## 1. 文档定位

本文设计需求 1、2、4 的内部实现、数据模型、任务模型、消息结构和协议兼容边界：

- 需求 1：能源配置、标准点、模型能效配置和有效表具解析。
- 需求 2：表具半小时用量核算、质量状态、人工修正、时日月归档、范围重算和页面查询。
- 需求 4：价格方案、表具价格绑定、成本事实和成本投影。

唯一需求基线是 `AI节能/index.html`。`xtwin-insight` 和 `xtwin-expr` 仅用于了解现有入口、调用方和历史行为，不作为新内部实现的约束。

本设计不迁移旧数据、不双写、不保留旧表语义兼容层。需求 1、2、4 范围内的既有 RPC、HTTP 路径和 protobuf 契约通过边界适配器继续兼容；兼容请求和新接口请求最终进入同一套新业务服务、任务链路和事实投影。

本设计不定义上线容量、延迟或性能验收阈值。待项目指标明确后，另行补充相应的容量规划和验收标准。

## 2. 设计原则

1. **事实优先**：半小时用量事实、自动核算版本、人工版本、成本事实、价格快照和审计链路是审计与追溯依据。
2. **投影可重建**：有效当前投影、用量汇总和成本汇总都可以从权威事实重建。
3. **快照执行**：任务提交时冻结配置、表具集合、测点映射和相关价格上下文，任务执行及重试不重新读取配置。
4. **意图有序**：按提交时预留的 `intent_revision` 决定覆盖顺序，不按任务完成时间决定业务结果。
5. **单一业务链路**：旧协议适配器只做协议转换，不能包含第二套核算逻辑。
6. **身份规范化**：领域内部优先使用结构化 `InstanceRef`，边界统一使用规范 `meter_id` URI。
7. **长期可追溯**：权威事实及其审计链路按当前部署数据库的业务生命周期保留，默认不设置删除型 TTL。



## 3. 总体架构

```text
旧 RPC / HTTP / protobuf                 新接口
          |                                |
          +-------------> 边界兼容适配层 <-+
                              |
                              v
                    新领域业务服务
                              |
          +-------------------+-------------------+
          |                                       |
          v                                       v
XTwin 模型、继承、实例、遥测点              配置对象
                                           EnergyType
                                         StandardPoint
                                      ModelEnergyConfig
                                        active / pending
          |                                       |
          +-------------------+-------------------+
                              v
              EffectiveEnergyResolver
                EffectiveMeterService
                              |
             +----------------+----------------+
             |                                 |
             v                                 v
        半小时槽任务                       人工范围重算任务
             |                                 |
             +----------------+----------------+
                              v
                       TaskSnapshot
          配置 + 有效表具集合 + 测点映射 + 槽范围
                              |
                              v
                  EffectiveMeterManifest
                              |
                              v
                      原始累计量事实
                              |
                              v
              上一槽尾样本 + 当前槽尾样本
                              |
                              v
       当前槽用量 = 当前槽尾值 - 上一槽尾值
                              |
                              v
                           核算表                                      人工 SET/CLEAR
                    自动核算版本 追加保留                              人工版本 追加保留
                              |                                             |
                              |                                             |
                              +----------------------+----------------------+
                                                     |
                                                     v
                                           EffectiveFactProjector
                                                     |
                                                     |
                                                     |
                                                     v
                             有效表：usage_points.samples
                                 有效用量当前投影
                                 唯一用量读取入口
                                      |
                         +------------+-------------+
                         |                          |
                         v                          v
                    UsageRollup                UsageFactChange
                    可重建投影                       |
                                                     v
需求 4：                                               |
PricePlan                                              |
    |                                                  |
    v                                                  |
PricePlanRevision                                      |
    |                                                  |
    v                                                  |
PriceSnapshot                                          |
    |                                                  |
    +--------------------+                             |
                         v                             |
          MeterPriceBindingState + Revision ------------+
                         |
                         v
                   CostingWorkflow
        有效用量 + 绑定版本 + 价格快照
                         |
                         v
                      成本事实表
                    CostFact 版本
                         |
                         v
                 有效成本当前投影
                    cost_samples
                         |
                         v
                      CostRollup
                      可重建投影
                         |
                         v
     UsageFactReader / CostFactReader / MeterDetailReader
                         |
                         v
          表具明细 / 需求 5、6 / 页面消费链路
```

所有模块初期位于 `xtwin-insight` 同一进程内，以明确的应用服务和存储接口隔离边界。后续可以独立部署，但不能改变事实、版本和快照契约。

### 3.1 领域模块

- `energyconfig`：以能源类型为聚合根，原子维护基础字段、标准点、模型能力和实际测点映射，负责 active/pending generation、校验和提升。
- `meterresolver`：按半小时边界读取 XTwin 模型、继承链、设备/组件实例和遥测点，通过版本化遥测序列解析器冻结物理取数键，生成不可变有效表具 manifest revision。
- `accounting`：半小时调度、范围重算、尾样本选择、用量差值和自动版本生成。
- `usagefact`：人工 `SET/CLEAR`、自动/人工覆盖合并、有效当前投影和变更事件。
- `usagerollup`：从有效用量事实重建当前汇总。
- `pricing`：价格方案修订和表具价格绑定修订。
- `costing`：匹配有效用量、绑定和价格快照，生成成本事实及当前投影。
- `workflow`：操作、快照、分片、租约、优先级、重试、outbox 和 reconciliation。



### 3.2 依赖方向

```text
XTwin + energyconfig
        -> meterresolver
        -> accounting
        -> usagefact
        -> usagerollup

pricing -> costing <- usagefact
```

`xtwin-expr` 不执行需求 1、2、4 的新核算逻辑。历史表达式实现只作为兼容语法和行为的参考。

## 4. 身份和跨域契约



### 4.1 部署隔离和能源类型身份

当前部署的数据库实例是能源配置、用量事实和成本事实的隔离边界。领域模型、关系库业务键、ClickHouse 业务键、任务快照和事件 envelope 均不引入 `project_id`。固定的 `project_root` 仅来自部署配置并用于访问 XTwin，不作为领域身份，不从请求上下文解析，也不持久化到需求 1、2、4 的业务键中。

需求 1 的配置聚合键是 `energy_type_id`。创建能源类型时由服务生成 UUID，创建后永久不变；旧协议的 `EnergyType.name` 映射到该 UUID。用户可见的能源类型名称独立保存，并在当前数据库实例内唯一。

需求 2、4 共用数据库实例级部署配置 `energy_billing.business_timezone`，默认值为 `Asia/Shanghai`，允许部署覆盖为其他合法 IANA 时区。服务启动时必须通过 `time.LoadLocation` 校验；浏览器时区、请求时区和进程 `time.Local` 都不能成为业务计算依据。半小时边界、日期范围、时/日/月周期和计费时段先在该时区解释，再转换成 UTC 存储。

`billing_runtime_identity` 记录首次生成事实时使用的业务时区和事实模型版本。已有事实后，部署配置中的时区与该记录不一致时，核算写服务拒绝启动并告警，不能静默改变历史周期键。业务时区变更属于未来独立数据迁移，不属于需求 2 页面能力。任务快照、用量修订、汇总投影和价格快照都保存业务时区，供审计与历史重算使用。

### 4.2 结构化身份

```go
type InstanceRef struct {
    TwinID        string
    ComponentName string
}
```

设备实例的 `ComponentName` 为空。组件实例的名称来自所属 Twin 的 `components[].name`，只在该 Twin 内唯一。

### 4.3 规范 meter_id

设备：

```text
xtwin://twins/{percent-encoded-twin-id}
```

组件：

```text
xtwin://twins/{percent-encoded-twin-id}/components/{percent-encoded-component-name}
```

编码和解析规则：

- 使用 RFC 3986 percent-encoding。
- 解析时使用 URI parser，对原始路径段 percent-decode 一次。
- 规范化后重新编码，使用大写十六进制；非保留字符可保持字面形式。
- 要求有效 UTF-8；不做大小写折叠或 Unicode 归一化。
- 不允许 query、fragment、userinfo 或 port。
- 业务键按规范化后的字节值比较。

存储规范 `meter_id`，同时保存派生的 `twin_id` 和 `component_name` 以便索引与展示。派生字段不可独立写入。

组件改名视为旧身份消失和新身份产生，不迁移历史事实。能源类型不是身份的一部分。

领域和事实统一使用 `instance_id` 表示该 URI；旧接口字段名为 `name` 或 `by_meters` 时，由兼容适配器转换为同一个规范身份。

### 4.4 遥测来源和物理序列键

领域身份、模型测点身份和时序存储键必须分离。`meter_id` 标识业务表具，模型测点名称标识配置引用，`metrics.samples.__name__` 标识现有时序数据源中的物理序列；三者不能相互替代，也不能由核算 worker 临时猜测。

`meterresolver` 内部定义纯解析接口，在构建 manifest 时把结构化遥测引用解析为物理序列键：

```go
type ResolveTelemetrySeriesInput struct {
    TwinID             string
    TwinName           string
    ComponentName      string
    ModelTelemetryName string
}

type TelemetrySourceRef struct {
    SourceKind            string
    SeriesKey             string
    SeriesKeyCodec        string
    TwinID                string
    TwinName              string
    ComponentName         string
    ModelTelemetryName    string
    ResolvedTelemetryName string
    Fingerprint           string
}

type TelemetrySeriesResolver interface {
    Resolve(input ResolveTelemetrySeriesInput) (TelemetrySourceRef, error)
}
```

解析器沿用 4.2 的 `InstanceRef` 判别规则：`ComponentName` 为空表示设备测点，非空表示组件测点；调用方必须直接传入本次 XTwin 一致性读取中枚举到的 `TwinID`、`TwinName` 和 `ComponentName`，不能从 `meter_id` 反解或自行补全。

首版固定 `SourceKind = METRICS_SAMPLES`、`SeriesKeyCodec = XTWIN_TELEMETRY_V1`。`TwinID` 是领域身份，`TwinName` 是 XTwin 返回并由现有采集链路用于物理序列前缀的名称快照；不能用规范 `meter_id` 或 `TwinID` 替代 `TwinName` 拼接物理序列键。

`XTWIN_TELEMETRY_V1` 编码规则为：

- 设备测点：`ResolvedTelemetryName = ModelTelemetryName`，`SeriesKey = TwinName + "_" + ModelTelemetryName`。例如 `0_103 + 1_123_0 -> 0_103_1_123_0`。
- 组件测点：`ModelTelemetryName` 必须是以 `_0` 结尾的标准三段式测点名；把最后一段 `0` 替换为实例 `ComponentName`，再计算 `SeriesKey = TwinName + "_" + ResolvedTelemetryName`。例如 `TwinName=0_1`、`ComponentName=1#1`、`ModelTelemetryName=1_1_0` 时，结果为 `ResolvedTelemetryName=1_1_1#1`、`SeriesKey=0_1_1_1_1#1`。

解析器不访问 XTwin、ClickHouse 或实时数据，不检查当前是否已有样本，只执行确定性格式校验和编码。没有样本不影响表具资格，由需求 2 产生 `MISSING`；空 `TwinName`、无效模型测点名、组件缺失名称或不能按当前 codec 编码时，候选 manifest 构建失败并返回稳定 reason：`EMPTY_TWIN_NAME`、`INVALID_MODEL_TELEMETRY_NAME`、`MISSING_COMPONENT_NAME` 或 `INVALID_COMPONENT_TELEMETRY`。

解析结果连同结构化引用一起冻结在 manifest 映射中。`Fingerprint` 固定为以下字符串数组经 UTF-8 JSON 紧凑编码后的 SHA-256 小写十六进制值，字段顺序不得改变：`[SourceKind, SeriesKeyCodec, TwinID, TwinName, ComponentName, ModelTelemetryName, ResolvedTelemetryName, SeriesKey]`。未来增加新 codec 只影响新 manifest；历史任务和范围重算继续使用快照中已经冻结的 `SeriesKey` 和 codec，不能用新规则重新解析历史来源。

## 5. 需求 1：能源配置与有效表具

### 5.1 聚合边界

`EnergyConfiguration` 是需求 1 唯一写聚合，聚合键为 `energy_type_id`。创建能源类型时由服务生成 UUID，创建后永久不变。当前数据库实例是配置隔离边界，不增加其他隔离键。

聚合内包含：

| 实体 | 职责 |
| --- | --- |
| `EnergyConfigurationState` | active、pending、promoting generation 指针，active manifest 指针、etag、内置标记和技术状态 |
| `EnergyConfigurationGeneration` | 能源类型基础字段、内容指纹、生效槽和审计信息 |
| `StandardPoint` | 能源类型内部的标准计量角色及其单位 |
| `ModelEnergyConfig` | 物模型的能源计量/供电能力 |
| `StandardPointMapping` | 物模型标准角色到实际遥测点的映射 |
| `EffectiveMeterManifest` | 某个槽使用的有效表具、映射和单位换算快照 |

配置写入以整个聚合为单位，不提供让标准点、模型能力或测点映射独立生效的写入口。active、promoting 和 pending 是短期运行状态，不作为长期业务历史；任务快照保存执行所需的完整配置，审计日志保存每次创建、替换、提升、失败、重试和删除。

### 5.2 能源类型和标准点

能源类型 generation 至少包含：

- `energy_type_id`：服务生成的稳定 UUID。
- 用户可见名称：1–99 字符，在当前数据库实例内唯一，创建后不可修改。
- 度量类型：一次能源或工质。
- 半小时增量上限：必填且大于 0，默认值为 1500，展示单位引用唯一累计量标准点单位。
- 标煤和 CO2 折算系数：可选十进制定点数，最多 6 位小数；只有一次能源参与相应折算。
- generation、内容指纹、request ID、创建人和创建时间。

`StandardPoint` 至少包含稳定的 `standard_point_id`、分类、角色、标准名称、`unit_id` 和排序。`standard_point_id` 在标准点首次持久化时由服务生成 UUID，生成后永久不变；客户端只能原样回传查询结果中已有的正式 ID，不能为新增标准点指定正式 ID。

创建或替换完整聚合时，同一次请求可能同时包含新增标准点及引用它的实际测点映射。边界契约必须使用仅在本次请求草稿内有效的关联方式，例如为新增标准点提供唯一 `client_key` 并让映射引用该 key，或者采用不需要跨列表引用的等价嵌套请求结构。服务在事务内为新增标准点分配正式 `standard_point_id`，解析全部临时引用后再校验并持久化。`client_key` 或等价临时引用不是领域身份，不写入配置 generation、映射、manifest、任务快照、事实或审计；响应只返回正式 `standard_point_id`。

标准点单位是唯一真源，能源类型不重复保存累计量单位和实时值单位；列表接口分别从唯一累计量和主瞬时量派生这两列。

标准点满足：

- 每个能源类型恰好一个唯一累计量。
- 瞬时量至少一个，且恰好一个主瞬时量，其余为过程瞬时量。
- 标准名称在能源类型内唯一，所有标准点必须选择单位。
- 唯一累计量不可删除；主瞬时量不可直接删除，但可以把其他瞬时量切换为主。
- 过程瞬时量不进入表具身份、差值、数据质量校验、计费或对象级自动指标。

### 5.3 模型能力和实际测点映射

`ModelEnergyConfig` 按 `energy_type_id + generation + model_id` 唯一，能力可以多选“能源计量”和“供电”，但至少选择一项。能力只声明实例是否可作为表具或供电源候选，不创建、删除或推断任何能耗计量关系或 `supplyPower`。

选择能源计量时：

- 唯一累计量和主瞬时量映射必填，过程瞬时量映射选填。
- 实际测点必须属于所选模型或由其父模型继承。
- 同一实际测点在同一模型配置内不能映射多个标准角色。
- 实际单位必须与标准点单位同量纲；运行值统一换算为标准点单位。
- 父模型配置与其子模型配置互斥，父模型配置对继承了对应实际测点的子模型生效。

只选择供电时不要求测点映射。取消能源计量或供电能力时，如果会使现有实例关系失去合法能力，保存失败并返回结构化影响清单；服务不级联删除关系，也不保留静默失效的悬挂关系。

配置提升时由单位库计算并冻结：

```text
standard_value = raw_value * scale + offset
```

累计量只允许 `offset = 0` 的线性换算，差值前先换算为标准单位。主瞬时量和过程瞬时量允许单位库定义的合法同量纲换算。

### 5.4 聚合校验

一次保存必须同时通过：

1. 能源类型字段、名称唯一性和 etag 校验。
2. 标准点数量、角色、名称和单位校验；已有正式 ID 必须属于当前聚合且不可变，新增标准点不能携带客户端指定的正式 ID。
3. 请求内新增标准点的临时关联标识必须唯一，每个映射必须唯一解析到一个已有或新增标准点，不能存在未解析、重复或混用的引用。
4. 模型存在性、候选类型和父子互斥校验。
5. 支持能力及必填映射完整性校验。
6. 实际测点归属、继承关系、重复映射和单位换算校验。
7. 取消能力的实例关系影响校验。

任一校验失败都不写入部分 pending。错误返回字段路径、资源引用、冲突项、受影响实例/关系和稳定 reason。

### 5.5 保存和边界提升

客户端携带 etag 提交完整聚合。校验通过后，服务在一个关系库事务中创建新的 pending generation，并更新状态指针、审计和 outbox。同一半小时边界前再次保存会整体替换旧 pending 并递增 generation。

服务在创建 pending generation 前为请求中的新增标准点分配正式 UUID，并把请求内临时引用全部解析为正式 `standard_point_id`；内容指纹、审计和 outbox 只使用解析后的正式聚合。相同 request ID 的幂等重试必须返回首次请求已经分配的相同正式 ID，不能再次生成另一组标准点。

`effective_slot_utc` 由服务按数据库时间计算，取部署业务时区中尚未封口的下一本地半小时边界并转换成 UTC，客户端不能指定。边界提升流程为：

```text
VALIDATED_PENDING
    -> 冻结 pending generation 和 XTwin 读取时点
    -> PROMOTING
    -> 重新校验模型、测点、单位和实例关系
    -> 构建候选 EffectiveMeterManifest
    -> 原子切换 active generation 和 active manifest
    -> ACTIVE
```

提升与该槽核算任务创建串行：先切换完整配置和 manifest，再创建任务快照。边界开始提升后，该 generation 固定为 promoting；期间的新保存进入下一边界的 pending，不覆盖正在提升的版本。

提升失败时 active 和旧 manifest 保持不变，不产生部分切换；失败原因进入技术状态、审计、日志、指标和告警。失败版本不无限自动重试，可显式重试或由新保存覆盖。新能源类型首次提升前没有 active，不参与表具解析和核算。

### 5.6 有效表具解析

`EffectiveMeterResolver` 的输入为：

```text
energy_type_id + config_generation
+ XTwin 一致性读取时点 + target_slot
```

解析顺序：

1. 读取 active 配置闭包。
2. 展开每个模型配置的继承子模型集合。
3. 只保留继承了全部必填实际测点的子模型；父子互斥保证一个实例最多命中一条配置。
4. 枚举命中模型的设备实例和组件实例。
5. 仅能源计量能力有效且累计量、主瞬时量映射完整的实例成为有效表具；只支持供电的实例不进入表具 manifest。
6. 为每个映射调用 `TelemetrySeriesResolver`，冻结结构化 `TelemetrySourceRef`、物理 `SeriesKey`、原单位、标准单位、换算参数和测点元数据指纹。
7. 校验同一能源类型、同一 manifest 中不同表具的唯一累计量不能解析到相同 `SourceKind + SeriesKey`；冲突时以 `DUPLICATE_SOURCE_SERIES` 拒绝整个候选 manifest，不能重复核算同一物理序列。
8. 生成不可变 manifest revision，供同一槽全部核算分片共享。

每个 manifest 成员至少包含：规范 `meter_id`、`InstanceRef`、实例实际模型、命中的配置模型、config generation 和 XTwin 读取时点。每个累计量、主瞬时量和已配置过程瞬时量映射至少包含对应 `standard_point_id`、结构化 `TelemetrySourceRef`、冻结的 `SourceKind`、`SeriesKey`、`SeriesKeyCodec`、单位换算参数、来源指纹及测点元数据指纹。

每个半小时边界都基于 active generation 和新的 XTwin 读取时点生成 manifest revision，即使配置 generation 没有变化：

- 新建实例、组件或改为匹配模型时，下一槽自动纳入。
- 删除实例、移除组件或改为不匹配模型时，下一槽停止生成，历史保留。
- 组件改名视为旧 `meter_id` 消失和新 `meter_id` 出现，不迁移历史。
- 单个表具没有遥测数据时仍保留在 manifest，由需求 2 产生缺失状态。
- 配置引用的模型测点被删除、改成不兼容量纲、不能解析为物理序列键或与其他表具的唯一累计量解析到相同物理序列时，本次 manifest 构建失败，继续使用上一完整 manifest 并触发技术告警，不发布半套清单。

同一解析器可以提供供电源候选投影，但供电候选不混入有效表具 manifest，也不自动创建 `supplyPower`。

### 5.7 内置配置和删除

标准包通过稳定 `builtin_key` 幂等初始化电、水、冷量，首次安装时由服务生成各自 UUID。三类能源预置原型规定的标准点和半小时增量上限 1500；电预置 9 个通用设备/组件模型的累计量和主瞬时量映射，默认只启用能源计量，不启用供电，不包含名称带“联通”的定制模型或电流映射。

直流电量仪的输入电能元数据未补齐 `kWh` 和有功电能语义时，标准包校验失败，不绕过单位校验。内置能源不可删除。自定义能源删除命令必须携带客户端最近读取聚合时取得的 `expected_etag`；服务在同一关系库事务中先比较当前 etag，不一致时返回 `ABORTED/ETAG_MISMATCH` 且不写入任何变更，再校验价格方案、模型能力关系及其他有效引用。etag 校验不能替代当前引用校验。校验通过后写入待生效 tombstone；生效后停止未来解析，历史事实、任务快照和审计保留。

当前方案不包含存量配置或事实迁移。未来迁移通过独立脚本完成，脚本使用新服务或离线校验器生成合法配置闭包。

### 5.8 产品接口视图和范围重算

已确认的产品原型不因 active/pending、promotion 或 manifest 技术状态而修改：不增加状态列、提示、弹窗、跳转或生效时间文案。产品 Get/List 始终返回 pending overlay；没有 pending 时返回 active。用户保存后看到目标配置，核算内部在边界切换前仍使用 active，这段短暂最终一致性是已接受的技术语义。

promotion 状态、失败原因、active/pending 差异和能力影响清单仅进入内部管理接口、错误 detail、日志、指标、告警和审计，不进入产品页面契约。

人工范围重算提交时，只对当前选中的 `meter_id + energy_type_id` 选择 pending（存在时）或 active（不存在时），一次性冻结配置内容及 generation、该能源类型的有效表具 manifest、所选表具的标准点和遥测点映射、部署业务时区和计算规则。执行及重试不读取后续配置变化；范围重算使用 pending 是权威写入，不是预览。

## 6. 需求 2：用量事实、质量与归档

### 6.1 半小时事实键和预期槽

```text
FactKey = (
  meter_id,
  energy_type_id,
  slot_start_utc,
  slot_width = 30m
)
```

槽在部署业务时区中表示为左闭右开区间 `[T0, T1)`，其中 `T1-T0` 为半小时；数据时间恰好等于 `T0` 的样本属于当前槽，恰好等于 `T1` 的样本属于下一槽。槽边界转换为 UTC 后以 `DateTime64` 保存。

只有已经封口且与表具 manifest 有效区间相交的槽才是预期槽。尚未封口的未来槽、表具首次生效前和停用后的槽不创建预期事实，也不标记 `MISSING`。每个预期 `FactKey` 最终都必须有 `usage_points.samples` 当前投影行；任务执行失败属于待恢复的技术故障，不能伪装成事实质量状态。

### 6.2 尾样本选择和用量计算

当前槽用量必须同时读取当前槽和紧邻上一槽的尾样本。尾值只允许从各自槽末的窗口选择，不能把整个槽内任意中间样本当作尾值：

```text
当前槽 S = [T0, T1)
上一槽 P = [T0-30m, T0)

W = min(30m, max(10m, 2 * acquisition_interval))
无法取得标准采集周期时 W = 10m

previous_tail = [T0-W, T0) 内事件时间最晚的样本
current_tail  = [T1-W, T1) 内事件时间最晚的样本
usage(S)      = normalize(current_tail) - normalize(previous_tail)
```

当前仓库没有标准采集周期的权威来源，首版统一使用允许的回退值 `W = 10m`。调度和快照结构保留采集周期解析接口；以后上游提供权威周期时按公式计算并冻结到任务快照，不改变事实键或质量模型。

保持现有 `metrics.samples` 数据源；它是上游采集产生的原始遥测时序表，不是有效用量事实或当前投影。核算 worker 按任务快照引用的 manifest 映射读取已经冻结的 `SourceKind + SeriesKey`，其中首版 `SourceKind = METRICS_SAMPLES`、`SeriesKey` 精确对应 `metrics.samples.__name__`。worker 不能根据 `meter_id`、`TwinID`、组件名或模型测点名重新拼接物理序列键，也不能在重试时调用 `TelemetrySeriesResolver`。

原始读取只使用冻结物理序列的事件时间和数值。每个窗口只按事件时间排序并选择最晚记录；同一事件时间存在重复记录时不定义确定性决胜规则，不扩展接收时间或稳定原始记录 ID。若重复记录的值不同，多次重算可能产生不同结果，该场景视为上游数据质量边界。

上一槽缺失时不能继续向前寻找更早槽的样本补基线。范围重算第一槽必须额外读取前一槽尾值窗口。尾样本选择不能跨越配置生效、唯一累计量实际测点切换、表具停用或规范身份边界；首次启用、重新启用或映射切换后的首个预期槽写入 `MISSING_PREVIOUS_TAIL`，当前尾值只作为下一槽基线。

### 6.3 事实状态和质量原因

半小时有效事实状态固定为：

```text
NORMAL           正常
MISSING          缺失
BAD              坏值
MANUAL_CORRECTED 人工修正
```

- `NORMAL`：有有效用能量，参与聚合和计费。
- `MISSING`：当前槽或上一槽没有尾值；用能量、成本为 `null`，不是零。
- `BAD`：尾值不可用、累计量回退或差值超过上限；用能量、成本为 `null`。
- `MANUAL_CORRECTED`：有效值来自人工 `SET`；参与聚合和计费，自动重算不能覆盖。

自动修订只允许 `NORMAL`、`MISSING`、`BAD`。`MANUAL_CORRECTED` 只出现在有效当前投影；`FAILED` 只属于 `WorkflowOperation/TaskShard`，不是事实状态。稳定 `reason_code` 独立于状态：

| 判定条件 | 状态 | `reason_code` |
| --- | --- | --- |
| 当前、上一尾值都缺失 | `MISSING` | `MISSING_BOTH_TAILS` |
| 当前尾值缺失 | `MISSING` | `MISSING_CURRENT_TAIL` |
| 上一尾值缺失 | `MISSING` | `MISSING_PREVIOUS_TAIL` |
| 当前尾值不是有效数值 | `BAD` | `INVALID_CURRENT_TAIL` |
| 上一尾值不是有效数值 | `BAD` | `INVALID_PREVIOUS_TAIL` |
| 标准化当前尾值小于上一尾值 | `BAD` | `COUNTER_ROLLBACK` |
| 差值严格大于能源类型半小时上限 | `BAD` | `INCREMENT_LIMIT_EXCEEDED` |
| 其余情况 | `NORMAL` | 空 |

两个尾值相等时用量为零且状态为 `NORMAL`；原始累计量为零本身不是异常。`COUNTER_ROLLBACK` 和 `INCREMENT_LIMIT_EXCEEDED` 只否定当前槽用量，数值有效的当前尾值仍可作为下一槽基线。无法解析为数值的尾值不能参与相邻槽计算，但状态仍为 `BAD`，不是“缺失”。配置或单位换算契约损坏属于任务失败和配置告警，不生成伪造的 `BAD` 事实。

旧引擎的零值删除、中位数补偿、自动表计复位推断和跨槽寻找更早样本全部退出需求 2 核算路径。

### 6.4 自动修订

`AutoUsageRevision` 追加写入，至少包含：

- `FactKey`、自动版本号和提交时预留的 `intent_revision`。
- 任务快照 ID、配置 generation、manifest ID 和业务时区。
- 上一槽尾样本和当前槽尾样本的事件时间、原值和标准化值；不存在的数据源字段不做伪造。
- 计算出的用量、自动状态和 `reason_code`。
- 原始输入指纹、计算指纹和稳定 `write_id`。

相同快照和相同已选输入产生相同指纹，重复执行不产生语义重复版本。同事件时间冲突重复可能导致所选输入变化，是已接受的非确定数据源边界。

### 6.5 人工 SET/CLEAR、锁定和冲突

人工操作只追加 `ManualUsageRevision`：

- `SET`：指定某个 `FactKey` 的有效用量并锁定当前事实；同一槽允许再次 `SET`。
- `CLEAR`：撤销人工覆盖并解除锁定，显示当前最新首选自动结果；不恢复首次人工修正前保存的旧值。

人工修订至少保存 `FactKey`、人工版本、`base_effective_revision`、前序版本、动作、原状态/原值、新值、原因、操作者 ID/名称、操作时间和 `request_id`。`SET` 的新用能量必填且大于等于零，原因选填；空原因原样保存为空，由 Reader 展示“—”。人工操作不修改原始累计量、尾样本和自动修订。

人工命令复用“表具能耗明细”页面编辑权限，不新增独立角色、确认或审批流程。`base_effective_revision` 提供乐观并发控制；陈旧页面提交返回版本冲突。相同 `request_id` 重试幂等。

`EffectiveFactProjector` 按以下顺序选择当前值：

1. 最新人工动作为 `SET` 时，选择该人工值并投影为 `MANUAL_CORRECTED`。
2. 最新人工动作为 `CLEAR` 或从未人工修正时，选择最高有效 `intent_revision` 的首选自动版本。
3. 没有可用自动版本时保留明确的无值自动状态。

人工锁定期间自动重算仍追加自动修订，但不能替换当前人工值。最新自动候选与人工值不同，或者自动候选为 `MISSING/BAD` 时，投影保存 `manual_conflict=true`、自动候选版本/状态/值和发现时间；冲突不增加新的事实状态。再次 `SET` 后按新人工值与最新自动候选重新判断，`CLEAR` 后清除人工冲突并切回最新自动结果。所有 `SET/CLEAR` 修订长期保留并可查询。

### 6.6 有效当前投影和时日月归档

`usage_points.samples` 是半小时有效用量唯一读取入口。每个预期 `FactKey` 有一行当前投影，保存有效版本、来源、状态、质量原因、用量、尾值、人工冲突、业务时区、计算时间、指纹和 `write_seq`；数值零与 `null` 严格区分。

`usage_points.rollups` 保存时、日、月当前归档，业务键为：

```text
RollupKey = (
  meter_id,
  energy_type_id,
  grain,              // HOUR / DAY / MONTH
  period_start_utc
)
```

归档至少保存 `period_end_utc`、`expected_count`、`normal_count`、`manual_count`、`missing_count`、`bad_count`、`valid_count`、`usage_total`、`rollup_status`、最后一个现存尾值及其事件时间、业务时区和 `write_seq`。最后尾值只用于页面参考，不参与聚合计算。日、月的预期半小时槽数按业务时区的实际周期计算，不能固定写死为 48 或固定天数。

时、日、月 `rollup_status` 只允许：

```text
COMPLETE
ABNORMAL
```

`NORMAL` 和 `MANUAL_CORRECTED` 计入 `valid_count` 和用量合计。只有 `valid_count == expected_count` 且不存在 `MISSING/BAD` 时为 `COMPLETE`；任一预期槽没有有效值时为 `ABNORMAL`。不单独返回“含人工修正”，但 `manual_count` 和人工审计链保留。

半小时当前事实变化后，汇总器重新扫描受影响周期的 `usage_points.samples` 并覆盖对应时、日、月投影，不做增量加减，避免重复消息和乱序执行造成累计偏差。汇总是可重建当前投影，不保存业务历史版本。

### 6.7 首次核算和自动修复

半小时边界 `T1` 到达时，调度器冻结该槽的 active 配置 generation、有效表具 manifest、测点映射、业务时区和每块表的 `W`。任务分片设置 `ready_at = T1 + W`，只有到达 `ready_at` 才能首次读取尾值和生成事实；相同 `ready_at` 的表具合并成批次，不为每块表创建独立定时器。首版因采集周期不可得，全部在 `T1 + 10m` 首次核算。

保持现有数据源意味着无法按接收事件精确发现晚到数据。自动修复采用周期回扫，沿用默认最近 3 个完整自然日并按部署业务时区每日执行。回扫重新选择各槽上一尾值和当前尾值，与已保存的输入指纹比较；发生变化时重新核算槽 `S` 和 `S+1`，随后重建用量汇总和成本。超出自动回扫范围的历史变化由页面范围重算处理。

任务提交时为每个 `FactKey` 预留单调递增的 `intent_revision`。后提交意图优先，即使其任务更晚完成；低版本输出不能覆盖高版本输出。范围重算生成的首选自动 lineage 不能被更早定时快照或修复任务改回。

### 6.8 人工范围重算

页面按当前选中的单个 `meter_id + energy_type_id` 创建异步 `WorkflowOperation`，一次完成用量和成本，不提供重算类型选择。请求包含起止日期和 `request_id`；日期按部署业务时区解释且首尾日期都包含，转换为 `[start_date 00:00, end_date + 1 day 00:00)`。

范围重算使用第 5.8 节冻结的配置和 manifest：

1. 为第一个目标槽额外读取上一槽尾值窗口。
2. 核算选择范围内全部半小时槽。
3. 额外核算范围结束后的第一个半小时槽，因为它使用最后目标槽的尾值作为基线。
4. 写入自动修订并更新未被人工锁定的当前投影。
5. 重建受影响时、日、月用量归档。
6. 冻结任务提交时的当前表具绑定，并按每个槽历史时点选择该当前绑定方案的价格版本，重算成本和成本汇总；历史价格修订及快照不被改写。

任务按自然日拆分分片并可独立重试。重叠任务允许执行，同一 `FactKey` 仍以后提交任务预留的 `intent_revision` 为准。人工锁定槽继续生成自动候选但不覆盖人工值；操作结果返回成功/失败槽数、`MISSING/BAD` 数量、人工锁定数量、待复核冲突数、阶段、进度和结构化失败原因。全部阶段完成后操作才标记成功。

## 7. 需求 4：计费方案、表具应用与成本

### 7.1 价格方案聚合

`PricePlan` 是需求 4 的写聚合根：

```text
PricePlan = (
  price_plan_id,      // 服务生成 UUID，永久不变
  display_name,
  energy_type_id,     // 创建后不可修改
  etag,
  applied_meter_count,
  created_by,
  created_at,
  deleted_at
)
```

当前数据库实例是隔离边界。方案名称 1–99 字符并在实例内唯一；同一能源类型允许创建多个方案，现有实现“一种能源只能有一个方案”的限制不进入新领域服务。编辑聚合时允许修改方案名称，但不能修改 `energy_type_id`。

删除方案命令必须携带客户端最近读取聚合时取得的 `expected_etag`。服务在同一关系库事务中先比较当前 etag，不一致时返回 `ABORTED/ETAG_MISMATCH` 且不写入任何变更，再校验当前应用表具数为零；etag 校验不能替代当前应用关系校验。删除采用逻辑 tombstone，使产品列表和普通 Get 返回 `NotFound`；价格版本、绑定审计、成本修订和价格快照长期保留。

### 7.2 不可变价格版本和 48 槽日历

方案明细对应 `PricePlanRevision`：

```text
PricePlanRevision = (
  price_plan_revision_id,
  price_plan_id,
  effective_date,
  pricing_type,
  currency = CNY,
  unit_id,
  business_timezone,
  raw_rule_fingerprint,
  calendar_fingerprint,
  created_by,
  created_at
)
```

同一方案当前生效日程中的 `effective_date` 唯一，按部署业务时区当天 `00:00` 生效。第一条版本允许使用历史、今天或未来日期，以支持新建方案后补算历史成本；方案已经存在生效版本后，只能新增未来版本。`effective_date <= 当前业务日期` 的日程项永久锁定，不允许编辑或删除。未来版本允许编辑、修改为其他未来业务日期或二次确认删除。

价格修订内容始终不可变。编辑未来版本时创建新的 `PricePlanRevision` 及 48 槽日历，并在同一事务中把未来生效日程切换到新修订；移动日期等价于原子删除旧未来日程项并新增新日程项；删除未来版本只从当前日程移除。被替代或移除的修订、原始规则和日历继续保留用于审计，但不再参与后续任务的价格选择。以上修订替换状态只存在于技术存储和审计，不增加产品原型状态。

版本有效区间为 `[effective_date, next_effective_date)`，`valid_to` 由服务按下一版本日期派生，不接受客户端作为权威输入。删除未来版本后，前一版本自然延续到下一版本或无限期。新增、编辑或删除未来版本都不自动改写已有成本事实。

产品只支持：

```text
FLAT   统一单价
TOU_3  峰 / 平 / 谷
TOU_4  尖 / 峰 / 平 / 谷
```

价格版本保存时把原始规则编译为当地墙钟一天的 48 个半小时槽，每个槽保存 `slot_index`、计价类别和单价。输入和编译满足：

- 起止时间只能使用 `00` 或 `30` 分，区间统一为 `[start,end)`。
- 所有时段连续、无重叠、无空洞地覆盖 `00:00` 至次日 `00:00`。
- `TOU_3` 的峰/平/谷和 `TOU_4` 的尖/峰/平/谷都必须至少有一个时段和一个单价。
- 单价必填、大于等于零、最多 2 位小数；零单价合法。
- 计价单位固定为 `CNY / 唯一累计量标准单位`，`unit_id` 从能源配置派生并冻结。
- 不支持按星期差异、阶梯价、分档价或运行时默认回退。

原始规则是审计真源，48 槽价格日历是可重建运行投影；两者指纹同时保存并在任务快照中冻结。成本运行时不遍历原始时段。夏令时春季跳时日只为实际存在的事实槽计价；秋季重复墙钟时段的两个 UTC 槽使用同一个当地日历槽和单价。

### 7.3 当前表具绑定和应用集合

计费方案应用关系不表达历史业务有效区间，只保存当前状态：

```text
MeterPriceBindingState = (
  meter_id,
  energy_type_id,
  price_plan_id,
  binding_revision,
  etag
)
```

`MeterPriceBindingRevision` 追加记录 `SET/CLEAR`、前后方案、操作者、操作时间和 `request_id`，只用于审计。一个方案可以应用多块表具；同一 `meter_id + energy_type_id` 最多一个当前方案。

候选来自当前能源类型的有效表具 manifest，设备和组件都使用规范 `meter_id`。只有具备唯一累计量映射的表具进入候选；是否绑定计量对象不影响应用。关键字、物模型筛选和分页由服务端完成。已被其他方案占用的表具返回占用方案并置灰，当前方案成员默认选中。

应用弹窗“确定”提交当前方案完整的已选表具集合、集合 etag 和 `request_id`。服务在一个关系库事务中重新校验方案、能源类型、表具资格和占用状态，计算新增/保留/移除集合，更新全部当前绑定和应用计数，追加修订、审计及 outbox。任一表具冲突则整体失败，不产生部分应用；取消或关闭弹窗不提交变化。

绑定保存后立即成为产品和后续任务的当前绑定，不增加技术状态。已经冻结的任务不受随后改绑影响。改绑本身不自动重算历史；A 改为 B 后，之后提交的任意历史范围重算都使用 B，再按每条明细的本地日期选择 B 的价格版本。若 B 在该日期没有价格版本，则为 `NO_PRICE_VERSION`，不能回退 A 或 B 的第一条版本。解绑后的后续任务没有方案；既有成本修订仍保留，只有再次重算相应历史范围时才生成无方案结果。

### 7.4 半小时成本修订和当前投影

`CostFactRevision` 独立于用量事实，沿用用量 `FactKey` 并至少保存：

```text
FactKey
cost_revision
cost_intent_revision
effective_usage_revision
binding_revision
price_plan_id
price_plan_revision_id
price_calendar_fingerprint
pricing_category
unit_price
unit_id
currency
cost_amount_raw
cost_status
reason_code
business_timezone
task_snapshot_id
write_id
calculated_at
```

半小时成本状态固定为：

```text
COMPUTED          已计算
NO_USAGE         无有效用量
NO_PRICE_PLAN    未应用计费方案
NO_PRICE_VERSION 当前方案在该日期无适用价格版本
```

- 有效用量状态为 `NORMAL/MANUAL_CORRECTED` 才进入计价；`MISSING/BAD` 生成 `NO_USAGE`，成本为 `null`。
- 任务快照没有当前绑定时生成 `NO_PRICE_PLAN`，成本为 `null`。
- 存在绑定但方案在明细本地日期没有版本时生成 `NO_PRICE_VERSION`，成本为 `null`。
- 价格配置损坏和执行异常属于任务失败、重试和告警，不写 `INVALID_PRICE/FAILED` 成本业务状态。
- 零单价生成数值零且状态为 `COMPUTED`。

状态按 `NO_USAGE -> NO_PRICE_PLAN -> NO_PRICE_VERSION -> COMPUTED` 的顺序判定：先判断有效用量，再判断当前绑定，最后判断历史日期价格版本。同一槽同时缺少用量和方案时固定为 `NO_USAGE`。成本修订保存判定时已经取得的上游身份和快照；因为前置状态而未解析的绑定、方案、价格版本、类别、单价及金额字段为 `null`，不能伪造占位版本。

计算时把 `slot_start_utc` 转到部署业务时区，用本地日期选择 `effective_date <= local_date` 的最新价格版本，再用 `slot_index = (hour * 60 + minute) / 30` 读取编译日历：

```text
cost_amount_raw = effective_usage * unit_price
```

用量、单价和成本使用十进制定点数。半小时成本保存高精度结果，不在事实层舍入；完整价格快照随成本修订保存，因此方案改名、改绑或逻辑删除后仍可独立解释。

`usage_points.cost_samples` 是半小时有效成本唯一读取入口。当前投影只接受更高 `cost_intent_revision` 且引用当前有效用量版本的结果；低版本成本不能覆盖新用量对应的结果。

### 7.5 时日月成本汇总

`usage_points.cost_rollups` 从半小时成本当前投影完整重建，保存：

```text
expected_count
computed_count
no_usage_count
no_price_plan_count
no_price_version_count
cost_amount_raw_total
flat_amount_raw
super_peak_amount_raw
peak_amount_raw
shoulder_amount_raw
off_peak_amount_raw
cost_completeness
```

成本完整性独立于用量 `rollup_status`：

- 全部预期槽均为 `COMPUTED`：`COMPLETE`。
- 至少一个槽为 `COMPUTED` 但仍有未计算槽：`PARTIAL`，返回已计算部分金额和各类缺失数量。
- 没有任何 `COMPUTED` 槽：`MISSING`，金额返回 `null`。

时、日、月只累加高精度半小时成本，不重新乘单价。Reader 最终输出人民币金额时使用 `ROUND_HALF_UP` 舍入到 2 位小数；已接受页面显示的半小时金额逐项相加可能与高精度周期汇总存在尾差。统一价只计入 `FLAT`，尖/峰/平/谷分类金额供下游指标查询且不重复计入总成本。日汇总预期数量以需求 2 的实际预期事实数为准，不固定为 48。

### 7.6 成本触发和范围重算

`UsageEffectiveChanged` 唤醒 `CostingWorkflow`。正常半小时、人工 `SET/CLEAR`、迟到数据修复和范围重算产生新的有效用量后，都生成对应成本修订和汇总。新增未来价格版本、应用、改绑或解绑方案本身不重算历史；计费方案页面不提供重算入口。对象成本不物化，也不创建对象成本重算任务。

范围重算沿用需求 2 的同一个 operation：先重算用量，选择人工锁后的有效值，再计算成本并重建两类汇总。提交时冻结当前绑定及该方案在范围内的不可变版本和编译日历，当前绑定用于整个历史范围；执行期间再次改绑不影响已冻结任务。成本阶段失败时 operation 不标记成功，但可独立重试且不重复生成用量语义版本。

每个 `FactKey` 预留单调 `cost_intent_revision`。后提交成本意图优先，旧任务即使后完成也不能覆盖新用量或新成本结果。对象层在后续查询时读取最新表具成本并按当前计量拓扑归集，不反向改写表具事实。

## 8. 任务模型



### 8.1 操作、快照和分片

```text
WorkflowOperation
    -> TaskSnapshot
    -> EffectiveMeterManifest
    -> TaskShard[]
    -> ReservedOutputRevision[]
```

`TaskSnapshot` 至少冻结：

- 配置资源内容和 generation。
- XTwin 视图版本或读取时点。
- 有效表具和测点映射，包括结构化 `TelemetrySourceRef`、物理 `SeriesKey`、`SeriesKeyCodec`、来源指纹和单位换算参数。
- 槽范围、部署业务时区、每块表的尾值窗口 `W` 和尾样本规则。
- 当前价格绑定、绑定修订、相关不可变价格版本和编译日历（成本任务）。
- 提交者、原因、关联请求和优先级。

`TaskShard` 是可重试的最小执行单元。关系库保存 `ready_at`、状态、租约、尝试次数、下次执行时间、检查点和结构化错误。正常半小时任务固定一个目标槽，按相同 `ready_at` 将表具合并成批次；批次大小由部署配置控制。页面范围重算对当前选中的单个 `meter_id + energy_type_id` 只创建一个 `WorkflowOperation` 和一份冻结快照，再按部署业务时区的自然日拆成多个可独立重试的分片。每个日分片包含该日全部预期半小时 `FactKey`；首个分片额外读取范围前一槽的尾值窗口，最后一个分片额外核算范围结束后的第一个半小时槽。存在夏令时的日期按实际槽数生成，不能固定为 48。

同一 operation 的日分片可以并行执行，因为每个槽直接读取自己的上一槽和当前槽原始尾值，不依赖前一分片先生成用量结果。用量、用量汇总、成本和成本汇总属于同一范围重算 operation 的连续阶段；某个日分片失败时已经确认的其他日事实不回滚，只重试失败分片或失败阶段。operation 只有在全部必要阶段完成后才标记成功。

### 8.2 优先级

- `INTERACTIVE`：人工 `SET/CLEAR`。
- `SCHEDULED`：正常半小时任务。
- `REPAIR`：迟到数据和失败重试。
- `BULK`：页面人工范围重算。
- `MAINTENANCE`：汇总重建和一致性校验。

取消只阻止尚未开始的分片，已经提交的事实不回滚。

### 8.3 租约、状态和重试

`TaskShard` 状态只允许：

```text
PENDING
RUNNING
RETRY_WAIT
SUCCEEDED
FAILED
CANCELLED
```

状态转换为：`PENDING -> RUNNING -> SUCCEEDED`；可重试技术错误执行 `RUNNING -> RETRY_WAIT -> RUNNING`；不可重试错误或尝试次数耗尽执行 `RUNNING -> FAILED`；取消只允许把尚未开始的 `PENDING/RETRY_WAIT` 分片切换为 `CANCELLED`。`MISSING/BAD` 是成功生成的业务事实，不触发任务重试；`FAILED` 只属于 operation/shard 技术状态。

worker 使用数据库时间原子领取满足以下条件的分片：状态为 `PENDING/RETRY_WAIT`、`ready_at <= now`、`next_attempt_at <= now`，并且租约不存在或已经过期。领取成功后写入 `lease_owner`，递增单调 `lease_epoch`，设置 `lease_deadline = now + 2m`，记录 `heartbeat_at` 并递增 `attempt_count`。多个 worker 竞争时只能有一个原子更新成功。

worker 每 30 秒续租一次，把 `lease_deadline` 延长为数据库当前时间之后 2 分钟；单次 shard 执行上限为 30 分钟。领取、续租、检查点、成功和失败更新都必须同时匹配 `operation_id + shard_id + lease_owner + lease_epoch + RUNNING`。续租失败后 worker 必须停止处理新的输出；已经提交的输出不回滚，由稳定 `write_id` 幂等确认。`lease_epoch` 是 fencing token，租约过期后恢复的旧 worker 不能更新新持有者的任务状态。

执行结果分为：

- 业务结果：`NORMAL/MISSING/BAD/MANUAL_CORRECTED` 及成本业务状态，正常提交事实并视为任务成功。
- `RETRYABLE`：ClickHouse 或关系库超时、死锁、依赖服务暂时不可用等临时技术错误，进入退避重试。
- `PERMANENT`：快照损坏、不支持的 payload version、冻结单位或测点契约非法等确定性技术错误，立即进入 `FAILED`。
- `UNCERTAIN_COMMIT`：ClickHouse 可能已写入但关系库未确认，先按 `write_id` reconciliation，不能直接生成新版本。

结构化错误至少保存 `error_class`、稳定 `error_code`、`failed_stage`、可观察 `error_detail` 和 `last_failed_at`；调度器不能解析错误文本决定是否重试。首版重试策略为：`SCHEDULED/REPAIR` 最多 8 次，`BULK` 最多 5 次，`MAINTENANCE` 最多 3 次。退避时间为 `min(30m, 10s * 2^(attempt_count-1))`，再乘 `[0.5, 1.5]` 的随机抖动。重试复用原 `snapshot_id`、`FactKey`、`intent_revision/cost_intent_revision`、输出版本和 `write_id`，只能更新租约、尝试次数、下次执行时间、错误和检查点。

尝试次数耗尽后 shard 进入终态 `FAILED`，清空租约，operation 标记失败并返回成功/失败分片及事实数量，同时产生告警和 reconciliation 待处理项；不能向事实投影写入 `FAILED`。reconciliation 可以自动补齐不确定提交、过期租约、未发布 outbox 和未推进投影，但不能在重试耗尽后立即循环创建新 operation。

“重试失败任务”创建新的 redrive operation，只复制原 operation 的失败分片，保存 `root_operation_id` 和递增的 `redrive_generation`，复用原快照及未完成输出的预留版本和 `write_id`，不形成新的业务意图。“重新计算”则创建全新 operation 和快照，并预留更高的业务意图版本。每日缺口回扫也可以在确认事实仍缺失后创建新的 `REPAIR` operation。旧 `FAILED` operation/shard 保持终态，不原地改回 `PENDING`。

shard 只有在全部输出已经按 `write_id` 确认、当前投影已经处理、关系库输出状态和 outbox 已持久化后才能标记 `SUCCEEDED`；不要求 outbox 消息已经实际发送。operation 从 shard 和阶段状态聚合进度，不定义会掩盖失败的 `PARTIAL_SUCCESS` 终态。

### 8.4 事务和恢复

自动事实提交顺序：

1. 关系库事务预留 `intent_revision`、输出版本和稳定 `write_id`。
2. 使用 `write_id` 幂等追加 ClickHouse 自动修订。
3. 按人工锁和最高有效 `intent_revision` 幂等更新半小时当前投影。
4. 关系库标记输出已写入，并写入 outbox。
5. 异步投递领域事件并重建用量汇总和成本。

ClickHouse 已写入但关系库未确认时，reconciliation 按 `write_id` 补齐；关系库已预留但 ClickHouse 未写入时重试。已写入的事实不通过回滚删除来解决跨库失败。

任务执行失败仅更新 `WorkflowOperation/TaskShard` 并按策略重试，不向 `usage_points.samples` 写入 `FAILED`，也不把技术失败伪装成 `MISSING`。重叠任务可以并行执行，但低 `intent_revision` 的自动修订不能覆盖更高版本；最新人工 `SET` 始终优先，`CLEAR` 后选择当前最高有效自动版本。

成本任务在关系库事务中预留 `cost_intent_revision`、成本输出版本和 `write_id`，幂等追加 ClickHouse 成本修订，再按成本意图和 `effective_usage_revision` 更新成本当前投影。用量阶段已提交而成本阶段失败时只重试成本，不回滚或重复生成用量版本。成本当前投影和关系库确认之间的中断沿用相同 `write_id` reconciliation 规则。

## 9. 消息结构

统一事件 envelope：

```text
event_id
event_type
aggregate_key
aggregate_revision
occurred_at
correlation_id
causation_id
payload_version
payload
```

关键事件：

- `EnergyConfigurationPendingSaved`
- `EnergyConfigurationActivated`
- `EnergyConfigurationActivationFailed`
- `EffectiveMeterManifestReady`
- `AutoUsageRevisionReady`
- `ManualUsageRevisionChanged`
- `UsageEffectiveChanged`
- `PricePlanChanged`
- `PriceBindingChanged`
- `CostRevisionReady`
- `CostEffectiveChanged`

消息采用至少一次投递。消费者按 `event_id`、聚合版本和 `effective_revision` 幂等；不依赖全局消息顺序。事件只负责唤醒和传播事实变化，关系库和 ClickHouse 是恢复依据。

## 10. 物理存储



### 10.1 关系库控制面

需求 1 使用以下逻辑表：

| 表 | 主键和用途 |
| --- | --- |
| `energy_config_state` | `energy_type_id`；active/pending/promoting 指针、active manifest、etag、内置标记和技术状态 |
| `energy_config_generation` | `(energy_type_id, generation)`；基础字段、内容指纹、生效槽和审计信息 |
| `energy_standard_point` | `(energy_type_id, generation, standard_point_id)` |
| `energy_model_config` | `(energy_type_id, generation, model_id)`；能源计量/供电能力 |
| `energy_point_mapping` | `(energy_type_id, generation, model_id, standard_point_id)` |
| `effective_meter_manifest` | `manifest_id`；generation、XTwin 读取时点、适用槽和内容指纹 |
| `effective_meter_member` | `(manifest_id, meter_id)`；实例和命中模型 |
| `effective_meter_mapping` | `(manifest_id, meter_id, standard_point_id)`；结构化遥测来源、`source_kind`、`source_series_key`、`source_key_codec`、来源/测点指纹和单位换算快照 |
| `energy_config_audit` | 配置创建、替换、提升、失败、重试和删除审计 |

配置保存时，generation 主表、标准点、模型配置、映射、状态指针、审计和 outbox 在一个事务内提交。新增标准点的正式 UUID 在该事务中分配或从相同 request ID 的幂等结果中恢复；请求内 `client_key` 或等价临时引用不写入任何持久化表。manifest 先写不可变候选记录，再在短事务中原子切换 active generation 和 active manifest。相同 request ID 和内容指纹不创建语义重复 generation 或 manifest。

需求 2 使用以下逻辑表：

| 表 | 主键和用途 |
| --- | --- |
| `billing_runtime_identity` | 单行部署身份；业务时区和事实模型版本 |
| `workflow_operation` | `operation_id`；操作类型、阶段、终态、分片/事实计数、`root_operation_id` 和 `redrive_generation` |
| `workflow_snapshot` | `snapshot_id`；配置、manifest、时区、范围和规则快照 |
| `workflow_shard` | `(operation_id, shard_id)`；状态、`ready_at`、`lease_owner/lease_epoch/lease_deadline/heartbeat_at`、`attempt_count/max_attempts/next_attempt_at`、检查点、结构化错误和完成时间 |
| `usage_fact_intent` | `FactKey`；当前最高预留 `intent_revision` 和输出 `write_id` |
| `manual_usage_revision` | `manual_revision_id`；追加保存人工 `SET/CLEAR` |
| `idempotency_request` | `(command_type, request_id)`；人工操作和范围重算幂等 |
| `outbox_event` | `event_id`；可靠事件发布 |
| `usage_audit_index` | `FactKey + occurred_at + revision`；人工操作和任务追溯索引 |

需求 4 使用以下逻辑表：

| 表 | 主键和用途 |
| --- | --- |
| `price_plan` | `price_plan_id`；名称、能源类型、etag、应用计数和逻辑删除 |
| `price_plan_revision` | `price_plan_revision_id`；不可变版本、生效日期、币种、单位、时区和指纹 |
| `price_plan_revision_schedule` | `(price_plan_id, effective_date)`；当前生效和未来日程指向的不可变修订 |
| `price_category_rate` | `(price_plan_revision_id, category)`；各类别两位小数单价 |
| `price_rule_period` | `(price_plan_revision_id, category, period_index)`；原始时段配置 |
| `price_calendar_slot` | `(price_plan_revision_id, slot_index)`；48 槽编译日历 |
| `meter_price_binding_state` | `(meter_id, energy_type_id)`；当前方案、绑定修订和 etag |
| `meter_price_binding_revision` | `binding_revision_id`；绑定、改绑和解绑审计链 |
| `cost_fact_intent` | `FactKey`；最高 `cost_intent_revision` 和输出 `write_id` |
| `price_plan_audit` | 方案创建、聚合替换、版本锁定、应用和删除审计 |

价格方案聚合保存时，基础信息、不可变未来修订、生效日程、类别单价、原始时段、48 槽日历、审计和 outbox 在一个关系库事务内提交。应用集合保存时，全部绑定状态、绑定修订、方案应用计数、幂等结果、审计和 outbox 在一个事务内提交；任一占用冲突整体回滚。

关系库还保存 reconciliation 状态。人工命令在同一事务中提交修订、幂等结果、输出预留、审计索引和 outbox。被任务快照引用的 generation/manifest 不删除；被成本修订或任务引用的价格版本、绑定修订和价格快照长期保留。

### 10.2 ClickHouse 事实面

现有 `metrics.samples` 是需求 2 的上游原始遥测数据源，物理序列键为 `__name__`。本设计不修改其表结构，也不把它作为用量事实读取入口；从结构化遥测引用到 `__name__` 的解析只在 manifest 构建阶段执行，解析结果持久化在 `effective_meter_mapping` 并由任务快照冻结。

使用以下逻辑表：

- `usage_points.auto_revisions`：自动用量版本，追加写。
- `usage_points.samples`：有效用量当前投影，唯一用量读入口。
- `usage_points.rollups`：用量当前汇总。
- `usage_points.cost_revisions`：成本事实版本，追加写。
- `usage_points.cost_samples`：有效成本当前投影。
- `usage_points.cost_rollups`：成本当前汇总。

`usage_points.auto_revisions` 只保存当前数据源实际提供的尾样本事件时间、原值和标准化值，不保存接收时间或稳定原始记录 ID。其自动状态只允许 `NORMAL/MISSING/BAD`；`MANUAL_CORRECTED` 只存在于 `usage_points.samples` 当前投影。

`usage_points.cost_revisions` 追加保存第 7.4 节完整价格快照和高精度成本；业务状态只允许 `COMPUTED/NO_USAGE/NO_PRICE_PLAN/NO_PRICE_VERSION`。`usage_points.cost_samples` 按 `cost_intent_revision + effective_usage_revision` 选择当前值。`usage_points.cost_rollups` 保存高精度金额、分类金额、各状态计数和 `COMPLETE/PARTIAL/MISSING`，Reader 输出时才舍入到人民币 2 位小数。

事实表按 UTC 月份组织分区，自动修订排序键优先为能源类型、表具、槽和修订；当前投影与汇总排序键优先服务能源类型、表具和周期查询。`meter_hash` 只能用于加速，身份判断仍比较完整规范 `meter_id`。

用量和成本使用十进制定点数，时间使用 UTC `DateTime64`。日、月和峰谷时段按部署业务时区解释，时区规则写入价格快照。

当前投影使用版本条件保证逻辑上的最新值，Reader 按完整业务键读取最大有效版本，不能依赖 ClickHouse 后台合并形成唯一性。

`UsageRollup` 和 `CostRollup` 只是可重建当前投影，不保存业务历史版本。它们只保留单调递增的技术 `write_seq` 防止并发乱序覆盖；每次从受影响周期的半小时当前投影完整重算，不做增量加减。汇总审计完全依赖用量事实、人工版本、成本事实、价格快照和审计链路。

用量事实、自动版本、人工版本、成本事实、价格快照、任务快照和审计链路按当前部署数据库的业务生命周期长期保留，默认不设置删除型 TTL。汇总投影可以整体删除并从权威事实重建。

## 11. 服务和 API 契约



### 11.1 新应用服务

内部应用服务包括：

- `EnergyConfigurationService`：需求 1 的聚合创建、完整替换、删除、pending overlay 读取和提升重试。核心方法包括 `CreateEnergyConfiguration`、`GetEnergyConfiguration`、`ListEnergyConfigurations`、`ReplaceEnergyConfiguration`、`DeleteEnergyConfiguration` 和 `RetryEnergyConfigurationActivation`。
- `EnergyConfigurationQueryService`：提供 `ListModelCandidates`、`ListTelemetryCandidates`、`ListEffectiveMeters` 和 `ExplainEffectiveMeter`；技术状态和失败原因只通过内部管理接口暴露，不改变产品原型。
- `MeterUsageService`：有效用量读取、自动/人工版本读取、`SET/CLEAR`、范围重算、解释和导出。
- `EnergyPricingService`：价格方案修订、表具绑定修订、价格校验和成本解释。
- `UsageFactReader`、`CostFactReader`、`MeterDetailReader`：需求 5、6 及页面消费者的唯一事实读取入口。

Reader 返回数值或空值状态、质量、完整性、来源版本、指纹和计算时间。下游不能直接访问 ClickHouse。

### 11.2 需求 2 页面与命令契约

`MeterUsageService` 对表具能耗明细页面提供：

```text
ListEnergyMeters
QueryMeterEnergyDetails
GetManualCorrectionAudit
SetManualCorrection
ClearManualCorrection
StartRangeRecompute
GetRangeRecomputeOperation
```

`ListEnergyMeters` 输入能源类型、空间节点、搜索词和当前查询范围。空间树由 XTwin 提供，服务端展开节点下的有效表具；空间过滤和名称搜索同时生效。结果返回规范 `meter_id`、表具名称、所属空间和范围完整性，按异常优先、名称稳定排序，支持原型默认选中首个异常表具。

`QueryMeterEnergyDetails` 输入：

```text
meter_id
energy_type_id
grain          // HALF_HOUR / HOUR / DAY / MONTH
range_start
range_end
page_size
page_token
```

半小时粒度关联读取 `usage_points.samples` 和 `usage_points.cost_samples`，每行至少返回周期起止、可空用量、可空成本、用量事实状态、成本状态、可空质量原因、上一/当前尾值、可空人工修订 ID 和人工冲突信息。时、日、月关联读取 `usage_points.rollups` 和 `usage_points.cost_rollups`，每行至少返回周期起止、可空用量合计、可空成本合计、用量 `rollup_status`、成本 `cost_completeness`、各用量质量及成本缺失计数，以及最后一个现存尾值和时间。

月、日、时下钻直接把当前行的周期起止作为下一粒度查询范围：

```text
MONTH -> DAY -> HOUR -> HALF_HOUR
```

Reader 固定以下展示语义：

- 用量合计只累加 `NORMAL` 和 `MANUAL_CORRECTED`；成本合计只累加 `COMPUTED` 槽。
- 半小时没有价格方案或没有适用价格版本时，成本返回 `null` 和对应状态，前端展示“—”；不能回退其他方案或第一条价格版本。
- `MISSING/BAD` 的半小时用量返回 `null`，对应成本状态为 `NO_USAGE` 且成本返回 `null`；趋势点返回空值形成断点，不能补零。
- 成本完整性独立于用量完整性：全部预期槽均为 `COMPUTED` 时返回 `COMPLETE`；部分槽已计算时返回 `PARTIAL`、部分成本和各缺失计数；没有任何已计算成本时返回 `MISSING` 且成本合计为 `null`。
- 范围汇总和完整性针对完整查询范围计算，不受明细分页影响。
- 所有预期槽都有 `NORMAL/MANUAL_CORRECTED` 有效值时为 `COMPLETE`，否则为 `ABNORMAL`。
- 查询范围内根本没有有效表具或可展示预期槽时返回空集合，由前端展示“暂无数据”。存在预期槽但没有尾值时必须返回 `MISSING`。
- “加载中”只由前端请求生命周期表达，不进入任何事实或汇总状态。
- 时、日、月最后尾值只作参考，聚合值始终来自半小时当前事实求和。

明细按周期起点降序并使用稳定 `page_token`，不能用易受事实更新影响的页码偏移。人工审计接口根据人工修订 ID 返回完整 `SET/CLEAR` 链，默认定位当前生效或最近一次人工操作；空人工原因统一展示“—”。

`SetManualCorrection` 和 `ClearManualCorrection` 携带 `base_effective_revision` 与 `request_id`。范围重算按第 6.8 节返回异步 operation；页面运行期间禁用重算按钮，但不向事实模型增加“运行中”状态。已确认的产品原型不因任务、投影或存储技术状态而增加字段、弹窗或状态标签。

### 11.3 需求 4 页面与命令契约

`EnergyPricingService` 提供：

```text
CreatePricePlan
GetPricePlan
ListPricePlans
ReplacePricePlan
DeletePricePlan
ListPricePlanMeterCandidates
GetPricePlanApplications
ReplacePricePlanApplications
```

`ListPricePlans` 支持方案名称模糊搜索、物模型筛选、`page_size` 和稳定 `page_token`。物模型筛选表示方案当前应用表具中至少一块表具的实际模型匹配该条件。列表返回能源名称、方案名称、当前应用表具数、创建时间和 etag，按创建时间倒序、`price_plan_id` 稳定排序；逻辑删除方案不返回。

`CreatePricePlan/ReplacePricePlan` 以整个聚合保存。客户端提交基础信息、所有已存在版本和目标未来版本，服务端要求已生效版本与权威记录完全一致，重新校验版本日期、时段、48 槽日历、价格精度、能源单位和 etag。试图修改或漏传已生效版本时整体拒绝。产品页面可以修改方案名称及未来版本，能源类型始终只读。

应用范围接口返回：

```text
meter_id
display_name
actual_model_id
actual_model_name
selected
occupied_by_other_plan
occupied_plan_name
```

候选查询支持关键字、物模型筛选和分页；当前方案应用集合单独返回集合 etag。`ReplacePricePlanApplications` 携带完整 `meter_id` 集合和 `request_id`，执行第 7.3 节的原子替换。

`MeterDetailReader` 在第 11.2 节字段基础上，半小时行返回成本状态、价格方案/版本、计价类别、单价、币种和高精度成本的两位小数展示值；时、日、月返回成本完整性、计算数量、三类缺失数量、部分成本及尖/峰/平/谷分类金额。前端不能根据金额是否为零推导成本状态。

已确认的产品原型不增加价格版本技术状态、绑定生效提示或成本重算入口。已生效/未来的操作可用性由服务按部署业务日期返回，前端据此启用或置灰现有编辑、删除动作。

### 11.4 旧协议兼容层

需求 1、2、4 范围内保留现有 `EnergyService` 服务名和生成的 protobuf 契约。以下方法和路径作为兼容入口：


| RPC                       | HTTP                                                                  |
| ------------------------- | --------------------------------------------------------------------- |
| `CreateEnergyType`        | `POST /xtwin/v3/energy/v1/configs`                                    |
| `GetEnergyType`           | `GET /xtwin/v3/energy/v1/types/{name}`                                |
| `UpdateEnergyType`        | `PATCH /xtwin/v3/energy/v1/types/{energy_type.name}`                  |
| `DeleteEnergyType`        | `DELETE /xtwin/v3/energy/v1/types/{name}`                             |
| `ListEnergyTypes`         | `GET /xtwin/v3/energy/v1/types`                                       |
| `CreatePricePlan`         | `POST /xtwin/v3/energy/v1/price_plans`                                |
| `GetPricePlan`            | `GET /xtwin/v3/energy/v1/price_plans/{name}`                          |
| `UpdatePricePlan`         | `PATCH /xtwin/v3/energy/v1/price_plans/{price_plan.name}`             |
| `DeletePricePlan`         | `DELETE /xtwin/v3/energy/v1/price_plans/{name}`                       |
| `ListPricePlans`          | `GET /xtwin/v3/energy/v1/price_plans`                                 |
| `ListBillings`            | `GET /xtwin/v3/energy/v1/logical_devices/-/billing:list`              |
| `BatchUpdateBillings`     | `PUT /xtwin/v3/energy/v1/logical_devices/-/billing:batchUpdate`       |
| `ListMeasurements`        | `GET /xtwin/v3/energy/v1/logical_devices/{name}/measurements:list`    |
| `UpdateMeasurement`       | `PUT /xtwin/v3/energy/v1/logical_devices/{name}/measurements`         |
| `BatchImportMeasurements` | `GET /xtwin/v3/energy/v1/logical_devices/-/measurements:batchImport`  |
| `BatchExportMeasurements` | `POST /xtwin/v3/energy/v1/logical_devices/-/measurements:batchExport` |


protobuf 字段编号、字段类型、请求/响应结构和 HTTP 动词路径不可修改。兼容适配器负责：

- protobuf 到领域命令及领域结果到旧响应的转换。
- proto3 默认值和旧字段存在性处理。
- 旧身份形式到规范 `meter_id` 的解析和规范化。
- 没有 `request_id` 时生成稳定幂等键。
- 新领域错误到旧 gRPC code、HTTP status 和可观察错误 detail 的映射。

旧的 `ListBillings`、`ListMeasurements` 和配置查询都从新 Reader 或配置服务的兼容视图读取，不能查询旧表。旧接口与任何新增接口都必须汇入相同的事实和投影服务。

### 11.5 兼容读写语义

- 旧 `EnergyType.name` 映射到稳定 `energy_type_id`；创建能源类型时 UUID 由服务生成并在响应中返回。
- 旧能源配置写操作写入 pending，写响应返回 pending 兼容视图。更新采用受限无损合并，不能覆盖旧协议无法表达的配置。
- 旧 `EnergyType.unit` 只映射唯一累计量标准点单位；旧 `usage_points` 只映射已有模型配置的累计量实际测点。
- 主瞬时量、过程瞬时量和能源计量/供电能力始终从目标配置闭包保留。旧请求缺少形成合法闭包所需的主瞬时量、尝试增加没有主瞬时量映射的新模型或出现其他不可无损情况时，返回 `FAILED_PRECONDITION`，不能自动猜测。
- 旧响应的 `unit` 和 `usage_points` 从 pending overlay（不存在时为 active）派生。
- 旧 `Get/ListEnergyTypes` 在存在 pending 时返回 pending overlay，否则返回 active；内部调度和核算始终只读 active 或任务快照。
- 旧 `PricePlan.name` 映射稳定 `price_plan_id`，旧能源类型名称映射 `energy_type_id`；同一能源类型允许存在多个方案。
- 旧价格方案创建/更新转换为完整聚合创建/替换。旧 `UpdatePricePlan` 只允许修改方案名称和未来版本；修改、删除或漏传已生效版本时返回 `FAILED_PRECONDITION`。
- 旧 `PriceSection.valid_from` 必须对应部署业务时区某日 `00:00`；`valid_to` 由新服务派生且客户端传值不是权威输入。
- 旧 `double` 单价规范化为十进制定点数；超过两位小数时拒绝。只接受 `FlatRate/TimeOfUse` 及全天同规则，阶梯、分档和按星期差异不能静默降级。
- 旧查询从新聚合派生当前兼容视图，返回派生 `valid_to` 和旧字段可表达的原始价格规则。
- 旧 `BatchUpdateBillings` 的 names 规范化为 `meter_id`，并转换为当前方案应用表具完整集合的原子替换；绑定修订和历史成本快照不删除。
- 旧 `Measurement` 的显式 `by_meters + usage_point` 转换为规范表具集合和测点映射。
- 旧 `by_aggregate` 只接受当前已有的 `sum(...)` 设备列表语法，并转换为新任务快照的成员集合。
- 当前已返回 `Unimplemented` 的批量导入/导出接口保留相同的错误行为，直到明确改变契约。
- 旧 `DeleteEnergyType/DeletePricePlan` protobuf 请求没有 etag 字段。兼容适配器按旧名称解析稳定 ID 并读取当时的当前 etag，再把该 etag 作为 `expected_etag` 调用同一领域删除命令；领域事务仍执行 etag 比较和当前引用校验。该适配只能防止适配器读取后发生的并发覆盖，不能识别旧客户端是否基于陈旧页面发起删除。



### 11.6 已确认的不可无损语义

以下限制必须写入兼容说明并通过测试固定，不能静默处理：

1. **旧数据可见性**：不迁移旧数据，切换前旧数据不保证通过新实现的旧接口继续可见；兼容范围覆盖切换后由旧接口产生和读取的新数据。
2. **active/pending 双态表达**：旧 protobuf 只有一个配置对象，因此旧查询使用 pending overlay；无法通过旧字段同时表达 active 和 pending。
3. **聚合表达式**：无法解析的表达式、无效设备、组件折叠或成员歧义必须显式返回既定错误，不能降级为空集合或所属设备。
4. **隐式测点推导**：候选不唯一时返回明确错误，不能依赖不稳定的“第一个匹配”。
5. **数值精度**：旧 `double` 字段按 protobuf 能力读写；新接口产生的超出 double 表达能力的精度通过旧接口读取时会按 double 规则表示。
6. **删除语义**：旧删除映射为逻辑 tombstone，使旧查询返回 `NotFound`，但历史事实和价格快照保留。
7. **能源配置表达能力**：旧 `EnergyType` 只能表达一个单位和累计量 `usage_points`，不能表达主/过程瞬时量或支持能力；旧写入只能执行受限无损合并，不能创建不完整配置或触发隐式推导。
8. **技术状态可见性**：产品查询返回 pending overlay，旧协议和已确认原型都不表达 active/pending、promotion、manifest 或降级状态；这些状态只对内部运维可见。
9. **价格日期精度**：新领域只接受业务日期；旧时间戳不是当地 `00:00` 时返回错误，不能静默截断。
10. **价格表达能力**：阶梯价、分档价、按星期变化、超过两位小数或不能编译为全天 48 槽的旧价格方案不属于需求 4，兼容写入明确失败。
11. **绑定历史**：旧接口只能表达当前应用集合。新领域同样以当前绑定决定后续历史重算；历史绑定仅作审计，不作为重新计价依据。
12. **删除并发意图**：新接口删除能源配置或计费方案必须提供 `expected_etag`，可以拒绝基于陈旧读取的删除；旧删除契约无法携带客户端读取到的 etag，只能由适配器读取当前 etag 后调用领域命令，因此不能无损表达陈旧页面删除检测。



## 12. 一致性、错误和恢复



### 12.1 领域不变量

- `energy_type_id` 由服务生成 UUID，创建后永久不变；当前数据库实例是隔离边界。
- 能源配置以完整聚合保存和提升，不存在部分 active 或部分 pending。
- `standard_point_id` 由服务在标准点首次持久化时生成 UUID，跨 generation 保持不变；请求内 `client_key` 或等价临时引用不能成为持久化身份，也不能出现在 manifest、任务快照、事实或审计中。
- 能源配置和计费方案的替换、删除都必须比较客户端最近读取时取得的 `expected_etag`；etag 不一致时不能写入 generation、tombstone、审计或 outbox，删除事务随后仍须校验当前有效引用。
- 每个能源类型恰好一个唯一累计量，至少一个瞬时量且恰好一个主瞬时量。
- 能源计量能力必须具备累计量和主瞬时量映射；只支持供电不要求测点映射。
- 实际测点必须来自模型继承闭包、与标准点同量纲，且不能重复映射到多个标准角色。
- 取消模型能力不能产生悬挂实例关系，也不能自动级联删除关系。
- 每个槽的配置 generation、XTwin 读取时点和有效表具 manifest 都被冻结；配置或实例变化只影响后续槽。
- 规范 `meter_id`、结构化遥测身份和 `metrics.samples.__name__` 是三个不同层次的身份；物理序列键只能由 manifest 构建阶段的版本化 `TelemetrySeriesResolver` 产生。
- 每个已配置测点的 `TelemetrySourceRef`、`SourceKind`、`SeriesKey`、`SeriesKeyCodec` 和来源指纹都必须随 manifest 冻结；worker 及其重试只能读取冻结结果，不能重新拼接或解析物理序列键。
- `XTWIN_TELEMETRY_V1` 中，设备序列为 `TwinName + "_" + ModelTelemetryName`；组件序列把标准三段式模型测点最后一段 `0` 替换为实例 `ComponentName` 后再加 `TwinName` 前缀。
- 同一能源类型、同一 manifest 中，不同表具的唯一累计量不能引用相同的 `SourceKind + SeriesKey`。
- 数据库实例只有一个经校验且持久化的业务时区；存在事实后不能通过部署配置静默更换。
- 每个预期 `FactKey` 都有有效当前投影行，即使数值为空。
- 用量只等于当前槽尾值减紧邻上一槽尾值，不跨槽借值、估算或补零。
- 自动事实状态只允许 `NORMAL/MISSING/BAD`；有效当前投影另允许 `MANUAL_CORRECTED`，任务失败不是事实状态。
- `MISSING/BAD` 的用量和成本为 `null`；合法零用量保持 `NORMAL`。
- 时、日、月汇总只允许 `COMPLETE/ABNORMAL`，并且只能从半小时当前投影重建。
- 已冻结任务不受后续配置、XTwin 或映射变化影响。
- `SET` 覆盖并锁定自动值，自动重算只产生候选和冲突；`CLEAR` 解除锁并显示当前最新首选自动版本。
- 低版本意图和低版本写入不能覆盖高版本结果。
- `price_plan_id` 由服务生成且永久不变；方案的 `energy_type_id` 创建后不可修改，同一能源类型允许多个方案。
- 已生效价格版本不可修改或删除；每个版本必须编译为连续、无重叠且覆盖全天的 48 个半小时价格槽。
- 同一 `meter_id + energy_type_id` 最多一个当前方案；后续历史重算使用任务提交时冻结的当前绑定，绑定修订只作审计。
- 成本必须指向明确的用量有效版本、绑定版本和完整价格快照。
- 成本业务状态只允许 `COMPUTED/NO_USAGE/NO_PRICE_PLAN/NO_PRICE_VERSION`；配置损坏和执行异常是任务失败，不是成本事实状态。
- `NO_USAGE/NO_PRICE_PLAN/NO_PRICE_VERSION` 不转换为零；只有零单价产生状态为 `COMPUTED` 的零成本。
- 半小时成本以高精度保存，汇总后才按 `ROUND_HALF_UP` 舍入为人民币 2 位小数。
- 成本汇总完整性只允许 `COMPLETE/PARTIAL/MISSING` 且独立于用量完整性；`PARTIAL` 返回部分成本和缺失计数，`MISSING` 的金额为 `null`。
- 任意汇总当前投影都可以从权威事实重建。
- 同一事件时间重复记录不定义确定性决胜规则；系统不能假称该边界下的结果可重复。
- 同一 shard 同时只有一个有效 `lease_epoch`；过期持有者不能续租、更新检查点或改变任务终态。
- shard 技术重试复用原快照、业务意图版本、输出版本和 `write_id`；尝试耗尽后进入 `FAILED`，不能伪造事实状态或立即循环创建新 operation。



### 12.2 错误映射

| gRPC code | 需求 1、2、4 场景 |
| --- | --- |
| `INVALID_ARGUMENT` | 配置字段、角色数量、单位、映射、日期范围、人工新用量、价格类型/时段/精度或应用集合不合法 |
| `ALREADY_EXISTS` | 能源类型名称、计费方案名称或同一方案生效日期重复 |
| `ABORTED` | 能源配置或计费方案替换/删除的聚合 etag、应用集合 etag、`base_effective_revision`、表具占用或并发保存冲突 |
| `FAILED_PRECONDITION` | 能力仍被实例关系引用、修改/删除已生效价格版本、删除仍有当前应用的方案、旧协议无法无损写入或运行身份不允许继续核算 |
| `PERMISSION_DENIED` | 删除内置能源、人工修正、范围重算、计费方案或应用关系操作权限不足 |
| `NOT_FOUND` | 能源类型、模型、标准点、遥测点、表具、半小时事实、计费方案或价格版本不存在 |
| `UNAVAILABLE` | XTwin、单位服务或任务基础设施暂时不可用 |

错误 detail 固定返回字段路径、资源引用、冲突项、受影响实例/关系和稳定 reason。兼容层固定参数错误、资源不存在、重复资源、绑定冲突、配置无效、权限错误、内部故障和未实现行为的 gRPC code、HTTP status 和 detail；新领域错误不能直接泄漏成不稳定的内部文本。

### 12.3 Reconciliation

reconciliation 检查：

- 部署业务时区是否与 `billing_runtime_identity` 一致。
- active/pending/promoting generation 指针及 active manifest 是否一致。
- manifest 的配置 generation、XTwin 读取时点、成员和映射指纹是否完整；每个已配置测点的结构化来源、`SourceKind`、`SeriesKey`、codec 和来源指纹是否一致，不同表具的唯一累计量是否发生物理序列冲突。
- promotion 失败候选是否留下不可推进状态，被任务引用的 generation/manifest 是否被错误回收。
- 关系库版本预留与 ClickHouse 写入是否一致。
- 每个已封口预期 `FactKey` 是否存在当前投影；缺行作为技术故障修复，不写伪造质量状态。
- 当前投影是否引用最高有效自动版本或最新人工 `SET`，`CLEAR` 后是否正确回落。
- 人工冲突标记是否匹配最新自动候选和人工值。
- 汇总质量计数、合计和 `COMPLETE/ABNORMAL` 是否可由当前事实重建。
- 计费方案版本、价格规则和 48 槽编译日历的指纹是否一致，已生效版本是否发生非法变更。
- 当前表具绑定唯一性、方案应用计数、绑定修订和应用审计是否一致。
- 成本当前投影是否匹配最高有效成本意图、有效用量版本、冻结绑定版本和完整价格快照。
- 成本汇总的高精度金额、分类金额、状态计数和 `COMPLETE/PARTIAL/MISSING` 是否可由成本当前投影重建。
- `RUNNING` shard 的租约是否过期，重新领取后 `lease_epoch` 是否单调且旧持有者写入被拒绝。
- 重试 shard 是否复用原快照、意图版本、输出版本和 `write_id`，尝试耗尽的 `FAILED` shard 是否停止自动领取。
- redrive operation 是否只复制原失败分片并正确关联 `root_operation_id/redrive_generation`，与产生新业务意图的重新计算严格区分。
- outbox 是否存在无法推进的中间状态。



## 13. 测试与验收



### 13.1 测试层次

- 单元测试：配置聚合不变量、单位换算、继承匹配、URI 编解码、设备/组件物理序列编码、来源指纹、兼容合并、错误映射、槽边界、尾样本选择、质量原因、差值、汇总、价格匹配、舍入和状态转换。
- 性质测试：generation 单调、重复请求幂等、保存/提升乱序不产生部分 active、重复消息及不存在同事件时间冲突重复时的相同快照重试结果稳定。
- 协议契约测试：旧 protobuf、HTTP 路径、序列化、错误状态和 detail。
- 集成测试：关系库配置事务、etag、指针切换、XTwin 模型继承和组件实例、ClickHouse、任务分片、租约抢占与续租、事实投影、成本和汇总重建。
- 故障注入：配置 promotion、候选 manifest、租约过期和旧 worker 恢复、跨库写入、确认、outbox、退避重试和尝试耗尽边界中断。
- 端到端测试：从 XTwin 配置到 Reader 的完整需求 1、2、4 链路。



### 13.2 必测场景

- 服务生成且永久保持 `energy_type_id`；配置、事实键、事件和存储不引入运行时生成的 `project_id`。
- 能源类型名称唯一且不可修改；标准角色数量、名称、单位和模型映射不合法时整体拒绝。
- 新增标准点由服务生成正式 `standard_point_id`；同请求内新增标准点及其映射通过临时引用正确解析，客户端指定正式 ID、重复/悬空临时引用整体拒绝；相同 request ID 重试返回相同正式 ID。
- 能源计量缺少累计量或主瞬时量时拒绝；只支持供电时无需映射；过程瞬时量不影响表具身份。
- 同量纲单位换算正确，累计量非零 offset 和不同量纲映射被拒绝。
- 父模型配置覆盖继承实际测点的子模型，父子重复配置不能形成歧义。
- 设备和组件实例生成规范 meter ID；实例/组件增删、模型变化和组件改名在下一槽 manifest 中正确体现。
- 设备 `TwinName=0_103`、模型测点 `1_123_0` 按 `XTWIN_TELEMETRY_V1` 解析为 `SeriesKey=0_103_1_123_0`；组件 `TwinName=0_1`、`ComponentName=1#1`、模型测点 `1_1_0` 解析为 `ResolvedTelemetryName=1_1_1#1` 和 `SeriesKey=0_1_1_1_1#1`。
- 空 TwinName、组件缺失名称、无效模型测点名和不能由当前 codec 编码的组件测点使候选 manifest 整体失败并返回稳定 reason，不发布半套清单。
- 不同表具的唯一累计量解析到相同 `SourceKind + SeriesKey` 时以 `DUPLICATE_SOURCE_SERIES` 拒绝候选 manifest；同一表具内同一实际测点重复映射多个标准角色仍由配置聚合校验拒绝。
- worker 只使用任务快照冻结的 `SeriesKey` 查询 `metrics.samples`；Twin/组件后续变化、resolver 代码或 codec 升级不能改变已冻结任务及历史范围重算的数据来源。
- 单个实例无遥测数据仍进入 manifest；配置引用的模型测点失效时保留上一完整 manifest 并产生技术告警。
- 新能源类型及其配置闭包创建、pending 修改、promoting 并发保存和边界提升。
- active 任务冻结后修改 pending，当前槽和下一槽使用不同快照。
- promotion 失败保持旧 active/manifest；取消能力存在实例关系时保存失败且不自动删除关系。
- 能源配置删除携带当前 etag 时仍执行引用校验；携带陈旧 etag 时返回 `ABORTED/ETAG_MISMATCH`，不写 tombstone、审计或 outbox。
- 旧 Update 只无损修改累计量字段，缺少主瞬时量或增加新模型时返回 `FAILED_PRECONDITION`。
- 内置电、水、冷量幂等初始化；电的 9 个默认模型、测点和能力与需求原型一致。
- 产品 Get/List 返回 pending overlay，核算只读 active；产品原型不增加技术状态。
- pending 尚未提升时进行范围重算，重算使用 pending。
- 业务时区配置合法性、UTC 转换、自然日/月边界和夏令时周期的动态 `expected_count`。
- 当前无权威采集周期时 `W = 10m`，首次核算只在 `slot_end + 10m` 之后执行。
- 槽边界样本归属、尾值窗口、缺少上一/当前/两个尾值、无效尾值、累计量回退和严格超限分别产生约定状态、原因和空值。
- 原始零值不被过滤、相等尾值形成 `NORMAL` 零用量；旧中位数修复、复位推断和跨槽借值不进入结果。
- 同一事件时间重复记录不对具体胜出值和跨次稳定性做断言；没有冲突重复时结果可重复且幂等。
- 首次启用、重新启用和累计量映射切换后的首槽只建立基线并产生 `MISSING_PREVIOUS_TAIL`。
- 坏值槽不参与汇总，但数值有效的当前尾值仍可作为下一槽基线；缺失尾值正确影响相邻槽。
- `NORMAL/MANUAL_CORRECTED` 参与时日月汇总，`MISSING/BAD` 排除；人工修正但无无值槽时汇总仍为 `COMPLETE`。
- 半小时状态固定四类，时日月只返回 `COMPLETE/ABNORMAL`；“暂无数据”和“加载中”不成为事实状态。
- 每日回扫最近 3 个完整自然日，晚到样本通过输入指纹变化修正当前槽和后一槽，窗口外由范围重算处理。
- `SET -> SET -> CLEAR` 的追加版本链、乐观并发、幂等、冲突标记和回落到当前最新自动结果。
- 定时、修复、范围重算并发时按 `intent_revision` 决定结果。
- 范围重算按业务日期、首槽额外基线和末尾后一槽执行，按自然日分片重试，同时完成用量、汇总、历史价格成本并保护人工锁。
- 正常半小时任务按相同 `ready_at` 合并表具批次；单表范围重算只创建一个 operation 和快照，按业务自然日拆分可并行、可独立重试的 shard，夏令时日期使用实际槽数。
- 两个 worker 并发领取同一 shard 时只有一个租约成功；续租使用数据库时间，租约过期重新领取后旧 `lease_epoch` 不能更新检查点、成功或失败状态。
- `MISSING/BAD` 作为成功业务事实不重试；`RETRYABLE/PERMANENT/UNCERTAIN_COMMIT` 分别按退避、立即失败和 `write_id` reconciliation 处理。
- `SCHEDULED/REPAIR` 8 次、`BULK` 5 次、`MAINTENANCE` 3 次重试上限及指数退避带抖动生效；尝试耗尽进入 `FAILED` 并停止自动领取。
- 技术 redrive 仅复制失败 shard，复用原快照、意图版本、输出版本和 `write_id`；显式重新计算创建新快照和更高业务意图版本。
- 半小时/时/日/月 Reader 的分页、下钻、趋势断点、成本“—”、完整范围汇总及“暂无数据”和 `MISSING` 区分。
- 计费方案 ID 稳定、名称唯一、能源类型创建后不可修改；同一能源类型可创建多个方案，删除有当前应用的方案被拒绝。
- 计费方案删除携带当前 etag 时仍校验当前应用集合为空；携带陈旧 etag 时返回 `ABORTED/ETAG_MISMATCH`，不写 tombstone、审计或 outbox。
- `FLAT/TOU_3/TOU_4` 分别编译为全天 48 槽；验证 `[start,end)` 边界、半小时刻度、连续覆盖、无重叠、必需类别、零价和最多两位小数，并拒绝阶梯、分档及星期差异。
- 第一条价格版本允许历史、当天或未来任意业务日期；已生效日程不可修改/删除，未来编辑创建替代修订并保留旧审计，移动或删除未来日程后派生有效区间随相邻版本正确变化。
- 计费方案候选包含具备唯一累计量映射的设备和组件表具，不依赖计量对象拓扑；关键字、物模型、分页、当前选中和其他方案占用置灰正确。
- 应用完整集合替换在 etag、表具资格或占用冲突时整体失败；成功时当前绑定、应用计数、修订、审计和 outbox 原子一致，`CLEAR` 审计保留。
- A 改绑 B 不立即改写既有成本且不影响已冻结任务；之后重算历史范围冻结 B，并按历史业务日期选择 B 的版本，B 无适用版本时为 `NO_PRICE_VERSION`，不回退 A 或 B 的第一条版本。
- 新建未来价格版本、应用、改绑和解绑不自动重算历史；无绑定时新成本为 `NO_PRICE_PLAN`，后续形成绑定也只影响新任务或显式范围重算。
- 正常核算、晚到修复、人工 `SET/CLEAR` 和范围重算都生成单调成本意图；成本阶段失败可独立重试，不重新生成或改写已经提交的用量事实。
- 半小时成本状态只出现 `COMPUTED/NO_USAGE/NO_PRICE_PLAN/NO_PRICE_VERSION`；零价为 `COMPUTED` 零成本，配置损坏和执行异常只形成任务失败。
- 成本汇总分别覆盖 `COMPLETE/PARTIAL/MISSING`；`PARTIAL` 返回部分成本和各缺失计数，`MISSING` 返回空金额，且不受用量汇总状态替代。
- 半小时成本不提前舍入，高精度分类汇总后以 `ROUND_HALF_UP` 输出人民币 2 位小数；允许已展示半小时金额之和与已展示汇总金额存在舍入差。
- 旧价格方案及应用接口在可无损范围内映射新聚合；修改已生效版本、非当地零点、超过两位小数及旧阶梯/星期规则明确失败，不恢复一能源一方案、首段回退或 `(start,end]` 行为。
- 旧能源类型和计费方案删除由兼容适配器读取当前 etag 后进入同一领域删除命令；覆盖其可防止的请求内竞争，并通过契约测试固定其不能检测旧客户端陈旧页面删除的限制。
- 删除汇总投影后从权威事实完整重建。
- 同一逻辑请求分别通过旧接口和新接口提交，领域结果、事实版本和投影结果等价。

运行监控记录配置 promotion 延迟和失败次数、manifest 构建耗时和成员数、降级能源类型、任务积压、投影滞后、结构化失败原因、outbox 和 reconciliation 异常。这些观测项不进入产品原型；本设计不为其规定阈值或上线承诺。

## 14. 上线切换

1. 创建全新的关系库控制面和 ClickHouse 事实面结构。
2. 写入并校验新的 active/pending 配置、价格方案和绑定。
3. 停止旧调度器，确认旧链路不再写入事实。
4. 在约定的半小时边界启用新调度器。
5. 所有调用方切换到旧协议兼容适配器或新 Reader/API；两者都进入同一新业务服务。
6. 新链路从切换边界开始产生事实，不迁移旧数据、不双写。
7. 存量配置和事实迁移不属于当前技术方案；未来通过独立脚本完成，并按新聚合规则校验后写入。
