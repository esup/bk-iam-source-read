# BK-IAM 核心概念

## 1. 领域模型总览

BK-IAM 是一个基于 **ABAC + RBAC 混合模型** 的细粒度权限管理系统。其核心领域模型由以下概念构成：

```
┌─────────────────────────────────────────────────────────────────┐
│                         System (接入系统)                        │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐ │
│  │ ResourceType │  │    Action    │  │ InstanceSelection     │ │
│  │  (资源类型)   │←─│   (操作)     │←─│  (资源选择方式)        │ │
│  └──────┬───────┘  └──────┬───────┘  └───────────────────────┘ │
│         │                 │                                     │
│         │          ┌──────▼───────┐                             │
│         │          │   Resource   │                             │
│         └─────────→│   (资源实例)  │                             │
│                    └──────────────┘                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Subject ──→ Group ──→ Policy ──→ Expression                   │
│  (被授权    (用户组)   (策略)     (条件表达式)                    │
│   对象)                                                         │
│                                                                 │
│  Subject 类型: user / department / group                        │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  鉴权请求 (Request)                                             │
│  = System + Subject + Action + Resources                        │
│                    ↓                                            │
│  鉴权结果 = PDP 评估 Policy 的 Expression 是否满足               │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. System — 接入系统

**定义**: 接入权限中心的业务系统，每个系统拥有独立的权限模型。

```go
// pkg/service/types/system.go
type System struct {
    ID             string                 // 系统唯一标识, 如 "demo", "bk_job"
    Name           string                 // 系统中文名
    NameEn         string                 // 系统英文名
    Clients        string                 // 允许调用的客户端列表
    ProviderConfig map[string]interface{} // 回调配置 (host, token, auth方式)
}
```

**核心关系**:
- 一个 System 下定义多个 `ResourceType` 和 `Action`
- 权限数据按 `system_id` 隔离，实现多租户
- 通过 `ProviderConfig` 配置与接入系统的回调通信（资源属性查询等）

---

## 3. Subject — 被授权对象

**定义**: 权限的承载者，即"谁"拥有权限。Subject 有三种类型：

| 类型 | 说明 | PK 范围 |
|------|------|---------|
| `user` | 单个用户 | 1 ~ 99 (纯用户), 100+ (带组/部门的用户) |
| `department` | 部门 | 1xxx |
| `group` | 用户组 | 2xxx |

```go
// pkg/abac/types/subject.go — 鉴权时的 Subject
type Subject struct {
    Type      string            // "user" / "department" / "group"
    ID        string            // 业务 ID, 如 "admin", "user001"
    Attribute *SubjectAttribute // 内部属性: PK, 部门列表
}

// pkg/service/types/subject.go — Service 层的 Subject
type Subject struct {
    PK   int64  // 内部主键, 全局唯一
    Type string
    ID   string
    Name string
}
```

**关键属性**:
- **PK** (Primary Key): 全局唯一的整数主键，用于缓存和数据库查询优化
- **Departments**: 用户所属的部门 PK 列表，用于组织架构权限继承

**Subject 之间的关系**:
- 用户可以加入 **用户组** (SubjectGroup)
- 用户可以属于 **部门** (SubjectDepartment)
- 部门也可以加入 **用户组**
- 权限可以通过 **用户 → 部门 → 用户组** 的链路继承

---

## 4. Action — 操作

**定义**: 接入系统中可被控制的操作行为，如"查看应用"、"编辑流水线"。

```go
// pkg/service/types/action.go
type Action struct {
    ID                   string               // 操作 ID, 如 "view_app"
    Name                 string               // 操作名称
    AuthType             string               // 鉴权类型: "abac" / "rbac" / "none" / "all"
    Type                 string               // 操作类型
    Sensitivity          int64                // 敏感度
    RelatedResourceTypes []ActionResourceType // 关联的资源类型
    RelatedActions       []string             // 关联的前置操作
    RelatedEnvironments  []ActionEnvironment  // 关联的环境属性
    Hidden               bool                 // 是否隐藏
    Version              int64                // 版本号
}
```

**核心属性**:

| 属性 | 说明 |
|------|------|
| **AuthType** | 决定该操作使用哪种鉴权模型: `abac`(属性条件)、`rbac`(资源实例匹配)、`none`(无鉴权)、`all`(两者都支持) |
| **RelatedResourceTypes** | 操作关联的资源类型列表，决定鉴权时需要传入哪些资源 |
| **RelatedActions** | 前置依赖操作，如"开发应用"依赖"访问开发者中心" |
| **RelatedEnvironments** | 支持的环境属性（如时间维度控制） |

**ActionResourceType** — 操作与资源类型的关联:

```go
type ActionResourceType struct {
    System      string // 资源类型所属系统
    ID          string // 资源类型 ID
    SelectionMode string // 选择模式
    InstanceSelections []map[string]interface{} // 资源选择方式
}
```

---

## 5. ResourceType — 资源类型

**定义**: 接入系统中可被权限控制的资源种类。

```go
// pkg/service/types/resource_type.go
type ResourceType struct {
    ID             string                   // 资源类型 ID, 如 "app", "pipeline"
    Name           string                   // 中文名
    Parents        []map[string]interface{} // 父资源类型 (层级关系)
    ProviderConfig map[string]interface{}   // 资源提供者配置 (回调 API 路径)
    Sensitivity    int64                    // 敏感度
    Version        int64                    // 版本号
}
```

**核心概念**:
- 资源类型属于某个 System，如 `demo:app`、`bk_ci:pipeline`
- 资源类型可以有 **父子层级** (Parents)，如"应用"属于"项目"
- 通过 **ProviderConfig** 配置资源实例的回调查询接口

---

## 6. Resource — 资源实例

**定义**: 鉴权请求中传入的具体资源实例。

```go
// pkg/abac/types/resource.go — 鉴权时的 Resource
type Resource struct {
    System    string    // 所属系统
    Type      string    // 资源类型
    ID        string    // 资源实例 ID
    Attribute Attribute // 资源属性 (map), 用于条件匹配
}

// RBAC 鉴权用的资源节点
type ResourceNode struct {
    System string
    Type   string
    ID     string
    TypePK int64
}
```

**IAM Path (`_bk_iam_path_`)**:

资源属性中的特殊字段，用于表示资源的层级路径：

```
2段式 (ABAC): /resource_type_id,resource_id/
3段式 (RBAC): /system_id,resource_type_id,resource_id/
```

系统会自动将 3 段式裁剪为 2 段式进行条件匹配。

---

## 7. Group — 用户组

**定义**: 将多个 Subject (用户/部门) 组织在一起，便于批量授权。

```go
// Group 本质上也是 Subject, type="group"
// 通过 subject_relation 表建立关联

type SubjectGroup struct {
    PK        int64     // 关系 PK
    GroupPK   int64     // 用户组的 subject_pk
    ExpiredAt int64     // 过期时间
    CreatedAt time.Time
}

type GroupMember struct {
    PK        int64     // 关系 PK
    SubjectPK int64     // 成员的 subject_pk
    ExpiredAt int64     // 过期时间
    CreatedAt time.Time
}
```

**关键特性**:
- 用户组本身也是一个 Subject (type="group")，可以拥有自己的策略
- 成员关系有 **过期时间** (ExpiredAt)，支持临时授权
- 支持 **用户 → 用户组** 和 **部门 → 用户组** 两种关联方式
- 每个用户组在每个系统下有 **AuthType** 配置 (group_system_auth_type 表)

---

## 8. Policy — 策略

**定义**: 描述"谁(Subject)对哪个操作(Action)在什么条件下(Expression)拥有权限"。

### 8.1 ABAC 策略

```
Policy = Subject + Action + Expression + ExpiredAt
```

```go
// 数据库存储结构
// policy 表: subject_pk + action_pk + expression_pk + expired_at

// 鉴权时使用的策略
type AuthPolicy struct {
    PK           int64  // 策略主键
    SubjectPK    int64  // 被授权的 Subject PK
    ExpressionPK int64  // 关联的表达式 PK
    ExpiredAt    int64  // 过期时间戳
}
```

**策略来源**:
1. **自定义权限**: 用户直接授予的策略 (template_id=0)
2. **权限模板**: 通过模板批量授予的策略 (template_id!=0)
3. **用户组继承**: 用户通过所属用户组获得的策略

### 8.2 RBAC 策略

```go
// rbac_group_resource_policy 表
type RbacGroupResourcePolicy struct {
    PK         int64
    GroupPK    int64   // 用户组 PK
    TemplateID int64
    SystemID   string
    ActionPKs  []int64 // 关联的多个操作 PK

    // 授权的具体资源实例
    ActionRelatedResourceTypePK int64
    ResourceTypePK int64
    ResourceID     string
}
```

RBAC 策略直接绑定到**具体资源实例**，不需要 Expression 条件评估。

### 8.3 临时策略

```go
// temporary_policy 表
type TemporaryPolicy struct {
    PK         int64
    SubjectPK  int64
    ActionPK   int64
    Expression string // 直接存储表达式, 不通过 expression_pk 关联
    ExpiredAt  int64
}
```

临时策略有独立的过期时间，到期后自动失效。

### 8.4 Policy ID 编码规则

| 策略类型 | ID 范围 | 说明 |
|----------|---------|------|
| ABAC | 0 ~ 500,000,000 | policy 表自增 ID |
| RBAC | 500,000,000 ~ 1,000,000,000 | 真实 PK + 500000000 偏移 |

---

## 9. Expression — 条件表达式

**定义**: 描述策略中的资源条件约束，是 ABAC 鉴权的核心判定逻辑。

### 9.1 存储格式

```go
// expression 表
type AuthExpression struct {
    PK         int64  // 表达式主键
    Expression string // JSON 格式的条件表达式
    Signature  string // MD5 签名, 用于去重
}
```

### 9.2 表达式结构 (PolicyCondition)

```json
{
    "StringEquals": {
        "demo.app.id": ["001"]
    }
}
```

字段格式: `{system}.{resource_type}.{attribute}`

**复合表达式**:
```json
{
    "AND": {
        "content": [
            {"StringEquals": {"demo.app.id": ["001"]}},
            {"StringPrefix": {"demo.app._bk_iam_path_": ["/project,1/"]}}
        ]
    }
}
```

### 9.3 支持的操作符

| 类别 | 操作符 | 说明 | 示例 |
|------|--------|------|------|
| **逻辑** | `AND` | 所有子条件都满足 | 组合多个条件 |
| | `OR` | 任一子条件满足 | 多条件选一 |
| **特殊** | `ANY` | 无条件通过 | 不限制资源 |
| **字符串** | `StringEquals` | 精确匹配 | `id eq "001"` |
| | `StringPrefix` | 前缀匹配 | `_bk_iam_path_ startsWith "/project,1/"` |
| | `StringContains` | 包含匹配 | `_bk_iam_path_ contains "/project,1/"` |
| **数值** | `NumericEquals` | 数值相等 | `hms eq 83000` |
| | `NumericGt/Gte/Lt/Lte` | 数值比较 | `hms > 83000` |
| **布尔** | `Bool` | 布尔匹配 | `enabled eq true` |

### 9.4 条件接口 (Condition)

```go
type Condition interface {
    GetName() string
    GetKeys() []string
    HasKey(f keyMatchFunc) bool
    Eval(ctx EvalContextor) bool           // 完全求值
    Translate(withSystem bool) (map[string]interface{}, error) // 翻译输出
}

type LogicalCondition interface {
    Condition
    PartialEval(ctx EvalContextor) (bool, Condition) // 部分求值
}
```

---

## 10. AuthType — 鉴权类型

**定义**: 决定操作使用哪种鉴权模型。

```go
const (
    AuthTypeNone int64 = 0  // 无需鉴权
    AuthTypeABAC int64 = 1  // 仅 ABAC (属性条件评估)
    AuthTypeRBAC int64 = 2  // 仅 RBAC (资源实例匹配)
    AuthTypeAll  int64 = 15 // ABAC + RBAC 都支持
)
```

| 鉴权类型 | 策略来源 | 评估方式 | 适用场景 |
|----------|----------|----------|----------|
| **ABAC** | policy 表 + expression 表 | 条件表达式求值 | 细粒度属性控制 |
| **RBAC** | rbac_group_resource_policy 表 | 资源实例直接匹配用户组 | 资源级授权 |
| **All** | 两者都有 | 先 RBAC 快速判定，再 ABAC 条件评估 | 混合场景 |

---

## 11. Request — 鉴权请求

**定义**: 一次鉴权调用的输入。

```go
// pkg/abac/types/request/request.go
type Request struct {
    System    string           // 接入系统 ID
    Subject   types.Subject    // 被授权对象 (谁)
    Action    types.Action     // 操作 (做什么)
    Resources []types.Resource // 资源列表 (对什么)
}
```

**鉴权流程**:

```
Request(System, Subject, Action, Resources)
    │
    ├── 1. 填充 Action 详情 (PK, AuthType, 关联资源类型)
    ├── 2. 校验 Resources 与 Action 关联类型是否匹配
    ├── 3. 填充 Subject 部门信息
    ├── 4. 获取 Subject 关联的用户组 PKs, 按 AuthType 分组
    │
    ├── 5. [RBAC 快速判定]
    │      ├── 解析资源节点 (System:Type:ID)
    │      ├── 查询该资源+操作授权了哪些用户组
    │      └── 判断用户是否在这些用户组中 → 通过则直接返回
    │
    ├── 6. [PRP 策略检索]
    │      ├── 普通权限: 三级缓存链 (Memory→Redis→DB)
    │      ├── 临时权限: Redis + DB
    │      └── RBAC 表达式: Redis + DB + Singleflight
    │
    └── 7. [PDP 策略评估]
           ├── 逐条策略评估 Expression
           ├── 条件求值: Eval(ctx) → bool
           └── 任一通过 → 返回 Allow
```

---

## 12. InstanceSelection — 资源选择方式

**定义**: 定义前端授权时如何选择资源实例。

```go
type InstanceSelection struct {
    ID                string // 选择方式 ID, 如 "app_view"
    Name              string
    IsDynamic         bool   // 是否动态选择
    ResourceTypeChain []map[string]interface{} // 资源类型链 (层级选择路径)
}
```

例如"查看应用"操作可能关联两种资源选择方式：
- `app_view`: 直接选择应用
- `project_view`: 先选择项目，再选择项目下的应用

---

## 13. 核心关系图

```
System ────────────────────────────────────────────────┐
  │                                                    │
  ├── defines ──→ ResourceType (资源类型)               │
  │                  │                                 │
  │                  ├── has parents (层级关系)          │
  │                  └── provider_config (回调配置)      │
  │                                                    │
  ├── defines ──→ Action (操作)                        │
  │                  │                                 │
  │                  ├── related_resource_types ──→ ResourceType
  │                  ├── auth_type (abac/rbac/all)    │
  │                  └── related_actions (前置依赖)     │
  │                                                    │
  └── defines ──→ InstanceSelection (资源选择方式)      │
                     │                                 │
                     └── resource_type_chain            │
                                                       │
Subject ←── member ── Group ──→ Policy ──→ Expression  │
  │                      │         │                    │
  │                      │         ├── ABAC: condition  │
  │                      │         ├── RBAC: resource   │
  │                      │         └── Temporary        │
  │                      │                              │
  ├── belongs_to ──→ Department                         │
  │                      │                              │
  └── joins ──→ Group (via subject_relation)            │
                     │                                  │
                     └── has auth_type per system ──────┘
                         (group_system_auth_type)
```

---

## 14. 关键常量

| 常量 | 值 | 说明 |
|------|-----|------|
| `PKAttrName` | `"pk"` | Subject/Action 内部 PK 属性名 |
| `AuthTypeAttrName` | `"auth_type"` | Action 鉴权类型属性名 |
| `DeptAttrName` | `"department"` | Subject 部门属性名 |
| `IamPath` | `"_bk_iam_path_"` | 资源层级路径属性名 |
| `IamEnv` | `"_bk_iam_env_"` | 环境属性前缀 |
| `IamEnvTzSuffix` | `"_bk_iam_env_.tz"` | 时区环境属性 |
| `rbacIDBegin` | `500000000` | RBAC 策略 ID 偏移基准 |
| `AnyExpressionPK` | `-1` | 无资源约束时的表达式 PK |

---

## 15. 核心数据表

| 表名 | 用途 | 核心字段 |
|------|------|----------|
| `subject` | 所有被授权对象 | pk, type, id, name |
| `subject_department` | 用户-部门关系 | subject_pk, department_pks |
| `subject_relation` | 用户/部门-用户组关系 | subject_pk, parent_pk, policy_expired_at |
| `system_info` | 接入系统 | id |
| `resource_type` | 资源类型 | pk, system_id, id |
| `action` | 操作 | pk, system_id, id |
| `action_resource_type` | 操作-资源类型关联 | system_id, action_id, resource_type_system, resource_type_id |
| `saas_action` | 操作 SaaS 配置 | auth_type, related_actions, description |
| `saas_resource_type` | 资源类型 SaaS 配置 | provider_config, parents |
| `saas_instance_selection` | 资源选择方式 | resource_type_chain, is_dynamic |
| `policy` | ABAC 策略 | subject_pk, action_pk, expression_pk, expired_at |
| `expression` | 条件表达式 | pk, expression, signature |
| `temporary_policy` | 临时策略 | subject_pk, action_pk, expression, expired_at |
| `rbac_group_resource_policy` | RBAC 策略 | group_pk, action_pks, resource_type_pk, resource_id |
| `subject_action_expression` | Subject-Action RBAC 聚合表达式 | subject_pk, action_pk, expression, signature, expired_at |
| `subject_action_group_resource` | Subject-Action 组资源详情 | subject_pk, action_pk, group_resource(JSON) |
| `group_system_auth_type` | 用户组在系统下的鉴权类型 | system_id, group_pk, auth_type |
| `subject_system_group` | 用户在系统下的组关系缓存 | system_id, subject_pk, groups(JSON) |
