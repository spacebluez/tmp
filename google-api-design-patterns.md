# 从 Google 官方 API 学接口设计:特殊场景的处理方式

整理自 Google 的 API 设计规范(AIP,google.aip.dev)和 googleapis 仓库里的公开 proto,以及 BigQuery、Google Ads、Vertex AI、Merchant API 几个产品的实际接口。重点放在普通 CRUD 之外的场景:长时间任务、批量、导入导出、部分失败、幂等、预检。每一节先给 proto 形状,再说 Google 自己哪个 API 在这么用,最后是照抄时容易踩的坑。

AIP 里 must / should / may 的措辞我都保留了,因为这三个词的区别是有实际意义的:must 是 linter 会报错的,should 是 code review 会被问的,may 是你自己决定。

整理时间:2026 年 9 月。AIP 会更新(AIP-233 在 2025 年 3 月刚大改过一次),用之前建议核对一下原文。

---

## 0. 一条贯穿全文的原则:OK 就得是 OK

后面所有模式的设计都绕着一个约束转:**RPC 返回 OK,客户端就有权认为请求完整成功了。** 错误不能藏在 200 的响应体里让客户端自己翻。

AIP-193 对部分错误的态度很直接:API 不应该支持部分错误,因为它绕开了错误码,把错误挪进响应体,逼客户端写一套专门的处理逻辑。但批量操作里偶尔确有必要——几万行数据因为一行有问题就整体失败,对用户不友好。

于是 Google 的解法是:凡是可能部分成功的,做成 LRO(长时操作)。`Operation` 本身的状态码只说明"我拿到了这个操作句柄",不代表底层任务的结果,客户端本来就得去看 body,所以在 body 里放部分失败信息不会误导人。

这条原则会在 LRO、批量、导入导出三节里反复出现。先记住它,后面的设计就都顺理成章。

---

## 1. LRO:长时操作

规范:AIP-151。核心 proto:`google/longrunning/operations.proto`。

### 1.1 什么时候该用

AIP 给的经验值是 10 秒。方法可能超过 10 秒,就返回 `Operation` 而不是最终结果。这个数字不绝对,用户对"多久算慢"的预期跟任务类型有关,但作为起点够用了。

导入导出(AIP-153)的要求更严:除非能保证永远不超过几秒,否则必须是 LRO。

### 1.2 方法声明

```protobuf
import "google/longrunning/operations.proto";

rpc ImportBooks(ImportBooksRequest) returns (google.longrunning.Operation) {
  option (google.api.http) = {
    post: "/v1/{parent=publishers/*}/books:import"
    body: "*"
  };
  option (google.longrunning.operation_info) = {
    response_type: "ImportBooksResponse"
    metadata_type: "ImportBooksMetadata"
  };
}
```

几条硬性要求:

- 返回类型必须是 `google.longrunning.Operation`,不能把这个 proto 复制到自己的包里改。
- 不能是流式响应。
- 必须带 `google.longrunning.operation_info`,且 `response_type` 和 `metadata_type` 都要填。两个类型要定义在当前文件或它 import 的文件里;跨包要写全限定名。
- 服务必须实现标准的 `google.longrunning.Operations` 服务(GetOperation / ListOperations / CancelOperation / DeleteOperation / WaitOperation)。不要自己定义一套"查任务状态"的接口。这条是为了让客户端库能统一处理所有 Google 风格 API 的 LRO。

### 1.3 response_type 和 metadata_type 别用 Empty

两个类型都不建议用 `google.protobuf.Empty`,即便现在确实没东西可放。原因是以后加字段时,从 Empty 换成别的类型是 breaking change;而从一个空 message 加字段是兼容的。

```protobuf
// 现在没内容,也定义一个空 message 占位
message ImportBooksResponse {}
message ImportBooksMetadata {}
```

Delete 方法是例外,response_type 用 Empty 可以。

### 1.4 metadata 放什么

metadata 是客户端每次 GetOperation 都会拿到的东西,AIP 明说它的用途是"进度、部分失败等信息"。常见内容:

```protobuf
message ImportBooksMetadata {
  google.protobuf.Timestamp create_time = 1;
  google.protobuf.Timestamp update_time = 2;

  // 进度。用计数而不是百分比,让客户端自己算,
  // 因为 total 在扫描阶段可能还不知道。
  int64 total_count = 3;
  int64 processed_count = 4;
  int64 succeeded_count = 5;
  int64 failed_count = 6;

  // 部分失败。见第 3 节批量方法对这个字段形状的规定。
  map<int32, google.rpc.Status> failed_requests = 7;

  // 可选:当前阶段的人可读描述
  string status_message = 8;
}
```

Google Cloud 大多数服务的 OperationMetadata 都是类似的一组字段:create_time、end_time、target(资源名)、verb(create/update/delete)、status_message、requested_cancellation、api_version。可以参考 `google/cloud/common/operation_metadata.proto`。

### 1.5 错误分三层放

这是 LRO 最容易搞混的地方。AIP-151 在 2025 年 2 月专门补了一段澄清:

| 错误发生的时机 | 放哪里 | 客户端怎么看到 |
|---|---|---|
| 操作还没开始(参数校验失败、无权限、配额不够) | 直接返回 RPC 错误,和普通方法一样 | 调用抛异常,没有 Operation |
| 执行过程中的致命错误(整个任务失败) | `Operation.error`(google.rpc.Status) | `done=true`,`error` 有值,`response` 为空 |
| 执行过程中的非致命错误(跳过了某几行) | metadata 里的字段,类型必须是 google.rpc.Status | `done=true`,`response` 有值,metadata 里有失败明细 |

三层都要遵守 AIP-193 的错误格式要求,也就是 `ErrorInfo` 必须有(见第 5 节)。

### 1.6 并行操作怎么处理

同一个资源上同时发起两个 LRO,AIP 允许三种策略,选一种并写进文档:

- 接受并排队,按顺序执行。
- 拒绝,返回 `ABORTED`,错误信息说明有操作正在进行。
- 声明式(declarative-friendly)资源可以让新操作抢占旧的,旧操作被标记为 ABORTED。

导入场景一般选第二种,同一张表并发导入基本没有好结果。

### 1.7 通过 LRO 创建的资源,List/Get 里要能看到

用 LRO 创建资源时,资源在 Operation 完成之前就应该出现在 List 和 Get 里,但要通过 state 字段(AIP-216)标明它还不可用,比如 `CREATING`。删除同理,`DELETING` 状态下还能 Get 到。

这样客户端不用同时轮询 Operation 和资源两个东西。

### 1.8 其他

- 过期:Operation 可以在完成后一段时间过期,经验值 30 天。
- 兼容性:改 `response_type` 或 `metadata_type` 都是 breaking change,当初定的时候多想一下。
- Operation 的 `name` 是资源名,格式一般是 `operations/{operation_id}` 或 `projects/*/locations/*/operations/*`。REST 下 GetOperation 就是 `GET /v1/{name}`。

### 1.9 REST 下 Operation 长什么样

进行中:

```json
{
  "name": "projects/p/locations/us/operations/op-123",
  "metadata": {
    "@type": "type.googleapis.com/mycompany.ImportBooksMetadata",
    "totalCount": "5000",
    "processedCount": "2100",
    "failedCount": "3"
  },
  "done": false
}
```

成功完成:

```json
{
  "name": "projects/p/locations/us/operations/op-123",
  "metadata": { "...": "..." },
  "done": true,
  "response": {
    "@type": "type.googleapis.com/mycompany.ImportBooksResponse",
    "importedCount": "4997",
    "reportUri": "gs://bucket/reports/op-123.ndjson"
  }
}
```

整体失败:

```json
{
  "name": "projects/p/locations/us/operations/op-123",
  "done": true,
  "error": {
    "code": 10,
    "message": "None of the requests succeeded, refer to metadata.failed_requests for individual error details",
    "details": [
      { "@type": "type.googleapis.com/google.rpc.ErrorInfo",
        "reason": "ALL_ROWS_FAILED", "domain": "import.mycompany.com" }
    ]
  }
}
```

`response` 和 `error` 是 oneof,不会同时出现。

---

## 2. LRO 还是 Job:任务要不要做成资源

规范:AIP-152。

LRO 是一次性的、用完即弃的句柄。有些场景它不够用:

- 任务要重复跑(每天凌晨同步一次)。
- 配置任务和执行任务需要不同的权限(运营配规则,系统定时跑)。
- 要看历史执行记录。

这时候把任务本身建模成资源,叫 `XxxJob`:

```protobuf
message ImportBooksJob {
  option (google.api.resource) = {
    type: "library.googleapis.com/ImportBooksJob"
    pattern: "publishers/{publisher}/importBooksJobs/{import_books_job}"
  };
  string name = 1;
  // 配置字段:数据源、映射规则、调度等
}

// 五个标准方法做配置
rpc CreateImportBooksJob(...) returns (ImportBooksJob);
rpc GetImportBooksJob(...) returns (ImportBooksJob);
rpc ListImportBooksJobs(...) returns (ListImportBooksJobsResponse);
rpc UpdateImportBooksJob(...) returns (ImportBooksJob);
rpc DeleteImportBooksJob(...) returns (google.protobuf.Empty);

// Run 触发一次执行,返回 LRO
rpc RunImportBooksJob(RunImportBooksJobRequest) returns (google.longrunning.Operation) {
  option (google.api.http) = {
    post: "/v1/{name=publishers/*/importBooksJobs/*}:run"
    body: "*"
  };
  option (google.longrunning.operation_info) = {
    response_type: "RunImportBooksJobResponse"
    metadata_type: "RunImportBooksJobMetadata"
  };
}
```

要点:

- 资源名以 `Job` 结尾,前缀是"动词 + 名词"。
- `Run` 方法返回 LRO,URI 以 `:run` 结尾,请求体只有一个 `name`。
- 需要历史记录时,在 Job 下挂一个 `executions` 子集合,每次 Run 返回的 Operation 指向对应的 execution 资源。给 execution 定义 Get、List、Delete 就够了,不需要 Create 和 Update(它是 Run 产生的)。

```
publishers/{publisher}/importBooksJobs/{job}/executions/{execution}
```

对表格导入这种场景,我的判断是:用户上传文件、点一下导入,这是 LRO;用户配一个"每天从 SFTP 拉文件导入",这是 Job。同一个系统里两者可以并存,底层共用一套执行逻辑。

BigQuery 的设计是个反例可以对照着看:它的 `Job` 资源既承载一次性任务也承载配置,`jobs.insert` 创建即执行,没有单独的 Run。这是 AIP 之前就定型的老设计,新 API 不建议照抄。

---

## 3. 批量方法与部分成功

规范:AIP-233(BatchCreate),AIP-234、235 同理。这一节在 2025 年 3 月刚被大改,加了完整的部分成功方案,比之前的版本具体得多。

### 3.1 同步批量必须原子

```protobuf
// 同步:要么全成,要么全失败
rpc BatchCreateBooks(BatchCreateBooksRequest) returns (BatchCreateBooksResponse) {
  option (google.api.http) = {
    post: "/v1/{parent=publishers/*}/books:batchCreate"
    body: "*"
  };
}
```

理由回到第 0 节:同步方法返回 OK,客户端就认为全成功了。在响应体里加一个 `failed` 字段表示部分失败,老客户端会忽略它,把失败当成功,这是行为上的 breaking change。

所以规则是:**同步批量必须原子;想要部分成功,必须走 LRO。** 已经上线的同步批量接口如果想支持部分成功,只能开新版本,不能在原版本上加字段。

### 3.2 异步批量的部分成功方案

```protobuf
rpc BatchCreateBooks(BatchCreateBooksRequest) returns (google.longrunning.Operation) {
  option (google.api.http) = {
    post: "/v1/{parent=publishers/*}/books:batchCreate"
    body: "*"
  };
  option (google.longrunning.operation_info) = {
    response_type: "BatchCreateBooksResponse"
    metadata_type: "BatchCreateBooksOperationMetadata"
  };
}

message BatchCreateBooksRequest {
  string parent = 1;
  // 最多 1000 条。注释里必须写上上限。
  repeated CreateBookRequest requests = 2 [(google.api.field_behavior) = REQUIRED];
  // 已上线的接口后加部分成功能力时,用这个开关保持默认行为不变
  bool return_partial_success = 3;
}

message BatchCreateBooksResponse {
  repeated Book books = 1;
}

message BatchCreateBooksOperationMetadata {
  // key 是 requests 里的下标,value 是那条单独调 CreateBook 时会返回的错误
  map<int32, google.rpc.Status> failed_requests = 1;
  // 其他进度字段
}
```

规则:

- metadata 里必须有 `map<int32, google.rpc.Status> failed_requests`,key 是请求在 `requests` 数组里的下标。
- value 要和单条 Create 方法返回的错误一致。也就是说批量接口不发明新的错误码,复用单条接口的。
- 服务端会自动重试的瞬时错误,不能放进 `failed_requests`。放进去的都是"你不改请求它就不会好"的错误。
- 全部失败时,`Operation.error` 必须设为 `ABORTED`,message 固定为 `None of the requests succeeded, refer to the BatchCreateBooksOperationMetadata.failed_requests for individual error details`。
- 部分成功时 `Operation.error` 不设,`response` 正常返回成功的那部分,失败的看 metadata。

### 3.3 为什么是 `map<int32, Status>`

AIP-233 的 rationale 里列了被否掉的几个方案,值得看一眼,因为它们正好是大家第一反应会想到的:

| 方案 | 为什么不行 |
|---|---|
| `repeated google.rpc.Status` | 对不上是哪条请求失败的 |
| `map<string, Status>`,key 用 request_id | 客户端得自己维护 request_id → 请求的映射;而且给每条子请求编 request_id 可能和 AIP-155 的幂等语义冲突 |
| `repeated FailedRequest { request; status }`,把原请求回显 | 回显请求体有数据敏感性问题 |

所以最终选了按下标的 map。这个决定也解释了 Google Ads 和 BigQuery 的老设计为什么都是"按下标定位"。

### 3.4 Google 自己的两个老实现

AIP-233 定型之前,Google 内部几个产品各自搞了一套。看它们有助于理解 AIP 为什么这么规定,也能借鉴一些细节。

**Google Ads `partial_failure`**

请求里有 `bool partial_failure`,响应里有 `google.rpc.Status partial_failure_error`。这是同步接口,严格说违反了第 3.1 条——但它是 AIP-233 改版之前就存在的设计,而且 Google Ads 一直不算在 Cloud API 规范体系里。

它的错误定位方式值得看:

```
GoogleAdsError {
  error_code { range_error: TOO_LOW }
  message: "Too low."
  trigger { string_value: "" }          // 触发错误的原始值
  location {
    field_path_elements { field_name: "operations"  index { value: 1 } }
    field_path_elements { field_name: "create" }
    field_path_elements { field_name: "campaign" }
  }
}
```

`location` 是一个结构化路径链,不是拼好的字符串,前端不需要解析。`trigger` 带上出错的原始值,方便用户对照。这两个细节在自己设计逐行错误时可以借用。

它还有一条文档里明确写的约束:请求内的操作如果互相依赖(比如先创建 A 再创建引用 A 的 B),不要开 partial_failure,应当整批原子失败。判断方法是看有没有用临时 ID 做跨操作引用。

**BigQuery `tabledata.insertAll`**

```json
{
  "insertErrors": [
    {
      "index": 1,
      "errors": [
        { "reason": "invalid", "location": "name",
          "message": "Conversion from int64 to string is unsupported." }
      ]
    },
    { "index": 0, "errors": [ { "reason": "stopped" } ] }
  ]
}
```

`index` 是行下标,`location` 放的是列名。HTTP 200,错误在 body 里——同样是 AIP-233 现在不允许的做法,但它是流式插入接口,一开始就定位在"尽力而为"。

`reason: "stopped"` 这个条目值得留意:第 0 行本身没错,是因为同批次的第 1 行失败而被中止。如果你的批量实现有事务分组,需要类似的 reason 来区分"自身错误"和"被连坐"。

### 3.5 请求消息的其他约束

- `parent` 字段可以"提升"到批量请求顶层。设了顶层 parent 之后,子请求的 parent 可以省略;两边都设则必须一致,不一致要报错。
- 其他字段也可以这样提升(比如 `validate_only`、`request_id`),规则相同。但必须唯一的字段不能提升,比如客户端指定的资源 ID。
- 请求消息里不能有其他必填字段。
- `requests` 字段的注释要写明单批上限。

---

## 4. 导入导出

规范:AIP-153。

### 4.1 方法形状

```protobuf
// 导入多个资源
rpc ImportBooks(ImportBooksRequest) returns (google.longrunning.Operation) {
  option (google.api.http) = {
    post: "/v1/{parent=publishers/*}/books:import"
    body: "*"
  };
  option (google.longrunning.operation_info) = {
    response_type: "ImportBooksResponse"
    metadata_type: "ImportBooksMetadata"
  };
}

// 向单个资源导入数据(比如给一本书导入页面)
rpc ImportPages(ImportPagesRequest) returns (google.longrunning.Operation) {
  option (google.api.http) = {
    post: "/v1/{book=publishers/*/books/*}:importPages"
    body: "*"
  };
  // ...
}
```

- 必须是 LRO,除非能保证永远几秒内完成。
- HTTP 动词 POST,body 为 `"*"`。
- URI 后缀 `:import` / `:export`。向单个资源导入时后缀带名词,`:importPages`;URI 变量用资源名(`book`)而不是 `name`。
- 多资源导入时 URI 里应当有 `parent`;需要跨多个 parent 导入,parent 位置允许用 `-` 通配(AIP-159)。指定了 parent 之后,导入的数据里如果有属于其他 parent 的,必须拒绝。

### 4.2 请求消息:两类配置分开放

导入请求里的配置天然分两类:关于数据源的(从哪读、凭证、格式),和关于数据本身的(字段映射、冲突策略)。AIP 要求前者用 oneof 包起来,后者放顶层:

```protobuf
message ImportBooksRequest {
  string parent = 1 [(google.api.field_behavior) = REQUIRED];

  // 数据源配置。即便现在只有一种来源,也必须放在 oneof 里,
  // 为以后加 GcsSource、BigQuerySource 留位置。
  oneof source {
    InlineSource inline_source = 2;
    GcsSource gcs_source = 3;
  }

  // 数据本身的配置,所有来源共用,放顶层
  string isbn_prefix = 4;
  ConflictPolicy conflict_policy = 5;
}

message InlineSource {
  repeated Book books = 1;
}

message GcsSource {
  repeated string uris = 1;
}
```

`oneof source` 即便只有一个成员也必须有,这条是 must。原因是 oneof 里加成员是兼容的,而把一个普通字段改成 oneof 成员是 breaking change。

内联数据源命名为 `InlineSource` / `InlineDestination`,里面是资源的 repeated 字段。导入和导出的 inline 格式必须一致——用户导出的东西要能原样导回去。

导出的 `oneof destination` 同理。导入和导出的配置可以不对称,比如从文件导入、导出到目录。

### 4.3 部分失败放 metadata

AIP-153 的原话是:虽然部分失败一般不鼓励,但导入导出方法应当把部分失败信息放在 metadata 里,每个错误是一个 `google.rpc.Status`。字段形状沿用 AIP-233 的 `map<int32, google.rpc.Status>`,key 是行号或条目下标。

### 4.4 实际案例:Vertex AI RAG 的 ImportRagFiles

这是 Google 自己在大批量导入场景下的做法,和 AIP-153 有一处明显偏离,值得知道。

响应(LRO 完成后的 response):

```
ImportRagFilesResponse {
  int64 imported_rag_files_count
  int64 failed_rag_files_count
  int64 skipped_rag_files_count
  oneof partial_failure_sink {
    string partial_failures_gcs_path
    string partial_failures_bigquery_table
  }
}
```

请求里对应地让用户指定一个 sink(GCS 路径或 BigQuery 表),所有部分失败写进去。响应体里一条明细都不放,只有三个计数和一个路径。

它偏离 AIP-153 的地方是:失败明细没放 metadata,而是外置到文件。原因不难猜,一次导入几十万个文件,metadata 里放几十万条 Status 不现实。所以实践上的规则大概是:

- 条目数在几百以内,明细放 metadata 的 `failed_requests`。
- 可能到几千以上,响应里只放计数,明细写文件,给一个路径。
- 两者可以同时做:metadata 里放前 N 条方便快速预览,文件里放全量。

这个接口后来还演进了一步:`partial_failure_sink` 被标记 deprecated,换成 `import_result_sink`,成功和失败的记录都写进去。用户要对账,只给失败列表不够。

---

## 5. 错误结构:AIP-193 的几条硬规定

完整内容看 AIP-193,这里只列和上面几节直接相关的。

### 5.1 ErrorInfo 是必须的

2023 年 5 月起,所有错误响应的 `details` 里必须有一个 `google.rpc.ErrorInfo`。只靠 `Status.code` 的十几个枚举值区分不了错误,客户端需要 `reason` + `domain` 这对机器可读的标识。

```
ErrorInfo {
  reason: "ROW_LIMIT_EXCEEDED"        // UPPER_SNAKE_CASE,≤63 字符,同一 domain 内唯一
  domain: "import.mycompany.com"      // 全局唯一,通常是服务域名
  metadata: {                          // 出现在错误文案里的所有动态值都要放这里
    "limit": "10000",
    "actual": "12345"
  }
}
```

`metadata` 那条规则容易被忽略:错误信息里如果写了"最多允许 10000 行,你传了 12345 行",那 10000 和 12345 必须同时以 key-value 形式出现在 metadata 里,不能让客户端去解析字符串。

### 5.2 每种 detail 类型最多出现一次

`details` 里不能有两个 `BadRequest`。多条字段错误要合并到同一个 `BadRequest.field_violations` 里。但 `BadRequest` 和 `PreconditionFailure` 可以共存。

### 5.3 Status.message 一旦被依赖就不能改

如果某个 RPC 历史上返回过没带 `ErrorInfo` 的错误,客户端可能已经在解析 `message` 文本了,此时 `message` 就成了 API 契约的一部分,改它是 breaking change。想改文案,加一个 `LocalizedMessage` 到 details 里。

从第一天就带 `ErrorInfo` 的 RPC 没有这个包袱,`message` 可以随时改。所以新接口一开始就把 `ErrorInfo` 加上,省掉以后的麻烦。

### 5.4 传输层的限制

gRPC 的 `Status.details` 序列化后 base64 编码放在 `grpc-status-details-bin` 这个 trailer 里。默认 metadata 上限一般是 8KB,base64 再膨胀约三分之一。在 details 里放几百条 `FieldViolation` 会撞到这个限制,失败形态还很难排查。

这也是为什么部分失败要放 LRO 的 metadata 而不是错误的 details:metadata 走的是普通响应体,没有这个限制。

---

## 6. 幂等:request_id

规范:AIP-155。

```protobuf
message ImportBooksRequest {
  // ...
  // 可选。36 个 ASCII 字符以内,建议用随机 UUID。
  // 提供了 request_id 的请求才有幂等保证。
  string request_id = 10 [(google.api.field_info).format = UUID4];
}
```

- 带了 `request_id` 就必须保证幂等:重复请求返回上一次成功的响应,而不是再执行一遍。
- 字段放在请求消息上,不放在资源上。
- 应当是可选的。
- 用 UUID 时要加 `(google.api.field_info).format = UUID4` 注解。
- 服务端保留 request_id 的时间窗口自己定,文档里写清楚。
- 如果重复请求到达时资源已经被后续操作改过,无法返回一模一样的历史响应,可以返回资源的当前状态代替。

对 LRO 的意义:客户端发起导入后网络断了,不知道服务端收到没有,带着同一个 request_id 重发,拿回的是同一个 Operation,不会导入两次。

批量请求里 `request_id` 可以提升到顶层(整批一个),但不能给每条子请求分配——这和 AIP-233 否掉 `map<string, Status>` 方案的理由是同一个。

---

## 7. 预检:validate_only

规范:AIP-163;对 LRO 的补充在 AIP-151。

```protobuf
message ImportBooksRequest {
  // ...
  // 为 true 时只做校验,不落库。
  bool validate_only = 11;
}
```

导入场景里这个开关很实用:用户上传文件后先跑一遍校验,看到问题列表,修完再真正导入。同一套请求、同一套错误结构复用两次。

对返回 LRO 的方法,`validate_only=true` 的响应有三种合法形式,选一种:

1. 立即返回一个 `done=true` 的 Operation,`response` 里是空的(或有内容的)响应消息。`name` 可以为空,服务端不需要为校验维护状态。这是推荐做法。
2. 直接返回错误(通常是 `INVALID_ARGUMENT`)。
3. 校验本身也很慢时,返回 `done=false` 的 Operation,`name` 必须有,客户端轮询。最终 `done=true`,成功放 `response`,失败放 `error`。

第 1 种的好处是客户端可以用处理普通 LRO 的同一段代码处理校验结果,不需要特殊分支。

有一个细节:`validate_only` 模式下部分失败信息放哪。如果走第 1 种形式,metadata 里可以带 `failed_requests`,和真实执行时的结构一致。校验发现 3 行有问题,就返回 `done=true`、`response` 为空消息、metadata 里 3 条 Status。这和 Google Ads 的行为一致:`validate_only=true` 时不返回 results,但 `partial_failure_error` 正常填。

---

## 8. 综合示例:表格导入接口

把上面几节拼起来,一个符合 AIP 风格的表格导入接口大致是这样。字段编号故意留了空隙。

```protobuf
syntax = "proto3";
package mycompany.dataimport.v1;

import "google/api/annotations.proto";
import "google/api/field_behavior.proto";
import "google/api/field_info.proto";
import "google/longrunning/operations.proto";
import "google/protobuf/timestamp.proto";
import "google/rpc/status.proto";

service RowImportService {
  // 向一张表导入行。可能耗时较长,返回 LRO。
  rpc ImportRows(ImportRowsRequest) returns (google.longrunning.Operation) {
    option (google.api.http) = {
      post: "/v1/{table=workspaces/*/tables/*}:importRows"
      body: "*"
    };
    option (google.longrunning.operation_info) = {
      response_type: "ImportRowsResponse"
      metadata_type: "ImportRowsMetadata"
    };
  }
}

message ImportRowsRequest {
  // 目标表。用资源名而不是 name。
  string table = 1 [(google.api.field_behavior) = REQUIRED];

  // 数据源。即便只有一种也放 oneof。
  oneof source {
    InlineSource inline_source = 2;
    UploadedFileSource uploaded_file_source = 3;
  }

  // 以下是数据本身的配置,所有来源共用
  ColumnMapping column_mapping = 10;
  ConflictPolicy conflict_policy = 11;

  // 为 true 时只校验不写入
  bool validate_only = 20;

  // 为 true 时允许部分成功;默认整批原子
  bool return_partial_success = 21;

  // 幂等标识,可选,建议 UUID
  string request_id = 22 [(google.api.field_info).format = UUID4];
}

message InlineSource {
  // 单批上限 1000 行,超出返回 INVALID_ARGUMENT
  repeated Row rows = 1;
}

message UploadedFileSource {
  // 之前通过上传接口拿到的文件句柄
  string file = 1;
  // 从第几行开始是数据(1-based,含表头则填 2)
  int32 data_start_row = 2;
  string sheet_name = 3;
}

message ImportRowsMetadata {
  google.protobuf.Timestamp create_time = 1;
  google.protobuf.Timestamp update_time = 2;

  // 进度
  int64 total_rows = 3;
  int64 processed_rows = 4;
  int64 succeeded_rows = 5;
  int64 failed_rows = 6;

  // 部分失败。key 是行号,口径和 UploadedFileSource.data_start_row 一致,
  // 即用户在 Excel 里看到的物理行号。
  // 最多保留前 100 条;全量见 ImportRowsResponse.report_uri。
  map<int32, google.rpc.Status> failed_requests = 7;
  bool failed_requests_truncated = 8;

  // 警告:已导入但需要用户留意的行。用 Status 是为了复用 ErrorInfo 的
  // reason/metadata 结构,code 固定为 OK。
  map<int32, google.rpc.Status> warnings = 9;
}

message ImportRowsResponse {
  int64 imported_rows = 1;
  int64 skipped_rows = 2;
  // 全量结果报告(NDJSON,每行一条,成功和失败都有)
  string report_uri = 3;
  google.protobuf.Timestamp report_expire_time = 4;
}
```

`failed_requests` 里每条 Status 的形状,复用 AIP-193:

```
Status {
  code: INVALID_ARGUMENT
  message: "Column E (email): invalid format"
  details: [
    ErrorInfo {
      reason: "INVALID_EMAIL_FORMAT"
      domain: "dataimport.mycompany.com"
      metadata: {
        "row": "43",
        "column": "E",
        "columnName": "email",
        "value": "abc@@x"
      }
    },
    BadRequest {
      field_violations: [
        { field: "rows[43].email", description: "invalid format" }
      ]
    },
    LocalizedMessage {
      locale: "zh-CN"
      message: "第 43 行 E 列(邮箱)格式不正确:abc@@x"
    }
  ]
}
```

几个设计决定说明一下:

- 行号用用户视角的物理行号,不用 0-based 数组下标。AIP-233 说 key 是"请求里的下标",对 inline 来源就是数组下标;对文件来源,用户看的是 Excel 行号,按那个来。两种来源的口径不同,文档里写清楚。
- 警告单独一个 map,不混进 `failed_requests`。`google.rpc.Status` 没有 severity 维度,混在一起客户端分不出哪些行其实导进去了。
- metadata 里的明细截断在 100 条,超出走报告文件。这是 Vertex AI 的做法和 AIP-233 的做法折中。
- `report_uri` 里成功失败都有,方便对账。这是跟着 Vertex AI 从 `partial_failure_sink` 到 `import_result_sink` 的演进学的。

REST 调用流程:

```
POST /v1/workspaces/w1/tables/t1:importRows
{ "uploadedFileSource": { "file": "files/f-9" , "dataStartRow": 2 },
  "validateOnly": true }
→ 200 { "name": "", "done": true,
        "metadata": { "failedRequests": { "43": {...}, "87": {...} } },
        "response": { "@type": ".../ImportRowsResponse" } }

// 用户修完文件,重新上传,正式导入
POST /v1/workspaces/w1/tables/t1:importRows
{ "uploadedFileSource": { "file": "files/f-10", "dataStartRow": 2 },
  "returnPartialSuccess": true,
  "requestId": "3f0c...-uuid" }
→ 200 { "name": "workspaces/w1/operations/op-77", "done": false }

GET /v1/workspaces/w1/operations/op-77
→ 200 { "name": "...", "done": false,
        "metadata": { "totalRows": "5000", "processedRows": "2100" } }

GET /v1/workspaces/w1/operations/op-77
→ 200 { "name": "...", "done": true,
        "metadata": { "totalRows": "5000", "succeededRows": "4997",
                      "failedRows": "3", "failedRequests": {...} },
        "response": { "importedRows": "4997", "reportUri": "..." } }
```

---

## 9. 速查表

| 场景 | 做法 | 出处 |
|---|---|---|
| 方法可能超过 10 秒 | 返回 `google.longrunning.Operation`,带 `operation_info` | AIP-151 |
| LRO 的 response/metadata 暂时没内容 | 定义空 message,不用 `Empty` | AIP-151 |
| LRO 启动前失败 | 普通 RPC 错误 | AIP-151 |
| LRO 执行中整体失败 | `Operation.error` | AIP-151 |
| LRO 执行中跳过部分条目 | metadata 里 `map<int32, google.rpc.Status>` | AIP-151, 233 |
| 同一资源并发 LRO | 排队,或返回 `ABORTED` | AIP-151 |
| 任务要重复跑 / 配置与执行分权 / 要看历史 | 做成 `XxxJob` 资源 + `:run` + `executions` 子集合 | AIP-152 |
| 导入导出 | `:import` / `:export`,POST,body `*`,LRO,`oneof source` | AIP-153 |
| 同步批量 | 必须原子 | AIP-233 |
| 批量要部分成功 | 必须 LRO;`failed_requests` map;全失败 `ABORTED` | AIP-233 |
| 已上线的同步批量想加部分成功 | 开新版本,不能在原版本加字段 | AIP-233 |
| 失败明细可能上千条 | 响应只放计数 + 报告文件路径 | Vertex AI 实践 |
| 所有错误 | details 里必须有 `ErrorInfo`;每种 detail 类型最多一个 | AIP-193 |
| 错误文案里的动态值 | 同时放进 `ErrorInfo.metadata` | AIP-193 |
| 幂等 | 可选 `request_id`,UUID4,重复返回上次结果 | AIP-155 |
| 预检 | `validate_only`;LRO 方法返回 `done=true` 的 Operation | AIP-163, 151 |

---

## 参考

- AIP 总索引:https://google.aip.dev/general
- AIP-151 长时操作:https://google.aip.dev/151
- AIP-152 Jobs:https://google.aip.dev/152
- AIP-153 导入导出:https://google.aip.dev/153
- AIP-155 请求标识:https://google.aip.dev/155
- AIP-163 变更校验(validate_only):https://google.aip.dev/163
- AIP-193 错误:https://google.aip.dev/193
- AIP-216 状态枚举:https://google.aip.dev/216
- AIP-233 批量创建:https://google.aip.dev/233
- googleapis 仓库(所有公开 proto):https://github.com/googleapis/googleapis
  - `google/longrunning/operations.proto`
  - `google/rpc/status.proto`、`error_details.proto`、`code.proto`
  - `google/cloud/common/operation_metadata.proto`
- API Linter(可以直接跑在自己的 proto 上检查 AIP 合规):https://linter.aip.dev/
- Google Ads 部分失败文档:https://developers.google.com/google-ads/api/docs/best-practices/partial-failures
- BigQuery `tabledata.insertAll`:https://cloud.google.com/bigquery/docs/reference/rest/v2/tabledata/insertAll
- Vertex AI `ragFiles.import`:https://cloud.google.com/vertex-ai/docs/reference/rest/v1/projects.locations.ragCorpora.ragFiles/import
