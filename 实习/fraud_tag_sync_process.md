# Fraud Tag 同步流程详解

  

## 1. 概述

  

Fraud Tag同步系统负责将Anti-Fraud服务的风控标签同步到CS User系统，支持**全量同步**和**增量同步**两种模式。该系统确保客服人员能够实时看到用户的风险标签，从而提供更精准的服务。

  

### 核心特点

  

| 特点 | 说明 |

|-----|------|

| **双模式同步** | 支持全量同步(Spex RPC)和增量同步(Hive Kafka) |

| **自动降级** | 增量同步前检查全量同步标记，未全量同步会自动降级为全量同步 |

| **实时通知** | Tag变更后通过Kafka通知DataSync服务 |

| **防御性编程** | 完善的错误处理和容错机制 |

| **动态Tag定义** | 自动创建未定义的Fraud Tag |

  

---

  

## 2. 数据源架构

  

```

┌──────────────────────────────────────────────────────────────┐

│ Anti-Fraud System │

│ ┌────────────────┐ ┌─────────────────────────┐ │

│ │ Fraud Engine │─────────────▶│ Hive Database │ │

│ │ (实时检测) │ │ (fraud_tag_shopee_sg) │ │

│ └────────┬───────┘ └──────────┬──────────────┘ │

└───────────┼──────────────────────────────────┼───────────────┘

│ │

│ Spex RPC │ Kafka

│ (全量) │ (增量)

▼ ▼

┌──────────────────────────────────────────────────────────────┐

│ CS User Service │

│ ┌────────────────────────────────────────────────────────┐ │

│ │ FraudTagFullSyncFromSpex FraudTagIncrementalSync │ │

│ │ (GetEntityFraudTags) FromHive │ │

│ └────────────────────────────────────────────────────────┘ │

│ │ │

│ ▼ │

│ ┌────────────────────────────────────────────────────────┐ │

│ │ SaveUserTagBatch / DeleteUserTagByKeyBatch │ │

│ └────────────────────────────────────────────────────────┘ │

│ │ │

│ ▼ │

│ ┌────────────────────────────────────────────────────────┐ │

│ │ user_field_extend_tab (分表存储) │ │

│ └────────────────────────────────────────────────────────┘ │

│ │ │

│ ▼ │

│ ┌────────────────────────────────────────────────────────┐ │

│ │ FraudTagUpdateSendKafka │ │

│ │ (通知DataSync) │ │

│ └────────────────────────────────────────────────────────┘ │

└──────────────────────────────────────────────────────────────┘

│

▼

┌──────────────────────────────────────────────────────────────┐

│ DataSync Service │

│ (消费user_info_change_topic，同步到其他系统) │

└──────────────────────────────────────────────────────────────┘

```

  

---

  

## 3. 全量同步流程：FraudTagFullSyncFromSpex

  

### 3.1 函数签名

  

```go

// internal/service/cs_user_tag.go

func (s *UserTagService) FraudTagFullSyncFromSpex(

ctx context.Context,

csUserID int64, // CS用户ID

region string, // CS用户所在region

grassRegion string, // Anti-Fraud Hive表所在的region

shopeeUserID int64, // Shopee用户ID

) (savedTags []*seller_cs_user.InhouseUserTag, err error)

```

  

**调用场景**：

1. 用户首次进线时

2. 增量同步检测到未全量同步时

3. 配置强制全量同步时(`AlwaysFullSyncFraudTag=true`)

  

---

  

### 3.2 完整流程分解

  

#### 阶段1：调用Anti-Fraud Spex接口（第471-475行）

  

```go

log.WithCtx(ctx).Infof("FraudTagFullSyncFromSpex, cs user id:%d, shopee user id:%d", csUserID, shopeeUserID)

  

// 🔥 调用Anti-Fraud Service的Spex接口

entityFraudTagsResponse, err := anti_fraud_tag.GetEntityFraudTags(ctx, shopeeUserID, grassRegion)

if err != nil {

log.WithCtx(ctx).Errorf("GetEntityFraudTags spex err:%s", err.Error())

return savedTags, err

}

```

  

**GetEntityFraudTags接口**：

```protobuf

message GetEntityFraudTagsRequest {

int64 entity_id = 1; // shopee_user_id

string grass_region = 2; // 区域

}

  

message GetEntityFraudTagsResponse {

repeated FraudTag tags = 1;

}

  

message FraudTag {

uint32 id = 1; // fraud_tag_id

string tag_name = 2; // 标签名称（如"可疑交易"）

string type = 3; // 类型

string severity = 4; // 严重程度

int64 mtime = 5; // 修改时间戳

int64 tag_expiry_timestamp = 6; // 过期时间

}

```

  

**返回示例**：

```json

{

"tags": [

{

"id": 1001,

"tag_name": "可疑交易",

"type": "fraud_transaction",

"severity": "high",

"mtime": 1738732800,

"tag_expiry_timestamp": 0

},

{

"id": 1002,

"tag_name": "账号异常",

"type": "account_risk",

"severity": "medium",

"mtime": 1738732900,

"tag_expiry_timestamp": 1740000000

}

]

}

```

  

---

  

#### 阶段2：检查并创建Tag定义（第480-498行）

  

```go

saveTags := []*seller_cs_user.SaveUserTagRequest{}

saveTagsMap := make(map[string]*seller_cs_user.SaveUserTagRequest)

  

// 遍历所有Fraud Tag

for _, fraudTag := range entityFraudTagsResponse.GetTags() {

// 1. 生成Tag Key: "fraud_tag_1001"

fraudTagKey := s.getFraudTagKey(fraudTag.GetId())

// 2. 检查Tag定义是否存在，不存在则创建

err = s.CreateFraudTagDefineIfNotExist(ctx, fraudTagKey, grassRegion, fraudTag.GetTagName())

if err != nil {

log.WithCtx(ctx).Errorf("CreateFraudTagDefineIfNotExist err:%s", err.Error())

return savedTags, err

}

// 3. 构建SaveUserTagRequest

elems := &seller_cs_user.SaveUserTagRequest{

TagKey: proto.String(fraudTagKey), // "fraud_tag_1001"

TagValue: proto.String(utils.JsonString(s.spexFraudTagToShopeeFraudTagInfo(fraudTag))),

Service: proto.String(constant.TagServiceNameFraud), // "ops-cs"

Scene: proto.String(seller_cs_user.Constant_SPEX_UPDATE.String()), // "SPEX_UPDATE"

AgentId: proto.Int64(constant.AgentIDSystem), // 1

TagValueUpdateTimestamp: proto.Int64(int64(fraudTag.GetMtime())),

}

saveTags = append(saveTags, elems)

saveTagsMap[fraudTagKey] = elems

}

```

  

**getFraudTagKey函数**：

```go

func (s *UserTagService) getFraudTagKey(fraudTagID uint32) string {

return fmt.Sprintf("%s%d", constant.FraudTagKeyPrefix, fraudTagID)

// FraudTagKeyPrefix = "fraud_tag_"

// 结果: "fraud_tag_1001"

}

```

  

**CreateFraudTagDefineIfNotExist函数（第758-775行）**：

```go

func (s *UserTagService) CreateFraudTagDefineIfNotExist(ctx context.Context, tagKey string, region string, fraudTagName string) error {

// 1. 检查Tag定义是否存在

_, errInfo := TagDefineServiceImpl.GetInfoByTagKey(ctx, tagKey)

if errInfo != nil {

if errInfo.BizErrCode == bizerr.ErrCodeNotFound {

// 2. 不存在，创建Tag定义

log.WithCtx(ctx).Infof("add fraud tag:%s,%s,%s", tagKey, region, fraudTagName)

err := TagDefineServiceImpl.AddFraudTag(ctx, tagKey, region, fraudTagName)

if err != nil {

log.WithCtx(ctx).Errorf("add fraud tag error:%s", err.Error())

return bizerr.ErrInternal.Wrap(err)

}

} else {

return errInfo

}

}

return nil

}

```

  

**插入user_tag_define_tab表**：

```sql

INSERT INTO user_tag_define_tab (

tag_key, tag_desc, service, value_type, status_flag, region

) VALUES (

'fraud_tag_1001', '可疑交易', 'ops-cs', 'string', 1, 'sg'

);

```

  

**Tag Value格式（spexFraudTagToShopeeFraudTagInfo）**：

```json

{

"fraud_tag_id": 1001,

"fraud_tag_name": "可疑交易",

"type": "fraud_transaction",

"severity": "high",

"tag_expiry_timestamp": 0,

"create_timestamp": 1738732800,

"modify_timestamp": 1738732800

}

```

  

---

  

#### 阶段3：添加全量同步标记（第500-508行）

  

```go

// 添加 has_pull_all_fraud_tags 标记

saveTags = append(saveTags, &seller_cs_user.SaveUserTagRequest{

TagKey: proto.String(constant.FraudHasPullAllFraudTags), // "has_pull_all_fraud_tags"

TagValue: proto.String(constant.TagBoolValueTrue), // "true"

Service: proto.String(constant.TagServiceNameCsUser), // "cs_user"

Scene: proto.String(seller_cs_user.Constant_SPEX_UPDATE.String()),

AgentId: proto.Int64(constant.AgentIDSystem),

TagValueUpdateTimestamp: nil,

})

```

  

**作用**：

- 标记该用户已完成全量同步

- 增量同步时会检查这个标记

- `has_pull_all_fraud_tags = "true"` 表示已全量同步

  

---

  

#### 阶段4：删除失效的Fraud Tag（第510-530行）

  

```go

// 获取该region下所有Fraud Tag定义

tagDefineTabs, errInfo := TagDefineServiceImpl.GetByServiceName(ctx, constant.TagServiceNameFraud, grassRegion)

if errInfo != nil {

log.WithCtx(ctx).Errorf("GetByServiceName err:%s", errInfo.Error())

return savedTags, errInfo

}

  

// 找出需要删除的Tag（在数据库中有定义，但Spex接口未返回）

deleteKeyList := []string{}

for _, tagDefineTab := range tagDefineTabs {

if _, has := saveTagsMap[tagDefineTab.TagKey]; !has {

// 该Tag在Spex返回中不存在，说明已失效，需要删除

deleteKeyList = append(deleteKeyList, tagDefineTab.TagKey)

}

}

  

var deletedCount int64

if len(deleteKeyList) > 0 {

deletedCount, errInfo = s.DeleteUserTagByKeyBatch(ctx, csUserID, region, deleteKeyList)

if errInfo != nil {

log.WithCtx(ctx).Errorf("DeleteUserTagByKeyBatch err:%s", errInfo.Error())

return savedTags, errInfo

}

}

  

log.WithCtx(ctx).Infof("FraudTagFullSyncFromSpex, delete tag count:%d", deletedCount)

```

  

**场景举例**：

```

数据库中的Fraud Tag定义: ["fraud_tag_1001", "fraud_tag_1002", "fraud_tag_1003"]

Spex接口返回的Tag: ["fraud_tag_1001", "fraud_tag_1004"]

  

需要删除的Tag: ["fraud_tag_1002", "fraud_tag_1003"] （已失效）

需要保存的Tag: ["fraud_tag_1001", "fraud_tag_1004"] （1001更新，1004新增）

```

  

---

  

#### 阶段5：批量保存Tag（第534-541行）

  

```go

// 批量保存Tag（包括Fraud Tag和全量同步标记）

hasUpdateTag, errInfo := s.SaveUserTagBatch(ctx, csUserID, region, saveTags)

if errInfo != nil {

log.WithCtx(ctx).Errorf("SaveUserTagBatch error:%#v, cs user id:%d", errInfo.Error(), csUserID)

return savedTags, errInfo

}

  

log.WithCtx(ctx).Infof("FraudTagFullSyncFromSpex, update tag count:%d", len(hasUpdateTag))

```

  

**SaveUserTagBatch内部逻辑**：

1. 查询现有Tag数据（Redis → DB）

2. 时间戳对比（避免旧数据覆盖新数据）

3. 值对比（只更新变化的Tag）

4. 数据库UPDATE/INSERT

5. 删除Redis缓存

  

---

  

#### 阶段6：发送Kafka通知（第543-546行）

  

```go

// 如果有Tag变更，发送Kafka通知

if deletedCount > 0 || len(hasUpdateTag) > 0 {

s.FraudTagUpdateSendKafka(ctx, csUserID, region, grassRegion, shopeeUserID)

}

```

  

---

  

#### 阶段7：返回保存的Tag（第547-556行）

  

```go

// 构建返回结果（排除全量同步标记）

for i, _ := range saveTags {

if saveTags[i].GetTagKey() == constant.FraudHasPullAllFraudTags {

continue // 跳过全量同步标记

}

savedTags = append(savedTags, &seller_cs_user.InhouseUserTag{

TagKey: saveTags[i].TagKey,

TagValue: saveTags[i].TagValue,

})

}

return savedTags, nil

```

  

---

  

## 4. 增量同步流程：FraudTagIncrementalSyncFromHive

  

### 4.1 函数签名

  

```go

// internal/service/cs_user_tag.go

func (s *UserTagService) FraudTagIncrementalSyncFromHive(

ctx context.Context,

data servicemodel.HiveFraudTag, // Hive Kafka消息数据

) (execResultMsg string, err error)

```

  

**HiveFraudTag数据结构**：

```go

type HiveFraudTag struct {

UserId int64 `json:"user_id"` // Shopee User ID

FraudTagId uint32 `json:"fraud_tag_id"` // Fraud Tag ID

CreateTimestamp int64 `json:"create_timestamp"` // 创建时间

ModifyTimestamp int64 `json:"modify_timestamp"` // 修改时间

Type string `json:"type"` // 类型

Reason string `json:"reason"` // 原因（对应tag_name）

Severity string `json:"severity"` // 严重程度

TagExpiryTimestamp int64 `json:"tag_expiry_timestamp"` // 过期时间

IsActiveUserTag string `json:"is_active_user_tag"` // 是否生效("1"=生效)

GrassRegion string `json:"grass_region"` // 区域

}

```

  

**Kafka消息示例**：

```json

{

"user_id": 888888,

"fraud_tag_id": 1001,

"create_timestamp": 1738732800,

"modify_timestamp": 1738732900,

"type": "fraud_transaction",

"reason": "可疑交易",

"severity": "high",

"tag_expiry_timestamp": 0,

"is_active_user_tag": "1",

"grass_region": "sg"

}

```

  

---

  

### 4.2 完整流程分解

  

#### 阶段1：检查CS用户是否存在（第604-613行）

  

```go

log.WithCtx(ctx).Infof("start FraudTagIncrementalSyncFromHive [%#v]", data)

  

// 根据ShopeeUserID查询CSUserID

csUserID, csUserRegion, errInfo := s.getCsUserID(ctx, data.UserId, data.GrassRegion)

if errInfo != nil {

if errInfo.BizErrCode == bizerr.ErrCodeCSUserNotFound {

execResultMsg = "cs user not found"

return execResultMsg, nil // CS用户不存在，直接返回（正常情况）

}

execResultMsg = errInfo.Error()

return execResultMsg, errInfo.Err

}

```

  

**getCsUserID实现**：

```go

func (s *UserTagService) getCsUserID(ctx context.Context, shopeeUserID int64, region string) (csUserID int64, csUserRegion string, errInfo *errinfo.ErrInfo) {

// 查询cs_user表

csUser, errInfo := CsUserServiceImpl.GetCsUserByShopeeUserID(ctx, shopeeUserID, region)

if errInfo != nil {

return 0, "", errInfo

}

return csUser.ID, csUser.Region, nil

}

```

  

---

  

#### 阶段2：检查强制全量同步开关（第616-619行）

  

```go

// 配置检查：是否总是执行全量同步

if config.GetFeatureSwitch().AlwaysFullSyncFraudTag {

_, err := s.FraudTagFullSyncFromSpex(ctx, csUserID, csUserRegion, data.GrassRegion, data.UserId)

return execResultMsg, err

}

```

  

**配置项**：

```yaml

feature_switch:

always_full_sync_fraud_tag: false # 是否总是全量同步

```

  

**使用场景**：

- 调试阶段或数据质量不稳定时

- 临时开启以修复数据不一致问题

  

---

  

#### 阶段3：检查全量同步标记（第620-635行）

  

```go

// 检查 has_pull_all_fraud_tags 标记

res, errInfo := s.GetUserTagByKeyBatch(ctx, csUserID, []string{constant.FraudHasPullAllFraudTags}, false)

if errInfo != nil {

log.WithCtx(ctx).Errorf("get has_pull_all_fraud_tags tag err:%#v", errInfo)

return execResultMsg, errInfo.Err

}

  

if len(res) != 1 {

err = fmt.Errorf("unexpected tag result length:%d", len(res))

log.WithCtx(ctx).Errorf(err.Error())

return execResultMsg, err

}

  

if res[0].GetNotExist() || res[0].GetTagValue() != constant.TagBoolValueTrue {

// ⚠️ 未进行过全量同步，需要先全量同步

log.WithCtx(ctx).Infof("has_pull_all_fraud_tags not exist or false, do full sync")

_, err := s.FraudTagFullSyncFromSpex(ctx, csUserID, csUserRegion, data.GrassRegion, data.UserId)

return execResultMsg, err

}

```

  

**降级逻辑**：

```

检查全量同步标记

├─ 标记不存在 → 触发全量同步

├─ 标记值 != "true" → 触发全量同步

└─ 标记值 == "true" → 继续增量同步

```

  

**为什么需要全量同步标记？**

- 确保数据完整性：增量同步只能处理新增/变更，无法处理删除

- 防止数据丢失：用户首次进线时必须先全量同步

- 数据一致性：增量同步依赖全量同步的基准数据

  

---

  

#### 阶段4：判断Tag是否有效（第640-641行）

  

```go

log.WithCtx(ctx).Infof("FraudTagIncrementalSync")

  

tagKey := s.getFraudTagKey(data.FraudTagId) // "fraud_tag_1001"

isValid := s.FraudTagRecordIsValid(data)

```

  

**FraudTagRecordIsValid函数（第778-788行）**：

```go

func (s *UserTagService) FraudTagRecordIsValid(data servicemodel.HiveFraudTag) bool {

// 检查1: is_active_user_tag必须为"1"

if data.IsActiveUserTag != constant.FraudTagIsActiveUserTagTrueValue {

return false // "0"表示已失效

}

// 检查2: 是否过期

if data.TagExpiryTimestamp > 0 && data.TagExpiryTimestamp <= time.Now().Unix() {

// 有过期时间，且已过期

return false

}

return true

}

```

  

**有效性判断逻辑**：

```

Tag有效条件:

1. is_active_user_tag == "1"

AND

2. (

tag_expiry_timestamp == 0 (无过期时间)

OR

tag_expiry_timestamp > NOW() (未过期)

)

```

  

**示例**：

```json

// 有效Tag

{"is_active_user_tag": "1", "tag_expiry_timestamp": 0} ✅

{"is_active_user_tag": "1", "tag_expiry_timestamp": 2000000000} ✅ (未来时间)

  

// 无效Tag

{"is_active_user_tag": "0", "tag_expiry_timestamp": 0} ❌ (已失效)

{"is_active_user_tag": "1", "tag_expiry_timestamp": 1000000000} ❌ (已过期)

```

  

---

  

#### 阶段5A：Tag有效 - 保存Tag（第644-671行）

  

```go

hasUpdate := false

  

if isValid {

// ========== Tag有效，保存/更新 ==========

// 1. 检查并创建Tag定义

err = s.CreateFraudTagDefineIfNotExist(ctx, tagKey, data.GrassRegion, data.Reason)

if err != nil {

log.WithCtx(ctx).Errorf("CreateFraudTagDefineIfNotExist err:%s", err)

return execResultMsg, err

}

// 2. 构建SaveUserTagRequest

saveReq := &seller_cs_user.SaveUserTagRequest{

TagKey: proto.String(tagKey), // "fraud_tag_1001"

TagValue: proto.String(utils.JsonString(s.hiveFraudTagToShopeeFraudTagInfo(data))),

Service: proto.String(constant.TagServiceNameFraud),

Scene: proto.String(seller_cs_user.Constant_SYNC_FROM_HIVE.String()), // "SYNC_FROM_HIVE"

AgentId: proto.Int64(constant.AgentIDSystem),

TagValueUpdateTimestamp: proto.Int64(data.ModifyTimestamp),

}

// 3. 保存Tag

hasUpdateTag, errInfo := s.SaveUserTagBatch(ctx, csUserID, csUserRegion, []*seller_cs_user.SaveUserTagRequest{saveReq})

if errInfo != nil {

log.WithCtx(ctx).Errorf("save inhouse tag err:%s", errInfo.Error())

return execResultMsg, errInfo

}

// 4. 标记是否有更新

if len(hasUpdateTag) > 0 {

hasUpdate = true

}

}

```

  

**hiveFraudTagToShopeeFraudTagInfo（第559-569行）**：

```go

func (s *UserTagService) hiveFraudTagToShopeeFraudTagInfo(data servicemodel.HiveFraudTag) *user_kafka.ShopeeFraudTagInfo {

return &user_kafka.ShopeeFraudTagInfo{

FraudTagId: proto.Uint32(data.FraudTagId),

FraudTagName: proto.String(data.Reason),

Type: proto.String(data.Type),

Severity: proto.String(data.Severity),

TagExpiryTimestamp: proto.Int64(data.TagExpiryTimestamp),

CreateTimestamp: proto.Int64(data.CreateTimestamp),

ModifyTimestamp: proto.Int64(data.ModifyTimestamp),

}

}

```

  

---

  

#### 阶段5B：Tag无效 - 删除Tag（第672-683行）

  

```go

} else {

// ========== Tag无效，删除 ==========

// 1. 删除Tag

deletedCount, errInfo := s.DeleteUserTagByKeyBatch(ctx, csUserID, csUserRegion, []string{tagKey})

if errInfo != nil {

log.WithCtx(ctx).Errorf("delete inhouse tag err:%s", errInfo.Error())

return execResultMsg, errInfo

}

// 2. 标记是否有更新

if deletedCount > 0 {

hasUpdate = true

}

}

```

  

**DeleteUserTagByKeyBatch实现**：

```sql

DELETE FROM user_field_extend_tab_X

WHERE cs_user_id = 12345

AND field_name IN ('fraud_tag_1001')

AND service = 'ops-cs'

```

  

---

  

#### 阶段6：发送Kafka通知（第685-688行）

  

```go

// 如果有更新，发送Kafka通知

if hasUpdate {

s.FraudTagUpdateSendKafka(ctx, csUserID, csUserRegion, data.GrassRegion, data.UserId)

}

  

return execResultMsg, err

```

  

---

  

## 5. Kafka通知流程：FraudTagUpdateSendKafka

  

### 5.1 完整实现（第691-741行）

  

```go

func (s *UserTagService) FraudTagUpdateSendKafka(ctx context.Context, csUserID int64, csUserRegion string, hiveDbGrassRegion string, shopeeUserID int64) {

// ========== 步骤1：功能开关检查 ==========

if !config.GetFeatureSwitch().EnableShopeeFraudTagChangeNotice {

return // 功能未启用，直接返回

}

log.WithCtx(ctx).Infof("FraudTagUpdateSendKafka:%d,csUserRegion:%s,hiveDbGrassRegion:%s", csUserID, csUserRegion, hiveDbGrassRegion)

// ========== 步骤2：获取所有Fraud Tag定义 ==========

tagDefineTabs, errInfo := TagDefineServiceImpl.GetByServiceName(ctx, constant.TagServiceNameFraud, hiveDbGrassRegion)

if errInfo != nil {

log.WithCtx(ctx).Errorf("GetByServiceName err:%s", errInfo.Error())

return

}

// ========== 步骤3：批量查询用户的Fraud Tag数据 ==========

tagKeyList := []string{}

for _, tagDefineTab := range tagDefineTabs {

tagKeyList = append(tagKeyList, tagDefineTab.TagKey)

}

fraudTagResp, errInfo := s.GetUserTagByKeyBatch(ctx, csUserID, tagKeyList, true)

if errInfo != nil {

log.WithCtx(ctx).Errorf("GetUserTagByKeyBatch err:%s", errInfo.Error())

return

}

// ========== 步骤4：过滤和反序列化Tag ==========

fraudTagList := []*user_kafka.ShopeeFraudTagInfo{}

for _, fraudTag := range fraudTagResp {

// 过滤1: 跳过不存在或值为空的Tag

if fraudTag.GetNotExist() || fraudTag.GetTagValue() == "" {

continue

}

// 反序列化Tag Value

temp := user_kafka.ShopeeFraudTagInfo{}

err := json.Unmarshal([]byte(fraudTag.GetTagValue()), &temp)

if err != nil {

log.WithCtx(ctx).Errorf("Unmarshal tag value err:%s,data[%s]", err, fraudTag.GetTagValue())

return

}

// 过滤2: 跳过已过期的Tag

if temp.GetTagExpiryTimestamp() > 0 && temp.GetTagExpiryTimestamp() <= time.Now().Unix() {

continue

}

fraudTagList = append(fraudTagList, &temp)

}

// ========== 步骤5：发送Kafka消息 ==========

err := s.publishUserInfoUpdate(ctx, &user_kafka.UserInfoChangeMessage{

Region: csUserRegion,

CsUserId: csUserID,

UserInfoChangeTypes: []user_kafka.UserInfoChangeType{user_kafka.UserInfoChangeType_SHOPEE_FRAUD_TAG},

FraudTagList: fraudTagList,

ShopeeUserId: proto.Int64(shopeeUserID),

})

if err != nil {

log.WithCtx(ctx).Errorf("publishUserInfoUpdate err:%s", err.Error())

}

}

```

  

---

  

### 5.2 Kafka消息格式

  

**publishUserInfoUpdate函数（第744-756行）**：

```go

func (s *UserTagService) publishUserInfoUpdate(ctx context.Context, msg *user_kafka.UserInfoChangeMessage) error {

// 功能开关控制

if !config.GetFeatureSwitch().ShopeeUsernameChangeNoticeEnable {

msg.ShopeeUsername = nil

msg.UserInfoChangeTypes = nil

}

// 设置上下文

ctx = kitutils.AppendRegionToCtx(ctx, msg.GetRegion())

ctx = kitutils.AppendSaasIDToCtx(ctx, config.ShopeeSaasID)

ctx = config.SetDefaultSaas(ctx, 0)

log.WithCtx(ctx).Infof("user_data_sync:publish user info change = [%v]", msg)

// 发送Kafka消息

return kafkaclient.KafkaDal.Publish(ctx, config.GetKafkaConfig().UserInfoChangeTopic, msg)

}

```

  

**Kafka Topic**: `user_info_change_topic`

  

**消息Protobuf定义**：

```protobuf

message UserInfoChangeMessage {

string region = 1; // "sg"

int64 cs_user_id = 2; // 12345

repeated UserInfoChangeType user_info_change_types = 3; // [SHOPEE_FRAUD_TAG]

repeated ShopeeFraudTagInfo fraud_tag_list = 4;

int64 shopee_user_id = 5; // 888888

}

  

message ShopeeFraudTagInfo {

uint32 fraud_tag_id = 1; // 1001

string fraud_tag_name = 2; // "可疑交易"

string type = 3; // "fraud_transaction"

string severity = 4; // "high"

int64 tag_expiry_timestamp = 5; // 0

int64 create_timestamp = 6; // 1738732800

int64 modify_timestamp = 7; // 1738732900

}

  

enum UserInfoChangeType {

SHOPEE_FRAUD_TAG = 1;

SHOPEE_ACCOUNT_STATUS = 2;

// ...

}

```

  

**实际Kafka消息示例**：

```json

{

"region": "sg",

"cs_user_id": 12345,

"user_info_change_types": ["SHOPEE_FRAUD_TAG"],

"fraud_tag_list": [

{

"fraud_tag_id": 1001,

"fraud_tag_name": "可疑交易",

"type": "fraud_transaction",

"severity": "high",

"tag_expiry_timestamp": 0,

"create_timestamp": 1738732800,

"modify_timestamp": 1738732900

},

{

"fraud_tag_id": 1004,

"fraud_tag_name": "账号异常",

"type": "account_risk",

"severity": "medium",

"tag_expiry_timestamp": 1740000000,

"create_timestamp": 1738732800,

"modify_timestamp": 1738733000

}

],

"shopee_user_id": 888888

}

```

  

---

  

## 6. 完整时序图

  

### 6.1 全量同步时序图

  

```

用户进线

│

▼

FraudTagFullSyncFromSpex(csUserID=12345, shopeeUserID=888888)

│

├─ 调用Anti-Fraud Spex ──────────────────────────────────┐

│ GetEntityFraudTags(888888, "sg") │

│ 返回: [Tag1001, Tag1002] │

│ │

├─ 检查并创建Tag定义 ─────────────────────────────────────┤

│ CreateFraudTagDefineIfNotExist("fraud_tag_1001") │

│ CreateFraudTagDefineIfNotExist("fraud_tag_1002") │

│ → INSERT user_tag_define_tab (如果不存在) │

│ │

├─ 添加全量同步标记 ──────────────────────────────────────┤

│ Tag: "has_pull_all_fraud_tags" = "true" │

│ │

├─ 查询现有Fraud Tag定义 ─────────────────────────────────┤

│ GetByServiceName("ops-cs", "sg") │

│ 返回: ["fraud_tag_1001", "fraud_tag_1002", "fraud_tag_1003"]

│ │

├─ 对比找出需要删除的Tag ──────────────────────────────────┤

│ 需要删除: ["fraud_tag_1003"] (Spex未返回，已失效) │

│ DeleteUserTagByKeyBatch(12345, ["fraud_tag_1003"]) │

│ → DELETE user_field_extend_tab │

│ │

├─ 批量保存Tag ───────────────────────────────────────────┤

│ SaveUserTagBatch([Tag1001, Tag1002, has_pull_all]) │

│ → 查询现有数据 (Redis→DB) │

│ → 时间戳对比 │

│ → 值对比 │

│ → UPDATE/INSERT user_field_extend_tab │

│ → DELETE Redis Cache │

│ │

├─ 发送Kafka通知 ─────────────────────────────────────────┤

│ FraudTagUpdateSendKafka() │

│ ├─ 查询所有Fraud Tag数据 │

│ ├─ 过滤已过期Tag │

│ └─ Publish to user_info_change_topic │

│ │

└─ 返回savedTags: [Tag1001, Tag1002]

```

  

---

  

### 6.2 增量同步时序图

  

```

Hive Kafka消息

│ {"user_id": 888888, "fraud_tag_id": 1001, "is_active_user_tag": "1", ...}

▼

FraudTagIncrementalSyncFromHive(data)

│

├─ 检查CS用户 ────────────────────────────────────────────┐

│ getCsUserID(888888, "sg") │

│ 返回: csUserID=12345, csUserRegion="sg" │

│ │

├─ 检查强制全量同步开关 ───────────────────────────────────┤

│ AlwaysFullSyncFraudTag = false │

│ → 继续增量同步 │

│ │

├─ 检查全量同步标记 ───────────────────────────────────────┤

│ GetUserTagByKeyBatch(["has_pull_all_fraud_tags"]) │

│ 返回: TagValue="true" │

│ → 已全量同步，继续增量同步 │

│ │

├─ 判断Tag有效性 ─────────────────────────────────────────┤

│ FraudTagRecordIsValid(data) │

│ 检查: is_active_user_tag == "1"? ✅ │

│ 检查: tag_expiry_timestamp未过期? ✅ │

│ → Tag有效 │

│ │

├─ 场景A: Tag有效 - 保存 ─────────────────────────────────┤

│ ├─ CreateFraudTagDefineIfNotExist("fraud_tag_1001") │

│ ├─ SaveUserTagBatch([SaveReq]) │

│ │ Scene: "SYNC_FROM_HIVE" │

│ │ TagValueUpdateTimestamp: data.ModifyTimestamp │

│ └─ hasUpdate = true (如果有变化) │

│ │

├─ 场景B: Tag无效 - 删除 ─────────────────────────────────┤

│ ├─ DeleteUserTagByKeyBatch(["fraud_tag_1001"]) │

│ └─ hasUpdate = true (如果删除成功) │

│ │

├─ 发送Kafka通知 ─────────────────────────────────────────┤

│ if hasUpdate: │

│ FraudTagUpdateSendKafka() │

│ │

└─ 返回execResultMsg

```

  

---

  

## 7. 关键配置项

  

### 7.1 Feature Switch

  

```yaml

feature_switch:

# Fraud Tag相关

always_full_sync_fraud_tag: false # 是否总是全量同步

enable_shopee_fraud_tag_change_notice: true # 是否发送Fraud Tag变更通知

# Account Status相关

enable_shopee_account_status_notice: true # 是否发送Account Status变更通知

shopee_username_change_notice_enable: false # 是否发送Username变更通知

```

  

### 7.2 Kafka配置

  

```yaml

kafka_config:

user_info_change_topic: "user_info_change_topic" # 用户信息变更Topic

```

  

### 7.3 常量定义

  

```go

// Tag Service名称

const (

TagServiceNameFraud = "ops-cs" // Fraud Tag的service

TagServiceNameCsUser = "cs_user" // CS User的service

)

  

// Fraud Tag相关

const (

FraudTagKeyPrefix = "fraud_tag_" // Fraud Tag Key前缀

FraudHasPullAllFraudTags = "has_pull_all_fraud_tags" // 全量同步标记

FraudTagIsActiveUserTagTrueValue = "1" // Tag生效标记

TagBoolValueTrue = "true" // 布尔值true

)

  

// Agent ID

const (

AgentIDSystem = 1 // 系统操作

)

```

  

---

  

## 8. 数据存储

  

### 8.1 user_tag_define_tab (Tag定义表)

  

```sql

-- Fraud Tag定义示例

id | tag_key | tag_desc | service | value_type | status_flag | region

---|-----------------|------------|---------|------------|-------------|-------

1 | fraud_tag_1001 | 可疑交易 | ops-cs | string | 1 | sg

2 | fraud_tag_1002 | 账号异常 | ops-cs | string | 1 | sg

3 | fraud_tag_1004 | 虚假信息 | ops-cs | string | 1 | sg

```

  

### 8.2 user_field_extend_tab (用户Tag数据表 - 分表)

  

```sql

-- 用户Fraud Tag数据示例 (user_field_extend_tab_5)

cs_user_id | field_name | field_value | service | scene

-----------|-----------------|--------------------------------------|---------|---------------

12345 | fraud_tag_1001 | {"fraud_tag_id":1001,"fraud_tag_... | ops-cs | SYNC_FROM_HIVE

12345 | fraud_tag_1004 | {"fraud_tag_id":1004,"fraud_tag_... | ops-cs | SPEX_UPDATE

12345 | has_pull_all_fraud_tags | true | cs_user | SPEX_UPDATE

```

  

**field_value JSON示例**：

```json

{

"fraud_tag_id": 1001,

"fraud_tag_name": "可疑交易",

"type": "fraud_transaction",

"severity": "high",

"tag_expiry_timestamp": 0,

"create_timestamp": 1738732800,

"modify_timestamp": 1738732900

}

```

  

---

  

## 9. 错误处理和边界情况

  

### 9.1 CS用户不存在

  

```go

// 增量同步时，CS用户不存在是正常情况

if errInfo.BizErrCode == bizerr.ErrCodeCSUserNotFound {

execResultMsg = "cs user not found"

return execResultMsg, nil // 返回nil error

}

```

  

**原因**：

- Hive中的用户可能未在CS系统注册

- 不应该报错，只记录日志

  

### 9.2 未全量同步时自动降级

  

```go

if res[0].GetNotExist() || res[0].GetTagValue() != constant.TagBoolValueTrue {

// 自动触发全量同步

_, err := s.FraudTagFullSyncFromSpex(ctx, csUserID, csUserRegion, data.GrassRegion, data.UserId)

return execResultMsg, err

}

```

  

**保证数据完整性**

  

### 9.3 Tag过期处理

  

```go

// 保存前检查

if data.TagExpiryTimestamp > 0 && data.TagExpiryTimestamp <= time.Now().Unix() {

return false // 已过期，不保存

}

  

// 发送Kafka前检查

if temp.GetTagExpiryTimestamp() > 0 && temp.GetTagExpiryTimestamp() <= time.Now().Unix() {

continue // 已过期，不发送

}

```

  

**双重检查确保不发送过期Tag**

  

---

  

## 10. 性能优化

  

| 优化点 | 实现方式 | 效果 |

|-------|---------|------|

| **批量操作** | SaveUserTagBatch批量保存 | 减少数据库交互 |

| **Tag定义缓存** | GetByServiceName结果可能缓存 | 减少DB查询 |

| **Spex RPC优化** | 一次调用获取全部Fraud Tag | 减少RPC次数 |

| **异步处理** | Kafka异步通知 | 不阻塞主流程 |

| **分表存储** | user_field_extend_tab按csUserID分表 | 提升查询性能 |

  

---

  

## 11. 监控指标

  

建议添加的Metrics：

  

```go

// Fraud Tag同步指标

FraudTagFullSyncCounter // 全量同步次数

FraudTagIncrementalSyncCounter // 增量同步次数

FraudTagSyncDuration // 同步耗时

FraudTagCreateCounter // Tag创建次数

FraudTagDeleteCounter // Tag删除次数

FraudTagKafkaPublishCounter // Kafka发送次数

```

  

---

  

## 12. 相关文件

  

| 文件路径 | 说明 |

|---------|------|

| `internal/service/cs_user_tag.go` | Fraud Tag同步核心实现 |

| `internal/model/servicemodel/cs_user_tag.go` | HiveFraudTag数据结构 |

| `internal/rpcclient/anti_fraud_tag/` | Anti-Fraud Spex客户端 |

| `internal/spex/api/user.go` | 触发同步的Spex API |

| `internal/consumer/handlers.go` | Kafka消费者Handler |

  

---

  

**版本信息**：

- 文档版本：1.0

- 创建日期：2026-02-05

- 最后更新：2026-02-05
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTMxNzE1NDk3N119
-->