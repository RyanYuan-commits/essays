## 1 ZSet 与跳表
Redis 速度快的原因之一就是它选择了「合理的数据编码」来实现其数据基本类型，其中有序集合 ZSet 的数据编码是这样的：当其中的元素个数小于 128 个，每个元素的值小于 64 字节时，使用压缩列表编码，否则使用 **跳表** 编码。
相比于 Redis 中的另一个集合类型 Set，ZSet 提供了 Score 分数的概念，并且围绕这个概念，提供了对 Score 的 **排序** 和 **范围查询** 的功能。
为了实现这些基本功能，Redis 需要一个 **高效** 的数据结构，跳表承担了这一责任。
接下来，让我们快速认识一下跳表这个简单高效的数据结构：
### 1.1 跳表基本介绍
跳跃表(简称跳表)由美国计算机科学家 William Pugh 于 1989 年提出的。他在论文[《Skip lists: a probabilistic alternative to balanced trees》](https://dl.acm.org/doi/epdf/10.1145/78973.78977)中详细介绍了跳表的数据结构和插入删除等操作。

> 跳表是一个使用随机性平衡而不是强制性平衡的数据结构，这使得他在算法简单性 和 插入删除性能上相较于平衡树有了很大的提升
> 跳表是一个随机化的数据结构，实质上就是一个可以进行二分查找的有序链表，跳表在有序链表的基础上，添加了多层的索引，通过索引来提升了查询的效率。跳表的性能和自平衡二叉树以及红黑树的查询性能相当，为 $O(logN)$，但是实现方式相比于前面两者要简单很多。

总结一下，跳表的实质就是一个可以进行二分查找的 **有序链表**，通过随机化的方式实现。
与顺序数组相比，顺序链表的查询速度非常非常慢；当在顺序链表中查询数据的时候，每次查找都需要一个个进行枚举，要找到所需节点的话，平均要查询 $n/2$ 个节点，那能不能稍微优化一下呢？
![[跳表链表图例.png|1000]]

现在，将链表节点两个为一组，提取前面的节点到上一层，使其成为索引，结构大概是这样的：
![[跳表链表案例优化.png|1000]]
此时的查询速度就会快很多，比如当我们需要查询节点 5 的时候，只需要在第一层定位到节点 4，在下层查找的时候，移动一次指针就可以查询到节点 5。类似的，在查询链表中某个节点的时候，首先会从上一层快速定位一个节点，再到下层进行查询，查询速度比之前优化了一倍。但是当节点数量很大的时候，与二分查找相比，这个速度依旧很慢。

二分查找在每次查找的时候，会将查找范围缩短为原本的二分之一，对于上面的案例中，再在上一层添加一层索引就可以实现这一效果:
![[跳表 3.png|1000]]
**理想情况**下，只需要保证上一层的节点个数为下一层节点个数的一半，这样当从最顶层开始查找的时候，每次跳跃，就能使搜索范围缩短为原来的二分之一。
这就是跳表的大致实现思路，但是这样完美的跳表（上一层的个数严格为下一层的一半）大概率是不存在的，作为一个链表，少不了增加和删除的操作，插入和删除可能会导致链表的整个结构都受到影响，而此时如果还是要保证 “跳表“ 完美，时间消耗就很大了，跳表使用随机化巧妙的解决了这个问题。
### 1.2 跳表的随机化算法
所谓随机化，简单理解一下就是抛硬币；
跳表的最底层是一个含有全部元素的有序列表，当插入一个节点的时候，抛一次硬币，如果正面就到上层变为节点，否则就只留在最底层，节点到倒数第二层的概率为 **二分之一**。
当到达倒数第二层的时候，再抛一次硬币，用这个结果来决定要不要到倒数第三层，此时节点到倒数第三层的概率就是 **四分之一**，以此类推......
最终，当样本足够多的时候，最后一层有 $n$ 个节点，倒数第二层就有 $\frac{n}{2}$ 个节点，倒数第三层就有 $\frac{n}{4}$ 个节点......
## 2 跳表简单实现
为了更好的理解跳表的实现方式，我们来手写一个简单的跳表。
### 类定义
#### 跳表节点类
```java
@Data  
static class SkipNode<T> {  

    int key;  
  
    T value;  
  
    SkipNode<T> right, down;  
  
    public SkipNode (int key, T value) {  
        this.key = key;  
        this.value = value;  
    }  
  
}
```
上面是跳表节点，其存储一个 k-v 值，k 是 int 类型，跳表会根据它来排序，v 是一个泛型，代表具体的值。
#### 跳表类
```java
static class SkipList<T> {  
  
    /**  
     * 头节点，入口  
     */  
    SkipNode<T> headNode;  
  
    /**  
     * 总层数
     */  
    int highLevel;  
  
    /**  
     * 抛硬币  
     */  
    ThreadLocalRandom random;  
  
    /**  
     * 最大层数  
     */  
    final int MAX_LEVEL = 32;  
  
    public SkipList(){  
        random = ThreadLocalRandom.current();  
        headNode = new SkipNode<>(Integer.MIN_VALUE, null);  
        highLevel = 1;  
    }
}
```
跳表类是访问跳表的入口，它管理着跳表的头节点、总层数、以及一个 Random 类，来负责“抛硬币“。
### 方法
#### 跳表查询方法
```java
public SkipNode<T> search(int key) {  
    // 从头节点开始查询  
    SkipNode<T> temp = headNode;  
    while (temp != null) {  
        if (temp.key == key) return temp;  
        // 右侧没有了，从下层开始寻找  
        else if (temp.right == null) temp = temp.down;  
        // 定位到一个区间，向下进行查询  
        else if (temp.right.key > key) temp = temp.down;  
        else temp = temp.right;  
    }  
    return null;  
}
```
查询是从最顶层头节点开始的，指针不断向右移动，如果发现下一个节点比 target 要大或者右侧没有节点了，就去下一层寻找。
#### 跳表删除方法
```java
public void delete(int key) {  
    SkipNode<T> temp = headNode;  
    while (temp != null) {  
		// 右侧没有了，向下找
        if (temp.right == null) temp = temp.down;  
        // 找到了需要删除的节点，在节点的右侧  
        else if (temp.right.key == key) {  
            temp.right = temp.right.right; 
            // 继续向下找 
            temp = temp.down;  
        }  
        else if (temp.right.key > key) temp = temp.down;  
        else temp = temp.right;  
    }  
}
```
还是从头节点开始遍历，直到找到需要删除的节点的上一个节点，具体删除方式和链表相同，使上一个节点指向他的 next.next 即可；
然后继续向下遍历，将每一层的节点都删除掉。
#### 跳表插入方法
```java
public void add(SkipNode<T> node) {  
    int key;  
    SkipNode<T> temp;  
    // 如果已存在，更新  
    if ((temp = search(key = node.key)) != null) {  
        while (temp != null) {  
            // 从 temp 开始查询  
            temp.value = node.value;  
            temp = temp.down;  
        }  
        return;  
    }
  
    // 存储需要插入的节点  
    Stack<SkipNode<T>> stack = new Stack<>();  
    // 从 header 开始寻找，找到插入的位置  
    temp = headNode;  
    while (temp != null) {  
        if (temp.right == null || temp.right.key > key) {  
            stack.add(temp);  
            temp = temp.down;  
        } else temp = temp.right;  
    }  
  
    int level = 1;  
    // 上层节点需要指向的节点  
    SkipNode<T> downNode = null;  
    while (!stack.isEmpty()) {  
        SkipNode<T> pop = stack.pop();  
        // 处理竖向  
        SkipNode<T> newNode = new SkipNode<>(node.key, node.value);  
        newNode.down = downNode;  
        downNode = newNode;  
  
        // 处理横向  
        newNode.right = pop.right;  
        pop.right = newNode;  
  
        // 查看是否还需要向上  
        if (level > MAX_LEVEL) break;  
        double num = random.nextDouble();  
        if(num > 0.5) break;  
  
        // 需要继续向上  
        level++;  
        if (level > highLevel) {  
            highLevel = level;  
            SkipNode<T> newHead = new SkipNode<>(Integer.MIN_VALUE, null);  
            newHead.down = headNode;  
            headNode = newHead;  
            stack.add(headNode);  
        }  
    }  
}
```
大致的操作流程是这样的：
1. 从最上层开始遍历，在遍历的过程中找到**待插入的左节点**，具体来说就是每层 **最后一个** key 比新插入的 key 小的节点，用栈存储起来。
2. 当遍历到最底层的时候，开始执行插入操作：取出栈中的节点，构建新的节点插入到后面。
3. 然后抛一次硬币，决定要不要继续插入，如果需要继续插入，继续执行第二步的操作。
4. 在节点上溢的时候，需要判断是不是已经到达最顶层，如果是的话，需要构造新的头节点，然后将头节点存入栈中。
#### 跳表打印方法
```java
public void printList() {  
    SkipNode<T> temp = headNode;  
    String format = "(%s, %s)";  
    // 存储每一行的开头  
    SkipNode<T> listHead = temp;  
    while (listHead != null && temp != null) {  
        System.out.print(String.format(format, temp.key, temp.value) + " ->" + "\t");  
        if (temp.right == null) {  
            temp = listHead.down;  
            listHead = listHead.down;  
            System.out.print("\n");  
        } else {  
            int num = -1;  
            int rightKey = temp.right.key;  
            SkipNode<T> down = temp.down;  
            while (down != null && down.key != rightKey) {  
                down = down.right;  
                num++;  
            }  
            for (int i = 0; i < num; i++) {  
                System.out.print("--------->" + "\t");  
            }  
            temp = temp.right;  
        }  
    }  
}
```
具体来说就是输出每一行，但是输出的时候要注同一行的元素下一行对齐，最终输出结果大致如下：
```java
(-2147483648, null) ->	(3, 🍌) ->	
(-2147483648, null) ->	(3, 🍌) ->	
(-2147483648, null) ->	--------->	(3, 🍌) ->	
(-2147483648, null) ->	(1, 🍎) ->	--------->	(3, 🍌) ->	
(-2147483648, null) ->	(1, 🍎) ->	(2, 🍊) ->	(3, 🍌) ->	(4, 🐶) ->	(5, 🫎) ->	
```
## 3 跳表时间复杂度计算
对于一个包含 n 个节点的跳表，每个节点的层数是独立随机确定的。一个节点成为第 k 层节点的概率为 $p^{k - 1}(1 - p)$。
根据概率论，跳表的期望最大层数 L 可以通过计算每个节点层数的期望值来得到，一个节点的层数 h 的期望值为：
$$E(h)=\sum_{k = 1}^{\infty}k\times p^{k - 1}(1 - p)=\frac{1}{1 - p}$$
当 $p = \frac{1}{2}$ 时，$E(h) = 2$当 $p = \frac{1}{4}$ 时，$E(h) = \frac{4}{3}$。
在实际应用中，为了避免层数过高，通常会设置一个最大层数限制 $(L_{max})$，一般取 $(L_{max}=\log_{\frac{1}{p}}n)$。
在跳表中进行查询操作时，从最高层开始，沿着每一层的指针向前移动，直到找到目标节点或者确定目标节点不存在。在每一层上，指针最多移动的次数不会超过该层的节点数。
由于跳表的每一层节点数是呈指数级递减的，第 i 层的节点数大约为 $(n\times p^i)$。
查询过程中，从最高层 L 开始，逐层向下移动。在每一层上，指针最多移动的次数不会超过该层的节点数。因此，在每一层上的移动次数的期望值为：
- 第 L 层：最多移动 $n\times p^L$ 次。
- 第 \(L - 1\) 层：最多移动 $n\times p^{L - 1}$ 次。
- ......
- 第 1 层：最多移动 $n\times p$ 次。

查询的总移动次数 T 等于在每一层上移动次数的总和。由于跳表的层数是 $log_{\frac{1}{p}}n$，因此：
$$T=\sum_{i = 0}^{\log_{\frac{1}{p}}n}n\times p^i$$

这是一个等比数列求和，根据等比数列求和公式 $(S_n=\frac{a(1 - r^n)}{1 - r})$（其中 a 是首项，r 是公比，n 是项数），可得：
$$T=n\times\frac{1 - p^{\log_{\frac{1}{p}}n + 1}}{1 - p}=n\times\frac{1 - \frac{1}{n}\times p}{1 - p}\approx\frac{n}{1 - p}$$

当 $p = \frac{1}{2}$ 时，$T = 2n$；当 $p = \frac{1}{4}$ 时，$T = \frac{4}{3}n$。
但是，由于跳表的层数是 $\log_{\frac{1}{p}}n$，在每一层上移动的次数是常数级别的，因此查询的时间复杂度主要取决于跳表的层数。

由于跳表的期望最大层数为 $L=\log_{\frac{1}{p}}n$，在每一层上的移动次数是常数级别的，因此跳表的查询时间复杂度为 $O(\log n)$。
## Redis ZSet 源码浅析
与上面实现的简单跳表相比，Redis 中的 ZSet 要更复杂一些。
### 数据结构
#### 跳表节点 zskiplistNode
```cpp
// 跳表节点
typedef struct zskiplistNode {
    double score;                       // 节点的分数，排序、查找使用
    sds ele;                            // 节点的值
    struct zskiplistNode *backward;     // 后退指针，指向前一个节点
    struct zskiplistLevel {
        struct zskiplistNode *forward;  // 前进指针，指向后一个节点
        unsigned int span;              // 跨度
    } level[];                          // 层级数组
} zskiplistNode;
```
- `score`：用于节点的排序与查找，跳表中的节点按照 `score` 从小到大排列。
- `ele`：存储节点的值，采用 `sds`（简单动态字符串）类型。
- `backward`：后退指针，指向当前节点的前一个节点，便于逆向遍历。
- `level`：层级数组，每个元素是一个 `zskiplistLevel` 结构体。`forward` 指向当前层级的下一个节点，`span` 表示从当前节点到 `forward` 指向节点所跨越的节点数量。
#### 跳表 zskiplist
```cpp
// 跳表
typedef struct zskiplist {
    struct zskiplistNode *header, *tail; // 头节点和尾节点
    unsigned long length;                // 跳表长度
    int level;                           // 当前最大层级，默认1
} zskiplist;
```
- `header` 和 `tail`：分别指向跳表的头节点与尾节点。
- `length`：记录跳表中节点的数量。
- `level`：当前跳表的最大层级，初始默认值为 1。
### 关键操作实现
#### 随机层级生成
```cpp
#define ZSKIPLIST_P 0.25
#define ZSKIPLIST_MAXLEVEL 32

int zslRandomLevel(void) {
	// 初始层数是1
    int level = 1;
    // 以 ZSKIPLIST_P 的概率提升层级，随机层数的值是 0.25
    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;
    // ZSKIPLIST_MAXLEVEL 最大层数是64
    return (level < ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```
在插入新节点时，需要通过随机函数为其确定一下插入哪些层级，节点每次上溢的概率为四分之一。
#### 创建跳表节点
```cpp
zskiplistNode* zslCreateNode(int level, double score, sds ele) {
    // 为节点分配内存，level 决定节点具有的层数
    zskiplistNode *zn = zmalloc(sizeof(*zn) + level * sizeof(struct zskiplistLevel));
    zn->score = score;  // 节点的分数
    zn->ele = ele;      // 节点的值
    return zn;
}
```
- 在创建跳表节点之前，会通过
#### 插入操作
```cpp
zskiplistNode* zslInsert(zskiplist *zsl, double score, sds ele) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;
    
    x = zsl->header;  // 从头节点开始
    for (i = zsl->level-1; i >= 0; i--) {  // 从最高层往下遍历
        while (x->level[i].forward && 
               (x->level[i].forward->score < score || 
               (x->level[i].forward->score == score && 
                sdscmp(x->level[i].forward->ele, ele) < 0))) {  // 查找插入位置
            rank[i] += x->level[i].span;
            x = x->level[i].forward;
        }
        update[i] = x;  // 记录每层的前驱节点
    }
    
    level = zslRandomLevel();  // 随机生成新节点的层数
    
    if (level > zsl->level) {  // 如果新节点层数超过当前最大层数
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        zsl->level = level;  // 更新跳表的层数
    }
    
    x = zmalloc(sizeof(*x)+level*sizeof(struct zskiplistLevel));  // 分配新节点内存
    x->score = score;
    x->ele = sdsdup(ele);
    for (i = 0; i < level; i++) {  // 插入新节点，并更新相关指针和跨度
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }
    x->backward = (update[0] == zsl->header) ? NULL : update[0];  // 更新 backward 指针
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;  // 如果新节点是最后一个节点，更新尾节点指针
    zsl->length++;  // 更新跳表的长度
    return x;
}
```
插入操作的步骤如下：
1. 从最高层开始查找插入位置，同时记录每层需要更新的节点，这一步和上面我们实现的基本思路相同。
2. 调用前面提到的随机函数来生成新节点的层级，和前面我们的实现相比，相当于将确定层级这一步提前进行了。
3. 若新节点的层级大于当前跳表的最大层级，需要更新跳表的最大层级。
4. 创建新节点并更新相应指针。
