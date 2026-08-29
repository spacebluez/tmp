# 表具能效计费前端 API 设计

## 1. 文档定位

本文定义需求 1、2、4 的前端 HTTP/JSON 对接契约，覆盖：

- 能源配置、标准测点、物模型能力和实际测点映射。
- 计费方案、价格版本和应用表具集合。
- 表具清单、半小时时日月明细、范围汇总和趋势。
- 人工 `SET/CLEAR`、修正审计和异步范围重算。
- 统一错误结构及旧接口到新领域服务的兼容映射。

需求和内部实现基线为：

- `AI节能/index.html` 及其能源配置、计费方案、表具能耗明细原型。
- `doc/能效2.5/2026-08-27-meter-energy-accounting-billing-design.md`。

新页面只调用本文定义的新领域接口。旧 `/xtwin/v3/energy/v1/...` 接口继续服务已有调用方，但不作为新页面的主要接口。

本文不定义 protobuf 字段编号，不暴露 promotion、manifest、reconciliation、任务租约、内容指纹等内部技术状态。

## 2. 通用约定

### 2.1 路径和消息格式

- 新接口统一前缀：`/xtwin/v3/energy/v2`。
- JSON 字段使用 `lowerCamelCase`。
- 创建、获取和替换成功时直接返回资源，不增加通用 `data` 包装。
- 列表返回资源数组、`nextPageToken` 和原型需要时的 `totalSize`。
- 前端不能直接访问关系库、ClickHouse 或内部 Reader。

### 2.2 身份

- `energyTypeId`、`pricePlanId`、`standardPointId` 和价格修订 ID 均为稳定 UUID。
- `energyTypeId`、`pricePlanId` 和 `standardPointId` 由服务生成。
- `meterId` 使用规范 URI：

```text
xtwin://twins/{percent-encoded-twin-id}
xtwin://twins/{percent-encoded-twin-id}/components/{percent-encoded-component-name}
```

`meterId` 包含 `/`，只能通过 query 或 JSON body 传递，不能作为普通 URL 路径段。

### 2.3 数值和空值

- 用量、尾值、单价、成本、半小时增量上限及折算系数使用十进制字符串。
- 合法零值返回 `"0"` 或等价补零形式，不能返回 `null`。
- `null` 表示没有业务值，前端按页面语义显示“—”或趋势断点。
- 前端不能根据金额是否为零推导成本状态。

### 2.4 时间

- 业务日期使用 `YYYY-MM-DD`。
- 查询请求的 `rangeStart/rangeEnd` 使用不带时区的本地时间，服务按部署配置 `energy_billing.business_timezone` 解释，区间为左闭右开。
- 查询结果中的周期起止使用带业务时区偏移的 RFC 3339。
- 审计和创建时间使用 UTC RFC 3339。
- 范围重算的 `startDate/endDate` 均包含在目标范围内。

### 2.5 分页和排序

- 首次请求的 `pageToken` 为空。
- 后续请求原样传回 `nextPageToken`。
- `pageToken` 是不透明稳定游标，前端不能解析或构造。
- 不能使用易受数据更新影响的页码偏移替代稳定游标。

### 2.6 并发和幂等

- 能源配置及计费方案替换携带资源 `etag`。
- 能源配置及计费方案删除通过 query 携带最近读取的 `etag`。
- `etag` 是不透明字符串，放入 query 时必须按 URL query value 编码，前端不能解析或改写。
- 应用表具集合替换携带 `applicationSetEtag`。
- 人工修正携带 `baseEffectiveRevision`。
- 能源配置保存、应用集合替换、人工修正和范围重算按各自契约携带 `requestId`。
- 相同 `requestId` 重试必须返回首次请求的相同业务结果。

## 3. 接口总览

### 3.1 能源配置

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| `GET` | `/energy-configurations` | 能源类型列表和稳定分页 |
| `POST` | `/energy-configurations` | 创建完整能源配置聚合 |
| `GET` | `/energy-configurations/{energyTypeId}` | 获取 pending overlay |
| `PUT` | `/energy-configurations/{energyTypeId}` | 完整替换能源配置聚合 |
| `DELETE` | `/energy-configurations/{energyTypeId}?etag=...` | 删除自定义能源 |
| `GET` | `/model-candidates` | 获取物模型候选 |
| `GET` | `/telemetry-candidates` | 获取实际测点候选 |

单位下拉继续使用平台已有单位服务，不新增能源领域单位候选接口。

### 3.2 计费方案和应用表具

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| `GET` | `/price-plans` | 计费方案列表 |
| `POST` | `/price-plans` | 创建完整计费方案聚合 |
| `GET` | `/price-plans/{pricePlanId}` | 获取方案和当前版本日程 |
| `PUT` | `/price-plans/{pricePlanId}` | 完整替换方案及未来版本 |
| `DELETE` | `/price-plans/{pricePlanId}?etag=...` | 逻辑删除方案 |
| `GET` | `/price-plans/{pricePlanId}/meter-candidates` | 获取应用表具候选 |
| `GET` | `/price-plans/{pricePlanId}/applications` | 获取当前应用集合 |
| `PUT` | `/price-plans/{pricePlanId}/applications` | 原子替换当前应用集合 |

### 3.3 表具能耗明细

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| `GET` | `/energy-meters` | 获取表具清单 |
| `POST` | `/meter-energy-details:query` | 查询范围汇总、趋势和分页明细 |
| `POST` | `/manual-usage-corrections:set` | 设置人工修正 |
| `POST` | `/manual-usage-corrections:clear` | 解除人工锁 |
| `GET` | `/manual-usage-corrections/{manualRevisionId}` | 获取完整人工修订链 |
| `POST` | `/range-recomputations` | 创建异步范围重算 |
| `GET` | `/range-recomputations/{operationId}` | 查询范围重算进度和结果 |

## 4. 公共枚举

### 4.1 能源配置

```text
MeasurementKind: PRIMARY_ENERGY | WORKING_FLUID
StandardPointCategory: CUMULATIVE | INSTANTANEOUS
StandardPointRole: UNIQUE_CUMULATIVE | PRIMARY_INSTANTANEOUS | PROCESS_INSTANTANEOUS
EnergyCapability: METERING | SUPPLY
```

### 4.2 用量和汇总

```text
Grain: HALF_HOUR | HOUR | DAY | MONTH
UsageStatus: NORMAL | MISSING | BAD | MANUAL_CORRECTED
UsageCompleteness: COMPLETE | ABNORMAL
```

质量原因：

```text
MISSING_BOTH_TAILS
MISSING_CURRENT_TAIL
MISSING_PREVIOUS_TAIL
INVALID_CURRENT_TAIL
INVALID_PREVIOUS_TAIL
COUNTER_ROLLBACK
INCREMENT_LIMIT_EXCEEDED
```

### 4.3 价格和成本

```text
PricingType: FLAT | TOU_3 | TOU_4
PricingCategory: FLAT | SUPER_PEAK | PEAK | SHOULDER | OFF_PEAK
CostStatus: COMPUTED | NO_USAGE | NO_PRICE_PLAN | NO_PRICE_VERSION
CostCompleteness: COMPLETE | PARTIAL | MISSING
```

### 4.4 范围重算

```text
OperationStatus: PENDING | RUNNING | SUCCEEDED | FAILED | CANCELLED
OperationStage: QUEUED | USAGE_RECOMPUTE | USAGE_ROLLUP | COST_RECOMPUTE | COST_ROLLUP | COMPLETED
```

## 5. 能源配置接口

### 5.1 获取能源类型列表

```http
GET /xtwin/v3/energy/v2/energy-configurations?pageSize=10&pageToken=...
```

响应只包含原型列表列、资源身份、删除并发控制和内置标记：

```json
{
  "energyConfigurations": [{
    "energyTypeId": "f3e8b240-9b93-43f7-9b0d-62b0fda8e5a3",
    "displayName": "电",
    "cumulativeUnitDisplayName": "kWh",
    "primaryInstantUnitDisplayName": "kW",
    "measurementKind": "PRIMARY_ENERGY",
    "tceFactor": "0.997000",
    "co2Factor": "0.122900",
    "createdAt": "2026-07-01T02:30:00Z",
    "builtin": true,
    "etag": "BwW/3Y..."
  }],
  "nextPageToken": "...",
  "totalSize": 3
}
```

约束：

- 累计量单位从唯一累计量标准点派生。
- 实时值单位从主瞬时量标准点派生。
- 不返回 `permissions`、`canEdit`、`canDelete` 等行级权限字段。
- `builtin=true` 用于隐藏或禁用内置能源删除入口。
- 自定义能源删除是否满足引用约束由删除命令实时校验。
- 按 `createdAt` 倒序、`energyTypeId` 稳定排序；默认 `pageSize=10`。

### 5.2 创建能源配置

```http
POST /xtwin/v3/energy/v2/energy-configurations
Content-Type: application/json
```

```json
{
  "requestId": "59b34f18-652e-4234-a75d-cb2e2540f48e",
  "displayName": "电",
  "measurementKind": "PRIMARY_ENERGY",
  "halfHourIncrementLimit": "1500",
  "tceFactor": "0.997000",
  "co2Factor": "0.122900",
  "standardPoints": [{
    "clientKey": "cumulative",
    "category": "CUMULATIVE",
    "role": "UNIQUE_CUMULATIVE",
    "displayName": "累计电量",
    "unitId": "kWh"
  }, {
    "clientKey": "main-instant",
    "category": "INSTANTANEOUS",
    "role": "PRIMARY_INSTANTANEOUS",
    "displayName": "实时功率",
    "unitId": "kW"
  }],
  "modelConfigurations": [{
    "modelId": "dtmi:xbrother:meter:SinglePhase;1",
    "capabilities": ["METERING"],
    "mappings": [{
      "standardPointRef": {"clientKey": "cumulative"},
      "telemetryPointId": "1_1123_0"
    }, {
      "standardPointRef": {"clientKey": "main-instant"},
      "telemetryPointId": "1_627_0"
    }]
  }]
}
```

服务生成 `energyTypeId` 和正式 `standardPointId`，成功返回完整资源及 `etag`。响应不回显 `clientKey`。

### 5.3 获取能源配置

```http
GET /xtwin/v3/energy/v2/energy-configurations/{energyTypeId}
```

产品接口始终返回 pending overlay；没有 pending 时返回 active。响应不暴露该选择过程。

读取结构示例：

```json
{
  "energyTypeId": "f3e8b240-9b93-43f7-9b0d-62b0fda8e5a3",
  "displayName": "电",
  "measurementKind": "PRIMARY_ENERGY",
  "halfHourIncrementLimit": "1500",
  "tceFactor": "0.997000",
  "co2Factor": "0.122900",
  "standardPoints": [{
    "standardPointId": "8fb7e606-59a2-4df0-8c26-dd1e50ad41a7",
    "category": "CUMULATIVE",
    "role": "UNIQUE_CUMULATIVE",
    "displayName": "累计电量",
    "unitId": "kWh"
  }, {
    "standardPointId": "2529dbd2-2c13-43ec-ad2a-92b30f7acfde",
    "category": "INSTANTANEOUS",
    "role": "PRIMARY_INSTANTANEOUS",
    "displayName": "实时功率",
    "unitId": "kW"
  }],
  "modelConfigurations": [{
    "modelId": "dtmi:xbrother:meter:SinglePhase;1",
    "modelDisplayName": "单相电量仪",
    "capabilities": ["METERING"],
    "mappings": [{
      "standardPointId": "8fb7e606-59a2-4df0-8c26-dd1e50ad41a7",
      "telemetryPointId": "1_1123_0",
      "telemetryPointDisplayName": "有功电能",
      "telemetryPointUnitDisplayName": "kWh"
    }, {
      "standardPointId": "2529dbd2-2c13-43ec-ad2a-92b30f7acfde",
      "telemetryPointId": "1_627_0",
      "telemetryPointDisplayName": "有功功率-总",
      "telemetryPointUnitDisplayName": "kW"
    }]
  }],
  "builtin": true,
  "etag": "BwW/3Y..."
}
```

读取响应不返回 generation、manifest、单位换算参数、内容指纹或 active/pending/promotion 状态。

### 5.4 替换能源配置

```http
PUT /xtwin/v3/energy/v2/energy-configurations/{energyTypeId}
Content-Type: application/json
```

请求使用创建接口的完整聚合结构，并增加：

```json
{
  "requestId": "7e515afd-81c2-4351-834a-4f31f48ee232",
  "etag": "BwW/3Y..."
}
```

标准点引用规则：

- `displayName` 必须与读取结果相同；能源类型名称创建后不可修改。
- 新标准点必须携带请求内唯一 `clientKey`，不能携带 `standardPointId`。
- 已有标准点必须原样回传 `standardPointId`，不能携带 `clientKey`。
- `standardPointRef` 必须且只能包含 `standardPointId` 或 `clientKey` 之一。
- 相同 `requestId` 重试返回首次分配的相同正式 ID。
- 标准点数组顺序作为展示顺序，前端不传内部 `sortOrder`。

已有映射示例：

```json
{
  "standardPointRef": {
    "standardPointId": "8fb7e606-59a2-4df0-8c26-dd1e50ad41a7"
  },
  "telemetryPointId": "1_1123_0"
}
```

保存按整个聚合原子校验，任一字段、模型、能力或映射失败都不产生部分 pending。

### 5.5 删除能源配置

```http
DELETE /xtwin/v3/energy/v2/energy-configurations/{energyTypeId}?etag={etag}
```

服务先比较 etag，再校验价格方案、模型能力关系和其他有效引用：

- etag 不一致：`ABORTED / ETAG_MISMATCH`。
- 内置能源：`PERMISSION_DENIED / BUILTIN_ENERGY_DELETE_FORBIDDEN`。
- 存在有效引用：`FAILED_PRECONDITION / ENERGY_CONFIGURATION_REFERENCED`，并返回结构化影响清单。
- 校验通过：写入待生效 tombstone，返回空响应。

删除成功返回 `204 No Content`。

### 5.6 物模型候选

```http
GET /xtwin/v3/energy/v2/model-candidates?query={query}
```

```json
{
  "models": [{
    "modelId": "dtmi:xbrother:meter:SinglePhase;1",
    "displayName": "单相电量仪",
    "parentModelId": null
  }]
}
```

返回资产/集合候选及父模型引用，前端按当前草稿所选集合执行父子互斥展示；服务保存时重新校验。

### 5.7 实际测点候选

```http
GET /xtwin/v3/energy/v2/telemetry-candidates
  ?modelId=dtmi%3Axbrother%3Ameter%3ASinglePhase%3B1
  &category=CUMULATIVE
  &unitId=kWh
```

```json
{
  "telemetryPoints": [{
    "telemetryPointId": "1_1123_0",
    "displayName": "有功电能",
    "unitId": "kWh",
    "unitDisplayName": "kWh",
    "sourceModelId": "dtmi:xbrother:meter:SinglePhase;1",
    "sourceModelDisplayName": "单相电量仪"
  }]
}
```

服务只返回分类和量纲兼容的当前模型或继承测点。前端按来源模型分组，保存时服务重新校验归属、继承、重复映射和单位换算。

## 6. 计费方案和应用表具接口

### 6.1 获取计费方案列表

```http
GET /xtwin/v3/energy/v2/price-plans
  ?query={方案名称}
  &modelId={实际物模型ID}
  &pageSize=10
  &pageToken=...
```

物模型筛选表示方案当前应用表具中至少一块表具的实际模型匹配。

```json
{
  "pricePlans": [{
    "pricePlanId": "8f5b0ac8-36a8-4db5-8a10-bff3527c8507",
    "energyDisplayName": "电",
    "displayName": "尖峰平谷电价 2026",
    "appliedMeterCount": 12,
    "createdAt": "2026-07-01T02:30:00Z",
    "etag": "BwW/9A..."
  }],
  "nextPageToken": "...",
  "totalSize": 1
}
```

不返回 `canEdit`、`canDelete` 或权限字段。删除条件由删除命令实时校验。
结果按 `createdAt` 倒序、`pricePlanId` 稳定排序；默认 `pageSize=10`。

### 6.2 计费方案资源

创建：

```http
POST /xtwin/v3/energy/v2/price-plans
```

获取：

```http
GET /xtwin/v3/energy/v2/price-plans/{pricePlanId}
```

替换：

```http
PUT /xtwin/v3/energy/v2/price-plans/{pricePlanId}
```

创建请求体是下方资源去掉 `pricePlanId`、`energyDisplayName`、`currency`、`unitDisplayName`、价格修订 ID、`editable`、`deletable` 和 `etag` 后的可写字段。替换请求体不携带路径中的 `pricePlanId`，保留已有价格修订 ID 和聚合 `etag`，并同样不携带其他响应只读字段。

响应资源示例：

```json
{
  "pricePlanId": "8f5b0ac8-36a8-4db5-8a10-bff3527c8507",
  "displayName": "尖峰平谷电价 2026",
  "energyTypeId": "f3e8b240-9b93-43f7-9b0d-62b0fda8e5a3",
  "energyDisplayName": "电",
  "currency": "CNY",
  "unitDisplayName": "kWh",
  "versions": [{
    "pricePlanRevisionId": "9d55267d-0abc-4b10-a5b2-c29cb2414f28",
    "effectiveDate": "2026-01-01",
    "pricingType": "TOU_4",
    "unitPrices": {
      "SUPER_PEAK": "1.350000",
      "PEAK": "1.050000",
      "SHOULDER": "0.680000",
      "OFF_PEAK": "0.320000"
    },
    "periods": [{
      "category": "OFF_PEAK",
      "startTime": "00:00",
      "endTime": "08:00"
    }, {
      "category": "PEAK",
      "startTime": "08:00",
      "endTime": "10:00"
    }, {
      "category": "SUPER_PEAK",
      "startTime": "10:00",
      "endTime": "12:00"
    }, {
      "category": "SHOULDER",
      "startTime": "12:00",
      "endTime": "24:00"
    }],
    "editable": false,
    "deletable": false
  }],
  "etag": "BwW/9A..."
}
```

统一单价版本：

```json
{
  "effectiveDate": "2026-09-01",
  "pricingType": "FLAT",
  "unitPrices": {"FLAT": "0.800000"},
  "periods": []
}
```

创建请求不传 `pricePlanId`、价格修订 ID、`etag`、`editable` 或 `deletable`。`currency` 固定为 `CNY`，单位由能源配置唯一累计量标准点派生，写请求不传。

完整替换规则：

- `energyTypeId` 必须原样回传且不可修改。
- 已生效版本必须携带原 `pricePlanRevisionId` 并原样回传。
- 新增未来版本不传修订 ID，由服务生成。
- 编辑未来版本时携带原修订 ID 和修改后的内容；服务创建新修订并返回新 ID，不修改原修订。
- 删除未来版本时从 `versions` 中移除。
- 删除或漏传已生效版本整体返回 `FAILED_PRECONDITION / PRICE_VERSION_LOCKED`。
- `editable/deletable` 是服务按部署业务日期返回的版本操作可用性，不是权限字段，写请求不得携带。
- `versions` 按 `effectiveDate` 倒序返回，与原型方案明细列表顺序一致。

价格规则：

- `effectiveDate` 是业务日期。
- 单价最多两位有效小数；响应可以按统一精度补零。
- 时段使用 `[startTime,endTime)`，只允许半小时刻度。
- `24:00` 只允许作为结束时间。
- 时段必须连续、无重叠、无空洞覆盖全天。
- 不支持阶梯、分档和按星期差异。

### 6.3 删除计费方案

```http
DELETE /xtwin/v3/energy/v2/price-plans/{pricePlanId}?etag={etag}
```

服务先比较 etag，再检查当前应用表具数：

- etag 不一致：`ABORTED / ETAG_MISMATCH`。
- 仍有当前应用：`FAILED_PRECONDITION / PRICE_PLAN_HAS_APPLICATIONS`。
- 校验通过：逻辑删除并返回空响应。

删除成功返回 `204 No Content`。

### 6.4 获取应用表具候选

```http
GET /xtwin/v3/energy/v2/price-plans/{pricePlanId}/meter-candidates
  ?query={关键字}
  &modelId={实际物模型ID}
  &pageSize=50
  &pageToken=...
```

```json
{
  "meters": [{
    "meterId": "xtwin://twins/0_708/components/branch-1",
    "displayName": "列头柜1-支路1",
    "actualModelId": "dtmi:xbrother:branch;1",
    "actualModelName": "交流输出支路",
    "selected": true,
    "occupiedByOtherPlan": false,
    "occupiedPlanName": null
  }],
  "nextPageToken": "...",
  "totalSize": 1
}
```

候选来自方案能源类型的当前有效表具 manifest。未绑定计量对象的表具仍可应用；已被其他方案占用的表具由前端置灰。

### 6.5 获取和替换当前应用集合

获取：

```http
GET /xtwin/v3/energy/v2/price-plans/{pricePlanId}/applications
```

```json
{
  "applicationSetEtag": "BwW/2Q...",
  "meters": [{
    "meterId": "xtwin://twins/0_708/components/branch-1",
    "displayName": "列头柜1-支路1",
    "actualModelId": "dtmi:xbrother:branch;1",
    "actualModelName": "交流输出支路"
  }]
}
```

替换：

```http
PUT /xtwin/v3/energy/v2/price-plans/{pricePlanId}/applications
```

```json
{
  "meterIds": ["xtwin://twins/0_708/components/branch-1"],
  "applicationSetEtag": "BwW/2Q...",
  "requestId": "5ce74124-c1fb-4d2b-b1d4-6c76758c5393"
}
```

`meterIds` 是当前方案目标完整集合。服务在一个事务中重新校验资格、占用和集合 etag；任一表具冲突则整体返回 `ABORTED`，不产生部分绑定。绑定成功不会自动重算历史成本。

## 7. 表具能耗明细接口

### 7.1 获取表具清单

```http
GET /xtwin/v3/energy/v2/energy-meters
  ?energyTypeId={energyTypeId}
  &spaceNodeId={spaceNodeId}
  &query={名称或编码}
  &rangeStart=2026-07-09T00:00:00
  &rangeEnd=2026-07-10T00:00:00
  &pageSize=20
  &pageToken=...
```

空间树由 XTwin 现有接口提供。服务端展开 `spaceNodeId` 的子节点，空间过滤和关键字搜索同时生效。

```json
{
  "meters": [{
    "meterId": "xtwin://twins/0_708/components/branch-1",
    "displayName": "列头柜1-支路1",
    "meterKind": "COMPONENT",
    "meterCode": "0_708",
    "spaceDisplayPath": "云谷园区 / A栋",
    "rangeCompleteness": "ABNORMAL"
  }],
  "nextPageToken": "...",
  "totalSize": 1
}
```

列表按异常优先、名称稳定排序，支持原型默认选中首个异常表具。响应不返回 manifest、generation 或任务状态。

### 7.2 查询表具明细

```http
POST /xtwin/v3/energy/v2/meter-energy-details:query
```

```json
{
  "meterId": "xtwin://twins/0_708/components/branch-1",
  "energyTypeId": "f3e8b240-9b93-43f7-9b0d-62b0fda8e5a3",
  "grain": "HALF_HOUR",
  "rangeStart": "2026-07-09T00:00:00",
  "rangeEnd": "2026-07-10T00:00:00",
  "pageSize": 20,
  "pageToken": ""
}
```

响应顶层：

```json
{
  "meter": {
    "displayName": "列头柜1-支路1",
    "meterKind": "COMPONENT",
    "meterCode": "0_708",
    "measurementObjectDisplayPath": "云谷园区 / A栋 / 列头柜1 / 支路1",
    "pricePlanId": "8f5b0ac8-36a8-4db5-8a10-bff3527c8507",
    "pricePlanDisplayName": "尖峰平谷电价 2026"
  },
  "usageUnitDisplayName": "kWh",
  "currency": "CNY",
  "summary": {},
  "trendPoints": [],
  "details": [],
  "nextPageToken": "...",
  "totalSize": 48
}
```

未绑定计量对象或未应用计费方案时，对应字段返回 `null`，由前端显示“未绑定计量对象”或“未应用计费方案”。

完整范围汇总：

```json
{
  "usageTotal": "1742.600000",
  "usageCompleteness": "ABNORMAL",
  "expectedCount": 48,
  "normalCount": 43,
  "manualCount": 1,
  "missingCount": 2,
  "badCount": 2,
  "costTotal": "1386.45",
  "costCompleteness": "PARTIAL",
  "computedCount": 44,
  "noUsageCount": 4,
  "noPricePlanCount": 0,
  "noPriceVersionCount": 0
}
```

`summary` 始终针对完整查询范围计算，不受 `details` 分页影响。

### 7.3 半小时明细行

```json
{
  "periodStart": "2026-07-09T12:00:00+08:00",
  "periodEnd": "2026-07-09T12:30:00+08:00",
  "previousTail": {
    "value": "160394.800000",
    "eventTime": "2026-07-09T11:59:24+08:00"
  },
  "currentTail": {
    "value": "160478.600000",
    "eventTime": "2026-07-09T12:29:17+08:00"
  },
  "usage": {
    "value": "82.000000",
    "status": "MANUAL_CORRECTED",
    "reasonCode": null,
    "effectiveRevision": "185",
    "manualRevisionId": "c627eef1-fc4c-4634-874e-e3197ee909d9",
    "manualConflict": true,
    "autoCandidateStatus": "NORMAL",
    "autoCandidateValue": "83.800000"
  },
  "cost": {
    "amount": "55.76",
    "currency": "CNY",
    "status": "COMPUTED",
    "pricePlanId": "8f5b0ac8-36a8-4db5-8a10-bff3527c8507",
    "pricePlanDisplayName": "尖峰平谷电价 2026",
    "pricePlanRevisionId": "9d55267d-0abc-4b10-a5b2-c29cb2414f28",
    "category": "SHOULDER",
    "unitPrice": "0.680000"
  }
}
```

规则：

- `MISSING/BAD` 的 `usage.value` 为 `null`，对应成本状态为 `NO_USAGE` 且金额为 `null`。
- 无当前绑定时成本状态为 `NO_PRICE_PLAN`。
- 有绑定但业务日期无适用版本时成本状态为 `NO_PRICE_VERSION`。
- `manualRevisionId` 用于打开审计弹窗。
- `effectiveRevision` 用于后续 `SET/CLEAR` 乐观并发。
- 自动候选字段只在人工锁场景返回；其他场景为 `null`。

### 7.4 时日月明细行

```json
{
  "periodStart": "2026-07-09T12:00:00+08:00",
  "periodEnd": "2026-07-09T13:00:00+08:00",
  "lastTail": {
    "value": "160556.900000",
    "eventTime": "2026-07-09T12:59:19+08:00"
  },
  "usage": {
    "total": "160.300000",
    "completeness": "COMPLETE",
    "expectedCount": 2,
    "normalCount": 1,
    "manualCount": 1,
    "missingCount": 0,
    "badCount": 0
  },
  "cost": {
    "total": "80.82",
    "completeness": "COMPLETE",
    "computedCount": 2,
    "noUsageCount": 0,
    "noPricePlanCount": 0,
    "noPriceVersionCount": 0,
    "categoryAmounts": {
      "FLAT": null,
      "SUPER_PEAK": null,
      "PEAK": null,
      "SHOULDER": "80.82",
      "OFF_PEAK": null
    }
  }
}
```

时、日、月不返回人工修订或自动候选，操作列通过当前行的 `periodStart/periodEnd` 下钻到下一粒度。`categoryAmounts` 返回当前价格类型各计价类别的归档金额；不适用或没有已计算金额的类别为 `null`，前端不能按缺失类别补零。

### 7.5 趋势点

```json
{
  "periodStart": "2026-07-09T09:00:00+08:00",
  "usageValue": null,
  "costValue": null
}
```

- 趋势点按时间升序返回完整查询范围。
- 分页明细按 `periodStart` 降序。
- 半小时 `MISSING/BAD` 返回空值形成断点，不能补零。

## 8. 人工修正和范围重算

### 8.1 设置人工修正

```http
POST /xtwin/v3/energy/v2/manual-usage-corrections:set
```

```json
{
  "meterId": "xtwin://twins/0_708/components/branch-1",
  "energyTypeId": "f3e8b240-9b93-43f7-9b0d-62b0fda8e5a3",
  "slotStart": "2026-07-09T12:00:00+08:00",
  "newUsage": "82.000000",
  "reason": "换表当刻补值",
  "baseEffectiveRevision": "184",
  "requestId": "4daf6813-5c4d-4d56-af2c-5ea6bdc6eebb"
}
```

`newUsage` 必须大于等于零。`reason` 可为 `null`。

### 8.2 清除人工修正

```http
POST /xtwin/v3/energy/v2/manual-usage-corrections:clear
```

```json
{
  "meterId": "xtwin://twins/0_708/components/branch-1",
  "energyTypeId": "f3e8b240-9b93-43f7-9b0d-62b0fda8e5a3",
  "slotStart": "2026-07-09T12:00:00+08:00",
  "baseEffectiveRevision": "185",
  "requestId": "77d09835-b5cd-45b1-bc9f-fde48ec2219c"
}
```

`CLEAR` 解除人工锁并回落到当前最新自动版本，不恢复首次修正前的旧值。

### 8.3 人工命令成功语义

`SET/CLEAR` 均返回 `200 OK`：

```json
{
  "manualRevisionId": "c627eef1-fc4c-4634-874e-e3197ee909d9",
  "action": "SET",
  "requestId": "4daf6813-5c4d-4d56-af2c-5ea6bdc6eebb",
  "acceptedAt": "2026-07-09T04:15:21Z"
}
```

成功表示：

1. `ManualUsageRevision` 已持久化。
2. `requestId` 幂等结果、审计和 outbox 已保存。
3. 后续投影任务已通过 outbox 可靠投递。
4. 服务立即返回上述成功响应。
5. 后台继续更新当前用量投影、时日月汇总和成本。

成功不表示当前用量、时日月汇总和成本已经全部刷新。响应不返回新的有效用量、成本、汇总或 `effectiveRevision`。

前端收到成功后关闭弹窗、提示提交成功并重新查询明细。查询行的 `manualRevisionId` 与命令响应一致时，表示人工修订已经进入当前用量投影；在此之前看到旧明细不代表命令失败，不增加新的事实状态或“投影中”字段。

### 8.4 获取人工修订链

```http
GET /xtwin/v3/energy/v2/manual-usage-corrections/{manualRevisionId}
```

```json
{
  "meterId": "xtwin://twins/0_708/components/branch-1",
  "energyTypeId": "f3e8b240-9b93-43f7-9b0d-62b0fda8e5a3",
  "slotStart": "2026-07-09T12:00:00+08:00",
  "revisions": [{
    "manualRevisionId": "c627eef1-fc4c-4634-874e-e3197ee909d9",
    "action": "SET",
    "previousStatus": "NORMAL",
    "previousUsage": "83.800000",
    "newUsage": "82.000000",
    "reason": "换表当刻补值",
    "operatorId": "user-1024",
    "operatorDisplayName": "王工",
    "operatedAt": "2026-07-09T04:15:21Z"
  }]
}
```

返回完整 `SET/CLEAR` 链。`CLEAR` 的 `newUsage` 为 `null`；空原因返回 `null`，前端显示“—”。

### 8.5 创建范围重算

```http
POST /xtwin/v3/energy/v2/range-recomputations
```

```json
{
  "meterId": "xtwin://twins/0_708/components/branch-1",
  "energyTypeId": "f3e8b240-9b93-43f7-9b0d-62b0fda8e5a3",
  "startDate": "2026-07-01",
  "endDate": "2026-07-09",
  "requestId": "c131f9ab-ced1-4b64-9250-d5d8a49f1c36"
}
```

返回 `202 Accepted`：

```json
{
  "operationId": "op-72bd...",
  "status": "PENDING",
  "stage": "QUEUED",
  "progressPercent": 0
}
```

一次 operation 依次完成用量、用量汇总、成本和成本汇总，不提供重算类型选择。

### 8.6 查询范围重算

```http
GET /xtwin/v3/energy/v2/range-recomputations/{operationId}
```

```json
{
  "operationId": "op-72bd...",
  "status": "RUNNING",
  "stage": "COST_RECOMPUTE",
  "progressPercent": 76,
  "result": null,
  "failure": null
}
```

成功时 `result` 返回最终处理结果：

```json
{
  "successSlotCount": 320,
  "failedSlotCount": 0,
  "missingCount": 4,
  "badCount": 2,
  "manualLockedCount": 1,
  "manualConflictCount": 1
}
```

失败时 `failure` 返回失败阶段和可展示原因：

```json
{
  "stage": "COST_RECOMPUTE",
  "code": "UNAVAILABLE",
  "message": "成本计算依赖暂时不可用"
}
```

`RUNNING/PENDING` 时 `result` 和 `failure` 均为 `null`；`SUCCEEDED` 时只有 `result` 非空；`FAILED` 时只有 `failure` 非空。只有用量、两类汇总和成本阶段全部完成才返回 `SUCCEEDED`。人工锁定槽仍生成自动候选，但不覆盖人工值。

## 9. 错误契约

### 9.1 错误结构

沿用 `google.rpc.Status`：

```json
{
  "code": 10,
  "message": "配置已被其他操作修改，请刷新后重试",
  "details": [{
    "@type": "type.googleapis.com/xtwin.api.insight.energy.v2.ErrorDetail",
    "reason": "ETAG_MISMATCH",
    "fieldViolations": [],
    "resourceRefs": [{
      "resourceType": "PRICE_PLAN",
      "resourceId": "8f5b0ac8-36a8-4db5-8a10-bff3527c8507",
      "displayName": "尖峰平谷电价 2026"
    }],
    "conflicts": [{
      "field": "etag",
      "resourceId": "8f5b0ac8-36a8-4db5-8a10-bff3527c8507"
    }],
    "impacts": []
  }]
}
```

- `message` 是可直接展示的本地化文本，前端不能据此判断业务分支。
- `reason` 是稳定机器码。
- `fieldViolations` 包含 `field` 和 `description`，字段路径按请求 JSON 表达。
- `resourceRefs` 返回相关资源引用。
- `conflicts` 返回并发、占用和重复配置冲突。
- `impacts` 返回删除或取消能力时受影响的实例和关系；每项包含 `resourceType`、`resourceId`、`displayName` 和稳定的 `relation`。

### 9.2 HTTP 和 gRPC 映射

| HTTP | gRPC | 场景 |
| --- | --- | --- |
| `400` | `INVALID_ARGUMENT` | 字段、单位、映射、日期、价格或应用集合不合法 |
| `409` | `ALREADY_EXISTS` | 能源名称、方案名称或生效日期重复 |
| `409` | `ABORTED` | 资源/集合 etag、有效版本或表具占用冲突 |
| `400` | `FAILED_PRECONDITION` | 资源仍被引用、已生效版本被修改或旧协议无法无损表达 |
| `403` | `PERMISSION_DENIED` | 平台权限不足或删除内置能源 |
| `404` | `NOT_FOUND` | 资源、事实或 operation 不存在 |
| `503` | `UNAVAILABLE` | XTwin、单位服务或任务基础设施不可用 |
| `500` | `INTERNAL` | 未分类内部故障 |

### 9.3 稳定 reason

```text
ETAG_MISMATCH
APPLICATION_SET_ETAG_MISMATCH
EFFECTIVE_REVISION_MISMATCH
ENERGY_NAME_ALREADY_EXISTS
ENERGY_NAME_IMMUTABLE
STANDARD_POINT_RULE_VIOLATION
MODEL_HIERARCHY_CONFLICT
METERING_MAPPING_REQUIRED
TELEMETRY_UNIT_INCOMPATIBLE
CAPABILITY_IN_USE
BUILTIN_ENERGY_DELETE_FORBIDDEN
ENERGY_CONFIGURATION_REFERENCED
PRICE_PLAN_NAME_ALREADY_EXISTS
PRICE_VERSION_DATE_ALREADY_EXISTS
PRICE_VERSION_LOCKED
PRICE_PERIOD_INVALID
PRICE_UNIT_PRICE_INVALID
PRICE_PLAN_HAS_APPLICATIONS
METER_NOT_ELIGIBLE
METER_OCCUPIED_BY_OTHER_PLAN
MANUAL_USAGE_INVALID
RANGE_RECOMPUTE_DATE_INVALID
```

并发类 reason 要求前端重新读取，不允许自动覆盖；字段错误根据 `fieldViolations[].field` 定位控件；删除和能力取消失败展示 `impacts`。

## 10. 旧接口兼容映射

旧服务名、protobuf 字段编号、字段类型和 HTTP 路径保持不变。适配器只做协议转换。

| 旧 RPC | 旧 HTTP 接口 | 新领域入口 | 兼容语义 |
| --- | --- | --- | --- |
| `CreateEnergyType` | `POST /xtwin/v3/energy/v1/configs` | `CreateEnergyConfiguration` | 名称映射服务生成的 ID；不能形成完整闭包时失败 |
| `GetEnergyType` | `GET /xtwin/v3/energy/v1/types/{name}` | `GetEnergyConfiguration` | 返回 pending overlay |
| `ListEnergyTypes` | `GET /xtwin/v3/energy/v1/types` | `ListEnergyConfigurations` | 从新聚合派生旧单位和 `usage_points` |
| `UpdateEnergyType` | `PATCH /xtwin/v3/energy/v1/types/{energy_type.name}` | `ReplaceEnergyConfiguration` | 受限无损合并并保留旧协议不可表达字段 |
| `DeleteEnergyType` | `DELETE /xtwin/v3/energy/v1/types/{name}` | `DeleteEnergyConfiguration` | 适配器读取当前 etag 后调用删除命令 |
| `CreatePricePlan` | `POST /xtwin/v3/energy/v1/price_plans` | `CreatePricePlan` | 仅接受可编译为新领域价格类型的规则 |
| `GetPricePlan` | `GET /xtwin/v3/energy/v1/price_plans/{name}` | `GetPricePlan` | 派生旧 `valid_to` |
| `ListPricePlans` | `GET /xtwin/v3/energy/v1/price_plans` | `ListPricePlans` | 从新方案聚合派生旧列表 |
| `UpdatePricePlan` | `PATCH /xtwin/v3/energy/v1/price_plans/{price_plan.name}` | `ReplacePricePlan` | 只允许修改名称和未来版本 |
| `DeletePricePlan` | `DELETE /xtwin/v3/energy/v1/price_plans/{name}` | `DeletePricePlan` | 适配器读取当前 etag，当前有应用时拒绝 |
| `ListBillings` | `GET /xtwin/v3/energy/v1/logical_devices/-/billing:list` | 当前绑定查询 | 从当前表具绑定派生旧 `Billing` |
| `BatchUpdateBillings` | `PUT /xtwin/v3/energy/v1/logical_devices/-/billing:batchUpdate` | `ReplacePricePlanApplications` | names 规范化为 meter ID 后完整替换 |
| `ListMeasurements` | `GET /xtwin/v3/energy/v1/logical_devices/{name}/measurements:list` | 配置兼容视图 + `UsageFactReader` | 从新配置和 Reader 派生，不查询旧表 |
| `UpdateMeasurement` | `PUT /xtwin/v3/energy/v1/logical_devices/{name}/measurements` | `MeasurementCompatibilityAdapter` | 转换显式表具和测点映射 |
| `BatchImportMeasurements` | `GET /xtwin/v3/energy/v1/logical_devices/-/measurements:batchImport` | 无 | 保持 `Unimplemented` |
| `BatchExportMeasurements` | `POST /xtwin/v3/energy/v1/logical_devices/-/measurements:batchExport` | 无 | 保持 `Unimplemented` |

`UpdateMeasurement.by_aggregate` 只兼容当前已有的 `sum(...)` 设备列表语法，解析后形成新任务快照成员集合。无法解析、成员歧义、无效设备或组件折叠必须显式失败。

### 10.1 不可无损场景

1. 不迁移旧数据，切换前旧事实不保证继续可见。
2. 旧协议不能同时表达 active/pending，只返回 pending overlay。
3. 旧能源类型不能表达主瞬时量、过程瞬时量和能力，不能创建不完整配置或隐式猜测测点。
4. 旧价格时间戳不是业务时区当地 `00:00` 时拒绝，不能截断。
5. 阶梯、分档、按星期差异、超过两位小数或无法覆盖全天 48 槽的规则拒绝。
6. 旧 `double` 不能保持新十进制定点数的全部表达精度。
7. 旧接口只表达当前应用集合；历史绑定只用于审计。
8. 旧删除请求不能携带客户端读取到的 etag。适配器读取当前 etag 不能识别旧客户端基于陈旧页面发起的删除。
9. 未提供 `requestId` 的旧写请求由适配器生成稳定幂等键。
10. 兼容请求最终进入同一新领域服务、事实和投影，不查询旧表、不双写。

## 11. 前端对接流程

### 11.1 能源配置

1. 列表读取 `energy-configurations`。
2. 新增页从平台单位服务、`model-candidates` 和 `telemetry-candidates` 取得候选。
3. 新标准点使用请求内 `clientKey`，保存响应改用正式 `standardPointId`。
4. 编辑页读取完整聚合，携带 `etag` 全量替换。
5. 删除使用列表读取到的 `etag`；冲突时刷新列表，引用失败时展示影响清单。

### 11.2 计费方案

1. 列表按名称、物模型和稳定游标查询。
2. 新增/编辑提交完整方案聚合。
3. 已生效版本根据服务返回的 `editable/deletable` 置灰。
4. 应用弹窗分别读取候选列表和当前完整应用集合。
5. 确定时提交完整 `meterIds`、`applicationSetEtag` 和 `requestId`。

### 11.3 表具能耗明细

1. 能源 Tab 来自能源配置列表，空间树来自 XTwin。
2. 左侧表具列表携带当前查询范围以计算完整性。
3. 右侧一次查询返回完整范围汇总、完整趋势和分页明细。
4. 半小时人工命令成功后重新查询，不能把短暂旧投影视为命令失败。
5. 范围重算轮询对应 operation；仅在 `SUCCEEDED` 后刷新完整明细。

## 12. 契约测试要点

- JSON 十进制数不经过二进制浮点往返。
- 规范 meter ID 在 query/body 中编码、解析和回显一致。
- energy/price 替换和删除的 etag 冲突返回 `ABORTED/ETAG_MISMATCH`。
- 新标准点的 `clientKey` 唯一、悬空和混用引用被整体拒绝；幂等重试返回相同正式 ID。
- 价格版本锁定、48 槽覆盖和两位小数校验固定。
- 应用集合 etag 或任一表具占用冲突时整体失败。
- 半小时 `MISSING/BAD` 的用量和成本为 `null`，零值保持数值零。
- 完整范围汇总和趋势不受明细分页影响。
- `SET -> SET -> CLEAR` 审计链、乐观并发和投影最终可见性固定。
- 范围重算只有全部阶段完成才返回 `SUCCEEDED`。
- 新旧接口提交等价逻辑请求时进入同一领域事实和投影。
