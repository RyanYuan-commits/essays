---
type: Gradle
sub-type: Basic Knowledge
---
Gradle 的设计哲学凝聚了过去几十年构建自动化领域的经验与教训, 其核心在于三大支柱: 灵活的基于任务的模型; 对性能的不懈追求; 表现力极强的领域特定语言 DSL. 

---

## 1	The Core Philosophy of Gradle

**基于任务的模型 (Task-Based Model)**：这是 Gradle 灵活性的源泉. 与 Maven 固定的、线性的生命周期阶段不同, Gradle 的核心是一个由 Tasks 构成的 DAG. 每个任务代表一个原子性的工作单元, 比如编译代码或运行测试. 你可以定义任务之间的依赖关系, Gradle 会确保它们按正确的顺序执行. 这种模型允许你精确地定义构建逻辑, 而不受预设阶段的束缚. 

**约定优于配置 (Convention over Configuration)**: Gradle 吸收了 Maven 的这一优点, 为标准项目类型提供了合理的默认配置. 例如, 它会默认在 `src/main/java` 目录下寻找 Java 源码. 然而, 与 Maven 僵化的模型不同, Gradle 的所有约定都是可以轻松覆盖和配置的, 这在满足特殊构建需求时提供了巨大的便利性. 

**丰富的构建脚本 (The DSL)**：Gradle 的构建脚本不仅仅是声明式的配置文件, 它们是**可执行的代码**. Gradle 提供了基于 Groovy 和 Kotlin 的 DSL, 允许你在需要时编写命令式逻辑, 从而赋予你解决复杂自动化问题的终极能力. 

**性能即特性 (Performance as a Feature)**：从诞生之初, Gradle 就将性能视为其核心特性之一. 它通过多种机制来避免不必要的工作, 从而显著加快构建速度, 这与在大型项目中可能变得缓慢的 Maven 形成了鲜明对比. 关键的性能特性包括: 

- **增量构建 (Incremental Builds)**：只重新构建自上次构建以来发生变化的部分; 
	
- **构建缓存 (Build Cache)**：重用之前任何构建所产生的输出;
	
- **Gradle 守护进程 (Daemon)**：在后台保持一个常驻进程, 将构建信息保留在内存中, 从而加速后续的构建. 

---

## 2	Domain-Sepcific Language

简称 DSL, 直译为领域特定语言, 是一种 **专门用来解决某一特定领域问题而设计** 的计算机语言; DSL 通常与通用编程语言 (General-Purpose Programming Language, GPP) 做对比:

- **设计目标不同**: DSL 用于解决某一个问题, 如定义构建流程, 数据库查询等, GPP 用于解决各种通用问题, 如构建系统, 应用等;
	
- **表达能力不同**: DSL 的表达能力有限, 而 GPP 是图灵完备的, 表达能力无限, 能够进行复杂的逻辑运算;

DSL 通常分为 External DSL 和 Internal DSL:

- External DSL: 拥有自己独立的语法, 解析器和编译器, 是一个完全独立的语言, 如 SQL, HTML, YAML, JSON 等;
- Internal DSL: 是在现有的编程语言的基础上构建出来的, 它利用宿主语言的语法, 通过库或 API 调用, 使其看起来像一门新语言, 最典型的案例就是 Gradle 的构建脚本.