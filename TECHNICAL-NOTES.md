# fsm-as-promised 技术说明：Promise 化的状态迁移机制

> 基于 `src/index.ts`（编译产物为 `lib/index.js`）与 `src/fsm-error.ts`，行号均指源码文件。

## 1. 同步/异步迁移如何被串进同一条 Promise 链

### 1.1 每个事件方法都是一条预装配的 Promise 链

状态机用 stampit 定义（`StateMachineStamp`，src/index.ts:124）。初始化时 `init`（src/index.ts:617）调用 `initTarget`（src/index.ts:576），后者对 `this.events` 里每个事件名调用 `buildEvent(name)`（src/index.ts:590-595），把生成的触发函数挂到目标对象上（src/index.ts:597）。

`buildEvent`（src/index.ts:477）返回闭包 `triggerEvent`（src/index.ts:485）。一次事件触发 = 构造一个 `options` 对象（src/index.ts:489-500：`{ name, from, to, args }`），然后把它喂进一条固定的 `.then` 链（src/index.ts:516-573）。

### 1.2 归一化：同步值、Promise、抛错一视同仁

链的起点是：

```ts
new this.factory.Promise(function (resolve) { resolve(options); })  // src/index.ts:517-519
```

这个"已 resolve 的 Promise 包裹 options"的技巧是整个库的核心：链上每一环要么是用户回调（`.bind(target, options)`），要么是 `_.identity` 透传（如 src/index.ts:528、533）。于是：

- 回调**同步返回**任意值 → 被 Promise 吸收，链继续；
- 回调**返回 Promise**（异步 onLeave/onEnter）→ 链自动等待其 settle；
- 回调**同步 throw 或返回 rejected Promise** → 统一变成链的 rejection，跳到末尾的 `.catch`（src/index.ts:572）。

注意链上各环节**不传递返回值**：每个回调拿到的都是同一个 `options`（闭包绑定），回调的返回值被丢弃，只有写进 `options.res` 的值最终由 `returnValue`（src/index.ts:274-276，`return options.res || options`）暴露给调用者。条件迁移的伪状态用 `Object.defineProperty` 把 `options.res` 桥接到 `this.responses` 存储上（`preprocessPseudoState`，src/index.ts:434-451），让"进入伪状态"和"离开伪状态"两条链共享结果。

Promise 实现可插拔：静态属性 `Promise`（src/index.ts:138）默认取 `global.Promise`，测试里通过 `StateMachine.Promise = promise` 换成 bluebird/q/when 等。

### 1.3 迁移类型与并发护栏

`type()`（src/index.ts:142-150）把迁移分三类：`NOOP`（from === to 或无 to，自环）、`GENERAL`（from 为 `*`）、`INTER`（普通跨状态）。`canTransition`（src/index.ts:189-215）据此加锁：

- `INTER`：若当前状态还有未完成的 NOOP 迁移（`noopTransitions` 非空）或 `inTransition` 为真，抛 `'Previous transition pending'`；否则置 `this.inTransition = true`（src/index.ts:209）。
- `NOOP`：仅检查 `inTransition`。

NOOP 迁移用 `options.id = v4()`（src/index.ts:504-506）登记进 `states[current].noopTransitions`（`onleavestate`，src/index.ts:267），完成时在 `onenterstate` 里删除（src/index.ts:249）——因为自环不改 `current`，只能靠这张表追踪"是否有异步迁移在飞"。

## 2. 一次事件的完整执行顺序

以 `events: [{ name: 'warn', from: 'green', to: 'yellow' }]` 触发 `fsm.warn()` 为例，链（src/index.ts:516-573）依次执行：

| # | 环节 | 代码位置 | 说明 |
|---|------|----------|------|
| 1 | `isValidEvent` | src/index.ts:520，定义于 235-241 | 当前状态不能触发该事件则抛 `FsmError('Invalid event in current state')` |
| 2 | `canTransition` | src/index.ts:521，定义于 189-215 | 并发检查；INTER 类型在此置 `inTransition = true` |
| 3 | `onleave{state}` 回调（`onleavegreen`） | src/index.ts:522-529 | 用户钩子，可返回 Promise 延迟/否决 |
| 4 | `onleave` 通用回调 | src/index.ts:530-534 | 同上 |
| 5 | 内部 `onleavestate` | src/index.ts:535，定义于 261-273 | NOOP 类型在此登记 `noopTransitions[id]` |
| 6 | `on{event}` 回调（`onwarn`） | src/index.ts:536-540 | 事件处理器本体 |
| 7 | `onenter{state}` 回调（`onenteryellow`） | src/index.ts:543-549 | 注意此刻 `current` 仍是 `green` |
| 8 | `onenter` 通用回调 | src/index.ts:550-554 | 目标是伪状态时跳过 |
| 9 | 内部 `onenterstate` | src/index.ts:555，定义于 243-260 | **状态在此才真正切换**：`inTransition = false`、`current = options.to`（src/index.ts:252-253），非伪状态时 `emit('state', ...)`（src/index.ts:255） |
| 10 | `onentered{state}` 回调（`onenteredyellow`） | src/index.ts:556-562 | 迁移已完成后的通知 |
| 11 | `onentered` 通用回调 | src/index.ts:563-570 | 同上 |
| 12 | `returnValue` | src/index.ts:571，定义于 274-276 | 把 `options.res || options` 作为 `fsm.warn()` 的 resolve 值 |
| — | `.catch(revert(options))` | src/index.ts:572，定义于 277-299 | 任一环节失败都汇聚到这里 |

关键点：**`current` 只在第 9 步被改写**。所以 `onleave*`、`on{event}`、`onenter*` 回调里读到的 `fsm.current` 都还是 `from` 状态；而第 1–8 步里任何错误发生时，状态天然还停留在 `from`，不需要"回滚状态值"，只需要回滚簿记（见下）。

### 2.1 错误传播路径

任何一环 throw/reject → 跳过中间所有环节 → 进入 `revert(options)` 生成的捕获函数（src/index.ts:277-299）：

- `INTER`：经 `instanceErrorHandler`（src/index.ts:172-188）把 `inTransition` 复位为 `false`（src/index.ts:286）；
- `NOOP`：从 `noopTransitions` 删掉该迁移 id（src/index.ts:291）；
- 然后**无条件 `throw err` 重新抛出**（src/index.ts:297），让 `fsm.warn()` 返回的 Promise 以原错误 reject，由调用方 `.catch` 接收。

`instanceErrorHandler` 的实例隔离逻辑：若错误是 `FsmError('Invalid event in current state')` 且 `err.instanceId` 不属于本实例（回调里触发了**另一台**状态机的事件失败），则**不**复位本机的簿记——避免别人的错误误清自己的 `inTransition`（配套测试 test/specs/error-isolation.js）。`instanceId` 由 `initTarget` 里的 `v4()` 生成（src/index.ts:578、603），`error()` 抛错时盖上（src/index.ts:165-171）。

### 2.2 条件迁移（choice 伪状态）

`isConditional`（src/index.ts:156）命中时走 `addConditionalEvent`（src/index.ts:333-424）：把 `from__event` 注册为伪状态，原事件变成"from → 伪状态"，再为每个候选目标注册伪事件 `伪状态--目标`。进入伪状态时，`onentered{伪状态}` 回调（src/index.ts:387-423）执行用户的 `condition`，根据其返回的索引/状态名再触发对应伪事件完成第二段迁移；无匹配则走 `no-choice` 伪事件回到 `from` 并报 `'Choice index out of range'`。`preprocessPseudoEvent`（src/index.ts:452-476）把第二段链上的 `options` 换成原始事件的 `pOptions`，所以用户回调看到的 `name/from` 仍是原始事件而非伪事件。

## 3. 与同步 FSM 在错误处理上的设计差异

1. **错误即 rejection，而非调用点异常**。同步 FSM（如 javascript-state-machine）里钩子抛错会沿调用栈炸到 `fsm.warn()` 的调用者；这里一切错误被 Promise 链吸收为 rejection，调用方必须 `.catch`，错误处理从"语法结构（try/catch）"变成"数据流（链的终点）"。

2. **状态切换被推迟到所有异步钩子完成之后**。同步 FSM 的迁移是原子的；本库把 `current = to` 放在链的第 9 环（src/index.ts:253），此前任何异步钩子失败，状态都还没动过——"回滚"退化为"不提交"，无需记录旧状态做恢复。`revert` 只需清理 `inTransition` 标志和 `noopTransitions` 登记表（src/index.ts:283-295），防止状态机被一次失败的异步迁移永久锁死。

3. **异步钩子天然具备"否决权"**。`onleave*` 返回 rejected Promise 即可在状态改变前取消迁移（test/specs/transition-lifecycle.js 验证失败后仍停留在 `from`），同步 FSM 要表达"异步等待用户确认再迁移"几乎做不到。

4. **并发错误是一等公民**。同步 FSM 不存在"迁移进行中"的概念；本库用 `inTransition` + `noopTransitions` 检测重入，报 `'Previous transition pending'` 并在 `FsmError` 上携带 `pending` 明细（src/index.ts:200-204，src/fsm-error.ts:27-29）。

5. **错误对象结构化、可跨实例隔离**。`FsmError`（src/fsm-error.ts:8-31）携带 `trigger`（事件名）、`current`（所在状态）、`instanceId`（实例 id），配合 `instanceErrorHandler` 实现多实例间的错误归因；还支持配置级 `error` 工厂（src/index.ts:113-121）让用户自定义抛出的错误类型（见 test/specs/error-handling.js 的 "Custom error handler"）。

6. **错误可被钩子自己吞掉、迁移继续**。因为链上各环节只看返回值，回调内部 try/catch 后正常返回，迁移照常走完（test/specs/error-handling.js 的 "Graceful error recovery"：在 `onwarn` 里捕获自身错误后，`onenteryellow` 依然执行，最终落到 `yellow`）。同步 FSM 里这当然也可行，但在这里它与"异步恢复"（catch 后返回新 Promise）统一成了同一种模式。
