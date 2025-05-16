### 1 关于 Read View
>Read View 用于在事务执行期间确定哪些版本的数据对当前事务可见。
#### 1.1 创建时机
下面是 MySQL 8.0.26 中创建 Read View 的代码，这是对普通的 SELECT 的处理，在查询开启前需要生成ReadView。
```cpp
else if (prebuilt->select_lock_type == LOCK_NONE) {  
	/* This is a consistent read */  
	/* Assign a read view for the query */  
	if (!srv_read_only_mode) {  
		trx_assign_read_view(trx);  
	}  
	prebuilt->sql_stat_start = FALSE;
}
```
而对于加锁读的语句：
```cpp
 else {
	wait_table_again:  
	err = lock_table(0, index->table,  prebuilt->select_lock_type == LOCK_S ? LOCK_IS : LOCK_IX,  thr);  
	if (err != DB_SUCCESS) {
		table_lock_waited = TRUE;
		goto lock_table_wait;
	}
	prebuilt->sql_stat_start = FALSE;  
}
```
则不会生成 Read View，而是先在尝试在表格上加一个意向锁。
#### 1.2 基本组成
Read View 有四个重要的字段：
- `m_ids` ：在创建 Read View 时，当前数据库中活跃事务（启动了但还没提交的事务）的 id 集合；
- `min_trx_id` ：在创建 Read View 时，m_ids 中最小的 id，也就是所有活跃事务中最早开启的事务的 id；
- `max_trx_id` ：创建 Read View 时当前数据库中应该给下一个事务的 id 值，值为当前全局事务中最大的事务 id + 1；
- `creator_trx_id` ：创建该 Read View 的事务的事务 id。
这几个字段圈定了一个以 id 为轴的几个范围：
![[ReadView 圈定的范围.png]]
当一个事务执行的过程中，它应该只能看到在其启动已经完成的事务，判断方式就是：
1. 如果一个事务的 ID 小于 `min_trx_id`，也就是当前活跃事务的最小 id 之前的事务，一定是已经提交了的。
2. 如果事务 ID 在 `min_trx_id` 和 `max_trx_id` 之间，则会存在已经提交和未提交的事务，此时就需要借助 `m_ids`（启动事务的时候活跃的事务，是一个 Set 类型），去检查这里面是否含有这个事务，如果不含有，则说明这个事务也是在事务之前提交的。
#### 1.3 事务 id 分配时机
会不会有这样的情况：有个事务未启动且未提交，而它的 ID 处于 `min_trx_id` 到 `max_trx_id` 这个范围内，在我们的事务执行的时候，这个事务执行完成并提交了，我们的事务应该可以看见这个事务的提交情况吗？
这种情况并不会发生，因为事务 id 的分配时机并不是创建后，在 MySQL 中有两种开启事务的方式：
1. `begin` 或 `start transaction` 命令；
2. `start transaction with consistent snapshot` 命令；
使用下面的语句可以看到当前所有事务的状态：
```sql
SELECT * FROM INFORMATION_SCHEMA.INNODB_TRX;
```
现在来观察一个案例，在这个案例中，先使用查询语句操作数据表，再使用更新语句：
![[事务 ID 的分配.png]]
1. 开始的时候，事务表中没有任何事务；
2. 然后我们执行 `BEGIN` 语句，可以观察到还是没有任何的数据产生；
3. 而此时执行第一条查询语句的时候，事务才真正的显露出来，此时有了事务 ID；
4. 执行修改语句的时候，事务的 ID 发生了变化。
这是因为在 MySQL 中，只读的事务分配的是一个虚拟的 `trx_id`，其值一定是大大超过当前执行修改数据的事务的，只有当事务真正的要修改数据库的时候，才会产生实际的事务 id。
trx_id 是 InnoDB 内部来维护的，维护了一个 max_trx_id 全局变量，每次需要申请一个新的 trx_id 时，就获得 max_trx_id 的当前值，然后并将 max_trx_id 加 1，这也是为什么我们连续启动两个修改数据的事务的时候，事务 ID 会是相邻的两个 ID。
但是即使分配的是虚拟的事务 ID，Read View 也是在执行 SELECT 语句后立刻创建的，所以延迟分配 ID 并不会影响 MVCC 的效果 ，来看这样一个案例：
#### 1.4 可见性判断
通过 Read View，可以圈定一个范围，哪个是当前执行的事务该看到的，哪个是当前事务不该看到的，可以看一下具体的代码，会更容易理解一些：
```cpp
/** Check whether the changes by id are visible.  
@param[in]  id transaction id to check against the view  
@param[in]  name table name  
@return whether the view sees the modifications of id. */  
[[nodiscard]] bool changes_visible(trx_id_t id,   const table_name_t &name) const {  
  // 检查 ID 是否大于零，如果否会直接返回异常
  ut_ad(id > 0);  

  // 如果 ID 小于等于 min_trx_id ，直接返回 true
  if (id < m_up_limit_id || id == m_creator_trx_id) {  
    return (true);  
  }  

  // 检查 ID 的准确性
  check_trx_id_sanity(id, name);  

  // 如果 ID 大于等于 max_trx_id 直接返回 false
  if (id >= m_low_limit_id) {  
    return (false);  
  } else if (m_ids.empty()) {
  // 如果 ID 在 min_trx_id 到 max_trx_id 之间
    return (true);  
  }  
  // 在 m_jds 中搜索 ID，如果不存在，返回 true
  const ids_t::value_type *p = m_ids.data();  
  
  return (!std::binary_search(p, p + m_ids.size(), id));  
}
```
### 2 MySQL 中的隐藏列
#### 2.1 数据结构
上面的 Read View 帮助我们解决了当前事务该看到什么记录，但并没有解决存储的问题，隐藏列通过存储历史记录解决了这个问题：
![[undo log 图例.png|900]]
每条记录中有两个隐藏列：
- `trx_id`，当一个事务对某条聚簇索引记录进行改动时，就会**把该事务的事务 id 记录在 trx_id 隐藏列里**；
- `roll_pointer`，每次对某条聚簇索引记录进行改动时，都会把旧版本的记录写入到 undo 日志中，然后**这个隐藏列是个指针，指向每一个旧版本记录**，于是就可以通过它找到修改前的记录。
通过 Read View 和 隐藏列，解决了当前事务应该看到哪些数据 和 不同事务看到的不同数据应该如何存储这两个问题。
#### 2.2 可重复读的实现方式
可重复读的要求是只能看到当前事务启动的时候的数据，那 Read View 就应该在事务一启动的时候生成。
在搜索的过程中不断的去通过当前持有的 Read View 中的 `change_visible` 方法去检查自己能否看到这一列，不能的话，就通过隐藏列的 undo log 来找到在本事务开启之前，这一条数据的状态。
#### 2.3 读已提交的实现方式
读已提交的实现方式就是在每次执行语句的时候都会创建一个 Read View，创建 Read View 的语句是这样的：
```cpp
ReadView *trx_assign_read_view(trx_t *trx) {  
  // ......
  
  if (srv_read_only_mode) {  
    // ......
  } else if (!MVCC::is_view_active(trx->read_view)) {  
	// 创建 Read View
    trx_sys->mvcc->view_open(trx->read_view, trx);  
  }  
  return (trx->read_view);  
}

static bool is_view_active(ReadView *view) {  
  ut_a(view != reinterpret_cast<ReadView *>(0x1));  
  return (view != nullptr && !(intptr_t(view) & 0x1));  
}
```
发现当前事务的 Read View 非活跃的时候，会创建一个 Read View，在读已提交的状态下，每执行完一条语句就会删除当前事务的 ReadView：
```cpp
else if (trx->isolation_level <= TRX_ISO_READ_COMMITTED &&  
           MVCC::is_view_active(trx->read_view)) {  
  mutex_enter(&trx_sys->mutex);  
  // 删除 Read View
  trx_sys->mvcc->view_close(trx->read_view, true);  
  mutex_exit(&trx_sys->mutex);  
}
```
此时这个事务就可以看到当前已提交修改，也就实现了读已提交。