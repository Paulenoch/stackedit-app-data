
# [BE] User Tag 对外接口能力技术方案

  

## 1 文档历史

  

| 修订日期 | 修订内容 | 修订版本 | 修订人 |

|----------|----------|----------|--------|

| 2026-02-05 | 新增文档 | v1.0 | 戴英特 |

  

## 2 背景

  

### 2.1 需求目标

  

User Tag 是 CS User 服务的核心能力之一，用于对用户进行标签化管理。当前需要将 User Tag 能力对外暴露，供以下上游系统使用：

  

- **Smart-KB / Chatbot Taskflow**：在 Issue 的 Condition 中选择「User ID User Tag」Function 做条件判断，在 Solution 中使用 Tag 信息

- **SOP**：在 SOP 流程中根据用户 Tag 进行路由和决策

- **WorkConsole**：在工作台侧展示用户 Tag 信息

  

具体需要提供三项能力：

  

| # | 能力 | 使用场景 |

|---|------|----------|

| 1 | 查询所有 User Tag 配置 | 上游获取全部 Tag 定义（名称、优先级、颜色等），用于展示或下拉选择 |

| 2 | 根据 UserID 计算 User Tag | 用户进线时实时触发 Tag 计算（基于多个 DataSource 并发计算），返回匹配的 Tag 列表 |

| 3 | 根据 UserID 查询其全部 Tag | 在 Issue Condition/Solution 中使用，查询用户当前已有的 Tag 列表（不触发计算） |

  

### 2.2 相关链接

  

- PRD：TODO

- Jira：TODO

- Chatbot API 管理页：`https://chatbot-admin.shopee.sg/chatbot/taskflow-portal/api-management-configuration`

- Function API Name：**User ID User Tag**（`sopGetUserTagByCsUserId` / `cs.console.workconsole.get_user_info_by_sop`）

  

## 3 方案

  

### 3.1 整体架构

  

本次对外提供三个 Spex 接口，整体调用链路如下：

  

```

上游调用方 cs/user 服务

┌──────────────────────┐ ┌──────────────────────────────────────┐

│ Smart-KB / Chatbot │ │ │

│ SOP Issue/Solution │ │ 接口1: get_cs_user_tag_config_list │

│ Taskflow │── Spex ──────────>│ 接口2: get_cs_user_tag_list │

│ WorkConsole │ │ (refresh=true → 触发计算) │

│ │ │ 接口3: get_cs_user_tag_list │

└──────────────────────┘ │ (refresh=false → 仅查询) │

└───────────┬──────────────────────────┘

│

┌───────────▼──────────────────────────┐

│ Tag 计算引擎 (接口2) │

│ DataSource Processors (并发执行): │

│ ├─ UserProfilePlatform (UPP) │

│ ├─ SellerOperationPlatform (SOP) │

│ ├─ CSUserProfile │

│ └─ CSSystem │

└───────────┬──────────────────────────┘

│

┌───────────▼──────────────────────────┐

│ 存储 & 外部依赖 │

│ ├─ MySQL (tag 定义表 & 用户 tag 表) │

│ ├─ Redis (限流锁 & 缓存) │

│ └─ UPP / SOP 等外部 API │

└──────────────────────────────────────┘

```

  

### 3.2 接口设计

  

#### 3.2.1 接口1：查询所有 User Tag 配置

  

查询系统中 Tag 的定义/配置信息，用于展示 Tag 列表、下拉选择等。

  

**Spex Command**: `seller.cs.user.get_cs_user_tag_config_list`

  

**代码位置**: `internal/spex/api/user_tag.go` → `GetCSUserTagConfigList`

  

##### Request

  

```protobuf

message GetCsUserTagConfigListRequest {

repeated int64 tag_id_list = 1; // 可选，指定要查询的 Tag ID 列表；为空则查全部

repeated int32 status_flag = 2; // 可选，过滤状态（1=启用，2=禁用）；为空则返回启用+禁用

optional Pageable pageable = 100; // 可选，分页参数

}

  

message Pageable {

optional int32 page = 1; // 页码，从1开始，默认1

optional int32 size_ = 2; // 每页数量，默认200，最大200

}

```

  

##### Response

  

```protobuf

message GetCsUserTagConfigListResponse {

optional string error_msg = 10000;

repeated CsUserTagConfig cs_user_tag_configs = 1;

optional Pagination pagination = 100;

}

  

message CsUserTagConfig {

optional int64 id = 1; // Tag ID

optional string tag_local_name = 2; // 本地名称（如"VIP用户"）

optional string tag_english_name = 3; // 英文名称（如"VIP User"）

optional int64 priority = 4; // 优先级（数字越大优先级越高）

optional string description = 5; // 描述

optional string promote_message = 6; // 推广消息

repeated string user_type_label = 7; // 用户类型标签（如 "seller", "buyer"）

optional bool visibility = 8; // 是否可见

optional string color_background = 9; // 背景颜色（如"#FF5733"）

optional string color_text = 10; // 文字颜色（如"#FFFFFF"）

optional int32 status_flag = 11; // 状态（1=启用，2=禁用）

}

```

  

##### 处理逻辑

  

1. 参数校验：`pageSize` 不得超过 200

2. 根据 `tag_id_list`、`status_flag` 组合查询条件

3. 查询 `cs_user_tag_rule_define_tab` 表获取 Tag 定义

4. 组装返回结果，包含分页信息

  

##### 异常处理

  

| 场景 | 处理 |

|------|------|

| `pageSize > 200` | 返回 `ERROR_INVALID_PARAMS` |

| DB 查询失败 | 返回 error_msg + 对应错误码 |

| 无匹配数据 | 返回空列表 + pagination.total = 0 |

  

##### 调用示例

  

```json

// Request: 查询所有启用的 Tag 配置

{

"status_flag": [1],

"pageable": { "page": 1, "size_": 50 }

}

  

// Response

{

"cs_user_tag_configs": [

{

"id": 100,

"tag_local_name": "VIP用户",

"tag_english_name": "VIP User",

"priority": 10,

"description": "高价值VIP客户",

"user_type_label": ["seller", "buyer"],

"visibility": true,

"color_background": "#FF5733",

"color_text": "#FFFFFF",

"status_flag": 1

}

],

"pagination": { "page": 1, "size": 50, "total": 12 }

}

```

  

**实现状态**: ✅ 已实现

  

---

  

#### 3.2.2 接口2：根据 UserID 计算 User Tag

  

传入用户 ID，触发 Tag 计算引擎实时计算，返回该用户匹配的 Tag 列表。

  

**Spex Command**: `seller.cs.user.get_cs_user_tag_list`

  

**代码位置**: `internal/spex/api/user_tag.go` → `GetCSUserTagList`

  

##### Request

  

```protobuf

message GetCSUserTagListRequest {

optional int64 cs_user_id = 1; // 必填，CS 用户 ID

optional bool refresh_cs_user_tags_with_rate_limited = 2; // 设为 true，触发实时计算（带限流）

}

```

  

##### Response

  

```protobuf

message GetCSUserTagListResponse {

optional string error_msg = 10000;

repeated CsUserTag tag_list = 1;

}

  

message CsUserTag {

optional int64 id = 1; // Tag ID

optional string english_name = 2; // 英文名称

optional int64 priority = 3; // 优先级

optional string local_name = 4; // 本地名称

}

```

  

##### 处理逻辑

  

整体流程分为三个阶段：**限流检查 → Tag 计算 → 结果组装**。

  

```

GetCSUserTagList(cs_user_id, refresh=true)

│

├─ 阶段1: 限流检查

│ Redis SetNX，key: "user_tag:calc:limit:{cs_user_id}"，TTL: 10s

│ ├─ 获取锁成功 → 进入阶段2

│ └─ 获取锁失败（被限流） → 跳过计算，直接读取 DB 中已有 Tag

│

├─ 阶段2: Tag 计算（CalcUserTags）

│ │

│ ├─ 2.1 获取用户基本信息

│ │ 调用 CsUserServiceImpl.GetCsUserInfo，获取 shopee_user_id, shop_id, region 等

│ │

│ ├─ 2.2 获取可用 DataSource 列表

│ │ 配置: CsUserTagCriteria.AvailableDataSource[tenantID][region]

│ │ 可选 DataSource:

│ │ ├─ UserProfilePlatform (UPP) → 调 UPP 接口检查用户画像 Segment

│ │ ├─ SellerOperationPlatform (SOP) → 调 SOP 接口批量查店铺标签

│ │ ├─ CSUserProfile → 查 CS 系统内用户属性（如 account_type）

│ │ └─ CSSystem → 查 CS 系统内部数据

│ │

│ ├─ 2.3 并发计算

│ │ 每个 DataSource 的 Processor 独立执行:

│ │ Prepare() → 预加载外部数据

│ │ Calc() → 对每个 tag rule 的 condition 逐一评估

│ │

│ ├─ 2.4 聚合结果

│ │ 根据 tag rule 的 MatchLogic 聚合:

│ │ - satisfy_any: 任一 condition 满足即命中

│ │ - satisfy_all: 所有 condition 满足才命中

│ │ - 自定义表达式: 按布尔表达式求值

│ │

│ └─ 2.5 对比现有 Tag → 写入 DB（新增/删除变化的 Tag）

│

└─ 阶段3: 结果组装

查询 Tag 定义 Map → 将 Tag ID 列表转为 CsUserTag 结构返回

```

  

##### 异常处理

  

| 场景 | 处理 |

|------|------|

| 用户被限流（10s 内重复计算） | `isThrottled=true`，跳过计算，读取 DB 已有 Tag 返回 |

| DataSource 配置为空 | 记录错误日志，返回对应错误码 |

| 外部 API 调用失败（UPP/SOP等） | 记录错误日志，该 DataSource 结果视为空，不影响其他 DataSource |

| DB 读写异常 | 返回 error_msg + 错误码 |

  

##### 限流说明

  

| 配置项 | 说明 | 默认值 |

|--------|------|--------|

| Redis Key | `user_tag:calc:limit:{cs_user_id}` | - |

| 锁过期时间 | `CsUserTagCriteria.CalcLimitExpireSecond` | 10s |

| 锁机制 | Redis SetNX（分布式锁） | - |

  

##### 调用示例

  

```json

// Request: 触发计算

{

"cs_user_id": 12345,

"refresh_cs_user_tags_with_rate_limited": true

}

  

// Response（计算完成后返回最新 Tag 列表）

{

"tag_list": [

{ "id": 100, "english_name": "VIP User", "local_name": "VIP用户", "priority": 10 },

{ "id": 103, "english_name": "High Value", "local_name": "高价值客户", "priority": 8 }

]

}

```

  

**实现状态**: ✅ 已实现

  

**备注**:

- DataSource 由服务端配置决定，**当前不支持**调用方在请求中指定 DataSource

- 若后续需要支持调用方指定 DataSource，需在 `GetCSUserTagListRequest` 中新增 `repeated string data_sources` 字段，并在计算逻辑中按该参数过滤

  

---

  

#### 3.2.3 接口3：根据 UserID 查询其全部 Tag

  

查询指定用户当前已有的所有 Tag（不触发重新计算）。这是 Smart-KB / Chatbot 中 **「User ID User Tag」Function** 的后端实现。

  

**Spex Command**: `seller.cs.user.get_cs_user_tag_list`（同接口2，`refresh = false`）

  

**代码位置**: `internal/spex/api/user_tag.go` → `GetCSUserTagList`

  

##### Request

  

```protobuf

message GetCSUserTagListRequest {

optional int64 cs_user_id = 1; // 必填，CS 用户 ID

optional bool refresh_cs_user_tags_with_rate_limited = 2; // 设为 false，仅查询不计算

}

```

  

##### Response

  

同接口2，返回 `tag_list`。

  

##### 处理逻辑

  

1. 根据 `cs_user_id` 查询 DB 中该用户已有的 Tag ID 列表（`GetCsUserTaggedIDs`）

2. 查询 Tag 定义 Map（`GetAllTagDefines`，本地缓存 2 小时）

3. 将 Tag ID 映射为 `CsUserTag` 结构返回

  

##### 与 Smart-KB / Chatbot 对接

  

在 Chatbot Taskflow 的 Issue Condition 和 Solution 中，可选择「User ID User Tag」Function 使用此能力。

  

**Function 配置信息**：

  

| 属性 | 值 |

|------|------|

| API Name | User ID User Tag |

| 旧 API 名 | `sopGetUserTagByCsUserId` |

| SOP 函数路径 | `cs.console.workconsole.get_user_info_by_sop` |

| Request | Chatbot user id / CS user id |

| Return | User Tag list，多个用逗号隔开 |

  

**对接时序**：

  

```

用户进线 / Issue 触发 Condition 检查

│

├─ 1. Chatbot/SOP 获取当前用户 CS User ID

│

├─ 2. 调用 Spex: seller.cs.user.get_cs_user_tag_list

│ Request: { cs_user_id: 12345, refresh_cs_user_tags_with_rate_limited: false }

│

├─ 3. cs/user 服务查询 DB，返回 tag_list

│ Response: { tag_list: [{ id:100, english_name:"VIP User", local_name:"VIP用户" }, ...] }

│

├─ 4. 上游将 tag_list 中的 local_name 拼接为逗号分隔字符串

│ 结果: "VIP用户,高价值客户"

│

└─ 5. Issue Condition 使用该字符串做匹配判断

示例: User ID User Tag 包含 "VIP用户" → 条件满足 → 执行 Solution

```

  

##### 使用场景示例

  

**Issue Condition 中**:

  

```

Condition: User ID User Tag = "VIP用户"

└─ 匹配时执行 Solution（如优先路由到高级客服）

  

Condition: User ID User Tag 包含 "高风险用户"

└─ 匹配时触发风控流程

```

  

**Solution 中**:

  

```

展示当前用户 Tag 信息给客服，辅助判断处理策略

```

  

##### 调用示例

  

```json

// Request: 仅查询

{

"cs_user_id": 12345,

"refresh_cs_user_tags_with_rate_limited": false

}

  

// Response

{

"tag_list": [

{ "id": 100, "english_name": "VIP User", "local_name": "VIP用户", "priority": 10 },

{ "id": 103, "english_name": "High Value", "local_name": "高价值客户", "priority": 8 }

]

}

  

// 上游拼接后的 Function 返回值: "VIP用户,高价值客户"

```

  

**实现状态**: ✅ 已实现

  

---

  

### 3.3 DataSource 配置说明

  

Tag 计算引擎支持以下 DataSource，按 `tenantID + region` 维度配置启用哪些 DataSource：

  

| DataSource | 常量 | 说明 | 外部依赖 |

|------------|------|------|----------|

| UserProfilePlatform | `UserProfilePlatform` | 用户画像平台，检查 UPP Segment | UPP API |

| SellerOperationPlatform | `SellerOperationPlatform` | 卖家运营平台，检查 SOP Private Tag | SOP API |

| CSUserProfile | `CSUserProfile` | CS 系统内用户属性（如 account_type） | 内部 DB |

| CSSystem | `CSSystem` | CS 系统内部数据 | 内部 DB |

  

**配置路径**: `CsUserTagCriteria.AvailableDataSource[tenantID][region]`

  

```json

// 示例配置

{

"available_data_source": {

"1": { // tenantID = 1 (Shopee)

"SG": ["UserProfilePlatform", "SellerOperationPlatform", "CSUserProfile", "CSSystem"],

"ID": ["UserProfilePlatform", "CSUserProfile"],

"TH": ["UserProfilePlatform", "CSUserProfile"]

}

}

}

```

  

---

  

## 4 接口汇总

  

| # | 能力 | Spex Command | 关键参数 | 实现状态 |

|---|------|-------------|----------|---------|

| 1 | 查询所有 Tag 配置 | `seller.cs.user.get_cs_user_tag_config_list` | `tag_id_list`(可选), `status_flag`(可选), `pageable` | ✅ 已实现 |

| 2 | 根据 UserID 计算 Tag | `seller.cs.user.get_cs_user_tag_list` | `cs_user_id`, `refresh=true` | ✅ 已实现 |

| 3 | 根据 UserID 查询全部 Tag | `seller.cs.user.get_cs_user_tag_list` | `cs_user_id`, `refresh=false` | ✅ 已实现 |

  

> 接口 2 和 3 是同一个 Spex Command，通过 `refresh_cs_user_tags_with_rate_limited` 字段区分「触发计算」和「仅查询」行为。

  

---

  

## 5 相关代码位置

  

| 文件 | 说明 |

|------|------|

| `internal/spex/sp_proto/seller.cs/user.proto` | Proto 定义（Request/Response） |

| `internal/spex/api/user_tag.go` | Spex 接口实现（GetCSUserTagList, GetCSUserTagConfigList） |

| `internal/spex/api/handler.go` | Spex Handler 注册 |

| `internal/service/user_tag/user_tag_service.go` | Tag 服务层（GetAllTagDefines, CalcUserTags, GetCsUserTaggedIDs） |

| `internal/service/user_tag/user_tag_calc.go` | Tag 计算引擎（Processor 并发调度） |

| `internal/service/user_tag_condition_processor/` | 各 DataSource 的 Processor 实现 |

| `internal/constant/user_tag.go` | DataSource 类型常量定义 |

| `internal/config/config.go` | GetAvailableDataSource 等配置读取 |

| `internal/http/router/router.go` | HTTP 路由（内部管理接口，非本次对外接口） |

  

---

  

## 6 Checklist

  

### 6.1 待确认事项

  

- [ ] 与 Smart-KB/Chatbot 确认：上游传的是 **CS user id** 还是 **Chatbot user id**（若为后者，需增加 ID 映射逻辑）

- [ ] 与 Smart-KB/Chatbot 确认：Function 返回值格式要求「逗号分隔字符串」还是「结构化列表」（当前返回结构化 tag_list，若需要逗号串可加薄封装或由调用方拼接）

- [ ] 是否需要支持调用方在请求中指定 DataSource（当前由服务端配置 `AvailableDataSource` 决定）

  

### 6.2 上线 Checklist

  

- [ ] 确认各市场（region）的 `AvailableDataSource` 配置已正确设置

- [ ] 确认 Tag 计算限流配置 `CalcLimitExpireSecond` 符合预期（默认 10s）

- [ ] 确认 Chatbot Taskflow 中 `User ID User Tag` Function 已订阅并配置 API 映射

- [ ] Smart-KB Function List 中确认 Function 名称已更新为 `User ID User Tag`

- [ ] 联调验证：上游调用三个接口均能正常返回
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTExNjc3NzkzOTYsMTExMTU3NzUyOCw5ND
kzNDA5ODRdfQ==
-->