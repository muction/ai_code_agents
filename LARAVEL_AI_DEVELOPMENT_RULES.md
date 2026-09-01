# Laravel AI 开发规范

> 本文件用于约束 AI 在 Laravel 项目中的分析、设计、编码、重构和审查行为。除非项目已有明确规范与本文件冲突，否则必须遵守本文件。

## 1. 核心原则

1. 遵循单一职责、依赖倒置、低耦合、高内聚。
2. 控制器只负责接收请求和调度，不承载业务流程。
3. Service 负责业务规则、业务流程和事务边界。
4. Model 只负责数据结构、关联关系、Eloquent 查询和持久化。
5. 不为了“分层”而机械增加无意义的类；每一层必须有明确职责。
6. 优先使用 Laravel 官方能力和约定，不重复封装框架已有功能。
7. 禁止把所有逻辑塞进 Controller、Model、Helper、Trait 或一个万能 Service。
8. 新代码必须兼容项目当前 PHP、Laravel 版本和既有目录结构，不得擅自升级依赖。

## 2. 标准调用方向

```text
Route
  -> FormRequest
  -> Controller
  -> Service / Action
  -> Model / Eloquent
  -> Resource / Response
```

外部系统调用：

```text
Service / Action -> Integration Client / Gateway -> External API
```

禁止反向依赖：Model 不得调用 Service，FormRequest 不得调用 Controller，Resource 不得执行业务流程。

## 3. 路由层（Route）

### 允许

- 定义 URI、HTTP 方法、控制器、路由名称。
- 配置 middleware、限流、鉴权和路由参数绑定。
- 按领域或版本组织路由文件。

### 禁止

- 在路由闭包中编写正式业务逻辑。
- 直接查询或修改数据库。
- 在路由层拼装复杂响应。

## 4. 请求验证层（FormRequest）

### 职责

- 请求字段格式、类型、长度、必填项、枚举值等输入验证。
- 当前用户是否有权发起该请求的基础授权判断。
- 提供明确、可本地化的验证消息。

### 规则

- Controller 不得堆积 `$request->validate()`；复杂接口必须使用 FormRequest。
- 使用 `$request->validated()` 获取可信数据，不得直接将 `$request->all()` 写入数据库。
- `prepareForValidation()` 只做输入标准化，不做业务查询或状态变更。
- 数据库唯一性等输入约束可使用 Laravel 验证规则；复杂业务资格判断必须交给 Service。

## 5. 控制器层（Controller）

### 职责

- 接收经过验证的请求。
- 获取路由参数和当前认证用户。
- 调用一个主要的 Service 或 Action 完成用例。
- 将结果交给 Resource 或统一响应组件。
- 返回正确的 HTTP 状态码。

### 禁止

- 编写大量业务逻辑、状态流转、金额计算或业务条件分支。
- 直接开启或提交数据库事务。
- 编写复杂 Eloquent 查询、循环查询或跨 Model 数据拼装。
- 直接调用第三方 API。
- 捕获所有 `Throwable` 后返回模糊错误。
- 在多个控制器中复制相同流程。

### 建议形态

```php
public function store(StoreOrderRequest $request, OrderService $service): OrderResource
{
    $order = $service->create(
        $request->user(),
        OrderData::from($request->validated())
    );

    return new OrderResource($order);
}
```

## 6. 业务服务层（Service / Action）

### Service 职责

- 实现业务规则、业务流程和跨实体协作。
- 调用多个 Model、领域服务或外部系统。
- 处理状态流转、业务计算、幂等性和一致性。
- 定义数据库事务边界。
- 派发事件、任务或通知。

### Action 的使用

- 一个清晰、独立的用例可使用 Action，例如 `CreateOrderAction`。
- 包含一组紧密相关业务能力时可使用 Service，例如 `OrderService`。
- 不得同时创建 Service 和 Action，仅互相转发而没有实际职责。

### 禁止

- 编写原生 SQL。
- 重复封装 `find`、`where`、`paginate` 等无业务含义的基础查询。
- 直接读取 HTTP Request、返回 JsonResponse 或依赖 Controller。
- 将无关领域逻辑集中到一个万能 Service。
- 吞掉异常或仅返回 `true/false` 隐藏失败原因。

### 事务规则

- 事务由 Service / Action 控制，不由 Controller 或 Model 控制完整业务事务。
- 使用 `DB::transaction()`，事务范围应尽量小。
- 默认不要在数据库事务内执行耗时外部 HTTP 调用。
- 需要事务提交后执行的事件、通知或队列，应使用 after-commit 机制。
- 并发更新关键数据时，应评估唯一索引、原子更新、悲观锁或幂等键。

## 7. 数据模型层（Model / Eloquent）

### 职责

- 定义表名、主键、时间戳、casts、fillable/guarded。
- 定义 Eloquent 关联关系。
- 定义与数据查询直接相关的 local scope。
- 封装有明确数据语义、可复用的 Eloquent 查询。
- 处理模型自身的轻量数据转换。

### 禁止

- 关心 HTTP、Request、Response、当前路由或控制器。
- 编排跨模块业务流程。
- 调用外部 API、发送通知或承担复杂状态机。
- 在 accessor 中执行查询、网络调用或副作用操作。
- 滥用 model event 隐式执行关键业务流程。
- 使用原生 SQL，除非项目负责人明确批准且 Eloquent/Query Builder 无法合理实现。

### 查询规范

- 优先使用 Eloquent；复杂查询可使用 Query Builder。
- 业务层提出“需要什么数据”，查询细节由 Model 的 scope 或具有明确语义的查询方法承载。
- 禁止创建只转发 Eloquent 基础方法的 Repository。
- 必须主动避免 N+1，按需使用 `with`、`load`、`withCount`。
- 大数据处理使用 `chunkById`、`lazyById` 或游标，禁止无界 `get()`。
- 查询必须明确字段和排序；分页接口必须设置合理上限。

## 8. DTO / Data 对象

- 当参数较多、跨层传递或需要明确类型时，使用 DTO/Data 对象。
- DTO 只承载数据，可做安全的类型转换，不执行查询和业务流程。
- 不要把 Request 对象传入 Service。
- 不要使用含义不明的多层数组传递核心业务数据。
- DTO 命名应体现用途，例如 `CreateOrderData`，不得使用笼统的 `CommonData`。

## 9. API Resource / 响应层

- API 输出统一使用 Resource、ResourceCollection 或项目统一响应类。
- Resource 只负责输出字段映射、格式转换和已加载关系展示。
- 禁止在 Resource 中查询数据库、调用 Service 或修改数据。
- 不得直接暴露 Model 全字段，尤其是密码、令牌、内部备注和软删除字段。
- 列表响应应保持统一的 data、meta、links 结构。
- HTTP 状态码必须符合语义：创建 201、无内容 204、验证失败 422、未认证 401、无权限 403、资源不存在 404、冲突 409。

## 10. 异常与错误处理

- 业务失败使用有语义的业务异常，例如 `InsufficientBalanceException`。
- 数据不存在优先使用 `findOrFail` 或领域化异常，禁止静默返回空数据掩盖问题。
- 在全局异常处理器中统一映射 HTTP 状态码和错误结构。
- 禁止在每个 Controller 重复 `try/catch`。
- 只捕获能够恢复、补偿或转换语义的异常；无法处理的异常继续抛出。
- 日志不得记录密码、Token、身份证号等敏感数据。
- 面向客户端的错误信息不得泄露 SQL、堆栈、服务器路径和内部配置。

## 11. 权限与安全

- 身份认证交给 Guard/Middleware；资源权限优先使用 Policy/Gate。
- FormRequest 的 `authorize()` 可调用 Policy，但不得复制复杂权限规则。
- 所有写操作必须考虑越权、批量赋值和资源归属校验。
- 禁止直接信任客户端传入的用户 ID、租户 ID、价格、状态等关键字段。
- 多租户项目的查询和写入必须始终带租户边界。
- 上传文件必须校验类型、大小和扩展名；保存路径和公开访问策略必须明确。

## 12. Event / Listener

- Event 表达已经发生的业务事实，例如 `OrderPaid`，使用过去式命名。
- Listener 负责后续反应，不应改变事件已经成立的核心事实。
- 核心同步流程不要依赖不可见、难追踪的事件链。
- Listener 必须考虑重复消费、失败重试和幂等性。
- 事务相关事件应在提交后派发。

## 13. Job / Queue

- 耗时、可重试、无需立即返回结果的工作放入队列。
- Job 参数应传稳定标识或可安全序列化的数据，避免传递巨大对象。
- 必须根据业务设置 tries、timeout、backoff 和 failed 处理。
- Job 必须可幂等执行，重试不得重复扣款、重复发放或重复通知。
- 禁止把关键业务校验完全推迟到 Job，导致接口先返回虚假成功。

## 14. 第三方集成层（Client / Gateway）

- 第三方 HTTP、支付、短信、存储等调用必须集中在独立 Client/Gateway。
- Service 依赖接口，不直接散落 `Http::` 调用。
- 统一处理认证、超时、重试、错误映射和日志脱敏。
- 外部响应必须转换为项目内部 DTO，禁止让第三方响应结构污染业务层。
- 涉及不可逆操作时必须设计幂等键、回调验签和对账机制。

## 15. Repository 使用边界

- Laravel 项目默认不强制 Repository 层。
- 只有以下情况可引入：需要替换数据源、聚合复杂持久化策略、跨存储实现统一接口，或项目已有明确规范。
- 禁止为每个 Model 自动生成 Repository。
- 禁止写 `UserRepository::find($id)` 这类纯 Eloquent 转发包装。
- Repository 若存在，只处理持久化抽象，不承载业务规则。

## 16. Observer / Trait / Helper 使用边界

### Observer

- 仅处理与模型生命周期紧密相关、低风险且可预期的副作用。
- 关键业务流程应显式放在 Service，不得隐藏在 Observer 中。

### Trait

- 只复用稳定、内聚的行为。
- 禁止用 Trait 拼装大型业务流程或制造隐式依赖。

### Helper

- 只放无状态、无副作用、通用的纯函数。
- 禁止把数据库查询、业务规则、外部调用放进全局 Helper。

## 17. 数据库与迁移

- 所有表结构变更必须通过 migration，禁止只改数据库不提交 migration。
- 表、字段、索引、外键、默认值和 nullable 必须符合业务语义。
- 高频查询字段、关联字段、唯一约束应建立合适索引。
- 金额使用 decimal 或最小货币单位整数，禁止使用 float/double。
- 状态值使用 PHP Enum 或项目统一常量，并设置数据库层合理约束。
- 外键删除策略必须明确，禁止无意识级联删除关键业务数据。
- 生产环境 migration 应评估锁表、数据回填和回滚风险。
- migration 的 `down()` 必须真实可回退；不可逆操作需明确说明。

## 18. 缓存与锁

- 缓存属于性能优化，不得成为唯一事实来源。
- 缓存 key 必须包含业务命名空间、实体标识和必要的租户标识。
- 写操作后必须明确缓存失效策略。
- 防止缓存穿透、击穿和无期限脏数据。
- 分布式锁必须设置超时，并使用 owner token 安全释放。
- 不要用缓存锁代替数据库唯一约束和最终一致性设计。

## 19. 日志与审计

- 日志应包含可追踪上下文，例如 request_id、user_id、tenant_id、业务单号。
- 使用结构化上下文数组，不拼接难以检索的长字符串。
- 普通日志、业务审计日志和异常日志应区分用途。
- 关键状态变更应保留操作人、前后状态、时间和来源。
- 禁止记录密码、完整 Token、银行卡号等敏感信息。

## 20. 命名与代码风格

- 遵循 PSR-12、Laravel 命名约定和项目现有代码风格。
- 类名、方法名必须表达业务意图，禁止 `handleData`、`doSomething` 等模糊命名。
- 方法保持短小；参数过多时使用 DTO。
- 优先类型声明、返回类型、Enum 和 readonly DTO。
- 注释解释“为什么”，代码本身表达“做什么”。
- 禁止保留无用代码、注释掉的大段旧实现和调试输出。
- 不得擅自引入新的设计模式、第三方包或基础架构。

## 20.1 类、方法与属性注释规范

### 总体要求

- 注释必须解释职责、业务语义、约束或设计原因，禁止只把类名、方法名翻译成中文。
- 注释必须与代码同步维护；修改业务行为时必须同步检查和更新相关 PHPDoc。
- PHPDoc 用于补充类型系统和业务含义，不能代替清晰命名、类型声明和合理拆分。
- 所有注释统一使用中文；框架类型、协议名、业务专有名词可保留英文。
- 禁止生成无价值注释、错误注释、过期注释和逐行复述代码的注释。

### 必须添加类级 PHPDoc 的类

以下业务类必须在类声明上方添加 PHPDoc：

- Controller、Service、Action。
- FormRequest、Policy。
- Job、Event、Listener、Notification。
- 第三方 Client、Gateway、Adapter。
- Repository（项目确有必要使用时）。
- DTO/Data、业务 Exception、领域 Enum。
- Observer、业务 Trait。
- 命令、定时任务及项目自定义中间件。

类级 PHPDoc 必须根据实际情况说明：

1. 类的核心职责。
2. 所属业务领域或处理的具体用例。
3. 明确的职责边界，即该类不负责什么。
4. 关键事务、幂等、权限或外部依赖约束。

禁止使用下面这种只重复类名的注释：

```php
/**
 * 订单服务类。
 */
class OrderService
{
}
```

推荐：

```php
/**
 * 负责订单创建与取消流程。
 *
 * 统一编排订单、库存和优惠数据，并定义本地数据库事务边界；
 * 支付请求由 PaymentGateway 处理，本类不直接调用第三方 HTTP 接口。
 */
class OrderService
{
}
```

### 必须添加方法级 PHPDoc 的方法

以下方法必须添加 PHPDoc：

- Controller、Service、Action 中的所有 public 业务方法。
- Client、Gateway、Repository 接口及实现中的 public 方法。
- Job 的 `handle()`，Listener 的 `handle()`，命令的 `handle()`。
- 存在业务规则、状态流转、事务、锁、幂等或外部调用的方法。
- 参数或返回值无法仅通过 PHP 类型完整表达业务含义的方法。
- 会抛出业务异常或调用方需要特殊处理异常的方法。
- 为兼容历史数据、第三方协议或特殊边界而存在的方法。

方法 PHPDoc 必须按需包含：

- 首行说明该方法完成的业务目的，不仅重复方法名。
- 关键前置条件、状态限制和副作用。
- `@param`：当参数业务语义、单位、格式、取值范围或数据结构不明显时必须填写。
- `@return`：当返回值业务含义、集合元素或特殊状态不明显时必须填写。
- `@throws`：列出调用方可预期并需要处理的业务异常。
- 数组必须尽量使用 array shape、泛型或 DTO，禁止只写无信息量的 `array`。

示例：

```php
/**
 * 创建待支付订单并锁定对应库存。
 *
 * 相同幂等键重复提交时返回原订单，不重复扣减可售库存。
 *
 * @param User $buyer 下单用户，必须属于当前租户
 * @param CreateOrderData $data 已完成格式校验的下单数据
 * @return Order 已加载订单明细的订单实例
 *
 * @throws InsufficientStockException 商品可售库存不足
 * @throws ProductUnavailableException 商品不可销售
 */
public function create(User $buyer, CreateOrderData $data): Order
{
    // ...
}
```

### 可以不添加方法级 PHPDoc 的情况

- 语义清楚且没有特殊规则的短小 private/protected 方法。
- Laravel 框架标准方法，且签名、类型和实现已完整表达含义，例如简单 Resource 的 `toArray()`。
- 简单 getter、setter、Enum 辅助方法或纯类型转换方法。
- Eloquent 关系方法在方法名、返回类型和关联键均清晰时，可以不写 PHPDoc。

即使属于以上情况，只要存在特殊业务含义、非默认关联键、兼容逻辑或副作用，仍必须添加说明。

### 属性与常量注释

- 属性必须声明准确 PHP 类型；类型和名称足以表达含义时，不强制重复注释。
- 无法从名称和类型判断单位、格式、作用域或生命周期的属性，必须添加 PHPDoc 或行尾说明。
- 数组属性必须标注元素类型或 array shape。
- 业务常量和非显然的配置值必须说明含义、单位及适用范围。
- 禁止使用没有说明的魔法数字、魔法字符串和状态码。
- DTO 构造参数必须使用明确名称和类型；涉及金额、时区、比例、距离等必须说明单位。

### Model 注释

- Model 类必须有类级 PHPDoc，说明对应业务实体、数据表和核心职责边界。
- Model 的数据库字段必须能够通过 IDE Helper、自动生成的 `@property`/`@property-read` 或项目统一方案被识别。
- 若项目手工维护 Model PHPDoc，则字段类型、nullable、关联属性必须准确并与 Migration 同步。
- 不得为了满足注释要求手工复制一份极易过期且无人维护的字段清单；优先使用项目已有自动生成方案。
- Eloquent scope 必须使用 `scopeXxx` 或 PHP 8 attribute 等项目既有规范，并说明非显然的筛选语义。
- 非默认外键、跨租户限制、包含软删除数据等特殊关联必须写明。

示例：

```php
/**
 * 订单持久化模型，对应 orders 表。
 *
 * 仅定义字段转换、关联关系和订单查询范围，不编排下单或支付流程。
 *
 * @property int $id 订单主键
 * @property int $tenant_id 所属租户主键
 * @property string $order_no 租户内唯一订单编号
 * @property OrderStatus $status 当前订单状态
 * @property-read User $buyer 下单用户
 */
class Order extends Model
{
}
```

### 接口与实现类注释

- 接口必须说明抽象能力、输入输出约定和失败语义。
- 实现类不得机械复制接口的全部 PHPDoc；若行为完全一致，可使用 `{@inheritDoc}` 或依赖 IDE 继承展示。
- 实现类存在额外限制、缓存、重试或第三方差异时，必须单独补充说明。

### 行内注释

以下代码必须添加行内或代码块注释，解释“为什么这样做”：

- 容易被误认为可以简化的特殊逻辑。
- 并发锁、原子更新、幂等判断和事务顺序。
- 第三方接口兼容、签名、重试和异常映射。
- 非直观算法、复杂计算公式和边界处理。
- 为修复特定 Bug 保留的兼容逻辑，应关联工单号或问题背景（如果存在）。
- 性能优化导致代码不直观的实现。

禁止：

```php
// 查询订单
$order = Order::findOrFail($id);

// 状态设置为已支付
$order->status = OrderStatus::PAID;
```

推荐：

```php
// 使用条件更新防止支付回调并发到达时重复推进状态和重复派发权益。
$updated = Order::query()
    ->whereKey($orderId)
    ->where('status', OrderStatus::PENDING)
    ->update(['status' => OrderStatus::PAID]);
```

### 注释审查标准

AI 完成代码后必须检查：

1. 新增业务类是否具有类级职责说明。
2. public 业务方法是否说明业务目的、关键规则和异常。
3. 参数的单位、格式、取值范围和 nullable 语义是否明确。
4. 复杂状态流转、事务、锁和第三方兼容逻辑是否说明原因。
5. 注释是否与当前代码一致，是否存在复制粘贴或过期内容。
6. 删除注释后若代码含义完全不受影响，该注释是否属于无价值重复。

## 21. 测试规则

- 新增或修改业务必须补充对应测试。
- 业务用例优先写 Feature Test；纯计算和独立规则写 Unit Test。
- 测试必须覆盖成功、验证失败、无权限、资源不存在、业务冲突和关键边界。
- 外部服务使用 Fake/Mock，不得在自动化测试中调用真实支付、短信等生产接口。
- 队列、事件、通知使用 Laravel Fake 验证派发行为。
- 禁止只验证 HTTP 200；必须验证响应结构、数据库结果和副作用。
- 修复 Bug 时先增加能复现问题的回归测试。

## 22. AI 修改代码的工作流程

AI 在生成或修改代码前必须：

1. 阅读项目的 `composer.json`，确认 PHP 与 Laravel 版本。
2. 阅读相关 Route、Controller、Request、Service、Model、Resource、Migration 和 Test。
3. 复用项目现有目录、命名、响应格式、异常体系和编码风格。
4. 明确本次需求属于哪一层，不跨层堆放逻辑。
5. 检查是否存在可复用实现，避免重复创建近似类。
6. 只修改完成需求所必需的文件，不进行无关重构。

AI 完成代码后必须：

1. 检查调用方向是否正确，是否存在反向依赖。
2. 检查 Controller 是否只负责调度。
3. 检查 Service 是否承载业务规则及正确事务边界。
4. 检查 Model 是否只包含 Eloquent 和数据相关逻辑。
5. 检查 N+1、无界查询、重复查询、缺失索引和并发风险。
6. 检查权限、租户边界、批量赋值和敏感字段泄露。
7. 检查异常、日志、队列重试和第三方调用幂等性。
8. 运行相关测试、代码格式化和项目已有静态检查。
9. 报告修改文件、关键设计、验证结果和仍存在的风险。

## 23. AI 强制自检清单

提交代码前，AI 必须逐项确认：

- [ ] Controller 中没有复杂查询、事务、业务计算和状态流转。
- [ ] FormRequest 只负责输入验证和请求授权。
- [ ] Service/Action 不依赖 HTTP Request 或 Response。
- [ ] Service/Action 中没有原生 SQL 和无意义基础查询封装。
- [ ] Model 中没有跨模块业务流程或外部 API 调用。
- [ ] Resource 中没有数据库查询和数据写入。
- [ ] 不存在明显 N+1 或循环内查询。
- [ ] 写操作已考虑事务、并发和幂等性。
- [ ] 权限、资源归属和租户边界已校验。
- [ ] 响应未泄露敏感字段和内部错误细节。
- [ ] 第三方调用具有超时、失败处理和必要的重试策略。
- [ ] 队列任务可安全重试。
- [ ] 数据库结构变更包含 migration、索引和字段说明。
- [ ] 新功能或 Bug 修复包含有效测试。
- [ ] 未引入无必要的 Repository、Trait、Helper 或第三方依赖。
- [ ] 新增业务类均有说明职责和边界的类级 PHPDoc。
- [ ] public 业务方法均有必要的业务说明、参数语义、返回值和 `@throws`。
- [ ] Model 字段与关联类型可被 IDE/静态分析识别，且与 Migration 一致。
- [ ] 事务、并发、幂等、算法及兼容逻辑均注释了设计原因。
- [ ] 不存在仅翻译类名、方法名或逐行复述代码的无效注释。
- [ ] 已运行项目允许的测试和检查命令，并如实报告结果。

## 24. 分层判断口诀

- 请求是否合法：FormRequest。
- 用户能否操作资源：Policy / Gate。
- 接口如何调度：Controller。
- 业务应该如何执行：Service / Action。
- 数据如何查询和保存：Model / Eloquent。
- 外部系统如何通信：Client / Gateway。
- 数据如何对外展示：Resource。
- 后续异步工作如何执行：Job / Listener。
- 错误如何统一呈现：Exception Handler。

当无法判断逻辑归属时，优先根据“该逻辑变化的原因”决定：因 HTTP 输入变化而变，归请求/控制器层；因业务规则变化而变，归 Service；因数据库结构或查询方式变化而变，归 Model/Eloquent；因第三方协议变化而变，归 Client/Gateway。
