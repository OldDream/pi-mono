# 🔄 深入理解工具执行模型：并行 (Parallel) vs 顺序 (Sequential)

当 `pi-agent-core` 的核心引擎向大模型发问，大模型如果发现当前的任务需要外部帮助，它会在报文里一口气吐出一个或多个“函数调用”（Tool Calls）的指令。

面对多个需要调用的系统工具，引擎应该如何执行？

这就是我们在初始化 Agent 时传入的 `toolExecution: "parallel" | "sequential"` 配置项大展身手的地方。接下来我们从**前端高并发处理**的视角，深入挖掘这两种模式的源码差异。

---

## 1. 为什么我们需要两种策略？

想象一下你是一个前端开发者，接到了这样的需求：

- **场景 A（互相独立）**：大模型说，帮我通过工具 A 读取 `index.js` 的源码，并且通过工具 B 读取 `package.json` 的源码。这两个操作没有任何先后依赖逻辑。如果串行，白白浪费网络与 I/O 耗时！此时我们需要**并行（Parallel）**。
- **场景 B（存在依赖）**：大模型说，先帮我通过工具 A 创建一个文件夹，再通过工具 B 在该文件夹里写入一个由 A 决定的配置文件。如果并发这两个动作，写文件的工具极可能会报错 `Directory not found`。此时我们必须**顺序执行（Sequential）**。

在 `pi-agent-core` 的设计中，大模型的一个 response.content 里可能包含多个 `type === "toolCall"` 的段落。入口方法非常简单：

```typescript
// agent-loop.ts -> executeToolCalls 方法
if (config.toolExecution === "sequential") {
  return executeToolCallsSequential(...);
}
return executeToolCallsParallel(...);
```

---

## 2. 顺序执行（Sequential）：稳扎稳打的过程

顺序执行的源码极其克制，本质就是一个 `for...of` 加上 `await`，就像我们在 `async` 函数里老老实实写循环一样。

### 执行流程
1. 用 `for (const toolCall of toolCalls)` 开启遍历。
2. 派发准备开始的 UI 事件 `tool_execution_start`。
3. 进入 **准备阶段（prepareToolCall）**：校验参数、走 `beforeToolCall` 拦截守卫。
4. 如果守卫拦截（状态为 `"immediate"`），立刻报错并装填结果。
5. Если 如果通过了校验，才会走到 **执行阶段（executePreparedToolCall）**，这里真正执行耗时的操作。
6. 最后进行 **收尾阶段（finalizeExecutedToolCall）**，触发生命周期钩子 `afterToolCall` 并组装 `ToolResultMessage`，推入结果数组。

这种模式的**最大缺点**是：如果大模型下发了 5 个网页内容读取工具调用，Agent 必须等待第一个网页从网络读取完毕返回，才会去发起第二个请求。对于现代计算机来说，这属于暴殄天物。

---

## 3. 并行执行（Parallel）：极限性能与“保序并发”黑魔法

默认情况下，`pi-agent-core` 采用并行策略。这里最让人拍案叫绝的一点是：**我们在并发请求后，如何保证组装出的回执顺序（Results），依然严格遵循大模型发号施令的原有先后次序？** 

如果次序不匹配，大模型拿到回执后上下文推理极可能发生错乱（它问了 1 是啥 2 是啥，你先回 2 是啥，大模型可能就对不上号了）。

### 源码解析：分层分步骤执行

我们来看看 `executeToolCallsParallel` 里的经典“保序并发”是怎样在前端 Promise 的加持下实现的：

#### 第一步：所有工具强行阻塞排队过“安检”
在真正发起网络/IO 并发之前，引擎先用了一个同步的 `for...of` 循环，**顺序地进行所有的准备工作 `prepareToolCall`。**

```typescript
// 1. 同步进行准备和安检
const runnableCalls: PreparedToolCall[] = [];
for (const toolCall of toolCalls) {
  const preparation = await prepareToolCall(...);
  
  // 被拦截的直接提前判处死刑
  if (preparation.kind === "immediate") {
      results.push(...); 
  } else {
      runnableCalls.push(preparation); // 允许放行的先入列
  }
}
```
**为什么“安检”不能并行？** 因为 `beforeToolCall` 这种拦截验证钩子可能有副作用或者可能互相冲突！顺序安检能避免幽灵 Bug。

#### 第二步：真正的并发引擎启动 (The Concurrent Engine)
此时我们有了一个准备好的待宰羔羊队列 `runnableCalls`，接下来引擎并**没有**普通地去写一把 `Promise.all`，而是做了一个极其精妙的映射（Mapping）：

```typescript
// 2. 将数组映射，把正在跑的 Promise 直接挂在对象属性上返回来
const runningCalls = runnableCalls.map((prepared) => ({
  prepared, // 任务描述
  execution: executePreparedToolCall(prepared, signal, emit), // 📌 注意：这里没有写 await！函数在此刻瞬间同步全部发起起跑！
}));
```
这就如同前端发起了 10 个 `fetch`，却没有等它们，直接把这 10 个未决的 `Promise` 拿在了手上！

#### 第三步：保序组装 (Ordered Resolution)
并发都已经发出去了，有些小任务已经算完，大任务还在跑。此时我们再掏出刚才带有原始严格顺序的 `runningCalls` 数组：

```typescript
// 3. 按照原有的顺序，依次去 await 这些挂在手里的 promise
for (const running of runningCalls) {
  const executed = await running.execution; // 必定按照数组原序收盘！
  
  results.push(
    await finalizeExecutedToolCall(..., executed, ...)
  );
}
```

> 💡 **前端启发**：
> 这段代码完美示范了当我们需要**同时兼具并发的极致速度和收敛时的绝对顺序**时该怎么写！你并不需要去搞复杂的索引映射表再拼回去，你只需要：**先把所有的 Promise 一次性触发并拿到它们的引用，接着按你需要的顺序对这些引用依次调用 `await`。** 就算前面的慢、后面的快，由于按需 await，塞进 `results.push` 里的元素一定跟原始顺序分毫不差。

---

## 4. 总结：你在业务里该怎么选？

- 如果你给大模型写的全是一些查询数据的 API，且大模型习惯一次性提问你很多问题（比如“查询今天天气”、“获取热门新闻”、“拿到用户个资”），请毫不犹豫遵守框架的**默认 Parallel 配置**。
- 如果你的 AI 系统在操盘你的 Git 仓库、操作 CI/CD 机器或者存在严重依赖上下文步骤状态的任务，强烈建议你改为 `toolExecution: 'sequential'`，宁可稍微牺牲速度换取绝对的安全。
