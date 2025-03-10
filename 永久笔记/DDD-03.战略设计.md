---
{}
---


# 核心概念关系图

![[DDD核心概念关系图.png]]
# 领域模型

![[领域模型1.png]]

# DDD 分层架构

![[DDD 分层架构图.png]]

# DDD 代码目录参考

![[DDD代码目录参考.png]]

# 核心域、通用语和支撑域

在领域不断划分的过程中，领域会细分为不同的子域，子域可以根据自身**重要性和功能属性**划分为三类子域，它们分别是：**核心域、通用域和支撑域**。

决定产品和公司核心竞争力的子域是**核心域**，它是业务成功的主要因素和公司的核心竞争力。没有太多个性化的诉求，同时被多个子域使用的通用功能子域是**通用域**。还有一种功能子域是必需的，但既不包含决定产品和公司核心竞争力的功能，也不包含通用功能的子域，它就是**支撑域**。

这三类子域相较之下，核心域是最重要的。通用域和支撑域如果对应到企业系统，举例来说的话，通用域则是你需要用到的通用系统，比如认证、权限等等，这类应用很容易买到，没有企业特点限制，不需要做太多的定制化。而支撑域则具有企业特性，但不具有通用性，例如数据代码类的数据字典等系统。

**那为什么要划分核心域、通用域和支撑域，主要目的是什么呢？** 实际上，它的目的主要是通过领域划分，区分不同子域在公司内的不同功能属性和重要性，从而公司可对不同子域采取不同的资源投入和建设策略，其关注度也会不一样。