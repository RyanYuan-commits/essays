行级锁加锁规则比较复杂，不同的场景，加锁的形式是不同的。**加锁的对象是索引，加锁的基本单位是临键锁**。
# 唯一索引等值查询
当我们用唯一索引进行等值查询的时候，查询的记录存不存在，加锁的规则也会不同，如果退化的锁就已经可以解决幻读的情况，锁就会退化，具体来说：
- 当查询的记录是「存在」的，在索引树上定位到这一条记录后，将该记录的索引中的 next-key lock 会 **退化成「记录锁」**。
- 当查询的记录是「不存在」的，在索引树找到第一条大于该查询记录的记录后，将该记录的索引中的 next-key lock 会 **退化成「间隙锁」**。
## 记录存在的情况
比如事务 A 执行了这样的语句，并且表中的数据是存在的：
```sql
BEGIN;
SELECT * FROM user_0 WHERE id = 1 FOR UPDATE;
-- 延迟执行 COMMIT
COMMIT;
```
此时，再提交一个事务 B 去执行修改语句的时候，事务 B 就会被阻塞。
```sql
UPDATE user_0 SET age = 19 WHERE id = 1;
```
我们可以通过这个语句来查看事务过程中都加了什么锁：
```sql
select * from performance_schema.data_locks;
```

| 列                     | Line1                                | Line2                                  |
| --------------------- | ------------------------------------ | -------------------------------------- |
| ENGINE                | InnoDB                               | InnoDB                                 |
| ENGINE_LOCK_ID        | 281473082985688:1076:281473097716656 | 281473082985688:10:4:2:281473097713664 |
| ENGINE_TRANSACTION_ID | 33306                                | 33306                                  |
| THREAD_ID             | 48                                   | 48                                     |
| EVENT_ID              | 52                                   | 52                                     |
| OBJECT_SCHEMA         | user_db                              | user_db                                |
| OBJECT_NAME           | user_0                               | user_0                                 |
| PARTITION_NAME        | 无                                    | 无                                      |
| SUBPARTITION_NAME     | 无                                    | 无                                      |
| INDEX_NAME            | 281473097716656                      | PRIMARY                                |
| OBJECT_INSTANCE_BEGIN | 281473097716656                      | 281473097713664                        |
| LOCK_TYPE             | **TABLE**                            | **RECORD**                             |
| LOCK_MODE             | IX                                   | X,REC_NOT_GAP                          |
| LOCK_STATUS           | GRANTED                              | GRANTED                                |
| LOCK_DATA             | 无                                    | 1                                      |
在事务 A 的执行过程中，输出了上面展示出来的两行内容，它们的锁类型分别是表锁和记录，表锁上加的是一个意向排他锁（IX）；
记录锁上加的是一个排他锁（X），且是一个记录锁（RECORD NOT GAP）。
- 如果 LOCK_MODE 为 `X`，说明是 next-key 锁
- 如果 LOCK_MODE 为 `X, REC_NOT_GAP`，说明是记录锁
- 如果 LOCK_MODE 为 `X, GAP`，说明是间隙锁
从上面的数据中，可以看出，此时加的是记录锁，这是因为，在这种情况下，记录锁已经完全可以保证不会出现幻读的现象了。
幻读的定义就是，当一个事务前后两次查询的结果集，不相同时，就认为发生幻读。
此时给索引 1 加上锁（注意，加锁的位置是索引），就可以保证新的索引为 1 的值不会被插入，而原本的索引值为 1 的数据也不会被删除了。
## 记录不存在的情况
表中的数据是这样的：
![[唯一索引等值查询记录不存在的情况.png|500]]

比如，此时我们需要查询索引值为 4 的数据，那肯定是无法查询到的，此时我们来看一下此时都加锁情况：

| 列                     | Line2                                | Line2                                  |
| --------------------- | ------------------------------------ | -------------------------------------- |
| ENGINE                | InnoDB                               | InnoDB                                 |
| ENGINE_LOCK_ID        | 281473082985688:1076:281473097716656 | 281473082985688:10:4:5:281473097713664 |
| ENGINE_TRANSACTION_ID | 33308                                | 33308                                  |
| THREAD_ID             | 48                                   | 48                                     |
| EVENT_ID              | 55                                   | 55                                     |
| OBJECT_SCHEMA         | user_db                              | user_db                                |
| OBJECT_NAME           | user_0                               | user_0                                 |
| PARTITION_NAME        | 无                                    | 无                                      |
| SUBPARTITION_NAME     | 无                                    | 无                                      |
| INDEX_NAME            | 281473097716656                      | PRIMARY                                |
| OBJECT_INSTANCE_BEGIN | 281473097716656                      | 281473097713664                        |
| LOCK_TYPE             | TABLE                                | RECORD                                 |
| LOCK_MODE             | IX                                   | X,GAP                                  |
| LOCK_STATUS           | GRANTED                              | GRANTED                                |
| LOCK_DATA             | 无                                    | 5                                      |

从上图可以看到，共加了两个锁，分别是：X 类型的意向锁（表锁） 、X 类型的间隙锁（行锁）。
此时我们去修改其他 id 的数据都是没有问题的，可以推断出，此时加的是 (3, 5) 这个范围的间隙锁；当然，添加 id 为 4 的数据会被阻塞。