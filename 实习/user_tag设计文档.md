# User Tag 系统设计文档

  

## 1. 系统概述

  

### 1.1 背景

  

User Tag系统是CS User服务的核心模块之一，用于对客服系统中的用户进行标签化管理。通过标签，客服人员可以快速识别用户特征、历史行为和风险等级，从而提供更精准的服务。

  

### 1.2 系统目标

  

- **统一标签管理**：提供统一的用户标签存储和管理能力

- **动态标签计算**：支持基于多数据源的规则引擎，动态计算用户标签

- **高性能访问**：通过多级缓存确保标签数据的高效访问

- **灵活扩展**：支持多种数据源和条件类型的扩展

  

### 1.3 核心功能

  

| 功能模块 | 说明 |

|---------|------|

| Inhouse User Tag | 存储外部系统推送的标签数据（如Fraud Tag） |

| CS User Tag Rule | 基于条件规则动态计算的用户标签 |

| Tag定义管理 | 标签元数据的增删改查 |

| Tag计算引擎 | 多数据源并发计算用户标签 |

  

---

  

## 2. 系统架构

  

### 2.1 整体架构图

  

```

┌─────────────────────────────────────────────────────────────────────┐

│ External Systems │

│ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────────┐ │

│ │Anti-Fraud│ │ UPP │ │ SOP │ │ Other Services │ │

│ └────┬─────┘ └────┬─────┘ └────┬─────┘ └──────────┬───────────┘ │

└───────┼─────────────┼─────────────┼────────────────────┼────────────┘

│ │ │ │

▼ ▼ ▼ ▼

┌─────────────────────────────────────────────────────────────────────┐

│ CS User Service │

│ ┌─────────────────────────────────────────────────────────────────┐│

│ │ API Layer ││

│ │ ┌──────────────┐ ┌──────────────┐ ┌──────────────────────┐ ││

│ │ │ HTTP APIs │ │ SPEX APIs │ │ Kafka Consumers │ ││

│ │ └──────┬───────┘ └──────┬───────┘ └──────────┬───────────┘ ││

│ └─────────┼─────────────────┼─────────────────────┼───────────────┘│

│ ▼ ▼ ▼ │

│ ┌─────────────────────────────────────────────────────────────────┐│

│ │ Service Layer ││

│ │ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ││

│ │ │ UserTagService │ │CSUserTagService │ │TagDefineService │ ││

│ │ │ (Inhouse Tag) │ │ (Rule Tag) │ │ (Definition) │ ││

│ │ └────────┬────────┘ └────────┬────────┘ └────────┬────────┘ ││

│ │ │ │ │ ││

│ │ │ ┌────────▼────────┐ │ ││

│ │ │ │ Tag Calc Engine │ │ ││

│ │ │ │ ┌───────────┐ │ │ ││

│ │ │ │ │ Processor │ │ │ ││

│ │ │ │ │ Factory │ │ │ ││

│ │ │ │ └───────────┘ │ │ ││

│ │ │ └─────────────────┘ │ ││

│ └───────────┼─────────────────────────────────────────┼────────────┘│

│ ▼ ▼ │

│ ┌─────────────────────────────────────────────────────────────────┐│

│ │ Data Layer ││

│ │ ┌──────────────┐ ┌──────────────┐ ┌──────────────────────┐ ││

│ │ │ Local Cache │ │ Redis │ │ MySQL │ ││

│ │ │ (2-24h) │ │ (Cache) │ │ (Persistence) │ ││

│ │ └──────────────┘ └──────────────┘ └──────────────────────┘ ││

│ └─────────────────────────────────────────────────────────────────┘│

└─────────────────────────────────────────────────────────────────────┘

```

  

### 2.2 模块划分

  

```

internal/

├── http/service/

│ └── user_tag.go # HTTP Handler

├── spex/api/

│ └── user_tag.go # SPEX API

├── service/

│ ├── cs_user_tag.go # Inhouse Tag服务

│ ├── cs_user_tag_define.go # Tag定义服务

│ └── user_tag/

│ ├── user_tag_service.go # CS Tag Rule服务

│ ├── user_tag_calc.go # Tag计算引擎

│ └── struct.go # 数据结构

├── service/user_tag_condition_processor/

│ ├── processor.go # Processor接口

│ ├── upp.go # UPP处理器

│ ├── sop.go # SOP处理器

│ ├── cs_user_profile.go # CS用户档案处理器

│ └── cs_system.go # CS系统处理器

├── model/dbmodel/

│ ├── user_tag_define_tab.go

│ └── cs_user_tag_rule_define_tab.go

└── repo/

└── user_tag相关仓库

```

  

---

  

## 3. 数据模型

  

### 3.1 数据库表设计

  

#### 3.1.1 user_tag_define_tab (Inhouse Tag定义表)

  

| 字段 | 类型 | 说明 |

|-----|------|------|

| id | BIGINT | 主键 |

| tag_key | VARCHAR(100) | 标签键名，唯一标识 |

| tag_desc | VARCHAR(255) | 标签描述 |

| service | VARCHAR(50) | 服务名称 |

| value_type | VARCHAR(20) | 值类型(string/int/bool) |

| validator | VARCHAR(255) | 验证器表达式 |

| status_flag | TINYINT | 状态(1启用/0禁用/-1删除) |

| region | VARCHAR(10) | 区域 |

| operator | VARCHAR(100) | 操作人 |

| ctime | BIGINT | 创建时间 |

| mtime | BIGINT | 修改时间 |

  

#### 3.1.2 cs_user_tag_rule_define_tab (Tag规则定义表)

  

| 字段 | 类型 | 说明 |

|-----|------|------|

| id | BIGINT | 主键 |

| tag_local_name | VARCHAR(100) | 本地化名称 |

| tag_english_name | VARCHAR(100) | 英文名称 |

| tag_desc | TEXT | 标签描述 |

| priority | BIGINT | 优先级 |

| status_flag | INT | 状态 |

| visibility | INT | 可见性(0隐藏/1显示) |

| color_background | VARCHAR(20) | 背景色 |

| color_text | VARCHAR(20) | 文字色 |

| user_type_label | JSON | 用户类型标签 |

| trigger_condition | JSON | 触发条件 |

| promote_message | TEXT | 提示消息 |

| tenant_id | BIGINT | 租户ID |

| region | VARCHAR(10) | 区域 |

| business_mtime | BIGINT | 业务修改时间 |

| operator | VARCHAR(100) | 操作人 |

| ctime | BIGINT | 创建时间 |

| mtime | BIGINT | 修改时间 |

  

#### 3.1.3 user_field_extend_tab (用户扩展字段表 - 分表)

  

| 字段 | 类型 | 说明 |

|-----|------|------|

| id | BIGINT | 主键 |

| cs_user_id | BIGINT | CS用户ID |

| field_name | VARCHAR(100) | 字段名 |

| field_value | TEXT | 字段值 |

| service | VARCHAR(50) | 服务名 |

| scene | VARCHAR(50) | 场景 |

| region | VARCHAR(10) | 区域 |

| tenant_id | BIGINT | 租户ID |

| agent_id | BIGINT | 操作人ID |

| status_flag | TINYINT | 状态 |

| ctime | INT | 创建时间 |

| mtime | INT | 修改时间 |

  

**分表策略**: 按cs_user_id取模分表

  

### 3.2 核心数据结构

  

#### 3.2.1 触发条件结构 (trigger_condition)

  

```json

{

"match_logic": "satisfy_any | satisfy_all | (c1 && c2) || c3",

"conditions": [

{

"field": "UPP|segment_id",

"operator": 1,

"value": {

"segment_ids": [123, 456]

}

},

{

"field": "SOP|tag_id",

"operator": 1,

"value": {

"tag_ids": [789]

}

}

]

}

```

  

#### 3.2.2 Tag计算用户信息

  

```go

type TagCalcUserInfo struct {

CsUserID int64

Region string

BusinessUserID int64

ShopID int64

BusinessUserType int32

TenantID int64

}

```

  

---

  

## 4. 核心流程

  

### 4.1 Inhouse Tag保存流程

  

```

┌──────────┐ ┌─────────────────┐ ┌──────────────────┐

│ 外部系统 │────▶│ SaveUserTagBatch│────▶│ 校验Tag Key定义 │

└──────────┘ └─────────────────┘ └────────┬─────────┘

│

┌─────────────────┐ ▼

│ 返回更新结果 │◀────┌──────────────────┐

└─────────────────┘ │ 查询现有Tag数据 │

▲ │ (Redis→DB) │

│ └────────┬─────────┘

┌───────┴───────┐ │

│ 删除Redis缓存 │ ▼

└───────────────┘ ┌──────────────────┐

▲ │ 数据对比 │

│ │ - 值是否变化 │

┌───────┴───────┐ │ - 时间戳对比 │

│ DB更新/插入 │◀──────└──────────────────┘

└───────────────┘

```

  

**关键代码**: `cs_user_tag.go#SaveUserTagBatch()`

  

### 4.2 CS User Tag计算流程

  

```

┌──────────────────────────────────────────────────────────────────┐

│ CalcUserTags 主流程 │

└──────────────────────────────────────────────────────────────────┘

│

▼

┌─────────────────┐

│ 限流检查 │ Redis锁(默认10秒)

│ CalcUserTag │

│ LimitCheck │

└────────┬────────┘

│ Pass

▼

┌─────────────────┐

│ 获取用户基础信息 │ GetCsUserInfo

└────────┬────────┘

│

▼

┌─────────────────┐

│ 获取解析好的 │ GetParsedTagDefine

│ Tag规则定义 │ (本地缓存2小时)

└────────┬────────┘

│

▼

┌────────────────────────────────────────┐

│ Tag计算引擎 (并发执行) │

│ ┌─────────┬─────────┬─────────────┐ │

│ │ UPP │ SOP │ CSUserProf │ │

│ │Processor│Processor│ Processor │ │

│ └────┬────┴────┬────┴──────┬──────┘ │

│ │ │ │ │

│ ▼ ▼ ▼ │

│ ┌─────────────────────────────────┐ │

│ │ Prepare (数据准备) │ │

│ │ - RPC调用外部服务 │ │

│ │ - 查询本地数据 │ │

│ └─────────────────────────────────┘ │

│ │ │

│ ▼ │

│ ┌─────────────────────────────────┐ │

│ │ Calc (条件计算) │ │

│ │ - 评估每个condition │ │

│ │ - 返回bool结果 │ │

│ └─────────────────────────────────┘ │

└────────────────────────────────────────┘

│

▼

┌─────────────────┐

│ 条件聚合 │ satisfy_any/all/表达式

└────────┬────────┘

│

▼

┌─────────────────┐

│ 保存计算结果 │ saveNewTagList

│ - 对比差异 │

│ - 事务更新DB │

│ - 清理缓存 │

└─────────────────┘

```

  

### 4.3 Fraud Tag同步流程

  

```

┌─────────────────────────────────────────────────────────────┐

│ Fraud Tag 增量同步 │

└─────────────────────────────────────────────────────────────┘

│

▼

┌─────────────────┐ ┌─────────────────────────┐

│ Hive Kafka消息 │────▶│ FraudTagIncrementalSync │

└─────────────────┘ │ FromHive │

└───────────┬─────────────┘

│

┌────────────────────┼────────────────────┐

│ │ │

▼ ▼ ▼

┌─────────────┐ ┌─────────────────┐ ┌──────────────┐

│检查CS User │ │检查是否已全量 │ │ 触发全量同步 │

│是否存在 │ │同步过 │───▶│ FromSpex │

└─────────────┘ │has_pull_all_ │ └──────────────┘

│fraud_tags │

└────────┬────────┘

│ 已全量同步

▼

┌─────────────────┐

│ 判断Tag是否有效 │

│ FraudTagRecord │

│ IsValid │

└────────┬────────┘

┌──────────┴──────────┐

▼ ▼

┌─────────────┐ ┌─────────────┐

│ 有效: 保存 │ │ 无效: 删除 │

│ SaveUserTag │ │ DeleteUser │

│ Batch │ │ TagByKey │

└──────┬──────┘ └──────┬──────┘

│ │

└──────────┬──────────┘

▼

┌─────────────────────┐

│ 发送Kafka通知 │

│ publishUserInfo │

│ Update │

└─────────────────────┘

```

  

---

  

## 5. Condition Processor设计

  

### 5.1 Processor接口

  

```go

type Processor interface {

// Prepare 数据准备阶段

// - 调用外部RPC获取数据

// - 查询本地数据库

Prepare(ctx context.Context, collectedValue interface{}) (hit bool, err error)

// IsValidDataSource 检查是否需要处理该数据源

IsValidDataSource(ctx context.Context, dataSource UserTagDataSource) bool

// Calc 计算单个condition是否满足

Calc(ctx context.Context, conditionItem *TagConditionForCalcItem) (result bool, err error)

}

```

  

### 5.2 数据源类型

  

| 数据源 | 说明 | Processor |

|-------|------|-----------|

| UPP | User Profile Platform | UserProfilePlatformProcessor |

| SOP | Seller Operation Platform | SellerOperationPlatformProcessor |

| CSUserProfile | CS用户档案 | CsUserProfileProcessor |

| CSSystem | CS系统数据 | CsSystemProcessor |

  

### 5.3 扩展新数据源

  

```go

// 1. 定义数据源常量

const NewDataSource UserTagDataSource = "NEW_SOURCE"

  

// 2. 实现DecodeAndCollectValue函数

func NewSourceDecodeAndCollectValue(ctx context.Context,

criteriaName string,

operator int64,

raw map[string]json.RawMessage,

currentCollectedValue interface{}) (decodedValue, collectValue interface{}, err error) {

// 解析条件值

// 收集需要查询的数据

}

  

// 3. 实现Processor

type NewSourceProcessor struct {

userInfo *servicemodel.TagCalcUserInfo

data interface{}

}

  

func (p *NewSourceProcessor) Prepare(ctx context.Context, collectedValue interface{}) (bool, error) {

// 获取外部数据

}

  

func (p *NewSourceProcessor) Calc(ctx context.Context, item *TagConditionForCalcItem) (bool, error) {

// 计算条件

}

  

// 4. 注册

func register() {

decodeAndCollectValueFuncMap[NewDataSource] = NewSourceDecodeAndCollectValue

processorFactoryMap[NewDataSource] = NewNewSourceProcessor

}

```

  

---

  

## 6. 缓存设计

  

### 6.1 缓存层次

  

```

┌─────────────────────────────────────────────────────────┐

│ 缓存架构 │

├─────────────────────────────────────────────────────────┤

│ L1: 本地缓存 (LocalCache) │

│ ├─ Tag规则定义: 2小时 │

│ ├─ 解析后的条件: 2小时 │

│ └─ 全量Tag定义: 12小时 │

├─────────────────────────────────────────────────────────┤

│ L2: Redis缓存 │

│ ├─ 用户Tag数据: 可配置 │

│ ├─ Tag定义版本号: 24小时 │

│ └─ 用户Tag ID列表: 可配置 │

├─────────────────────────────────────────────────────────┤

│ L3: MySQL数据库 │

│ └─ 持久化存储 │

└─────────────────────────────────────────────────────────┘

```

  

### 6.2 缓存Key设计

  

| 用途 | Key格式 | TTL |

|-----|---------|-----|

| Tag版本号 | `SHOPEE:CS:INHOUSE:USER:TAG_DEFINE:LIST:VERSION:{env}` | 24h |

| 用户Tag数据 | `SHOPEE:CS:INHOUSE:USER:TAG:{cs_user_id}:{tag_key}` | 配置 |

| 用户Tag ID列表 | `cs_user_tag_id:{cs_user_id}` | 配置 |

| 全量Tag定义 | `all_cs_user_tag:{tenant_id}:{region}` | 12h |

| 解析后条件 | `all_parsed_tag_define:{tenant_id}:{region}` | 2h |

  

### 6.3 缓存更新策略

  

**Tag定义更新**:

```

1. 修改数据库

2. 更新Redis版本号 (UUID)

3. 各服务实例检测版本变化

4. 触发本地缓存更新

```

  

**用户Tag数据更新**:

```

1. 修改数据库

2. 删除Redis缓存

3. 下次读取时重建缓存

```

  

---

  

## 7. API接口

  

### 7.1 HTTP接口

  

#### 添加Tag规则

```

POST /api/user-tag/define

Request:

{

"tag_local_name": "VIP用户",

"tag_english_name": "VIP_USER",

"description": "VIP用户标签",

"status_flag": 1,

"visibility": true,

"color_background": "#FFD700",

"color_text": "#000000",

"user_type_label": ["buyer", "seller"],

"trigger_condition": {...},

"promote_message": "这是VIP用户"

}

Response:

{

"code": 0,

"data": {"is_success": true}

}

```

  

#### 更新Tag规则

```

PUT /api/user-tag/define/{id}

```

  

#### 启用/禁用Tag

```

POST /api/user-tag/define/able

Request: {"id": 123, "status_flag": 1}

```

  

#### 获取Tag列表

```

GET /api/user-tag/define/list?page=1&size=20

```

  

### 7.2 SPEX接口

  

#### GetCSUserTagList

```protobuf

message GetCSUserTagListRequest {

int64 cs_user_id = 1;

bool refresh_cs_user_tags_with_rate_limited = 2;

}

  

message GetCSUserTagListResponse {

repeated CsUserTag tag_list = 1;

string error_msg = 2;

}

```

  

#### GetCSUserTagConfigList

```protobuf

message GetCsUserTagConfigListRequest {

repeated int64 tag_id_list = 1;

repeated int32 status_flag = 2;

Pageable pageable = 3;

}

  

message GetCsUserTagConfigListResponse {

repeated CsUserTagConfig cs_user_tag_configs = 1;

Pagination pagination = 2;

}

```

  

---

  

## 8. 性能优化

  

### 8.1 并发计算

  

- Tag计算引擎并发处理多个数据源

- 使用WaitGroup和Channel协调

- 单个数据源超时不影响整体计算

  

### 8.2 限流控制

  

- 用户维度限流: 同一用户10秒内只计算一次

- 使用Redis锁实现分布式限流

  

### 8.3 批量操作

  

- 批量读取Tag: Redis MGET

- 批量写入: 数据库事务批量INSERT

  

### 8.4 预编译优化

  

- 复杂表达式在加载时预编译

- 运行时直接执行编译后的表达式

  

---

  

## 9. 监控指标

  

| 指标名 | 类型 | 说明 |

|-------|------|------|

| TagCalcRequestDuration | Histogram | 计算请求耗时 |

| TagCalcPrepareDuration | Histogram | 数据准备耗时 |

| TagCalcCalcSummary | Summary | 计算耗时统计 |

| InhouseUserTagCacheCounter | Counter | 缓存命中统计 |

| InhouseUserTagDBReadCounter | Counter | 数据库读取次数 |

| InhouseUserTagDBUpdateCounter | Counter | 数据库更新次数 |

  

---

  

## 10. 配置项

  

```yaml

cs_user_tag_criteria:

enable: true # 是否启用Tag计算

calc_limit_expire_second: 10 # 限流过期时间(秒)

  

user_tag_segment_limit:

limit_check_enable: true # 是否启用数量限制

limit_check_values:

UPP|segment_id: 100 # UPP segment最大数量

SOP|tag_id: 30 # SOP tag最大数量

  

feature_switch:

enable_shopee_fraud_tag_change_notice: true # Fraud Tag变更通知

always_full_sync_fraud_tag: false # 总是全量同步

```

  

---

  

## 11. 错误处理

  

| 错误码 | 说明 | 处理方式 |

|-------|------|---------|

| ErrCodeCSUserNotFound | CS用户不存在 | 跳过处理 |

| ErrCodeNotFound | Tag定义不存在 | 返回错误 |

| ErrInvalidParams | 参数无效 | 返回错误 |

| ErrDbOperate | 数据库操作失败 | 记录日志,返回错误 |

| ErrRedisOperate | Redis操作失败 | 降级处理 |

  

---

  

## 12. 附录

  

### 12.1 相关服务

  

- **Anti-Fraud Service**: 提供Fraud Tag数据

- **User Profile Platform (UPP)**: 提供用户画像Segment

- **Seller Operation Platform (SOP)**: 提供卖家运营Tag

- **DataSync Service**: 接收用户信息变更通知

  

### 12.2 相关Topic

  

- `user_info_change_topic`: 用户信息变更通知

  

### 12.3 文档版本

  

| 版本 | 日期 | 作者 | 说明 |

|-----|------|------|------|

| 1.0 | 2026-02-04 | System | 初始版本 |
<!--stackedit_data:
eyJoaXN0b3J5IjpbMzk3NjQ4MDA5XX0=
-->