# User Tag 计算流程详解

  

## 1. 概述

  

User Tag计算引擎是基于**规则引擎**的动态标签计算系统，通过配置化的条件规则，从多个数据源（UPP、SOP、CS System等）并发获取数据并进行条件评估，最终计算出用户应该拥有哪些标签。

  

### 核心特点

  

| 特点 | 说明 |

|-----|------|

| **规则驱动** | 基于数据库配置的规则，无需修改代码即可新增标签 |

| **并发计算** | 多个数据源并发Prepare和Calc，提升性能 |

| **灵活表达式** | 支持satisfy_any、satisfy_all和自定义布尔表达式 |

| **限流控制** | 用户维度限流，防止重复计算 |

| **插件化扩展** | Processor接口，轻松扩展新数据源 |

  

---

  

## 2. 核心入口：CalcUserTags

  

### 函数签名

  

```go

// internal/service/user_tag/user_tag_service.go

func (d *defaultCSUserTagService) CalcUserTags(

ctx context.Context,

csUserID int64

) (tagIDList []int64, isThrottled bool, err error)

```

  

**返回值**：

- `tagIDList`: 计算出的Tag ID列表

- `isThrottled`: 是否被限流（true表示未执行计算）

- `err`: 错误信息

  

---

  

## 3. 完整流程分解

  

### 阶段1：功能开关检查（第495-497行）

  

```go

if !config.GetConfigBySaasRegion(ctx, tenantID, region, config.GetCsUserTagCriteria().Enable) {

return nil, isThrottled, nil // 功能未启用，直接返回

}

```

  

**配置项**：

```yaml

cs_user_tag_criteria:

enable: true # 是否启用Tag计算

```

  

---

  

### 阶段2：限流检查（第506-510行）

  

```go

if !d.CalcUserTagLimitCheck(ctx, csUserID) {

calcStatus = metrics.TagCalcStatusThrottled

log.WithCtx(ctx).Infof("CalcUserTagLimitCheck not pass:%d", csUserID)

return nil, true, nil // 被限流，返回isThrottled=true

}

```

  

**限流实现**：

```go

func (d *defaultCSUserTagService) CalcUserTagLimitCheck(ctx context.Context, csUserID int64) bool {

redisClient, _ := cache.RedisManager.Client(ctx)

lockKey := cache.GetUserTagCalcLimitLockKey(csUserID)

// lockKey格式: "user_tag:calc:limit:{cs_user_id}"

// 尝试设置Redis Key，NX=不存在才设置，EX=过期时间

expireSecond := config.GetCsUserTagCriteria().CalcLimitExpireSecond // 默认10秒

ok, err := redisClient.SetNX(ctx, lockKey, "1", time.Duration(expireSecond)*time.Second)

if err != nil || !ok {

return false // 限流

}

return true // 通过

}

```

  

**限流逻辑**：

- 同一用户10秒内只能计算一次

- 使用Redis SetNX实现分布式锁

- 避免重复计算浪费资源

  

---

  

### 阶段3：获取用户基础信息（第514-542行）

  

```go

// 1. 调用CsUserService获取用户信息

getCsUserInfoResponse, errInfo := user_service.CsUserServiceImpl.GetCsUserInfo(ctx, &servicemodel.GetCSUserInfoRequest{

CsUserId: csUserID,

Region: region,

})

  

// 2. 解析BusinessUserID

var bizUserID int64

if getCsUserInfoResponse.BusinessUserID != "" {

bizUserID, _ = strconv.ParseInt(getCsUserInfoResponse.BusinessUserID, 10, 64)

}

  

// 3. 构建Tag计算用户信息

tagUserInfo := &servicemodel.TagCalcUserInfo{

CsUserID: csUserID, // CS用户ID

Region: region, // 区域(sg/tw/id等)

BusinessUserID: bizUserID, // 业务用户ID(如ShopeeUserID)

ShopID: getCsUserInfoResponse.ShopId, // 店铺ID

BusinessUserType: getCsUserInfoResponse.BusinessUserType, // 业务类型(Shopee/Lazada等)

TenantID: tenantID, // 租户ID

}

```

  

**TagCalcUserInfo结构**：

```go

type TagCalcUserInfo struct {

CsUserID int64

Region string

BusinessUserID int64 // Shopee User ID或Lazada User ID

ShopID int64

BusinessUserType int32 // 1=Shopee, 2=Lazada, 3=TikTok

TenantID int64

}

```

  

---

  

### 阶段4：获取解析好的Tag规则（第545-550行）

  

```go

parsedTagDefine, err := d.GetParsedTagDefine(ctx)

```

  

#### 4.1 GetParsedTagDefine实现

  

```go

func (d *defaultCSUserTagService) GetParsedTagDefine(ctx context.Context) (*servicemodel.TagConditionInRegion, error) {

region := kitutils.RegionFromCtx(ctx)

tenantID := kitutils.TenantIDFromCtx(ctx)

// 🔑 使用本地缓存，2小时有效期

key := fmt.Sprintf(cache_wrapper.KeyAllParsedTagDefine, tenantID, region)

return cache_wrapper.WrapLocalCache(ctx, key, time.Hour*2, func() (*servicemodel.TagConditionInRegion, error) {

// 1. 从数据库获取所有Tag规则

allTagDefines, err := d.GetAllTagDefines(ctx)

// 2. 解析JSON条件，预编译表达式

rules := make([]*servicemodel.UserTagRule, 0, len(allTagDefines))

for _, tagDefine := range allTagDefines {

rules = append(rules, tagDefine)

}

// 3. 调用ParseTagDefineJSON

return ParseTagDefineJSON(ctx, true, rules...) // onlyEnable=true

})

}

```

  

#### 4.2 ParseTagDefineJSON核心逻辑（第339-401行）

  

```go

func ParseTagDefineJSON(ctx context.Context, onlyEnable bool, tagListInRegion ...*servicemodel.UserTagRule) (*servicemodel.TagConditionInRegion, error) {

res := &servicemodel.TagConditionInRegion{}

res.CollectedValue = make(map[constant.UserTagDataSource]interface{})

for _, tagRuleDefineTab := range tagListInRegion {

// ========== 步骤1：过滤条件 ==========

if onlyEnable && tagRuleDefineTab.StatusFlag != constant.StatusFlagEnable {

continue // 只要启用的Tag

}

if len(tagRuleDefineTab.TriggerCondition.Conditions) == 0 {

continue // 没有条件的Tag跳过

}

input := tagRuleDefineTab.TriggerCondition

temp := &servicemodel.TagDefineForCalc{}

temp.MatchLogic = input.MatchLogic // satisfy_any / satisfy_all / 自定义表达式

// ========== 步骤2：预编译表达式 ==========

if input.MatchLogic != auto_center.ConditionLogicSatisfyAll &&

input.MatchLogic != auto_center.ConditionLogicSatisfyAny {

// 自定义布尔表达式，需要预编译

expressionExecutor := utils.NewExpressionExecutor(input.MatchLogic)

err := expressionExecutor.Prepare() // 预编译，如 "(c1 && c2) || c3"

if err != nil {

return nil, err

}

temp.MatchLogicExecutor = expressionExecutor

}

temp.TagID = tagRuleDefineTab.ID

temp.Conditions = make([]*servicemodel.TagConditionForCalcItem, len(input.Conditions))

// ========== 步骤3：解析每个Condition ==========

for conditionIndex, condition := range input.Conditions {

item := &servicemodel.TagConditionForCalcItem{}

item.Field = condition.Field // 如 "UPP|segment_id"

item.Operator = condition.Operator // 1=In, 2=NotIn

// 拆分Field: "UPP|segment_id" -> dataSource="UPP", criteriaName="segment_id"

split := strings.Split(condition.Field, "|")

if len(split) != 2 {

return res, fmt.Errorf("condition.Field must be 'DataSource|CriteriaName'")

}

conditionDataSource := constant.UserTagDataSource(split[0]) // "UPP"

item.DataSource = conditionDataSource

item.CriteriaName = split[1] // "segment_id"

// ========== 步骤4：调用DecodeAndCollectValue ==========

// 获取数据源对应的解码函数

processor, found := user_tag_condition_processor.GetDecodeAndCollectValueFunc(conditionDataSource)

if !found {

return nil, fmt.Errorf("processor not found:%s", conditionDataSource)

}

// 解码Value并收集需要查询的数据

// condition.Value: {"UPP|segment_id": ["123", "456"]}

decodedValue, collectValue, err := processor(

ctx,

item.CriteriaName, // "segment_id"

condition.Operator, // 1

condition.Value, // map[string]json.RawMessage

res.CollectedValue[item.DataSource], // 当前已收集的值

)

if err != nil {

return nil, err

}

item.DecodedValue = decodedValue // 当前condition的值: [123, 456]

res.CollectedValue[item.DataSource] = collectValue // 累积所有UPP segment: [123, 456, 789, ...]

temp.Conditions[conditionIndex] = item

}

res.TagConditionList = append(res.TagConditionList, temp)

}

return res, nil

}

```

  

**ParseTagDefineJSON输出示例**：

```go

&TagConditionInRegion{

// 收集所有需要查询的数据（用于批量查询）

CollectedValue: {

"UPP": UppValue{123, 456, 789}, // 所有Tag规则中涉及的UPP Segment

"SOP": SopValue{1001, 1002}, // 所有Tag规则中涉及的SOP Tag

},

// 每个Tag的条件定义

TagConditionList: [

{

TagID: 100,

MatchLogic: "satisfy_any",

Conditions: [

{

DataSource: "UPP",

CriteriaName: "segment_id",

Operator: 1, // In

DecodedValue: [123, 456],

},

{

DataSource: "SOP",

CriteriaName: "tag_id",

Operator: 1,

DecodedValue: [1001],

},

],

},

{

TagID: 101,

MatchLogic: "(c1 && c2) || c3",

MatchLogicExecutor: <预编译的表达式执行器>,

Conditions: [...],

},

],

}

```

  

---

  

### 阶段5：创建计算处理器并执行计算（第555-557行）

  

```go

// 创建UserTagCalcProcessor

tagCalcProcessor := NewUserTagCalcProcessor(ctx, tagUserInfo, parsedTagDefine)

  

// 执行计算

tagIDList, err = tagCalcProcessor.Calc(ctx)

```

  

---

  

## 4. UserTagCalcProcessor.Calc 核心计算逻辑

  

### 4.1 初始化（第131-150行）

  

```go

func (s *UserTagCalcProcessor) Calc(ctx context.Context) (tagIDList []int64, err error) {

costRecorder := utils.NewCostRecorder()

t1 := time.Now()

wg := &sync.WaitGroup{}

conditionResultChan := make(chan tagCalcResult)

// 获取该region配置的可用数据源列表

dataTypeList := config.GetAvailableDataSource(s.UserInfo.TenantID, s.UserInfo.Region)

// 例如: ["UPP", "SOP", "CSUserProfile", "CSSystem"]

if len(dataTypeList) == 0 {

return tagIDList, fmt.Errorf("no data source config")

}

ctxWithCancel, CancelF := context.WithCancel(ctx)

defer CancelF()

costRecorder.Remark("start calc")

```

  

---

  

### 4.2 并发启动所有数据源Processor（第151-157行）

  

```go

for _, dataSource := range dataTypeList {

// 并发启动每个数据源的Processor

err := s.processorCalc(ctxWithCancel, wg, dataSource, conditionResultChan, costRecorder)

if err != nil {

return tagIDList, err

}

}

```

  

**processorCalc函数分析**（第50-129行）：

  

```go

func (s *UserTagCalcProcessor) processorCalc(

ctx context.Context,

wg *sync.WaitGroup,

dataSource constant.UserTagDataSource,

conditionResultChan chan<- tagCalcResult,

costRecorder *utils.CostRecorder,

) error {

// 1. 获取或创建Processor

processor, found, err := s.getPersonalProcessor(ctx, dataSource)

if err != nil {

return err

}

wg.Add(1)

newCtx := kitutils.CopyContextWithCID(ctx)

// 2. 启动goroutine执行

go func(ctx context.Context, dataSource constant.UserTagDataSource) {

t1 := time.Now()

prepareResult := metrics.TagCalcPrepareResultFailed

defer func() {

wg.Done()

if err := recover(); err != nil {

prepareResult = metrics.TagCalcPrepareResultPanic

log.WithCtx(ctx).Errorf("panic recoverd:%s\n%s", err, string(debug.Stack()))

}

metrics.TagCalcPrepareDuration.WithLabelValues(string(dataSource), s.UserInfo.Region, prepareResult).Observe(float64(time.Since(t1).Milliseconds()))

}()

// ========== 阶段A：Prepare - 数据准备 ==========

hit, err := processor.Prepare(ctx, s.tagList.CollectedValue[dataSource])

if err != nil {

log.WithCtx(ctx).Errorf("%s processor.Prepare: %s", dataSource, err)

return

}

prepareResult = metrics.TagCalcPrepareResultSuccess

costRecorder.Remark(fmt.Sprintf("%s Prepare Done", dataSource))

if !hit {

return // 没有数据需要处理，直接返回

}

// ========== 阶段B：Calc - 条件计算 ==========

for _, tagDefineForCalc := range s.tagList.TagConditionList {

for conditionIndex, condition := range tagDefineForCalc.Conditions {

// 检查这个condition是否属于当前数据源

if !processor.IsValidDataSource(ctx, condition.DataSource) {

continue

}

// 调用Processor.Calc计算condition结果

result, err := processor.Calc(ctx, condition)

if err != nil {

log.WithCtx(ctx).Errorf("tag define:%s, %s", utils.ToJson(tagDefineForCalc), err)

result = false // 出错当false处理

}

// 构建结果并发送到channel

tagResult := tagCalcResult{

TagID: tagDefineForCalc.TagID,

ConditionIndex: conditionIndex,

Result: result,

}

select {

case conditionResultChan <- tagResult:

case <-time.After(10 * time.Millisecond):

log.WithCtx(ctx).Errorf("send to result channel failed")

}

}

}

costRecorder.Remark(fmt.Sprintf("%s Calc Done", dataSource))

}(newCtx, dataSource)

return nil

}

```

  

---

  

### 4.3 收集并聚合结果（第159-289行）

  

```go

finishedChan := make(chan struct{})

  

go func() {

defer func() {

if err := recover(); err != nil {

log.WithCtx(ctx).Errorf("panic recoverd:%s\n%s", err, string(debug.Stack()))

}

}()

// ========== 初始化数据结构 ==========

// condition结果映射: map[tagID][conditionIndex]*bool

conditionResultMap := make(map[int64][]*bool)

// tag定义映射: map[tagID]*TagDefineForCalc

tagIDMap := make(map[int64]*servicemodel.TagDefineForCalc)

// tag计算结果: map[tagID]bool

tagCalcResultMap := make(map[int64]bool)

// 初始化

for _, tagDefineForCalc := range s.tagList.TagConditionList {

tagIDMap[tagDefineForCalc.TagID] = tagDefineForCalc

conditionResultMap[tagDefineForCalc.TagID] = make([]*bool, len(tagDefineForCalc.Conditions))

}

// ========== 从channel接收并处理结果 ==========

for calcResult := range conditionResultChan {

log.WithCtx(ctx).Infof("condition result: tag[%d], condition index[%d], result[%v]",

calcResult.TagID, calcResult.ConditionIndex, calcResult.Result)

// ========== 提前退出检查 ==========

if len(tagCalcResultMap) == len(s.tagList.TagConditionList) {

break // 所有Tag都有结果了

}

// 如果该Tag已有最终结果，跳过

if _, has := tagCalcResultMap[calcResult.TagID]; has {

continue

}

// ========== 记录condition结果 ==========

result := calcResult.Result

conditionResultMap[calcResult.TagID][calcResult.ConditionIndex] = &result

tagDefineForCalc := tagIDMap[calcResult.TagID]

// ========== 根据MatchLogic聚合结果 ==========

switch tagDefineForCalc.MatchLogic {

// ========== 场景1: satisfy_any (任一满足即可) ==========

case auto_center.ConditionLogicSatisfyAny:

if calcResult.Result == true {

// 只要有一个为true，整个Tag结果就是true

tagCalcResultMap[calcResult.TagID] = true

if len(tagCalcResultMap) == len(s.tagList.TagConditionList) {

break

}

continue

}

// 检查是否所有condition都返回了

allConditionReturned := true

for _, b := range conditionResultMap[calcResult.TagID] {

if b == nil {

allConditionReturned = false

break

}

}

// 所有condition都返回了，但都是false

if allConditionReturned {

tagCalcResultMap[calcResult.TagID] = false

}

// ========== 场景2: satisfy_all (全部满足才行) ==========

case auto_center.ConditionLogicSatisfyAll:

if calcResult.Result == false {

// 只要有一个为false，整个Tag结果就是false

tagCalcResultMap[calcResult.TagID] = false

if len(tagCalcResultMap) == len(s.tagList.TagConditionList) {

break

}

continue

}

// 检查是否所有condition都返回了

allConditionReturned := true

for _, b := range conditionResultMap[calcResult.TagID] {

if b == nil {

allConditionReturned = false

break

}

}

// 所有condition都返回了，且都是true

if allConditionReturned {

tagCalcResultMap[calcResult.TagID] = true

}

// ========== 场景3: 自定义布尔表达式 ==========

default:

if tagDefineForCalc.MatchLogicExecutor == nil {

log.WithCtx(ctx).Errorf("MatchLogicExecutor should not be nil, tagID:%d", tagDefineForCalc.TagID)

return

}

conditionLength := len(tagDefineForCalc.Conditions)

calcValue := make([]bool, conditionLength)

allConditionReturned := true

// 收集所有condition的结果

for conditionIndex, b := range conditionResultMap[calcResult.TagID] {

if b == nil {

allConditionReturned = false

} else {

calcValue[conditionIndex] = *b

}

}

if !allConditionReturned {

continue // 还有condition未返回，继续等待

}

// 所有condition都返回了，执行表达式

// 例如: MatchLogic="(c1 && c2) || c3", calcValue=[true, false, true]

// 结果: (true && false) || true = true

expressionResult, err := tagDefineForCalc.MatchLogicExecutor.GetResult(calcValue)

if err != nil {

log.WithCtx(ctx).Errorf("GetResult err:%s", err)

return

}

tagCalcResultMap[calcResult.TagID] = expressionResult

}

}

// ========== 提取满足条件的Tag ID列表 ==========

for tagID, result := range tagCalcResultMap {

if result {

tagIDList = append(tagIDList, tagID)

}

}

close(finishedChan)

}()

  

// ========== 等待所有数据源Processor完成 ==========

wg.Wait()

close(conditionResultChan)

  

// ========== 等待结果聚合完成 ==========

<-finishedChan

  

return tagIDList, nil

```

  

---

  

## 5. Processor接口详解

  

### 5.1 Processor接口定义

  

```go

// internal/service/user_tag_condition_processor/processor.go

type Processor interface {

// Prepare 数据准备阶段

// collectedValue: ParseTagDefineJSON阶段收集的需要查询的值

// 返回: hit=是否有数据, err=错误

Prepare(ctx context.Context, collectedValue interface{}) (hit bool, err error)

// IsValidDataSource 检查该condition是否属于当前Processor

IsValidDataSource(ctx context.Context, dataSource UserTagDataSource) bool

// Calc 计算单个condition是否满足

// 返回: result=true/false, err=错误

Calc(ctx context.Context, conditionItem *TagConditionForCalcItem) (result bool, err error)

}

```

  

---

  

### 5.2 UPP Processor实现示例

  

```go

// internal/service/user_tag_condition_processor/upp.go

  

type UserProfilePlatformProcessor struct {

userInfo *servicemodel.TagCalcUserInfo

uppResult map[uint64]bool // segment_id -> 是否在该segment中

}

  

// Prepare: 批量查询用户的UPP Segment

func (p *UserProfilePlatformProcessor) Prepare(ctx context.Context, collectedValueInterface interface{}) (hit bool, err error) {

if collectedValueInterface == nil {

return false, nil

}

collectedValue := collectedValueInterface.(UppValue) // []uint64{123, 456, 789}

if len(collectedValue) == 0 {

log.WithCtx(ctx).Debugf("no collectedValue,ignore")

return true, nil

}

if p.userInfo.BusinessUserID == 0 || p.userInfo.BusinessUserType != int64(common_constant.Shopee) {

log.WithCtx(ctx).Debugf("no shopee user id,ignore")

return true, nil

}

log.WithCtx(ctx).Infof("upp query tag data:%d", p.userInfo.CsUserID)

// 🔥 批量查询用户的UPP Segment

// 输入: shopeeUserID=888888, segmentIDs=[123, 456, 789]

// 输出: {123: true, 456: false, 789: true} 表示用户在123和789这两个segment中

uppSegmentWithCache, err := service.UppSegmentServiceImpl.CheckUppSegmentWithCache(

ctx,

p.userInfo.Region,

p.userInfo.BusinessUserID,

collectedValue,

)

if err != nil {

return false, err

}

p.uppResult = uppSegmentWithCache

return true, nil

}

  

// IsValidDataSource: 检查是否是UPP数据源

func (p *UserProfilePlatformProcessor) IsValidDataSource(ctx context.Context, dataSource constant.UserTagDataSource) bool {

return dataSource == constant.UserProfilePlatformDataSource

}

  

// Calc: 计算condition是否满足

func (p *UserProfilePlatformProcessor) Calc(ctx context.Context, conditionItem *servicemodel.TagConditionForCalcItem) (result bool, err error) {

dropdownListValue, ok := conditionItem.DecodedValue.(UppValue) // [123, 456]

if !ok {

return false, fmt.Errorf("DecodedValue type error")

}

switch common_constant.FilterOperator(conditionItem.Operator) {

case common_constant.FilterOperatorIn:

// In操作符: 用户只要在任一个segment中即为true

for _, checkGroupID := range dropdownListValue {

if p.uppResult[checkGroupID] == true {

return true, nil

}

}

return false, nil

case common_constant.FilterOperatorNotIn:

// NotIn操作符: 用户不在所有segment中才为true

for _, checkGroupID := range dropdownListValue {

if p.uppResult[checkGroupID] == true {

return false, nil

}

}

return true, nil

default:

return false, fmt.Errorf("not support operator:%d", conditionItem.Operator)

}

}

```

  

**UPP Processor执行示例**：

```

Prepare阶段:

用户: shopeeUserID=888888

需要查询的segments: [123, 456, 789]

调用UPP RPC:

CheckUppSegment(888888, [123, 456, 789])

返回:

{123: true, 456: false, 789: true}

存储到: p.uppResult

  

Calc阶段:

Condition1: 用户在 [123, 456] 任一segment中?

dropdownListValue = [123, 456]

Operator = In

检查: p.uppResult[123] = true ✅

结果: true

Condition2: 用户不在 [789] 中?

dropdownListValue = [789]

Operator = NotIn

检查: p.uppResult[789] = true ❌

结果: false

```

  

---

  

## 6. 保存计算结果：saveNewTagList

  

### 6.1 核心流程（第570-658行）

  

```go

func (d *defaultCSUserTagService) saveNewTagList(

ctx context.Context,

tenantID int64,

region string,

csUserID int64,

tagIDList []int64, // 计算出的Tag列表

costRecorder *utils.CostRecorder,

) (calcStatus string, err error) {

calcStatus = metrics.TagCalcStatusFailed

// ========== 步骤1：获取现有Tag列表 ==========

existTagIDList, err := d.GetCsUserTaggedIDs(ctx, csUserID)

if err != nil {

return calcStatus, err

}

costRecorder.Remark("getExistTag")

// ========== 步骤2：对比差异 ==========

toAdd, toRemove := tagIDListCompare(existTagIDList, tagIDList)

log.WithCtx(ctx).Debugf("%d user tag, now: %#v, exist:%#v, to remove:%#v, to add:%#v",

csUserID, tagIDList, existTagIDList, toRemove, toAdd)

if len(toAdd) == 0 && len(toRemove) == 0 {

// Tag无变更

calcStatus = metrics.TagCalcStatusSuccessAndNoChange

return calcStatus, nil

}

// ========== 步骤3：转换为field_name ==========

toRemoveFieldName := []string{}

for _, id := range toRemove {

toRemoveFieldName = append(toRemoveFieldName, strconv.FormatInt(id, 10))

}

// ========== 步骤4：数据库事务操作 ==========

txDB, err := db.DalDB.Conn(ctx, &dbmodel.UserFieldExtendTab{},

kit_db.WithShardKey([]interface{}{csUserID}),

kit_db.WithSkipCheck(false, true))

err = db.NewTransaction(txDB, &sql.TxOptions{Isolation: sql.LevelReadCommitted}, func(tx *gorm.DB) error {

// ========== 删除旧Tag ==========

if len(toRemoveFieldName) > 0 {

deleteRowsAffected, err := d.userFieldExtendRepo.NewTransaction(tx, csUserID).

DeleteByCsUserIDBatch(csUserID, toRemoveFieldName...)

if err != nil {

return err

}

log.WithCtx(ctx).Infof("%d deleted tag id:%#v", csUserID, toRemove)

}

// ========== 插入新Tag ==========

insertList := []*dbmodel.UserFieldExtendTab{}

for _, tagID := range toAdd {

insertList = append(insertList, &dbmodel.UserFieldExtendTab{

Region: region,

CsUserId: csUserID,

FieldName: strconv.FormatInt(tagID, 10), // Tag ID作为field_name

FieldValue: "", // CS Tag的field_value为空

Service: constant.ServiceCSUser, // "cs_user"

Scene: seller_cs_user.Constant_CS_USER_TAG_RULE.String(), // "CS_USER_TAG_RULE"

AgentID: constant.AgentIDSystem, // 1

TenantID: tenantID,

StatusFlag: constant.StatusFlagEnable,

})

}

if len(insertList) > 0 {

err := d.userFieldExtendRepo.NewTransaction(tx, csUserID).CreateBatch(insertList)

if err != nil {

return err

}

log.WithCtx(ctx).Infof("%d created tag id:%#v", csUserID, toAdd)

}

return nil

})

if err != nil {

return calcStatus, err

}

costRecorder.Remark("saveTag")

// ========== 步骤5：清理缓存 ==========

d.cacheMgr.RemoveCache(cache_wrapper.KeyCSUserTagID, csUserID)

calcStatus = metrics.TagCalcStatusSuccessAndHasChange

return calcStatus, nil

}

```

  

**tagIDListCompare函数**：

```go

func tagIDListCompare(exist, now []int64) (toAdd, toRemove []int64) {

existMap := make(map[int64]bool)

for _, id := range exist {

existMap[id] = true

}

nowMap := make(map[int64]bool)

for _, id := range now {

nowMap[id] = true

}

// 找出需要添加的(now中有但exist中没有)

for _, id := range now {

if !existMap[id] {

toAdd = append(toAdd, id)

}

}

// 找出需要删除的(exist中有但now中没有)

for _, id := range exist {

if !nowMap[id] {

toRemove = append(toRemove, id)

}

}

return toAdd, toRemove

}

```

  

---

  

## 7. 完整调用链时序图

  

```

用户进线

│

▼

CalcUserTags(csUserID=12345)

│

├─ 功能开关检查 ────────────────────────────────────┐

│ │

├─ 限流检查 (Redis SetNX) ─────────────────────────┤

│ Key: "user_tag:calc:limit:12345" │

│ EX: 10秒 │

│ │

├─ 获取用户信息 ────────────────────────────────────┤

│ GetCsUserInfo(12345) │

│ 返回: {BusinessUserID: 888888, ShopID: 5566} │

│ │

├─ 获取解析好的Tag规则 (本地缓存2h) ───────────────┤

│ GetParsedTagDefine() │

│ 返回: TagConditionInRegion │

│ │

├─ 创建UserTagCalcProcessor ───────────────────────┤

│ │

├─ tagCalcProcessor.Calc() ────────────────────────┤

│ │ │

│ ├─ 并发启动Processor ──────────────────────────┤

│ │ ├─ UPP Processor │

│ │ │ ├─ Prepare: CheckUppSegment(888888, [123,456,789])

│ │ │ │ 返回: {123:true, 456:false, 789:true}

│ │ │ └─ Calc: 对每个condition计算 │

│ │ │ 发送结果到channel │

│ │ │ │

│ │ ├─ SOP Processor │

│ │ │ ├─ Prepare: GetShopTagBatch(5566, [1001,1002])

│ │ │ └─ Calc: 计算并发送结果 │

│ │ │ │

│ │ ├─ CSUserProfile Processor │

│ │ └─ CSSystem Processor │

│ │ │

│ ├─ 收集condition结果 (从channel接收) ───────────┤

│ │ TagID=100, ConditionIndex=0, Result=true │

│ │ TagID=100, ConditionIndex=1, Result=false │

│ │ TagID=101, ConditionIndex=0, Result=true │

│ │ ... │

│ │ │

│ ├─ 聚合结果 ────────────────────────────────────┤

│ │ Tag100: satisfy_any, Condition0=true │

│ │ => Tag100=true │

│ │ Tag101: satisfy_all, Condition0=true, Condition1=false

│ │ => Tag101=false │

│ │ │

│ └─ 返回tagIDList: [100, 103, 105] │

│ │

├─ saveNewTagList() ───────────────────────────────┤

│ │ │

│ ├─ 获取现有Tags: [100, 102, 104] │

│ ├─ 对比差异: │

│ │ toAdd: [103, 105] │

│ │ toRemove: [102, 104] │

│ │ │

│ ├─ 数据库事务: │

│ │ DELETE field_name IN ('102', '104') │

│ │ INSERT field_name VALUES ('103'), ('105') │

│ │ │

│ └─ 清理缓存: RemoveCache(KeyCSUserTagID, 12345)│

│ │

└─ 返回: tagIDList=[100, 103, 105], isThrottled=false

```

  

---

  

## 8. 核心数据结构

  

### 8.1 数据库表

  

**cs_user_tag_rule_define_tab (Tag规则定义表)**:

```sql

id | tag_local_name | trigger_condition | status_flag | region

---|----------------|---------------------------------------|-------------|-------

100| VIP用户 | {"match_logic":"satisfy_any",...} | 1 | sg

101| 高价值客户 | {"match_logic":"(c1&&c2)||c3",...} | 1 | sg

```

  

**user_field_extend_tab (用户Tag数据表)**:

```sql

cs_user_id | field_name | field_value | service | scene

-----------|------------|-------------|----------|------------------

12345 | 100 | | cs_user | CS_USER_TAG_RULE

12345 | 103 | | cs_user | CS_USER_TAG_RULE

```

  

### 8.2 trigger_condition JSON结构

  

```json

{

"match_logic": "satisfy_any",

"conditions": [

{

"field": "UPP|segment_id",

"operator": 1,

"value": {

"UPP|segment_id": ["123", "456"]

}

},

{

"field": "SOP|tag_id",

"operator": 1,

"value": {

"SOP|tag_id": ["1001"]

}

}

]

}

```

  

---

  

## 9. 性能优化设计

  

| 优化点 | 实现方式 | 效果 |

|-------|---------|------|

| **并发计算** | 多个Processor并发执行Prepare和Calc | 耗时从串行相加变为最慢的那个 |

| **批量查询** | CollectedValue收集所有需要的ID，一次RPC查询 | 减少RPC调用次数 |

| **提前退出** | satisfy_any遇到true立即返回 | 减少不必要的计算 |

| **预编译表达式** | ParseTagDefineJSON时预编译布尔表达式 | 运行时直接执行，无需解析 |

| **本地缓存** | 解析后的Tag规则缓存2小时 | 避免重复解析JSON和编译表达式 |

| **限流控制** | 10秒内同一用户只计算一次 | 防止重复计算浪费资源 |

  

---

  

## 10. 监控指标

  

| 指标名 | 类型 | 标签 | 说明 |

|-------|------|------|------|

| `TagCalcRequestDuration` | Histogram | region, status | 整体计算耗时 |

| `TagCalcPrepareDuration` | Histogram | dataSource, region, result | Prepare阶段耗时 |

| `TagCalcCalcSummary` | Summary | region | Calc阶段耗时统计 |

  

**Status枚举**：

- `success_and_has_change`: 计算成功且Tag有变化

- `success_and_no_change`: 计算成功但Tag无变化

- `throttled`: 被限流

- `failed`: 计算失败

  

---

  

## 11. 实际使用示例

  

### 示例1：用户进线时触发计算

  

```go

// Session创建时

func CreateSession(ctx context.Context, req *CreateSessionReq) {

// ... 其他逻辑

// 计算用户Tag

tagIDList, isThrottled, err := user_tag.CSUserTagServiceImpl.CalcUserTags(ctx, csUserID)

if err != nil {

log.WithCtx(ctx).Errorf("CalcUserTags failed: %v", err)

}

if isThrottled {

log.WithCtx(ctx).Infof("CalcUserTags throttled for user %d", csUserID)

} else {

log.WithCtx(ctx).Infof("User %d tags: %v", csUserID, tagIDList)

}

}

```

  

### 示例2：通过SPEX API触发计算

  

```go

// internal/spex/api/user_tag.go

func (s *UserTagAPI) GetCSUserTagList(ctx context.Context, req *GetCSUserTagListRequest) {

// 是否需要刷新Tag（带限流）

if req.GetRefreshCsUserTagsWithRateLimited() {

tagIDList, _, err := user_tag.CSUserTagServiceImpl.CalcUserTags(ctx, req.CsUserId)

// ... 处理结果

}

// 获取Tag列表

tags, err := user_tag.CSUserTagServiceImpl.GetCsUserTagList(ctx, req.CsUserId)

return tags

}

```

  

---

  

## 12. 相关文件

  

| 文件路径 | 说明 |

|---------|------|

| `internal/service/user_tag/user_tag_service.go` | Tag服务主文件 |

| `internal/service/user_tag/user_tag_calc.go` | Tag计算引擎 |

| `internal/service/user_tag_condition_processor/processor.go` | Processor接口定义 |

| `internal/service/user_tag_condition_processor/upp.go` | UPP Processor实现 |

| `internal/service/user_tag_condition_processor/sop.go` | SOP Processor实现 |

| `internal/service/user_tag_condition_processor/cs_user_profile.go` | CS用户档案Processor |

| `internal/service/user_tag_condition_processor/cs_system.go` | CS系统Processor |

  

---

  

**版本信息**：

- 文档版本：1.0

- 创建日期：2026-02-05

- 最后更新：2026-02-05
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTUyNDM3NTEwXX0=
-->