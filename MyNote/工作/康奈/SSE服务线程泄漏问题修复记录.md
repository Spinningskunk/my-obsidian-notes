# SSE 服务线程泄漏问题修复记录

## 业务逻辑梳理

### 一、修复前的业务流程

#### 1. 客户端建立连接 `connect()`

```
前端发起 SSE 请求
  → connect(userType, userId, organId)
  → 生成 clientId（格式：mt-sse-{userType}-{organId}-{userId}）
  → 若 clientId 已存在：调用 close(clientId) 关闭旧连接（仅移除 map，未关闭 emitter）
  → 创建 SseEmitter(0L)（永不超时）
  → 注册三个回调：
      onCompletion → EMITTER_MAP.remove(clientId)
      onTimeout    → 什么都不做（注释说"连接保持"）
      onError      → EMITTER_MAP.remove(clientId) 被注释掉，什么都不做
  → EMITTER_MAP.put(clientId, emitter)
  → sendMessage() 发送"连接成功"消息
  → startHealthCheck(clientId) 启动心跳
  → 返回 emitter 给前端，HTTP 长连接建立
```

#### 2. 心跳维持 `startHealthCheck()`

```
startHealthCheck(clientId)
  → executor.execute(...)  向 CachedThreadPool 提交一个新线程
  → 线程内：
      while(true)
        sleep(5秒)
        if EMITTER_MAP 里没有 clientId → break 退出
        sendMessage(clientId, 心跳消息)  ← @Async 异步发送
```

#### 3. 发送消息 `sendMessage()`

```
sendMessage(clientId, sseMessage)  ← @Async 由 sseTaskExecutor 线程池执行
  → 从 EMITTER_MAP 取出 emitter
  → emitter 为 null → 打日志，EMITTER_MAP.remove（多余操作），return
  → emitter.send(message)
      成功 → 打日志
      IOException → 打日志，什么都不做（注释说"不移除连接"）
```

#### 4. 客户端断开连接

```
正常断开（前端主动调用）：
  → close(clientId)
  → EMITTER_MAP.remove(clientId)  ← 只移除 map，未调用 emitter.complete()
  → 心跳线程下次醒来发现 map 里没有 clientId → break 退出 ✓

异常断开（断网/关浏览器/网络抖动）：
  → onError 触发
  → EMITTER_MAP.remove 被注释掉，map 未清理
  → 心跳线程：map 里还有我 → 继续运行
  → sendMessage 拿到已死的 emitter → IOException
  → catch 里不移除连接
  → 心跳线程永不退出 ✗  ← 线程泄漏根源
```

#### 5. 问题总结图

```
异常断开
  ↓
onError（map 未清理）
  ↓
心跳线程永不退出（每5秒 sleep/唤醒）
  ↓
sendMessage 每次抛 IOException（不清理）
  ↓
线程永久占用，随连接次数累积
  ↓
CachedThreadPool 无上限扩张
  ↓
JVM 线程/内存耗尽，崩溃
```

---

### 二、修复后的业务流程

#### 1. 客户端建立连接 `connect()`

```
前端发起 SSE 请求
  → connect(userType, userId, organId)
  → 生成 clientId
  → 若 clientId 已存在：调用 close(clientId) → cleanup()（取消心跳 + complete + 移除 map）
  → 创建 SseEmitter(30分钟)（超时兜底）
  → 注册三个回调：
      onCompletion → 取消心跳 + EMITTER_MAP.remove（不调 emitter.complete()，已完成）
      onTimeout    → cleanup(clientId)
      onError      → cleanup(clientId)
  → EMITTER_MAP.put(clientId, emitter)
  → sendMessage() 发送"连接成功"消息
  → startHeartbeat(clientId) 启动心跳
  → 返回 emitter 给前端
```

#### 2. 心跳维持 `startHeartbeat()`

```
startHeartbeat(clientId)
  → scheduler.scheduleAtFixedRate(...)  向 ScheduledThreadPoolExecutor 提交定时任务
  → 返回 ScheduledFuture，存入 HEARTBEAT_MAP
  → 每5秒执行一次：
      if EMITTER_MAP 里没有 clientId → 取消自身 future，return
      sendMessage(clientId, 心跳消息)  ← @Async 异步发送
```

#### 3. 发送消息 `sendMessage()`

```
sendMessage(clientId, sseMessage)  ← @Async 由 sseTaskExecutor 线程池执行
  → 从 EMITTER_MAP 取出 emitter
  → emitter 为 null → 打日志，return
  → emitter.send(message)
      成功 → 打日志
      IOException → 打日志，调用 cleanup(clientId)  ← 立即清理
```

#### 4. 统一清理 `cleanup()`

```
cleanup(clientId)
  → HEARTBEAT_MAP.remove(clientId) 取出 ScheduledFuture
  → future.cancel(false)  精确取消心跳任务，不依赖业务逻辑
  → EMITTER_MAP.remove(clientId) 取出 emitter
  → emitter.complete()  正确关闭底层 HTTP 连接
```

#### 5. 客户端断开连接

```
正常断开（前端主动调用）：
  → close(clientId) → cleanup()
  → 心跳任务被 cancel()，立即停止 ✓
  → emitter.complete()，HTTP 连接释放 ✓

异常断开（断网/关浏览器）：
  → onError 触发 → cleanup()
  → 心跳任务被 cancel()，立即停止 ✓
  → emitter.complete()，HTTP 连接释放 ✓

30分钟无活动：
  → onTimeout 触发 → cleanup()
  → 兜底清理，防止僵尸连接 ✓

服务重启/关闭：
  → DisposableBean.destroy()
  → 遍历所有连接调用 cleanup()
  → scheduler.shutdown()
  → 优雅退出 ✓
```

#### 6. 修复后流程图

```
任意断开方式（正常/异常/超时）
  ↓
统一触发 cleanup(clientId)
  ↓
future.cancel()  +  emitter.complete()  +  map.remove()
  ↓
心跳任务精确取消，HTTP 连接释放，map 清理
  ↓
线程数始终 = 2（ScheduledThreadPoolExecutor 固定）
  ↓
JVM 资源稳定
```

---

## 问题发现

**时间：** 2026-03-17

**现象：** JVM 被打满、崩溃

**诊断数据：**
- `EMITTER_MAP.size() = 7`（仅7个活跃连接）
- `pool-6-thread-xxx` 线程数：185+（远超连接数）
- 线程状态：`TIMED_WAITING`（卡在 `sleep(5)` 上）

**结论：** 每次用户连接/断开都会泄漏一个心跳线程

---

## 根本原因分析

### 1. 线程退出条件失效

```java
// 原代码
executor.execute(() -> {
    while (true) {
        TimeUnit.SECONDS.sleep(5);
        if (!EMITTER_MAP.containsKey(clientId)) {
            break;  // 唯一退出条件
        }
        sendMessage(clientId, sseMessage);
    }
});
```

线程退出依赖 `EMITTER_MAP` 中是否存在 `clientId`，但：

### 2. onError 回调未清理 map

```java
emitter.onError((e) -> {
    EMITTER_MAP.remove(clientId);  // 被注释掉
    log.error(...);
});
```

客户端断网/关闭浏览器 → `onError` 触发 → map 未清理 → 心跳线程永不退出

### 3. 连锁反应

```
客户端断开
  → onError 触发，map 未清理
  → 心跳线程：map 里还有我，继续运行
  → sendMessage 拿到已死的 emitter
  → emitter.send() 抛 IOException
  → catch 里"不移除连接"（代码注释明确写了）
  → 下次心跳继续，无限循环
  → 线程永不退出
```

### 4. 其他问题

- `close()` 只移除 map，未调用 `emitter.complete()`，底层 HTTP 连接未释放
- `SseEmitter(0L)` 永不超时，僵尸连接永久占用资源
- `CachedThreadPool` 无上限，大量连接 = 大量线程

---

## 修复方案

### 核心改动

| 问题 | 修复 |
|------|------|
| 线程泄漏 | 用 `ScheduledFuture.cancel()` 强制取消，不依赖业务逻辑 |
| 僵尸连接 | `cleanup()` 统一清理：取消心跳 + `emitter.complete()` + 移除 map |
| 发送失败不清理 | catch 里调用 `cleanup()` |
| 无限超时 | 设置 30 分钟超时，`onTimeout` 调用 `cleanup()` |
| 线程池无上限 | 用 `ScheduledThreadPoolExecutor(2)` 替代 `CachedThreadPool`，固定2线程可支撑 ~2000 在线用户 |

### 代码变更

#### 1. 替换线程池

```java
// 旧
private final ExecutorService executor = Executors.newCachedThreadPool();

// 新
private final ScheduledExecutorService scheduler = new ScheduledThreadPoolExecutor(2,
        r -> {
            Thread t = new Thread(r, "sse-heartbeat");
            t.setDaemon(true);
            return t;
        });
private static final Map<String, ScheduledFuture<?>> HEARTBEAT_MAP = new ConcurrentHashMap<>();
```

#### 2. 修改心跳逻辑

```java
// 旧：无限循环 + sleep
private void startHealthCheck(String clientId) {
    executor.execute(() -> {
        while (true) {
            TimeUnit.SECONDS.sleep(5);
            if (!EMITTER_MAP.containsKey(clientId)) break;
            sendMessage(clientId, sseMessage);
        }
    });
}

// 新：ScheduledFuture 管理
private void startHeartbeat(String clientId) {
    ScheduledFuture<?> future = scheduler.scheduleAtFixedRate(() -> {
        if (!EMITTER_MAP.containsKey(clientId)) {
            ScheduledFuture<?> self = HEARTBEAT_MAP.remove(clientId);
            if (self != null) self.cancel(false);
            return;
        }
        SseMessage heartbeat = new SseMessage()...;
        sendMessage(clientId, heartbeat);
    }, HEARTBEAT_INTERVAL, HEARTBEAT_INTERVAL, TimeUnit.SECONDS);
    HEARTBEAT_MAP.put(clientId, future);
}
```

#### 3. 统一清理方法

```java
private void cleanup(String clientId) {
    // 取消心跳任务
    ScheduledFuture<?> future = HEARTBEAT_MAP.remove(clientId);
    if (future != null) {
        future.cancel(false);
    }
    // 关闭 emitter
    SseEmitter emitter = EMITTER_MAP.remove(clientId);
    if (emitter != null) {
        try {
            emitter.complete();
        } catch (Exception ignored) {
        }
    }
}
```

#### 4. 修复 close() 和回调

```java
@Override
public void close(String clientId) {
    cleanup(clientId);
    log.info(...);
}

// connect() 中
emitter.onCompletion(() -> cleanup(clientId));
emitter.onTimeout(() -> cleanup(clientId));
emitter.onError(e -> cleanup(clientId));

// sendMessage() 中
} catch (IOException e) {
    sendFlag = false;
    log.error(...);
    cleanup(clientId);  // 发送失败立即清理
}
```

#### 5. 添加服务关闭钩子

```java
@PreDestroy
public void destroy() {
    log.info("【{}】服务关闭，清理所有连接，当前连接数:【{}】", moduleName, EMITTER_MAP.size());
    EMITTER_MAP.keySet().forEach(this::cleanup);
    scheduler.shutdown();
}
```

---

## 修复效果

### 修复前
- 7 个连接 → 185+ 个线程
- 线程永不退出，持续泄漏
- JVM 资源耗尽，服务崩溃

### 修复后
- N 个连接 → 2 个心跳线程（固定）
- 连接断开时 `ScheduledFuture.cancel()` 精确取消任务
- `emitter.complete()` 正确释放底层 HTTP 连接
- 30 分钟超时兜底，防止僵尸连接

---

## 验证方法

### 1. Arthas 监控

```bash
# 查看线程数
thread

# 查看 EMITTER_MAP 大小
ognl '@cn.konne.service.impl.SseServiceImpl@EMITTER_MAP.size()'

# 查看 HEARTBEAT_MAP 大小
ognl '@cn.konne.service.impl.SseServiceImpl@HEARTBEAT_MAP.size()'

# 查看调度器状态
ognl '@cn.konne.service.impl.SseServiceImpl@scheduler'
```

### 2. 压测验证

- 模拟 100 个客户端连接/断开
- 观察线程数是否稳定在 2 个
- 观察 `EMITTER_MAP` 和 `HEARTBEAT_MAP` 是否正确清理

---

## 关键点总结

1. **不要用 `while(true) + sleep` 实现定时任务**，用 `ScheduledExecutorService`
2. **不要让业务逻辑控制线程生命周期**，用 `ScheduledFuture.cancel()` 外部控制
3. **SSE 连接必须调用 `emitter.complete()`**，否则底层 HTTP 连接不释放
4. **所有异常路径都要清理资源**：`onError`、`onTimeout`、`sendMessage` 失败
5. **设置合理超时**，不要用 `SseEmitter(0L)` 永不超时

---

## 调度器线程数配置说明

### 为什么 2 个线程够用？

`ScheduledThreadPoolExecutor(2)` 不是"同时处理 2 个连接"，而是"2 个工作线程执行所有连接的心跳任务"。

**工作原理：**
```
N 个连接 → N 个心跳任务（ScheduledFuture）
              ↓
      调度器的任务队列
              ↓
    2 个工作线程轮流执行
```

**为什么轻量？**
- 心跳任务只是"提交异步发送"，不等待网络 IO
- `sendMessage` 标注了 `@Async`，真正的发送由 `sseTaskExecutor` 线程池处理
- 单个心跳任务耗时 < 1ms

**吞吐量计算：**
```
假设 1000 个在线用户，每 5 秒一次心跳
1000 个任务 ÷ 2 个线程 = 每个线程 500 个任务
500 × 1ms = 500ms

结论：2 个线程在 0.5 秒内处理完 1000 个连接的心跳，还有 4.5 秒空闲
```

### 如何调整？

根据实际在线用户数调整：

| 在线用户数 | 建议线程数 | 说明 |
|-----------|-----------|------|
| < 500 | 1 | 单线程足够 |
| 500 ~ 2000 | 2 | 当前配置 |
| 2000 ~ 5000 | 4 | 提高并发能力 |
| > 5000 | - | 建议评估 WebSocket 或消息队列方案 |

**修改方式：**
```java
// 修改构造函数的第一个参数
private final ScheduledExecutorService scheduler = new ScheduledThreadPoolExecutor(1, ...);
```

---

**修复人：** HeKun
**修复日期：** 2026/03/17

---

## 二次修复：onCompletion 触发连接立即断开（2026/03/19）

### 现象

修复后前端 SSE 连接在 2ms 内断开，回退旧代码则正常。

### 根本原因

`onCompletion` 回调中调用了 `cleanup(clientId)`，而 `cleanup()` 内部会调用 `emitter.complete()`。

Spring MVC 在 `return emitter` 后会调用一次 `complete()` 来完成 HTTP 响应写入，这触发了 `onCompletion` → `cleanup()` → `EMITTER_MAP.remove(clientId)`，导致连接刚建立就被从 map 里清掉，心跳任务也被取消，前端收到连接关闭信号后立即重连。

```
return emitter
  → Spring MVC 调用 emitter.complete()（完成 HTTP 写入）
  → 触发 onCompletion 回调
  → cleanup(clientId)
  → EMITTER_MAP.remove(clientId)  ← 连接刚建立就被清掉
  → future.cancel()               ← 心跳任务被取消
  → emitter.complete()            ← 重复调用（no-op，但已造成破坏）
```

### 修复

`onCompletion` 不能调 `cleanup()`，因为 emitter 已经完成，不能再调 `complete()`。只需手动取消心跳并移除 map：

```java
emitter.onCompletion(() -> {
    log.info("【{}】连接完成！clientId:【{}】", moduleName, clientId);
    // emitter 已完成，不能再调 complete()，只清理 map 和心跳任务
    ScheduledFuture<?> future = HEARTBEAT_MAP.remove(clientId);
    if (future != null) future.cancel(false);
    EMITTER_MAP.remove(clientId);
});
```

`onTimeout` 和 `onError` 仍然调 `cleanup()`（需要主动调 `emitter.complete()` 释放连接）。
