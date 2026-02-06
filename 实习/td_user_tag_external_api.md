
# TD: User Tag 对外接口能力

  

## 1. 背景

  

User 项目需要向外部系统（Smart-KB、Chatbot Taskflow、SOP Issue/Solution 等）提供 User Tag 相关能力。经分析，需要对外暴露三个核心接口：

  

1 | 查询所有 User Tag 配置 | 上游获取全部 Tag 定义列表（如名称、优先级、颜色等），用于展示或下拉选择

2 | 根据 UserID 计算 User Tag | 根据用户 ID 实时触发 Tag 计算（基于多个 DataSource 并发计算），返回计算后的 Tag 列表

3 | 根据 UserID 查询其全部 Tag | 查询指定用户当前已有的所有 Tag，用于 Issue Condition 判断、Solution 展示等

  
  
  

## 2. 接口详细设计

  

### 2.1 接口1：查询所有 User Tag 配置

  

**Spex Command**: `seller.cs.user.get_cs_user_tag_config_list`

  

**实现位置**: `internal/spex/api/user_tag.go` → `GetCSUserTagConfigList`

  

**用途**: 上游获取系统中全部 Tag 的定义/配置信息，用于展示 Tag 列表、下拉选择等。

  

#### Request

  

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

  

#### Response

  

```protobuf

message GetCsUserTagConfigListResponse {

	optional string error_msg = 10000;

	repeated CsUserTagConfig cs_user_tag_configs = 1;

	optional Pagination pagination = 100;

}

  

message CsUserTagConfig {

	optional int64 id = 1; // Tag ID

	optional string tag_local_name = 2; // 本地名称

	optional string tag_english_name = 3; // 英文名称

	optional int64 priority = 4; // 优先级

	optional string description = 5; // 描述

	optional string promote_message = 6; // 推广消息

	repeated string user_type_label = 7; // 用户类型标签（如 "seller", "buyer"）

	optional bool visibility = 8; // 是否可见

	optional string color_background = 9; // 背景颜色

	optional string color_text = 10; // 文字颜色

	optional int32 status_flag = 11; // 状态（1=启用，2=禁用）

}

  

message Pagination {

	optional int32 page = 1;

	optional int32 size = 2;

	optional int64 total = 3;

}

```

  

#### 调用示例

  

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

		"visibility": true,

		"color_background": "#FF5733",

		"color_text": "#FFFFFF",

		"status_flag": 1

	}

	],

	"pagination": { "page": 1, "size": 50, "total": 12 }

}

```

  

#### 实现状态: 已实现

  

---

  

### 2.2 接口2：根据 UserID 计算 User Tag

  

**Spex Command**: `seller.cs.user.get_cs_user_tag_list`

  

**实现位置**: `internal/spex/api/user_tag.go` → `GetCSUserTagList`

  

**用途**: 传入用户 ID，触发 Tag 计算引擎实时计算（或直接读取已有数据），返回该用户匹配的 Tag 列表。

  

#### Request

  

```protobuf

message GetCSUserTagListRequest {

	optional int64 cs_user_id = 1; // 必填，CS 用户 ID

	optional bool refresh_cs_user_tags_with_rate_limited = 2; // 是否触发实时计算

	// true = 触发计算（带限流，同一用户 10s 内只计算一次）

	// false = 仅读取已有 tag（等同接口3）

}

```

  

#### Response

  

```protobuf

message GetCSUserTagListResponse {

	optional string error_msg = 10000;

	repeated CsUserTag tag_list = 1; // 用户的 Tag 列表

}

  

message CsUserTag {

	optional int64 id = 1; // Tag ID

	optional string english_name = 2; // 英文名称

	optional int64 priority = 3; // 优先级

	optional string local_name = 4; // 本地名称

}

```

#### 调用示例

  

```json

// Request: 触发计算

{

	"cs_user_id": 12345,

	"refresh_cs_user_tags_with_rate_limited": true

}

  

// Response

{

	"tag_list": [

		{ "id": 100, "english_name": "VIP User", "local_name": "VIP用户", "priority": 10 },

		{ "id": 103, "english_name": "High Value", "local_name": "高价值客户", "priority": 8 }

	]

}

```

  

#### 实现状态: 已实现

  

**备注**:

- DataSource 由服务端配置决定，**当前不支持**调用方在请求中指定 DataSource。

- 若后续需要支持调用方指定 DataSource，需在 `GetCSUserTagListRequest` 中新增 `repeated string data_sources` 字段，并在 `CalcUserTags` 中按该参数过滤。

  

---

  

### 2.3 接口3：根据 UserID 查询其全部 Tag

  

**Spex Command**: `seller.cs.user.get_cs_user_tag_list`（同接口2，`refresh = false`）

  

**实现位置**: `internal/spex/api/user_tag.go` → `GetCSUserTagList`

  

**用途**: 查询指定用户当前已有的所有 Tag（不触发重新计算），用于：

- Smart-KB / Chatbot 的 **「User ID User Tag」Function** 在 Issue Condition 和 Solution 中使用

  

#### Request

  

```protobuf

message GetCSUserTagListRequest {

optional int64 cs_user_id = 1; // 必填，CS 用户 ID

optional bool refresh_cs_user_tags_with_rate_limited = 2; // 设为 false，仅查询不计算

}

```

  

#### Response

  

同接口2，返回 `tag_list`。

  

#### 调用示例

  

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

```

  

#### 与 Smart-KB / Chatbot 对接说明

  

根据文档，「User ID User Tag」Function 的对外约定为：

  

| API Name | Request | Return | Description |

|----------|---------|--------|-------------|

| User ID User Tag | Chatbot user id / CS user id | User Tag list | 返回 user tag list，多个用逗号隔开 |

  

**对接方式**:

- 上游（workconsole / chatbot）调用 Spex `get_cs_user_tag_list`，传入 `cs_user_id`

- 拿到 `tag_list` 后，将 `local_name`（或 `english_name`）拼接为逗号分隔字符串，即为 Function 的返回值

- 示例: `"VIP用户,高价值客户"`

  

#### 实现状态: 已实现

  

---

  

## 3. 接口汇总

  

| # | 能力 | Spex Command | 关键参数 | 实现状态 |

|---|------|-------------|----------|---------|

| 1 | 查询所有 Tag 配置 | `seller.cs.user.get_cs_user_tag_config_list` | `tag_id_list`(可选), `status_flag`(可选), `pageable` | 已实现 |

| 2 | 根据 UserID 计算 Tag | `seller.cs.user.get_cs_user_tag_list` | `cs_user_id`, `refresh_cs_user_tags_with_rate_limited=true` | 已实现 |

| 3 | 根据 UserID 查询全部 Tag | `seller.cs.user.get_cs_user_tag_list` | `cs_user_id`, `refresh_cs_user_tags_with_rate_limited=false` | 已实现 |

  

> 注：接口 2 和 3 是同一个 Spex Command，通过 `refresh_cs_user_tags_with_rate_limited` 字段区分「触发计算」和「仅查询」行为。

  
  

---

  

## 4. 待确认事项

  

- [ ] 与 Smart-KB/Chatbot 确认：上游传的是 **CS user id** 还是 **Chatbot user id**（若为后者，需增加 ID 映射逻辑）

- [ ] 与 Smart-KB/Chatbot 确认：Function 返回值格式要求「逗号分隔字符串」还是「结构化列表」（当前返回结构化 tag_list，若需要逗号串可加薄封装）

- [ ] 是否需要支持调用方在请求中指定 DataSource（当前由服务端配置决定）


<!--stackedit_data:
eyJoaXN0b3J5IjpbOTQ5MzQwOTg0XX0=
-->