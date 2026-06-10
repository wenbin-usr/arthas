# Arthas 目标类目标方法增强机制详解

## 一、整体架构概览

Arthas 对目标类目标方法的增强分为 **6 个核心环节**：

```
用户命令 (watch/trace/monitor/stack/tt)
    │
    ▼
EnhancerCommand.enhance()          ← 命令入口，组装 Enhancer + AdviceListener
    │
    ▼
Enhancer.enhance(inst)             ← 类匹配 + 注册 Transformer + 触发 retransform
    │
    ▼
Enhancer.transform()               ← JVM 回调，ASM + ByteKit 字节码注入
    │
    ▼
SpyAPI → SpyImpl                   ← BootstrapClassLoader 桥接层
    │
    ▼
AdviceListenerManager → AdviceListener  ← Listener 查找与回调
    │
    ▼
具体命令逻辑 (WatchAdviceListener / TraceAdviceListener / ...)
```

### 核心类职责

| 类 | 职责 | 文件 |
|---|---|---|
| `EnhancerCommand` | 所有增强命令的抽象基类，组装 Enhancer 和 AdviceListener | [EnhancerCommand.java](file:///d:/workspace/java_projects/source_projects/arthas/core/src/main/java/com/taobao/arthas/core/command/monitor200/EnhancerCommand.java) |
| `Enhancer` | 实现 `ClassFileTransformer`，负责字节码注入 | [Enhancer.java](file:///d:/workspace/java_projects/source_projects/arthas/core/src/main/java/com/taobao/arthas/core/advisor/Enhancer.java) |
| `SpyInterceptors` | 定义 ByteKit 注解驱动的拦截器，指定注入位置 | [SpyInterceptors.java](file:///d:/workspace/java_projects/source_projects/arthas/core/src/main/java/com/taobao/arthas/core/advisor/SpyInterceptors.java) |
| `SpyAPI` | BootstrapClassLoader 加载的桥接类，所有 ClassLoader 可见 | [SpyAPI.java](file:///d:/workspace/java_projects/source_projects/arthas/spy/src/main/java/java/arthas/SpyAPI.java) |
| `SpyImpl` | SpyAPI 的实际实现，负责查找并回调 AdviceListener | [SpyImpl.java](file:///d:/workspace/java_projects/source_projects/arthas/core/src/main/java/com/taobao/arthas/core/advisor/SpyImpl.java) |
| `AdviceListenerManager` | 管理 (ClassLoader, 类名, 方法) → List\<AdviceListener\> 的映射 | [AdviceListenerManager.java](file:///d:/workspace/java_projects/source_projects/arthas/core/src/main/java/com/taobao/arthas/core/advisor/AdviceListenerManager.java) |
| `AdviceWeaver` | 管理 AdviceListener 的注册/注销/暂停/恢复 | [AdviceWeaver.java](file:///d:/workspace/java_projects/source_projects/arthas/core/src/main/java/com/taobao/arthas/core/advisor/AdviceWeaver.java) |
| `TransformerManager` | 统一管理所有 ClassFileTransformer，按优先级链式调用 | [TransformerManager.java](file:///d:/workspace/java_projects/source_projects/arthas/core/src/main/java/com/taobao/arthas/core/advisor/TransformerManager.java) |

---

## 二、以 `watch` 命令为例的完整执行流程

以用户输入 `watch demo.MathGame primeFactors '{params, returnObj, throwExp}' -x 2` 为例，跟踪从命令输入到结果输出的完整链路。

### 阶段 1：命令解析与实例化

1. Arthas Shell 接收到命令字符串，CLI 框架解析参数
2. 创建 `WatchCommand` 实例，设置字段：
   - `classPattern = "demo.MathGame"`
   - `methodPattern = "primeFactors"`
   - `express = "{params, returnObj, throwExp}"`
   - `expand = 2`
3. 调用 `WatchCommand.process(process)`（继承自 `EnhancerCommand`）

```java
// EnhancerCommand.process() — 入口
@Override
public void process(final CommandProcess process) {
    process.interruptHandler(new CommandInterruptHandler(process));  // Ctrl+C 处理
    process.stdinHandler(new QExitHandler(process));                 // q 退出处理
    enhance(process);  // 开始增强
}
```

### 阶段 2：组装 Enhancer 与 AdviceListener

`EnhancerCommand.enhance()` 方法：

```java
protected void enhance(CommandProcess process) {
    Session session = process.session();
    if (!session.tryLock()) {
        // 防止并发增强，同一时间只能有一个增强操作
        process.end(-1, "someone else is enhancing classes, pls. wait.");
        return;
    }

    Instrumentation inst = session.getInstrumentation();

    // 1. 获取 AdviceListener — WatchCommand 返回 WatchAdviceListener
    AdviceListener listener = getAdviceListenerWithId(process);
    // WatchCommand.getAdviceListener():
    //   return new WatchAdviceListener(this, process, GlobalOptions.verbose || this.verbose);

    // 2. 创建 Enhancer（实现 ClassFileTransformer）
    Enhancer enhancer = new Enhancer(
        listener,                                    // WatchAdviceListener
        listener instanceof InvokeTraceable,         // false（watch 不需要 trace）
        skipJDKTrace,                                // false
        getClassNameMatcher(),                       // WildcardMatcher("demo.MathGame")
        getClassNameExcludeMatcher(),                // null
        getMethodNameMatcher(),                      // WildcardMatcher("primeFactors")
        this.lazy,                                   // false
        this.hashCode                                // null
    );

    // 3. 注册 listener 和 enhancer 到 process（用于命令结束时清理）
    process.register(listener, enhancer);

    // 4. 执行增强
    effect = enhancer.enhance(inst, this.maxNumOfMatchedClass);
}
```

### 阶段 3：类匹配与触发 retransform

`Enhancer.enhance(inst)` 方法：

```java
public synchronized EnhancerAffect enhance(final Instrumentation inst, int maxNumOfMatchedClass) {
    // 步骤 1：搜索匹配的类
    // 利用 Instrumentation.getAllLoadedClasses() 找到所有已加载的类
    // 然后通过 classNameMatcher 过滤
    this.matchingClasses = GlobalOptions.isDisableSubClass
            ? SearchUtils.searchClass(inst, classNameMatcher)          // 精确匹配
            : SearchUtils.searchSubClass(inst, SearchUtils.searchClass(inst, classNameMatcher)); // 含子类

    // 步骤 2：过滤无法增强的类
    // - Arthas 自身的类（SpyAPI、Enhancer 等）
    // - BootstrapClassLoader 加载的类（unsafe 模式除外）
    // - 已经是代理类/合成类的
    List<Pair<Class<?>, String>> filtedList = filter(matchingClasses);

    // 步骤 3：数量限制检查
    if (matchingClasses.size() > maxNumOfMatchedClass) {
        affect.setOverLimitMsg("...");
        return affect;
    }

    // 步骤 4：注册 ClassFileTransformer
    // TransformerManager 内部维护了 watchTransformers / traceTransformers 两个列表
    // watch 命令的 transformer 加入 watchTransformers
    ArthasBootstrap.getInstance().getTransformerManager().addTransformer(this, isTracing);

    // 步骤 5：触发 JVM retransform
    // 这会触发 JVM 回调 Enhancer.transform() 方法
    if (GlobalOptions.isBatchReTransform) {
        inst.retransformClasses(classArray);  // 批量
    } else {
        for (Class<?> clazz : matchingClasses) {
            inst.retransformClasses(clazz);   // 逐个
        }
    }

    return affect;
}
```

### 阶段 4：字节码注入（JVM 回调 transform）

JVM 调用 `Enhancer.transform()`，这是最核心的环节：

```java
@Override
public byte[] transform(ClassLoader inClassLoader, String className,
        Class<?> classBeingRedefined, ProtectionDomain protectionDomain,
        byte[] classfileBuffer) {

    // === 前置检查 ===

    // 1. 检查 classloader 能否加载 SpyAPI（BootstrapClassLoader 加载的类）
    //    如果不能，放弃增强（因为注入的 SpyAPI 调用将无法解析）
    if (inClassLoader != null) {
        inClassLoader.loadClass(SpyAPI.class.getName());
    }

    // 2. 二次过滤：transform 过程中可能诞生新类，需要再次匹配
    if (matchingClasses != null && !matchingClasses.contains(classBeingRedefined)) {
        return null;  // 不增强
    }

    // === 字节码解析 ===

    // 3. 用 ASM 解析原始字节码为 ClassNode
    ClassNode classNode = new ClassNode(Opcodes.ASM9);
    ClassReader classReader = AsmUtils.toClassNode(classfileBuffer, classNode);
    classNode = AsmUtils.removeJSRInstructions(classNode);  // 移除 JSR 指令

    // === 解析 Interceptor（ByteKit 注解驱动） ===

    // 4. 从 SpyInterceptors 中解析带 ByteKit 注解的拦截器
    DefaultInterceptorClassParser parser = new DefaultInterceptorClassParser();

    // 方法级拦截器（ENTER / EXIT / EXCEPTION_EXIT）
    List<InterceptorProcessor> interceptorProcessors = new ArrayList<>();
    interceptorProcessors.addAll(parser.parse(SpyInterceptor1.class));  // @AtEnter
    interceptorProcessors.addAll(parser.parse(SpyInterceptor2.class));  // @AtExit
    interceptorProcessors.addAll(parser.parse(SpyInterceptor3.class));  // @AtExceptionExit

    // === 遍历匹配的方法 ===

    // 5. 找出所有匹配的方法
    List<MethodNode> matchedMethods = new ArrayList<>();
    for (MethodNode methodNode : classNode.methods) {
        if (!isIgnore(methodNode, methodNameMatcher)) {
            // 忽略抽象方法、<clinit>、不匹配的方法
            matchedMethods.add(methodNode);
        }
    }

    // === 防重复增强检查 ===

    // 6. 创建 GroupLocationFilter，检查方法中是否已包含 SpyAPI 调用
    //    如果已有 atEnter/atExit/atExceptionExit 调用，则跳过
    GroupLocationFilter groupLocationFilter = new GroupLocationFilter();
    groupLocationFilter.addFilter(new InvokeContainLocationFilter(
        "java/arthas/SpyAPI", "atEnter", LocationType.ENTER));
    groupLocationFilter.addFilter(new InvokeContainLocationFilter(
        "java/arthas/SpyAPI", "atExit", LocationType.EXIT));
    groupLocationFilter.addFilter(new InvokeContainLocationFilter(
        "java/arthas/SpyAPI", "atExceptionExit", LocationType.EXCEPTION_EXIT));

    // === 对每个方法注入字节码 ===

    for (MethodNode methodNode : matchedMethods) {
        if (AsmUtils.isNative(methodNode)) {
            continue;  // 跳过 native 方法
        }

        // 7. 创建 MethodProcessor，在指定位置注入 SpyAPI 调用
        MethodProcessor methodProcessor = new MethodProcessor(
            classNode, methodNode, groupLocationFilter);

        for (InterceptorProcessor interceptor : interceptorProcessors) {
            // ByteKit 根据注解找到 ENTER/EXIT/EXCEPTION_EXIT 位置
            // 将 SpyAPI.atEnter/atExit/atExceptionExit 的调用指令插入到字节码中
            List<Location> locations = interceptor.process(methodProcessor);
        }

        // 8. 注册 Listener 映射
        //    建立 (ClassLoader, className, methodName, methodDesc) → AdviceListener 的映射
        AdviceListenerManager.registerAdviceListener(
            inClassLoader, className, methodNode.name, methodNode.desc, listener);

        affect.addMethodAndCount(inClassLoader, className, methodNode.name, methodNode.desc);
    }

    // === 输出增强后的字节码 ===

    // 9. 将 ClassNode 转回 byte[]
    byte[] enhanceClassByteArray = AsmUtils.toBytes(classNode, inClassLoader, classReader);

    // 10. 可选：dump 增强后的 .class 文件到磁盘
    dumpClassIfNecessary(className, enhanceClassByteArray, affect);

    return enhanceClassByteArray;  // JVM 使用这个新的字节码替换原类
}
```

**注入效果示意**（以 `primeFactors` 方法为例）：

```java
// 原始方法
public List<Integer> primeFactors(int number) {
    // ... 原始逻辑 ...
    return result;
}

// 增强后的等效代码
public List<Integer> primeFactors(int number) {
    // === 注入 atEnter ===
    SpyAPI.atEnter(MathGame.class, "primeFactors|(I)Ljava/util/List;", this, new Object[]{number});

    try {
        // ... 原始逻辑 ...
        List<Integer> result = ...;

        // === 注入 atExit ===
        SpyAPI.atExit(MathGame.class, "primeFactors|(I)Ljava/util/List;", this,
                      new Object[]{number}, result);
        return result;
    } catch (Throwable t) {
        // === 注入 atExceptionExit ===
        SpyAPI.atExceptionExit(MathGame.class, "primeFactors|(I)Ljava/util/List;", this,
                               new Object[]{number}, t);
        throw t;
    }
}
```

### 阶段 5：Spy 桥接（运行时回调）

当目标方法被调用时，注入的 `SpyAPI` 静态方法被执行：

```java
// SpyAPI 内部维护一个 AbstractSpy 单例
// 默认是 NopSpy（空操作），Enhancer 初始化时替换为 SpyImpl
public class SpyAPI {
    private static volatile AbstractSpy spyInstance = NOPSPY;

    public static void atEnter(Class<?> clazz, String methodInfo, Object target, Object[] args) {
        spyInstance.atEnter(clazz, methodInfo, target, args);  // 委托给 SpyImpl
    }
}
```

`SpyImpl` 根据方法信息查找对应的 `AdviceListener` 并回调：

```java
// SpyImpl.atEnter()
public void atEnter(Class<?> clazz, String methodInfo, Object target, Object[] args) {
    ClassLoader classLoader = clazz.getClassLoader();

    // 解析 methodInfo: "primeFactors|(I)Ljava/util/List;"
    String[] info = StringUtils.splitMethodInfo(methodInfo);
    String methodName = info[0];   // "primeFactors"
    String methodDesc = info[1];   // "(I)Ljava/util/List;"

    // 从 AdviceListenerManager 查找注册的 Listener
    List<AdviceListener> listeners = AdviceListenerManager.queryAdviceListeners(
        classLoader, clazz.getName(), methodName, methodDesc);

    if (listeners != null) {
        for (AdviceListener listener : listeners) {
            listener.before(clazz, methodName, methodDesc, target, args);
        }
    }
}
```

### 阶段 6：WatchAdviceListener 执行具体逻辑

```java
// WatchAdviceListener.before() — 方法进入时
@Override
public void before(ClassLoader loader, Class<?> clazz, ArthasMethod method,
                   Object target, Object[] args) throws Throwable {
    threadLocalWatch.start();  // 开始计时
    if (command.isBefore()) {
        watching(Advice.newForBefore(loader, clazz, method, target, args));
    }
}

// WatchAdviceListener.afterReturning() — 方法正常返回时
@Override
public void afterReturning(ClassLoader loader, Class<?> clazz, ArthasMethod method,
                           Object target, Object[] args, Object returnObject) throws Throwable {
    Advice advice = Advice.newForAfterReturning(loader, clazz, method, target, args, returnObject);
    if (command.isSuccess()) {
        watching(advice);
    }
    finishing(advice);
}

// watching() — 核心输出逻辑
private void watching(Advice advice) {
    double cost = threadLocalWatch.costInMillis();  // 计算耗时

    // 1. 评估条件表达式（如 '#cost>100'）
    boolean conditionResult = isConditionMet(command.getConditionExpress(), advice, cost);
    if (!conditionResult) return;

    // 2. 执行 OGNL 表达式获取结果（如 '{params, returnObj, throwExp}'）
    Object value = getExpressionResult(command.getExpress(), advice, cost);

    // 3. 构建输出模型
    WatchModel model = new WatchModel();
    model.setTs(LocalDateTime.now());
    model.setCost(cost);
    model.setValue(new ObjectVO(value, command.getExpand()));  // expand=2 控制展开深度
    model.setClassName(advice.getClazz().getName());
    model.setMethodName(advice.getMethod().getName());

    // 4. 输出到终端
    process.appendResult(model);
    process.times().incrementAndGet();
}
```

---

## 三、`trace` 命令的特殊处理

`trace` 命令与 `watch` 的关键区别在于：**trace 需要在目标方法的每个子方法调用前后也注入字节码**。

### 3.1 额外的 Interceptor

在 `Enhancer.transform()` 中，当 `isTracing = true` 时，额外注册 `SpyTraceInterceptor`：

```java
if (this.isTracing) {
    interceptorProcessors.addAll(parser.parse(SpyTraceInterceptor1.class));  // @AtInvoke, whenComplete=false
    interceptorProcessors.addAll(parser.parse(SpyTraceInterceptor2.class));  // @AtInvoke, whenComplete=true
    interceptorProcessors.addAll(parser.parse(SpyTraceInterceptor3.class));  // @AtInvokeException
}
```

这些 Interceptor 使用 `@AtInvoke` 注解，会在目标方法的**每一个方法调用指令（INVOKEVIRTUAL/INVOKESTATIC 等）前后**注入 `SpyAPI.atBeforeInvoke/atAfterInvoke/atInvokeException`。

### 3.2 注入效果示意

```java
// 原始方法
public void run() {
    int num = random.nextInt(100);     // 方法调用1
    List<Integer> factors = primeFactors(num);  // 方法调用2
    System.out.println(factors);       // 方法调用3
}

// trace 增强后的等效代码
public void run() {
    SpyAPI.atEnter(...);  // ENTER 拦截（与 watch 相同）

    SpyAPI.atBeforeInvoke(clazz, "java/util/Random|nextInt|(I)I", this);
    int num = random.nextInt(100);
    SpyAPI.atAfterInvoke(clazz, "java/util/Random|nextInt|(I)I", this);

    SpyAPI.atBeforeInvoke(clazz, "demo/MathGame|primeFactors|(I)Ljava/util/List;", this);
    List<Integer> factors = primeFactors(num);
    SpyAPI.atAfterInvoke(clazz, "demo/MathGame|primeFactors|(I)Ljava/util/List;", this);

    SpyAPI.atBeforeInvoke(clazz, "java/io/PrintStream|println|(Ljava/lang/Object;)V", this);
    System.out.println(factors);
    SpyAPI.atAfterInvoke(clazz, "java/io/PrintStream|println|(Ljava/lang/Object;)V", this);

    SpyAPI.atExit(...);  // EXIT 拦截
}
```

### 3.3 TraceAdviceListener 的调用树

`TraceAdviceListener` 实现了 `InvokeTraceable` 接口，在 `invokeBeforeTracing/invokeAfterTracing` 中维护一棵调用树：

```java
// TraceAdviceListener
@Override
public void invokeBeforeTracing(ClassLoader classLoader, String tracingClassName,
        String tracingMethodName, String tracingMethodDesc, int tracingLineNumber) {
    threadLocalTraceEntity(classLoader).tree.begin(
        tracingClassName, tracingMethodName, tracingLineNumber, true);
}

@Override
public void invokeAfterTracing(...) {
    threadLocalTraceEntity(classLoader).tree.end();
}
```

最终输出效果：

```
`---ts=2024-01-01 12:00:00;thread_name=main;id=1;is_daemon=false;priority=5
    `---[0.50ms] demo.MathGame:run()
        +---[0.05ms] java.util.Random:nextInt()       # 子调用1
        +---[0.20ms] demo.MathGame:primeFactors()     # 子调用2
        `---[0.03ms] java.io.PrintStream:println()    # 子调用3
```

---

## 四、注入位置详解

ByteKit 通过注解驱动的方式，定义了以下注入位置：

| 注解 | 注入位置 | 对应 SpyAPI 方法 | 用途 |
|------|----------|-----------------|------|
| `@AtEnter` | 方法体第一条指令之前 | `atEnter()` | watch/trace/monitor/stack/tt 的方法进入 |
| `@AtExit` | 所有 return 指令之前 | `atExit()` | watch/trace/monitor/stack/tt 的方法正常返回 |
| `@AtExceptionExit` | catch 块中、throw 之前 | `atExceptionExit()` | watch/trace/monitor/stack/tt 的方法异常退出 |
| `@AtInvoke` | 每个方法调用指令前后 | `atBeforeInvoke()` / `atAfterInvoke()` | trace 的子调用追踪 |
| `@AtInvokeException` | 每个方法调用的异常处理 | `atInvokeException()` | trace 的子调用异常追踪 |
| `@AtLine` | 指定行号位置 | `atLine()` | watch/trace 的行级增强 |

### SpyAPI → SpyImpl → AdviceListener 回调映射

| SpyAPI 方法 | SpyImpl 查询方式 | 回调的 Listener 方法 |
|---|---|---|
| `atEnter()` | `queryAdviceListeners(classLoader, className, methodName, methodDesc)` | `listener.before()` |
| `atExit()` | 同上 | `listener.afterReturning()` |
| `atExceptionExit()` | 同上 | `listener.afterThrowing()` |
| `atBeforeInvoke()` | `queryTraceAdviceListeners(classLoader, className, owner, methodName, methodDesc)` | `listener.invokeBeforeTracing()` |
| `atAfterInvoke()` | 同上 | `listener.invokeAfterTracing()` |
| `atInvokeException()` | 同上 | `listener.invokeThrowTracing()` |
| `atLine()` | `queryLineAdviceListeners(classLoader, className, methodName, methodDesc, lineNumber)` | `listener.atLine()` |

---

## 五、防重复增强机制

多个命令可能对同一个类的同一个方法进行增强（如同时执行 `watch` 和 `trace`），Arthas 通过以下机制避免重复注入：

### 5.1 GroupLocationFilter

在 `Enhancer.transform()` 中，创建 `GroupLocationFilter` 检查方法字节码中是否已包含 `SpyAPI` 的调用：

```java
// 检查 ENTER 位置是否已有 atEnter 调用
LocationFilter enterFilter = new InvokeContainLocationFilter(
    "java/arthas/SpyAPI", "atEnter", LocationType.ENTER);

// 检查 EXIT 位置是否已有 atExit 调用
LocationFilter existFilter = new InvokeContainLocationFilter(
    "java/arthas/SpyAPI", "atExit", LocationType.EXIT);

// 检查 EXCEPTION_EXIT 位置是否已有 atExceptionExit 调用
LocationFilter exceptionFilter = new InvokeContainLocationFilter(
    "java/arthas/SpyAPI", "atExceptionExit", LocationType.EXCEPTION_EXIT);
```

如果某个位置已经注入过，`MethodProcessor` 会跳过该位置，不会重复注入。

### 5.2 多 Listener 共享注入点

虽然字节码只注入一次，但 `AdviceListenerManager` 支持同一个方法注册多个 `AdviceListener`。当 `SpyImpl` 回调时，会遍历所有注册的 Listener：

```java
List<AdviceListener> listeners = AdviceListenerManager.queryAdviceListeners(...);
for (AdviceListener listener : listeners) {
    listener.before(clazz, methodName, methodDesc, target, args);
}
```

### 5.3 Transformer 优先级

`TransformerManager` 内部维护了三个优先级列表：

```java
// TransformerManager 的 transform 链
for (ClassFileTransformer t : reTransformers) {     // 优先级最高
    byte[] result = t.transform(...);
}
for (ClassFileTransformer t : watchTransformers) {  // 优先级中等
    byte[] result = t.transform(...);
}
for (ClassFileTransformer t : traceTransformers) {  // 优先级最低
    byte[] result = t.transform(...);
}
```

---

## 六、关键设计要点

### 6.1 SpyAPI 的 BootstrapClassLoader 加载

`SpyAPI` 的包名是 `java.arthas`，通过 `-Xbootclasspath/a` 参数附加到 BootstrapClassLoader。这确保了：

- **所有 ClassLoader 都能访问**：无论目标类由哪个 ClassLoader 加载，都能解析 `SpyAPI` 的符号引用
- **避免 ClassLoader 隔离问题**：不会出现 `NoClassDefFoundError`

### 6.2 NopSpy 默认实现

`SpyAPI` 初始化时使用 `NopSpy`（所有方法为空实现），避免在 Arthas 未 attach 时注入的代码抛出 NPE。只有在 `Enhancer` 初始化后才替换为 `SpyImpl`。

### 6.3 懒加载模式（`--lazy` 参数）

当目标类尚未加载时，使用 `--lazy` 参数可以让 Arthas 等待类首次加载时自动增强：

```java
if (isLazy) {
    // 注册为 addTransformer(transformer, false)，使其在类首次定义时也能工作
    ArthasBootstrap.getInstance().getTransformerManager().addLazyTransformer(this);
}
```

### 6.4 命令生命周期管理

- **注册**：`process.register(listener, enhancer)` 将 listener 注册到 `AdviceWeaver`
- **清理**：`AdviceListenerManager` 定时清理已终止的 Process 对应的 Listener
- **重置**：`reset` 命令调用 `Enhancer.reset()` 移除 Transformer 并重新 `retransformClasses` 恢复原始字节码

### 6.5 类增强缓存

```java
// 防止同一个类被重复增强
private final static Map<Class<?>, Object> classBytesCache = new WeakHashMap<>();
```

使用 `WeakHashMap` 避免内存泄漏，当类被 GC 时缓存自动清除。

---

## 七、各命令的 AdviceListener 实现

| 命令 | AdviceListener | 关键行为 |
|------|---------------|---------|
| `watch` | `WatchAdviceListener` | 在 before/afterReturning/afterThrowing 中执行 OGNL 表达式并输出结果 |
| `trace` | `TraceAdviceListener` | 维护调用树，在 invokeBeforeTracing/invokeAfterTracing 中构建树节点 |
| `monitor` | `MonitorAdviceListener` | 统计方法调用次数、成功/失败次数、平均耗时 |
| `stack` | `StackAdviceListener` | 在 before 中获取当前线程的调用栈并输出 |
| `tt` | `TimeTunnelAdviceListener` | 记录每次调用的入参、返回值、异常，支持回放 |

---

## 八、总结

Arthas 的增强机制核心是 **JVM Instrumentation + ASM 字节码注入 + Spy 桥接 + Listener 回调** 四层架构：

1. **Instrumentation 层**：利用 `retransformClasses()` 触发 JVM 回调，无需重启
2. **ASM + ByteKit 层**：在方法的 ENTER/EXIT/EXCEPTION_EXIT/INVOKE/LINE 位置精确注入 `SpyAPI` 调用
3. **Spy 桥接层**：`SpyAPI`（BootstrapClassLoader）→ `SpyImpl` → `AdviceListenerManager` 查找 Listener
4. **Listener 层**：各命令实现自己的 `AdviceListener`，在回调中执行诊断逻辑

这套架构的优点是：
- **解耦**：字节码注入逻辑（Enhancer）与诊断逻辑（AdviceListener）完全分离
- **可扩展**：新增命令只需实现 `AdviceListener`，无需修改字节码注入逻辑
- **防重复**：通过 `GroupLocationFilter` 避免同一位置被多次注入
- **多命令共存**：多个命令可以同时增强同一个方法，各自独立工作
