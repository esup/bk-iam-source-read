# Expression 条件表达式深度分析

## 一、概述

Expression 是 BK-IAM 权限引擎的核心判定单元。每一条权限策略（Policy）都关联一个 Expression，
它描述了"在什么条件下，该策略允许/拒绝某个操作"。

Expression 贯穿整个鉴权流程：

```
Policy (JSON string)
   ↓  解析
PolicyCondition (map 结构)
   ↓  反序列化
Condition 树 (接口对象，可执行 Eval / PartialEval)
   ↓  评估
true / false / 剩余条件
   ↓  翻译（可选）
ExprCell (输出给接入系统的表达式)
```

---

## 二、Expression 数据模型

### 2.1 存储层：AuthExpression

```go
// pkg/service/types/policy.go
type AuthExpression struct {
    PK         int64  `msgpack:"p"`   // 主键
    Expression string `msgpack:"ea"`  // 原始 JSON 字符串
    Signature  string `msgpack:"s"`   // MD5 签名，用于缓存 key
}
```

- `Expression` 是 JSON 字符串，存储在数据库 `expression` 表中
- `Signature` 是 `Expression` 的 MD5 哈希，用于去重和缓存索引
- 一个 Expression 可以被多条 Policy 共享（通过 ExpressionPK 关联）

### 2.2 中间层：PolicyCondition

```go
// pkg/abac/pdp/types/policy_condition.go
type PolicyCondition map[string]map[string][]interface{}
//                    ^^^^^^^^ ^^^^^^^^ ^^^^^^^^^^^^^^
//                    操作符    字段名    字段值列表
```

示例：

```json
{
  "StringEquals": {
    "bk_job.script.id": ["1", "2"]
  }
}
```

字段命名规则：`{system}.{type}.{attr}`，例如 `bk_job.script.id`。

### 2.3 执行层：Condition 接口

```go
// pkg/abac/pdp/condition/init.go
type Condition interface {
    GetName() string
    GetKeys() []string
    HasKey(f keyMatchFunc) bool
    GetFirstMatchKeyValues(f keyMatchFunc) ([]interface{}, bool)
    Eval(ctx types.EvalContextor) bool
    Translate(withSystem bool) (map[string]interface{}, error)
}
```

---

## 三、条件类型体系

### 3.1 操作符注册表

```go
// pkg/abac/pdp/condition/operator/types.go
const (
    AND            = "AND"
    OR             = "OR"
    ANY            = "Any"
    Bool           = "Bool"
    StringPrefix   = "StringPrefix"
    StringEquals   = "StringEquals"
    StringContains = "StringContains"
    NumericEquals  = "NumericEquals"
    NumericGt      = "NumericGt"
    NumericGte     = "NumericGte"
    NumericLt      = "NumericLt"
    NumericLte     = "NumericLte"
)
```

### 3.2 类型继承关系

```
Condition (interface)
├── baseCondition (struct)          ← 叶子条件基类
│   ├── AnyCondition                ← 恒真条件
│   ├── StringEqualsCondition       ← 字符串精确匹配
│   ├── StringPrefixCondition       ← 字符串前缀匹配
│   ├── StringContainsCondition     ← 字符串包含匹配
│   ├── BoolCondition               ← 布尔匹配
│   └── NumericCompareCondition     ← 数值比较（eq/gt/gte/lt/lte 共用）
│
└── LogicalCondition (interface)    ← 逻辑条件接口（含 PartialEval）
    └── baseLogicalCondition (struct)
        ├── AndCondition            ← 逻辑与
        └── OrCondition             ← 逻辑或
```

### 3.3 各类条件详解

#### AnyCondition — 恒真条件

```go
func (c *AnyCondition) Eval(ctx types.EvalContextor) bool {
    return true  // 永远返回 true
}
```

- 表示"无限制"，任何资源都允许
- 当 Action 不关联资源类型时，Expression 为空字符串 `""`，解析为 AnyCondition
- Translate 输出：`{"op": "any", "field": "", "value": []}`

#### StringEqualsCondition — 字符串精确匹配

```go
func (c *StringEqualsCondition) Eval(ctx types.EvalContextor) bool {
    return c.forOr(ctx, func(a, b interface{}) bool {
        return a == b
    })
}
```

- 字段值之间是 **OR** 关系（任一匹配即通过）
- 支持属性值为数组的情况（数组元素与表达式值做笛卡尔积匹配）
- Translate 规则：
  - 单值 → `{"op": "eq", "field": "...", "value": "..."}`
  - 多值 → `{"op": "in", "field": "...", "value": [...]}`

#### StringPrefixCondition — 字符串前缀匹配

```go
func (c *StringPrefixCondition) Eval(ctx types.EvalContextor) bool {
    return c.forOr(ctx, func(a, b interface{}) bool {
        // 支持 IAM Path 末尾通配符: /biz,1/set,*/ → /biz,1/set,
        if strings.HasSuffix(c.Key, IamPathSuffix) && strings.HasSuffix(bStr, ",*/") {
            bStr = bStr[0 : len(bStr)-2]
        }
        return strings.HasPrefix(aStr, bStr)
    })
}
```

- 用于 IAM Path 层级路径匹配
- 支持 `*` 通配符（路径末尾节点为任意）
- Translate 规则：
  - 单值 → `{"op": "starts_with", "field": "...", "value": "..."}`
  - 多值 → `{"op": "OR", "content": [...]}`（每个值一个 starts_with，用 OR 连接）
  - **永远不会**输出 `starts_with [x, y]` 这种形式

#### StringContainsCondition — 字符串包含匹配

```go
func (c *StringContainsCondition) Eval(ctx types.EvalContextor) bool {
    return c.forOr(ctx, func(a, b interface{}) bool {
        return strings.Contains(aStr, bStr)
    })
}
```

- Translate 规则同 StringPrefix：单值直接输出，多值用 OR 连接

#### BoolCondition — 布尔匹配

```go
func (c *BoolCondition) Eval(ctx types.EvalContextor) bool {
    // 不支持多值，不支持数组属性
    return valueBool == exprBool
}
```

- 严格限制：只允许单个布尔值，属性值也不能是数组
- Translate 输出：`{"op": "eq", "field": "...", "value": true/false}`

#### NumericCompareCondition — 数值比较

统一实现 5 种数值操作符：

| 操作符 | 比较函数 | Translate Op | 多值 Translate |
|--------|----------|--------------|----------------|
| NumericEquals | `eval.ValueEqual` | `eq` | `in` |
| NumericGt | `eval.Greater` | `gt` | 报错 |
| NumericGte | `eval.GreaterOrEqual` | `gte` | 报错 |
| NumericLt | `eval.Less` | `lt` | 报错 |
| NumericLte | `eval.LessOrEqual` | `lte` | 报错 |

- `NumericEquals` 是唯一支持多值的数值操作符（多值时 Translate 为 `in`）
- `gt/gte/lt/lte` 严格限制单值

#### AndCondition — 逻辑与

```go
func (c *AndCondition) Eval(ctx types.EvalContextor) bool {
    for _, condition := range c.content {
        if !condition.Eval(ctx) {
            return false  // 短路求值：任一 false 即返回
        }
    }
    return true
}
```

- 反序列化时 field 必须为 `"content"`
- content 是递归解析的子条件数组

#### OrCondition — 逻辑或

```go
func (c *OrCondition) Eval(ctx types.EvalContextor) bool {
    for _, condition := range c.content {
        if condition.Eval(ctx) {
            return true  // 短路求值：任一 true 即返回
        }
    }
    return false
}
```

---

## 四、PartialEval — 部分评估

PartialEval 是 BK-IAM 的**查询模式**核心：当鉴权请求没有携带完整资源时，
系统不会直接返回 false，而是返回"剩余未评估的条件"，交给接入方做二次判定。

### 4.1 接口定义

```go
type LogicalCondition interface {
    Condition
    PartialEval(ctx types.EvalContextor) (bool, Condition)
    //                ^^^^   ^^^^^^^^^
    //                是否通过  剩余条件
}
```

### 4.2 AND 的 PartialEval 逻辑

```
遍历子条件：
  - AND/OR  → 递归 PartialEval
    - 子结果 false → 整体返回 (false, nil)
    - 子结果 ANY   → 跳过（恒真无需保留）
    - 子结果有剩余 → 加入 remainedContent
  - ANY     → 跳过（恒真）
  - 叶子条件 → 检查 ctx 是否有对应资源类型
    - 有资源 → 直接 Eval
      - true  → 继续
      - false → 整体返回 (false, nil)
    - 无资源 → 加入 remainedContent

结果汇总：
  remainedContent 为空 → 返回 (true, Any)
  remainedContent 1个  → 返回 (true, 该条件)
  remainedContent 多个 → 返回 (true, AND(remainedContent))
```

### 4.3 OR 的 PartialEval 逻辑

```
遍历子条件：
  - AND/OR  → 递归 PartialEval
    - 子结果 true 且剩余 ANY → 整体返回 (true, Any)
    - 子结果 true 且有剩余   → 加入 remainedContent
    - 子结果 false           → 忽略（OR 中 false 不影响）
  - ANY     → 整体返回 (true, Any)
  - 叶子条件 → 检查 ctx 是否有对应资源类型
    - 有资源 → 直接 Eval
      - true  → 整体返回 (true, Any)
      - false → 忽略
    - 无资源 → 加入 remainedContent

结果汇总：
  remainedContent 为空 → 返回 (false, nil)
  remainedContent 1个  → 返回 (true, 该条件)
  remainedContent 多个 → 返回 (true, OR(remainedContent))
```

### 4.4 两种评估模式对比

| 维度 | EvalPolicies | PartialEvalPolicies |
|------|-------------|---------------------|
| 用途 | auth 鉴权 | query 资源筛选 |
| 输入 | 完整请求 + 策略列表 | 部分请求 + 策略列表 |
| 输出 | `(isPass, policyID, err)` | `([]Condition, []policyID, err)` |
| 无资源时 | 返回 false | 返回剩余条件 |
| 策略匹配 | 任一通过即返回 true | 收集所有通过策略的剩余条件 |

---

## 五、表达式解析流程

### 5.1 JSON → Condition 树

```
Expression (JSON string)
    ↓ expressionToCondition()
    ↓
    ├── 空字符串 / "[]" → AnyCondition
    ├── 以 "{" 开头     → newExprToCondition()  [新版格式]
    └── 其他            → oldExprToCondition()  [旧版格式，兼容]
```

**新版格式**（直接 PolicyCondition）：

```json
{
  "AND": {
    "content": [
      {"StringEquals": {"bk_job.script.id": ["1"]}},
      {"StringPrefix": {"bk_job.script._bk_iam_path_": ["/bk_job,script/"]}}
    ]
  }
}
```

**旧版格式**（ResourceExpression 数组）：

```json
[
  {
    "system": "bk_job",
    "type": "script",
    "expression": {
      "StringEquals": {"id": ["1"]}
    }
  }
]
```

旧版需要通过 `ToNewPolicyCondition(system, type)` 将字段加上 `{system}.{type}.` 前缀，
转换为新版格式后再解析。

### 5.2 反序列化缓存

```go
// pkg/cacheimpls/local_unmarshaled_expression.go
func GetUnmarshalledResourceExpression(
    expression string,
    signature string,
    timestampNano int64,
) (condition.Condition, error)
```

- Key: `signature`（MD5 哈希）
- 使用 `GetAfterExpirationAnchor` 实现时间锚点缓存
- 缓存未命中时调用 `translate.PolicyExpressionToCondition()` 反序列化
- 结果缓存到 `LocalUnmarshaledExpressionCache`（go-cache，无 TTL）

---

## 六、评估上下文 EvalContext

### 6.1 结构

```go
type EvalContext struct {
    *request.Request
    objSet pdptypes.ObjectSetInterface
}
```

### 6.2 ObjectSet 属性查找

```
ObjectSet.data:
  "bk_job.script"  → {"id": "1", "_bk_iam_path_": ["/bk_job,script/"]}
  "bk_ci.pipeline" → {"id": "p1", "name": "test"}
  "bk_job._bk_iam_env_" → {"tz": "Asia/Shanghai", "hms": 143025}
```

属性查找规则（`GetAttribute(key)`）：

```
key = "bk_job.script.id"
  → LastIndexByte('.') 拆分
  → _type = "bk_job.script"
  → attrName = "id"
  → 返回 objSet.data["bk_job.script"]["id"]
```

### 6.3 环境属性注入

```go
func (c *EvalContext) InitEnvironments(cond Condition, currentTime time.Time) error
```

当 Condition 中包含 `_bk_iam_env_` 后缀的 key 时：

1. 提取时区值 `tz`（必须唯一）
2. 根据当前时间和时区计算环境值：
   - `tz`: 时区字符串，如 `"Asia/Shanghai"`
   - `hms`: 时分秒合并值，如 `143025`（14:30:25）
3. 注入到 ObjectSet：key = `{system}._bk_iam_env_`

环境属性缓存：`GenTimeEnvsFromCache` 使用 `tz+unix` 作为 key，
同一秒内相同时区的请求共享环境值。

### 6.4 IAM Path 标准化

```go
func standardizeIamPaths(iamPaths interface{}) interface{}
```

自动将 3 段式 IAM Path 裁剪为 2 段式：

```
输入: /bk_job,script,s1/         (3段式)
输出: /script,s1/                (2段式)

输入: /bk_job,script,s1/bk_ci,pipeline,p1/  (多节点3段式)
输出: /script,s1/pipeline,p1/               (多节点2段式)
```

---

## 七、Translate — 表达式翻译

### 7.1 目的

将内部表达式（`{system}.{type}.{attr}` 格式）翻译为接入方可理解的格式。

### 7.2 翻译规则

当前版本（v1）：`withSystem = false`，去掉 system 前缀。

```
内部: {"field": "bk_job.script.id", "op": "eq", "value": "1"}
输出: {"field": "script.id", "op": "eq", "value": "1"}
```

未来版本（v2）：`withSystem = true`，保留完整路径。

### 7.3 条件合并优化

`mergeContentField` 对同一字段的多条 eq/in 条件进行合并：

```
输入: [
  {"field": "script.id", "op": "eq", "value": "1"},
  {"field": "script.id", "op": "in", "value": ["2", "3"]},
  {"field": "pipeline.name", "op": "eq", "value": "test"}
]

输出: [
  {"field": "script.id", "op": "in", "value": ["1", "2", "3"]},
  {"field": "pipeline.name", "op": "eq", "value": "test"}
]
```

### 7.4 翻译输出格式

| 条件类型 | 单值输出 | 多值输出 |
|---------|---------|---------|
| StringEquals | `{op: "eq", field, value}` | `{op: "in", field, value: [...]}` |
| StringPrefix | `{op: "starts_with", field, value}` | `{op: "OR", content: [...]}` |
| StringContains | `{op: "string_contains", field, value}` | `{op: "OR", content: [...]}` |
| Bool | `{op: "eq", field, value}` | 报错 |
| NumericEquals | `{op: "eq", field, value}` | `{op: "in", field, value: [...]}` |
| NumericGt/Gte/Lt/Lte | `{op: "gt/gte/lt/lte", field, value}` | 报错 |
| AND | `{op: "AND", content: [...]}` | — |
| OR | `{op: "OR", content: [...]}` | — |
| Any | `{op: "any", field: "", value: []}` | — |

---

## 八、Expression 三级缓存

### 8.1 架构

```
GetExpressionsFromCache(actionPK, expressionPKs)
    ↓
L1: memoryRetriever (go-cache, TTL=60s)
    ↓ miss
L2: redisRetriever (Redis, TTL=7天+随机60s抖动)
    ↓ miss
L3: databaseRetriever (MySQL, expression 表)
```

### 8.2 各层职责

| 层级 | 存储介质 | Key 格式 | TTL | 特点 |
|------|---------|---------|-----|------|
| L1 Memory | go-cache | `{expressionPK}` | 60s | ChangeList 检测过期 |
| L2 Redis | Redis Hash | `{expressionPK}` | 7天+随机抖动 | Pipeline 批量读写 |
| L3 Database | MySQL | PK 查询 | — | 最终数据源 |

### 8.3 缓存失效机制

```
策略变更 → alterPolicies
    ↓
batchDeleteExpressionsFromMemory
    ↓
1. 删除本地缓存 (LocalExpressionCache.Delete)
2. 写入 ChangeList (Redis ZSet, member=expressionPK, score=timestamp)
3. 截断过期 ChangeList 条目

同时:
batchDeleteExpressionsFromRedis
    ↓
批量删除 Redis 缓存 (BatchDelete)
```

### 8.4 缓存失效规则

- 只有用户自定义的（`template_id=0`）Expression 才会通过 alterPolicies 更新
- 来自模板的（`template_id!=0`）不会更新，只会新增和删除

---

## 九、baseCondition 求值机制

### 9.1 forOr — 核心求值辅助

```go
func (c *baseCondition) forOr(ctx EvalContextor, fn func(a, b) bool) bool {
    attrValue := ctx.GetAttr(c.Key)  // 获取资源属性值

    switch vs := attrValue.(type) {
    case []interface{}:  // 属性是数组
        for _, av := range vs {        // 遍历数组元素
            for _, v := range c.Value { // 遍历表达式值
                if fn(av, v) { return true }
            }
        }
    default:            // 属性是单值
        for _, v := range c.Value {
            if fn(attrValue, v) { return true }
        }
    }
    return false
}
```

- value 之间是 **OR** 关系（任一匹配即返回 true）
- 属性为数组时，数组元素与表达式值做笛卡尔积匹配

### 9.2 HasKey / GetFirstMatchKeyValues

用于 PartialEval 和环境属性检测：

```go
// baseCondition: 直接检查自己的 Key
func (c *baseCondition) HasKey(f keyMatchFunc) bool {
    return f(c.Key)
}

// baseLogicalCondition: 递归检查所有子条件
func (c *baseLogicalCondition) HasKey(f keyMatchFunc) bool {
    for _, condition := range c.content {
        if condition.HasKey(f) { return true }
    }
    return false
}
```

---

## 十、完整鉴权流程中的 Expression

```
1. 收到鉴权请求 (system, subject, action, resources)
       ↓
2. PRP: 获取策略列表 (GetPoliciesFromCache)
       ↓  每条策略包含 ExpressionPK
3. PRP: 获取表达式 (GetExpressionsFromCache)
       ↓  L1→L2→L3 三级缓存
4. PDP: 反序列化表达式 (GetUnmarshalledResourceExpression)
       ↓  JSON → Condition 树（带缓存）
5. PDP: 构建评估上下文 (NewEvalContext)
       ↓  资源属性 → ObjectSet
       ↓  IAM Path 标准化
6. PDP: 初始化环境 (InitEnvironments)
       ↓  检测 tz → 计算 hms → 注入 ObjectSet
       ↓
7. PDP: 评估 (EvalPolicies / PartialEvalPolicies)
       ↓
   ┌─────────────────────────────────────────┐
   │  EvalPolicies (auth 模式)               │
   │  逐条策略 → cond.Eval(ctx)              │
   │  任一 true → 返回 (true, policyID)      │
   │  全部 false → 返回 (false, -1)          │
   ├─────────────────────────────────────────┤
   │  PartialEvalPolicies (query 模式)       │
   │  逐条策略 → PartialEval(ctx)            │
   │  收集所有通过策略的剩余条件              │
   │  返回 ([]Condition, []policyID)         │
   └─────────────────────────────────────────┘
       ↓
8. (query 模式) Translate 剩余条件
       ↓  去掉 system 前缀
       ↓  合并同字段 eq/in
       ↓
9. 返回给接入方
```

---

## 十一、关键设计决策

### 11.1 为什么 Expression 独立存储？

- 多条策略可以共享同一个 Expression（通过 ExpressionPK 关联）
- 减少存储空间，提高缓存命中率
- Expression 的 Signature（MD5）可以作为全局唯一标识

### 11.2 为什么需要 PartialEval？

- 查询模式下，接入方需要知道"哪些资源有权限"
- 系统无法在不知道资源的情况下做出完整判定
- PartialEval 将"已知资源"的条件评估掉，返回"未知资源"的条件
- 接入方拿到剩余条件后，可以在自己的系统中做资源级过滤

### 11.3 为什么 StringPrefix 支持通配符？

- IAM Path 表示资源层级路径：`/biz,1/set,*/`
- `*` 表示"该层级可以是任意资源"
- 实现方式：裁剪掉 `,*/` 后缀变为 `/biz,1/set,`，然后做前缀匹配
- 只在 key 以 `_bk_iam_path_` 结尾时启用

### 11.4 为什么数值比较限制单值？

- `gt/gte/lt/lte` 语义上只应与单个阈值比较
- 多值比较在权限场景下没有明确业务含义
- 只有 `NumericEquals` 支持多值（等价于 `in` 操作）

---

## 十二、文件索引

| 文件路径 | 职责 |
|---------|------|
| `pkg/abac/pdp/condition/init.go` | Condition 接口定义、工厂注册 |
| `pkg/abac/pdp/condition/operator/types.go` | 操作符常量定义 |
| `pkg/abac/pdp/condition/base_condition.go` | 叶子条件基类（forOr 求值） |
| `pkg/abac/pdp/condition/base_logical_condition.go` | 逻辑条件基类（递归 key 收集） |
| `pkg/abac/pdp/condition/and.go` | AND 条件（Eval + PartialEval） |
| `pkg/abac/pdp/condition/or.go` | OR 条件（Eval + PartialEval） |
| `pkg/abac/pdp/condition/any.go` | Any 恒真条件 |
| `pkg/abac/pdp/condition/string_equals.go` | 字符串精确匹配 |
| `pkg/abac/pdp/condition/string_prefix.go` | 字符串前缀匹配（IAM Path） |
| `pkg/abac/pdp/condition/string_contains.go` | 字符串包含匹配 |
| `pkg/abac/pdp/condition/bool.go` | 布尔匹配 |
| `pkg/abac/pdp/condition/numeric_cmp.go` | 数值比较（5 种操作符） |
| `pkg/abac/pdp/evaluation/evaluation.go` | 策略评估入口（Eval + PartialEval） |
| `pkg/abac/pdp/evalctx/context.go` | 评估上下文（ObjectSet + 环境属性） |
| `pkg/abac/pdp/translate/translate.go` | 表达式翻译（去 system 前缀 + 合并） |
| `pkg/abac/pdp/types/policy_condition.go` | PolicyCondition 类型 + 旧版转换 |
| `pkg/abac/pdp/types/object.go` | ObjectSet 属性集合 |
| `pkg/abac/pdp/types/util.go` | InterfaceToPolicyCondition 反序列化辅助 |
| `pkg/abac/prp/expression/init.go` | Expression 三级缓存入口 |
| `pkg/abac/prp/expression/types.go` | Retriever 接口定义 |
| `pkg/abac/prp/expression/memory.go` | L1 Memory Retriever |
| `pkg/abac/prp/expression/redis.go` | L2 Redis Retriever |
| `pkg/abac/prp/expression/database.go` | L3 Database Retriever |
| `pkg/cacheimpls/local_unmarshaled_expression.go` | 反序列化缓存（JSON → Condition） |
