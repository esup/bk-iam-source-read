# 蓝鲸权限中心 (BK-IAM) 项目分析

## 1. 项目概述

**项目名称**: 蓝鲸智云-权限中心 (BlueKing-IAM)  
**开发语言**: Go 1.22  
**开源协议**: MIT  
**所属组织**: THL A29 Limited (腾讯公司)  

蓝鲸权限中心是一个身份与访问管理系统，用于安全地控制系统资源的访问权限。它是蓝鲸智云 PaaS 平台的核心组件之一，提供基于属性的访问控制 (ABAC) 和基于角色的访问控制 (RBAC) 能力。

---

## 2. 技术栈

| 分类 | 技术选型 |
|------|----------|
| **Web 框架** | Gin (`github.com/gin-gonic/gin`) |
| **ORM/数据库** | sqlx + MySQL (`github.com/jmoiron/sqlx`, `github.com/go-sql-driver/mysql`) |
| **缓存** | Redis (`github.com/go-redis/redis/v8`) + 本地内存缓存 (`github.com/wklken/go-cache`, `github.com/TencentBlueKing/gopkg/cache/memory`) |
| **消息队列** | Redis-based RMQ (`github.com/adjust/rmq/v4`) |
| **配置管理** | Viper + Cobra (`github.com/spf13/viper`, `github.com/spf13/cobra`) |
| **日志** | Logrus + Zap (`github.com/sirupsen/logrus`, `go.uber.org/zap`) |
| **监控** | Prometheus (`github.com/prometheus/client_golang`) |
| **错误追踪** | Sentry (`github.com/getsentry/sentry-go`) |
| **API 文档** | Swagger (`github.com/swaggo/swag`) |
| **分布式锁** | Redis Lock (`github.com/bsm/redislock`) |
| **序列化** | JSON Iterator + MsgPack (`github.com/json-iterator/go`, `github.com/vmihailenco/msgpack/v5`) |
| **测试** | Ginkgo/Gomega + GoMock + GoSqlMock + Gomonkey |

---

## 3. 项目结构

```
bk-iam
├── cmd/                    # 命令行入口
│   ├── iam.go              # 主服务启动 (bk-iam)
│   ├── worker.go           # 异步任务 Worker 启动 (bk-iam worker)
│   ├── checker.go          # 检查命令
│   ├── transfer.go         # 数据迁移命令
│   ├── init.go             # 初始化配置/日志/数据库等
│   └── version.go          # 版本信息
├── pkg/
│   ├── abac/               # ABAC 权限模型核心
│   │   ├── pap/            # 策略管理 (Policy Administration Point)
│   │   ├── pdp/            # 策略决策 (Policy Decision Point)
│   │   ├── pip/            # 策略信息 (Policy Information Point)
│   │   ├── prp/            # 策略检索 (Policy Retrieval Point)
│   │   └── types/          # 权限模型数据类型
│   ├── api/                # API 视图层 (Controller)
│   │   ├── basic/          # 基础 API (/ping, /metrics)
│   │   ├── common/         # 公共处理逻辑
│   │   ├── debug/          # 调试 API (/api/v1/debug)
│   │   ├── engine/         # 引擎 API (/api/v1/engine)
│   │   ├── model/          # 模型注册/变更 API (/api/v1/model)
│   │   ├── open/           # 开放 API (/api/v1/open)
│   │   ├── policy/         # 策略查询/鉴权 API (/api/v1/policy)
│   │   └── web/            # SaaS Web API (/api/v1/web)
│   ├── cache/              # 缓存基础设施
│   │   ├── cleaner/        # 缓存清理器
│   │   └── redis/          # Redis 缓存封装
│   ├── cacheimpls/         # 缓存实现 (30+ 缓存实例)
│   ├── component/          # 外部组件集成
│   ├── config/             # 配置结构定义
│   ├── database/           # 数据库层
│   │   ├── dao/            # 数据访问对象 (47+ 文件)
│   │   ├── edao/           # 扩展 DAO
│   │   └── sdao/           # 服务 DAO
│   ├── locker/             # 分布式锁
│   ├── logging/            # 日志系统
│   ├── metric/             # 监控指标
│   ├── middleware/         # HTTP 中间件
│   ├── server/             # HTTP 服务器 & 路由
│   ├── service/            # 业务逻辑层 (50+ 文件)
│   ├── task/               # 异步任务系统
│   ├── util/               # 工具函数
│   └── version/            # 版本管理
├── docs/                   # 文档 & Swagger
├── test/                   # 测试初始化 SQL & 脚本
└── vendor/                 # Go 依赖
```

---

## 4. 架构分层

```
┌─────────────────────────────────────────────────────┐
│                    API Layer (api/)                   │
│  basic │ web │ policy │ open │ model │ engine │ debug│
├─────────────────────────────────────────────────────┤
│               Middleware Layer (middleware/)          │
│  Auth │ RateLimit │ Metrics │ Logger │ Recovery │ Audit│
├─────────────────────────────────────────────────────┤
│                 ABAC Engine (abac/)                   │
│     PAP   │   PDP   │   PIP   │   PRP              │
├─────────────────────────────────────────────────────┤
│               Service Layer (service/)               │
│  action │ group │ policy │ subject │ resource │ ...  │
├─────────────────────────────────────────────────────┤
│              Cache Layer (cacheimpls/)               │
│  本地缓存 (memory) │ Redis 缓存 │ 缓存清理器         │
├─────────────────────────────────────────────────────┤
│              Database Layer (database/)              │
│        dao (47+ DAO) │ edao │ sdao │ mysql          │
├─────────────────────────────────────────────────────┤
│           Infrastructure                             │
│  MySQL │ Redis │ RMQ │ Prometheus │ Sentry          │
└─────────────────────────────────────────────────────┘
```

**调用链路**: `api → [abac] → cache|service → database/*dao → database`

---

## 5. 核心模块分析

### 5.1 ABAC 权限引擎

采用 XACML 标准的 ABAC (Attribute-Based Access Control) 架构:

| 组件 | 职责 |
|------|------|
| **PAP** (Policy Administration Point) | 策略管理：创建、修改、删除权限策略 |
| **PDP** (Policy Decision Point) | 策略决策：评估请求，执行策略计算，返回允许/拒绝 |
| **PIP** (Policy Information Point) | 策略信息：提供策略评估所需的属性信息 |
| **PRP** (Policy Retrieval Point) | 策略检索：从存储中检索相关策略 |

### 5.2 多级缓存体系

系统实现了 **三级缓存** 架构以优化鉴权性能:

1. **L1 - 本地内存缓存** (`memory.Cache` / `gocache.Cache`): 热点数据，如 Subject、Action、SubjectPK 等
2. **L2 - Redis 分布式缓存** (`redis.Cache`): 共享数据，如策略、表达式、系统信息等
3. **L3 - 数据库**: 最终数据源

缓存实例 30+ 个，覆盖 Subject、Action、Policy、Expression、ResourceType 等核心实体。每个缓存支持独立配置过期时间和随机抖动，防止缓存雪崩。

### 5.3 API 路由体系

| 路由组 | 路径前缀 | 用途 |
|--------|----------|------|
| basic | `/ping`, `/metrics` | 健康检查、监控指标 |
| web | `/api/v1/web`, `/api/v2/web` | SaaS 前端接口 |
| policy | `/api/v1/policy`, `/api/v2/policy` | 策略查询与鉴权 |
| open | `/api/v1/open` | 对外开放接口 |
| model | `/api/v1/model` | 权限模型注册与变更 |
| engine | `/api/v1/engine` | IAM 引擎内部接口 |
| debug | `/api/v1/debug` | 调试管理接口 |

### 5.4 中间件链

- **RequestID**: 请求唯一标识
- **Recovery**: Panic 恢复 + Sentry 上报
- **Metrics**: Prometheus 指标采集
- **ClientAuth**: AppCode/AppSecret 认证 (支持 JWT/APIGateway)
- **RateLimit**: 接口限流
- **Audit**: 操作审计日志
- **WebLogger / APILogger**: 分类请求日志
- **SuperClient / ShareClient**: 特殊客户端权限识别

### 5.5 异步任务系统

通过 `cmd/worker.go` 启动独立的 Worker 进程，基于 Redis RMQ 实现异步消息消费:
- 处理 Subject-Action 变更事件
- 执行缓存失效与重建
- 支持独立部署和水平扩展

### 5.6 数据库设计

- **双数据库支持**: DefaultDB (IAM 业务数据) + BKPaaSDB (蓝鲸 PaaS 平台数据，用于 app_code/app_secret 验证)
- **DAO 模式**: 47+ 个 DAO 文件，采用 sqlx 进行 SQL 操作
- **连接池管理**: 支持配置 MaxOpenConns、MaxIdleConns、ConnMaxLifetime
- **TLS 支持**: 数据库连接支持 TLS 加密

---

## 6. 启动流程

```
main() → cmd.Execute()
  └── Start()
       ├── 1. 加载配置 (Viper)
       ├── 2. 初始化日志 (Logrus/Zap)
       ├── 3. 初始化 Sentry (错误追踪)
       ├── 4. 初始化 Prometheus 指标
       ├── 5. 初始化 MySQL 连接
       ├── 6. 初始化 Redis 连接
       ├── 7. 初始化 RMQ Producer
       ├── 8. 初始化 30+ 缓存实例
       ├── 9. 初始化策略缓存设置
       ├── 10. 初始化认证配置
       ├── 11. 初始化组件
       ├── 12. 初始化配额/Worker/开关
       ├── 13. 监听系统信号 (SIGINT/SIGTERM)
       └── 14. 启动 HTTP Server (Gin)
```

---

## 7. 配置体系

核心配置结构 (`config.Config`):

- **Server**: 监听地址、超时设置
- **Database**: 多数据库实例 (支持 TLS)
- **Redis**: 多 Redis 实例 (支持 Sentinel 模式/TLS)
- **Sentry**: 错误追踪 DSN
- **Quota**: 模型/API/Web 配额管理
- **Cache/PolicyCache**: 缓存开关与过期策略
- **Logger**: 7 类独立日志配置 (System/API/SQL/Audit/Web/Component/Worker)
- **Worker**: 异步任务并发参数
- **Switch**: 功能开关
- **Crypto**: 加密密钥管理

---

## 8. 关键设计特点

1. **高性能鉴权**: 三级缓存 + Singleflight 防并发穿透 + 随机过期防雪崩
2. **ABAC + RBAC 混合模型**: 同时支持属性级和角色级的权限控制
3. **多系统隔离**: 通过 system_id 实现多租户权限隔离
4. **优雅停机**: 支持 Graceful Shutdown，可配置超时时间
5. **可观测性**: Prometheus 指标 + Sentry 错误追踪 + 分类日志
6. **灵活部署**: API Server 和 Worker 可独立部署
7. **配额管理**: 支持全局配额和按系统自定义配额
8. **分布式锁**: 基于 Redis 的分布式锁，用于并发控制

---

## 9. PRP 策略检索 — 三级缓存 Retriever 链式模式

PRP (Policy Retrieval Point) 是鉴权流程中获取策略数据的核心模块，对 Policy 和 Expression 均实现了**三级缓存链式检索**。

### 9.1 Retriever 接口定义

```go
// pkg/abac/prp/policy/types.go
type Retriever interface {
    retrieve(pks []int64) (policies []AuthPolicy, missingSubjectPKs []int64, err error)
    setMissing(policies []AuthPolicy, missingSubjectPKs []int64) error
}

// 下一层检索函数签名
type MissingRetrieveFunc func(pks []int64) ([]AuthPolicy, []int64, error)
```

每个 Retriever 持有一层缓存和指向下一层的 `MissingRetrieveFunc`，形成链式调用。

### 9.2 链式组装

```go
// pkg/abac/prp/policy/init.go
func GetPoliciesFromCache(system string, actionPK int64, subjectPKs []int64) {
    l3 := newDatabaseRetriever(actionPK)              // L3: MySQL
    l2 := newRedisRetriever(system, actionPK, l3.retrieve)  // L2: Redis → miss → L3
    l1 := newMemoryRetriever(system, actionPK, l2.retrieve) // L1: Memory → miss → L2 → L3
    policies, _, _ := l1.retrieve(subjectPKs)
}
```

**调用链**: `L1(Memory) → miss → L2(Redis/Pipeline) → miss → L3(MySQL/DAO)`

### 9.3 各层实现细节

| 层级 | 存储结构 | Key 设计 | 过期策略 | 特殊机制 |
|------|----------|----------|----------|----------|
| **L1 Memory** | `go-cache` 本地内存 | `{system}:{actionPK}:{subjectPK}` | 60s TTL | ChangeList 变更检测 |
| **L2 Redis** | Redis Hash (Pipeline) | key=`{system}:{subjectPK}`, field=`{actionPK}` | 7天 + 随机60s抖动 | BatchHGet/BatchHSet |
| **L3 Database** | MySQL `policy` 表 | `system_id + action_pk + subject_pks` | - | 最终数据源 |

**Expression 缓存** 结构类似，Key 维度为 `expressionPK`，同样有三级缓存链。

### 9.4 ChangeList 缓存失效机制

本地内存缓存通过 **ChangeList** (Redis Sorted Set) 实现高效的变更通知：

```
ChangeList (Redis ZSet):
  key:   "policy:{system}:{actionPK}"
  member: subjectPK (string)
  score:  unix timestamp (变更时间)
```

**工作流程**:
1. **写入时**: 策略变更 → `AddToChangeList()` 将 subjectPK 以当前时间戳写入 ZSet → `Truncate()` 清理过期条目
2. **读取时**: `FetchList()` 查询最近 60s 内的变更记录 → 对比本地缓存的 timestamp → 若缓存比 ChangeList 旧则视为 miss
3. **保护机制**: 最多取 1000 条变更记录，防止大量变更影响鉴权 API 性能

```go
// pkg/abac/prp/common/changelist.go
type ChangeList struct {
    Type     string  // "policy" 或 "expression"
    TTL      int64   // 60 秒
    MaxCount int64   // 1000 条
    KeyPrefix string
}
```

### 9.5 缓存空值填充

当某个 subjectPK 在数据库中**没有策略**时，系统仍会缓存空值：
- **Memory 层**: 缓存空的 `[]AuthPolicy{}`，TTL=60s
- **Redis 层**: 缓存空字符串 `""`，TTL=7天
- **目的**: 防止缓存穿透，避免频繁查询数据库

---

## 10. PDP 策略决策 — 条件评估与策略翻译

### 10.1 PDP 鉴权入口流程

```go
// pkg/abac/pdp/entrance.go - Eval() 核心流程
func Eval(request Request) (bool, error) {
    // 1. 填充 Action 详情 (从缓存获取 action 的属性: PK, 关联资源类型, authType)
    fillActionDetail(&request.Action)

    // 2. 校验 Action 与 Resource 的合法性
    ValidateActionResource(request.Action, request.Resources)

    // 3. 填充 Subject 部门信息 (用于组织架构继承鉴权)
    fillSubjectDepartments(&request.Subject)

    // 4. 获取 RBAC 类型的用户组 PKs
    effectGroupPKs := getEffectAuthTypeGroupPKs(request.Subject, request.Action)

    // 5. RBAC 快速判定 (如果 action 支持 RBAC)
    if rbacEval(request, effectGroupPKs) == pass {
        return true
    }

    // 6. 查询 ABAC 策略 (通过 PRP 三级缓存)
    policies := prp.PolicyManager.ListBySubjectAction(...)

    // 7. 策略评估 (条件表达式求值)
    return evaluation.EvalPolicies(ctx, policies)
}
```

### 10.2 条件评估系统 (Condition)

条件系统采用**工厂模式 + 接口多态**设计：

```go
// pkg/abac/pdp/condition/init.go
type Condition interface {
    GetName() string
    GetKeys() []string           // 条件中包含的所有属性 key
    HasKey(f keyMatchFunc) bool  // 是否匹配某个 key
    GetFirstMatchKeyValues(f keyMatchFunc) ([]interface{}, bool)
    Eval(ctx EvalContextor) bool // 核心：条件求值
    Translate(withSystem bool) (map[string]interface{}, error) // 翻译为接入方可理解的表达式
}

type LogicalCondition interface {
    Condition
    PartialEval(ctx EvalContextor) (bool, Condition) // 部分求值，返回剩余条件
}
```

**支持的操作符** (通过 `conditionFactories` 注册):

| 操作符 | 类型 | 说明 |
|--------|------|------|
| `AND` | 逻辑 | 所有子条件都满足 |
| `OR` | 逻辑 | 任一子条件满足 |
| `ANY` | 特殊 | 无条件通过 |
| `StringEquals` | 字符串 | 精确匹配 |
| `StringPrefix` | 字符串 | 前缀匹配 (用于 `_bk_iam_path_`) |
| `StringContains` | 字符串 | 包含匹配 (用于 RBAC 路径) |
| `Bool` | 布尔 | 布尔值匹配 |
| `NumericEquals/Gt/Gte/Lt/Lte` | 数值 | 数值比较 (用于时间 hms 等) |

### 10.3 策略评估模式

| 模式 | 函数 | 用途 |
|------|------|------|
| **完全评估** | `EvalPolicies()` | `auth` 接口：逐条策略评估，任一通过即返回 true |
| **部分评估** | `PartialEvalPolicies()` | `query` 接口：筛选通过的策略，返回剩余条件供接入方二次判定 |

**部分评估 (PartialEval)** 的核心思想：
- 如果请求中**缺少**某些资源属性，则将该条件作为"剩余条件"返回
- 接入方可根据剩余条件自行判定或提示用户补充资源

### 10.4 表达式翻译 (Translate)

将内部条件表达式转换为接入系统可理解的格式：

```go
// 内部表达式: {system}.{type}.{attr} eq abc
// 翻译后v1: {type}.{attr} eq abc  (当前默认，去除 system 前缀)
// 翻译后v2: {system}.{type}.{attr} eq abc (未来支持)
```

翻译过程还会**合并同字段条件**：多个 `eq`/`in` 相同 field 的条件合并为一个 `in` 条件。

### 10.5 评估上下文 (EvalContext)

```go
// pkg/abac/pdp/evalctx/context.go
type EvalContext struct {
    *request.Request        // 包含 System, Action, Subject, Resources
    objSet ObjectSetInterface // 资源属性集合，key = "{system}.{type}"
}
```

- 资源属性通过 `{system}.{resource_type}` 为 key 存入 ObjectSet
- 条件求值时通过 `GetAttr("{system}.{type}.{attr}")` 获取属性值
- 支持**环境属性**注入：时区(tz)、时分秒(hms) 等，用于时间维度的权限控制
- IAM Path 标准化：自动将 3 段式路径 `/system,type,id/` 裁剪为 2 段式 `/type,id/`

---

## 11. RBAC 策略缓存与 Singleflight

### 11.1 RBAC 表达式缓存

RBAC 策略采用**两级缓存** (Redis → Database)，不使用本地内存缓存：

```go
// pkg/abac/prp/rbac/redis.go
type PolicyRedisRetriever struct {
    subjectActionExpressionService    // 查询 subject_action_expression 表
    subjectActionGroupResourceService // 查询 subject_action_group_resource 表
    groupAlterEventService            // 发送组变更事件
    G singleflight.Group              // 防并发穿透
}
```

**缓存 Key**: `{subjectPK}:{actionPK}` → value 为序列化的 `SubjectActionExpression`

### 11.2 Singleflight 防并发

当缓存过期需要重新计算时，使用 `singleflight.Group` 确保同一 `subjectPK:actionPK` 只有一个请求真正执行计算：

```go
key := subjectPK + ":" + actionPK
expI, err, _ := r.G.Do(key, func() (interface{}, error) {
    return r.refreshSubjectActionExpression(subjectPK, actionPK)
})
```

### 11.3 过期刷新与变更事件

当发现缓存数据已过期时：
1. 通过 Singleflight 调用 `refreshSubjectActionExpression()`
2. 从 `subject_action_group_resource` 表重新计算表达式
3. 更新 Redis 缓存
4. **异步**发送 `GroupAlterEvent` 通知其他节点更新

---

## 12. Engine 策略管理 — ABAC/RBAC 双类型

### 12.1 Policy ID 编码规则

为区分 ABAC 和 RBAC 策略的 ID，系统采用 ID 偏移编码：

| 策略类型 | 数据库表 | ID 范围 |
|----------|----------|----------|
| ABAC | `policy` | 0 ~ 500,000,000 |
| RBAC | `rbac_group_resource_policy` | 500,000,000 ~ 1,000,000,000 |

```go
const rbacIDBegin = 500000000

// 输入 PK → 真实 PK: pk - 500000000
// 真实 PK → 输出 PK: realPK + 500000000
```

### 12.2 EnginePolicy 统一输出

```go
type EnginePolicy struct {
    Version    string
    ID         int64                  // ABAC=原始PK, RBAC=PK+500000000
    ActionPKs  []int64               // ABAC=单操作, RBAC=多操作
    SubjectPK  int64
    Expression map[string]interface{} // 翻译后的表达式
    TemplateID int64
    ExpiredAt  int64
    UpdatedAt  int64
}
```

- ABAC 策略: `ActionPKs` 为单个 action，Expression 来自 expression 表
- RBAC 策略: `ActionPKs` 为多个 action，Expression 动态构造 (`eq` 或 `string_contains`)

### 12.3 RBAC 表达式动态构造

```go
// 资源类型相同: pipeline.id eq 123
// 资源类型不同: pipeline._bk_iam_path_ string_contains "/project,1/"
```

---

## 13. PRP PolicyManager 策略聚合

`PolicyManager.ListBySubjectAction()` 是鉴权时获取策略的总入口，聚合三种策略来源：

```
1. 普通权限 (ABAC)
   └── policy.GetPoliciesFromCache() → 三级缓存链

2. 临时权限 (Temporary)
   └── temporary.NewPolicyCacheRetriever() → Redis + DB

3. RBAC 权限 (仅当 action.authType == RBAC 时)
   └── rbac.NewPolicyRedisRetriever() → Redis + DB + Singleflight
```

**去重机制**: 通过 `ExpressionSignature` (MD5) 去重，确保相同表达式不被重复评估。

**Debug 支持**: 每个步骤通过 `debug.Entry` 记录中间结果，便于问题排查。

---

## 14. 依赖关系图

```
                    ┌──────────────┐
                    │   main.go    │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │   cmd/iam    │ ←── Cobra CLI
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
       ┌──────▼──────┐ ┌──▼───┐ ┌──────▼──────┐
       │  server/    │ │ task/│ │  config/    │
       │  router.go  │ │worker│ │  config.go  │
       └──────┬──────┘ └──────┘ └─────────────┘
              │
    ┌─────────┼─────────┬──────────┐
    │         │         │          │
┌───▼───┐ ┌──▼──┐ ┌───▼───┐ ┌───▼────┐
│ api/  │ │abac/│ │cache/ │ │service/│
│views  │ │PDP  │ │impls  │ │logic   │
└───────┘ │PAP  │ │       │ │        │
          │PIP  │ │redis  │ │        │
          │PRP  │ │memory │ │        │
          └─────┘ └───────┘ └────────┘
                            │
                    ┌───────▼───────┐
                    │  database/    │
                    │  dao │ mysql  │
                    └───────────────┘
```
