---
title: 引用逃逸与不可变性思考
date: 2026-01-13
slug: reference-escape-and-immutability
categories: 
  - code
tags:
  - java
  - clean-code
---

## 背景

我们项目的订单流转系统，核心是一个通用的流程引擎。它的标准执行链路如下：

1. 准备数据：Service 层查询订单，获取订单的“历史累积提交数据”（`allSubmission`）。
2. 构建上下文：使用前端的当前提交数据与累积数据，构建 `FlowContext`。
3. 引擎执行：引擎根据当前步骤号和 Context，执行各流程结点的逻辑（校验、计算）。
4. 结果处理：引擎返回 `FlowResponse`，其中包含已经校验的输入、下一步的步骤和前端展示数据，Service 层合并已经校验的输入到 `allSubmission`，并更新入库。

为了通用性，我们使用了 `Map<String, Object>` 来承载数据。

在开发某个特定步骤时，我们需要将“后端处理结果”持久化到数据库，但现在的 `FlowResponse` 仅设计了返回给请前端的数据字段。为了快速实现，我们尝试直接在流程引擎的执行器中修改 `FlowContext` 里的 `allSubmission` Map。神奇的是，这样的实现方式居然生效了，数据被成功更新到了数据库。

## 分析

在发现这种方式可以实现数据更新后，我们开始思考，为什么这样可以奏效？查阅构建 `FlowContext` 的代码，很容易就可以发现问题：

```java
    Application application = applicationRepository.findByApplicationNo(applicationNo);
    // build flow context
    FlowContext context = FlowContext.builder()
            .currentSubmission(submittedData)
            .allSubmission(application.getAllSubmission())
            .build();
    // execute flow engine
    // merge filtered input into application's allSubmission
    // update database
    applicationRepository.save(application)
```

这段构建 flow context 代码，使 `FlowContext` 中的 `allSubmission` 和 `Application` 中的 `allSubmission` 指向了同一个 Map 对象引用。这就导致了在执行流程引擎时，任何针对 flow context 中的 `allSubmission` 的改动，都会隐式地更改持久层实体，最终导致了数据中的数据改动。
我们更新数据库的方式是一个典型的引用逃逸导致的隐式副作用。虽然这样方式实现了需要的功能，但却破坏了封装，让原本只读的 context，变成了可读写的后门，数据的流向也因此变得混乱，可能导致其他难以追踪的 bug。

## 解决方法

发现这个问题后，我们进行了以下的修复：

1. 防御性复制
   在构建 `FlowContext` 时，对 `allSubmission` 进行深拷贝，切断引用关联。

   ```java
    FlowContext context = FlowContext.builder()
            .currentSubmission(submittedData)
    //      .allSubmission(application.getAllSubmission())
            .allSubmission(deepCopyOfMap(application.getAllSubmission()))
            .build();
   ```

    这样即使 flow context 被改变，也不会影响到持久层对象。

2. 添加后端字段
    既然后端执行结果需要持久化到数据库，那么就应该正视这个流程引擎设计上的缺陷，显示在 `FlowResponse` 中增加后端的更新数据字段。

    ```java
    public record FlowResponse(
            String message,
            String nextStep,
            FlowStatus flowStatus,
            NodeStatus nodeStatus,
            Map<String, Object> filteredInput,
            // for backend updated
            Map<String, Object> backendData,
            // for frontend response
            Map<String, Object> frontendData) {
    }
    ```

    这样使得数据流向变清晰：`input -> engine -> backendData`，Service 层可以很明确知道，backendData 的结果就是需要更新到持久层的数据。

3. 不可变性
   最后，我们重新考虑了 `FlowContext` 和 `FlowResponse` 在业务流转中的作用。很显然，它们都是数据的不可变快照，只是负责将数据从一个地方传递到另一个地方，它们本身就不应该支持更改。
   因此，我们利用了 JDK 14+的 `Record` 特性和 `Collections.unmodifiableMap` 工具方法，构造了不可变的 `FlowContext` 和 `FlowResponse`，从类型系统层面，禁止了对这两个对象的修改。

## 思考

《Effective Java》中有这么一条建议，“使可变性最小化”。不可变性在这里，不仅是确保线程安全的手段，也是设计上的防腐剂。
试想我们一开始就践行最小可变性原则，将 `FlowContext` 设计为不可变的，那么这个问题会如何发展？

- 首先，当我们尝试执行 `context.getAllSubmission().put(...)` 的操作时，程序会直接抛出异常。
- 然后，这种阻碍会使得我们开始思考，“如果不能直接修改 context，那后端的执行结果应该放在哪里？”
- 最后，这种别扭的感觉将直接暴露 `FlowResponse` 缺少后端回写字段的设计缺陷。

不可变性在这里把“依靠自觉的代码规范”转化为了“编译器强制的硬性约束”，从而使我们在编码阶段就能注意到代码中的坏味道，迫使我们做出更合理的设计。
