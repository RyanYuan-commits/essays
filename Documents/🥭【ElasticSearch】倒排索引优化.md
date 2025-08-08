![[倒排索引思维导图.png]]
## 1 Posting List 优化

解决 Posting List 需要存储数据过多导致的空间浪费问题。

### 1.1 索引帧（Frame Of Reference）

适用于分散紧密的情况，存储的是前后两个数字的差值。

![[Frame Of Reference 算法.png|center|700]]

### 1.2 咆哮位图（Roaring Bitmap）

适用于稀疏分散的情况，存储的是一个坐标值，坐标由某个数字跟这个 ID 做除法和数字对这个 ID 取余构成。

![[咆哮位图压缩算法.png|center|700]]

## 2 Term Index 优化

### 2.1 前言

ES 用户搜索的大致过程是这样的：

- 当用户搜索一个词项时，首先会通过词项索引（Term Index）中找到该词项在词项字典中的位置；
- 然后通过词项索引定位到该词项对应的 **Post List**；
- 最终，根据 **Post List** 获取相关文档，并返回给用户。

我们可以将词项索引理解为一个 K-V 结构，其 key 值就是 term 本身，而 value 就是指向 Term Dictionary 对应项的指针。

对于搜索引擎这样，Term Dictionary 动辄就是以 “亿“ 起步的，那如何压缩就成了一个迫切需要解决的问题，我们从最经典的字符串压缩算法字典树开始讲起。

### 2.2 字典树 Trie

如果存储的是英文单词，那无论任何一个词项，无外乎由 26 个英文字母组成，这也就意味越多的词项就会造成的越多的数据 “重复”，也就是说词项之间会有很多个公共部分，如 “abandon” 和 “abandonment” 就共享了公共前缀 “abandonment”。

Lucene 在存储这种有重复字符的数据的时候，只会存储一次，也就是哪怕有一亿个以 abandon 为前缀的词项，“abandom” 这个前缀也只会存储一次。

这里就用到了一种我们经常用到的一种数据结构：Trie 即字典树，也叫前缀树，下面我们以 Term Dictionary：（es、estech、eslint、rtech）为例，演示一下 Trie 是如何存储 Term Dictionary 的。

在线体验 Trie：[https://www.cs.usfca.edu/~galles/visualization/Trie.html](https://www.cs.usfca.edu/~galles/visualization/Trie.html)

![[前缀树演示.png]]

在上面的案例中，公共前缀 es 虽然被重复利用了，但是重复部分 tech 没有被重复的利用，这是前缀树存在的一个问题。
### 2.3 从 FSM 到 FSA

通常我们在计算机的语言中标示一件事物，都会通过某种数学模型来描述。

假如现在我们要描述一件事：张三一天的所有活动；这里我们采用了一种叫做 FSM（Finite State Machine）的抽象模型，这种模型使用原型的节点标示某个“状态”，状态之间可以互相转换，但是转换过程是无向的。

比如睡觉醒了可以去工作，工作累了可以去玩手机；或者工作中想去上厕所等等。在这个模型中，标示状态的节点是有限多个的，但状态的转换的情况是无限多的，同一时刻只能处于某一个状态，并且状态的转换是无序切循环的。

![[FSM 案例.png|center|600]]

这种模型并不适合存储 Term Dictionary，但是它将数据存储在边上的思想却很有价值，在 FSM 基础上衍生出了 FSA（Finite State Acceptor）。

![[FSA 案例.png|center|800]]

相较于 FSM，FSA 增加了 Entry 和 Final 的概念，也就是由状态转换的不确定性变为了确定，由闭环变为了单向有序，这一点和 Trie 是类似的；

但是不同的是，FSA 的 Final 节点是唯一的，也是因为这个原因，FSA 在录入和 Trie 相同的 Term Dictionary 数据的时候，从第三步开始才表现出了区别，即尾部复用。如果在第三步的时候还不太明显，那第四步中就可以清楚的看到FSA在后缀的处理上更加高效。

至此，FSA 已经满足了对 Term Dictionary 数据高效存储的基本要求，但是仍然不满足的一个问题就是，FSA 无法存储 key-value 的数据类型，所以 FST 在 FSA 基础上为每一个出度添加了一个 output 属性，用来存储每个 term 的 value 值。

### 3 FST 有限状态转换机
### 3.1 FST 插入案例

下面插入这些 K-V 对，看一下 FST 是如何存储的：（es/10、estech/5、eslint/8、rtech/16）

![[FST 案例.png|center|600]]

1. 当第一个词项 es/10 被写入的时候，其输出的值被保存在第一个节点的出度上；当数据从 FST 中读取的时候，计算每个节点对应的出度以及终止节点的 Final Output 的总和就能得到存储的 value 值。
2. 第二个词项 estech/5 被写入的时候，其输出值 5 和 es 的输出值 10 产生了冲突，此时为了保证两者均存储正确，FST 会将 10 拆分成两个 5，一个 5 仍然作为 0 号节点的出度，而另一个 5 就需要找一个合适的位置存放；
   而其存放在任何位置都会影响 estech/5 的计算，为了避免这个问题，将数据存储在 Final 节点上，作为其 Final Output 存储。
3. 剩余两个词项的存储过程比较简单，这里将不多赘述，直接看上面的图例即可。

#### 3.2 Lucene 中 FST 的构建过程

在 Lucene 中，通过一个泛型来描述 FST 的数据结构，`org.apache.lucene.util.fst.FST`。

对于 FST 实例对象的构造，Lucene 利用了建造者模式，提供了一个 Builder 类来便捷的获取 FST 实例对象，FST 构造过程中涉及到的大部分代码和使用到的 infrastructure 都作为方法或者内部类在 Builder 类中封装。

FST 构造过程中的节点，使用 Builder 中的一个静态内部类：UnCompiledNode 来表示：

```java
public static final class UnCompiledNode<T> implements Node {  
	/* 表示当前节点所属的 FST 构造器实例 */
	final Builder<T> owner;  
	/*当前节点的出边（arc）数量，表示从该节点可以直接到达的子节点的数量*/
	public int numArcs;  
	/*表示从当前节点出发的所有弧（arc）*/
	public Arc<T>[] arcs;  
	/* 当前节点的累积输出值，即从根节点到达该节点路径上的所有弧的输出值之和。*/
	public T output;  
	/* 表示当前节点是否是一个“终止节点” */
	public boolean isFinal;  
	/* 表示以当前节点为起点的子树中所有路径对应的输入项的总数 */
	public long inputCount;  
	/* 节点的深度 */  
	public final int depth;

	// 其他方法。。。
}
```

FST 中的边也是使用 Builder 中的一个静态内部类 Arc 来表示：

```java
public static class Arc<T> {  
	/* 表示从当前节点通过该弧到达目标节点的输入符号，即该弧上的字符或标记。*/
	public int label; 
	/* 表示该弧连接的目标节点，即通过此弧后可以到达的节点 */
	public Node target;  
	/*表示从当前节点经过该弧到达的目标节点是否是一个终止状态。*/
	public boolean isFinal;  
	/* 表示通过该弧从当前节点到达目标节点时的增量输出值 */
	public T output;  
	/* 在路径上某些节点既是中间节点又是终止状态时（例如共享前缀的路径），nextFinalOutput 用于区分中间状态和终止状态的输出值。 */
	public T nexLucene 中使用了一个泛型来描述 FST，org.apache.lucene.util.fst.FST
```java
public final class FST<T> implements Accountable {}
```

Lucene 中使用建造者模式，构建了一个 Builder 类，在注释中提到，它的作用为：Builds a minimal FST (maps an IntsRef term to an arbitrary output) from pre-sorted terms with outputs；也就是通过一个提前排序好的、具有输出（output）的词项（term）组来构建一个最小的 FST 结构。设计 FST 构建过程的大部分代码都被封装在这个类中。

在 FST 对象的构建过程中又用一个 Node 类型的对象来描述 FST 模型中的节点，这个节点是 org.apache.lucene.util.fst.Builder 的内部接口：

```java
static interface Node {  
  boolean isCompiled();  
}
```

它有两个实现类，也是 org.apache.lucene.util.fst.Builder 的静态内部类，分别是 UnCompiledNode 和 CompiledNode，它们的区别就是是否已经 “Compiled“，暂时可以理解为是否经过了某种处理，经过了处理的节点就是 Compiled节点。


![[frontier 数组.png]]

未经过处理的 Node，也就是 UnCompiledNode 被存放在一个名为 frontier 的数组中，假设此时插入的是第一个词项，此时当前 Term 所有的字符都不会被处理，因为 FST 构建是遵循 **尾部冻结** 规则的。

FST 最终会被构建为一个 FST 对象，这个对象最终会转化为一个二进制对象存储在一个 ByteStore 中，在 Lucene 中，ByteStore 中封装了一个 byte 类型的数组 current，这个数组就是用来存储经过处理之后的节点，也就是 CompiledNode 的。

那么什么时机才是 “Compiled” 这个动作最好的时机呢？也就是什么时候才是我们从 UnCompiledNode 数组中摘下来节点并且计算结果存放在 current 中的最好时机呢？

我们知道，FST 插入的数据是按照字典序来排序的，这里假设插入顺序是 abd、abdc、abe，由于字典排序，当出现 abe 的时候，就说明第三个数字是 d 的 Term 将永远不会在将来的插入中，这个节点将不会有任何变化了，此时就是将这个数据落盘的最好时机。

![[CompiledNode 示意图.png]]

还是按照上面的那个插入顺序，当插入 abe 的时候，frontier 中的情况如上图所示，其中 S_d 节点是被执行了 freezeTail 操作，成为了一个 Compiled 节点。
