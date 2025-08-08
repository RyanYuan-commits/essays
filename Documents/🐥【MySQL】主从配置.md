> [!warning] 注意
>  主从复制属于增量复制，在开启之前，务必保证主库和从库的数据已经完全相同；先将主库锁定，再手动进行主从的第一次同步。
## 1 主库配置
```ini
# 开启日志
log-bin = mysql-bin
# 设置服务id，主从需要有不同的 id
server-id = 1
# 设置需要同步的数据库
binlog-do-db = course_db
# 屏蔽系统库同步
binlog-ignore-db = mysql 
binlog-ignore-db = information_schema
binlog-ignore-db = performance_schema
```
## 2 从库配置
```ini
# 开启日志
log-bin = mysql-bin
# 设置服务id，主从需要有不同的 id
server-id = 2
# 设置需要同步的数据库
replicate_wild_do_table = course_db.%
# 屏蔽系统库同步
replicate_wild_ignore_table = mysql.%
replicate_wild_ignore_table = information_schema.%
replicate_wild_ignore_table = performance_schema.%
```

## 3 确认主节点的文件名以及位点
```ini
# 确认文件名以及位点
SHOW MASTER status;
```
![[确认主节点的文件名以及位点.png]]
## 4 主从同步配置
```ini
# 切换到从服务器
mysql -udb_sync -p123456
# 先停止同步
STOP SLAVE;
# 修改从库指向主库，使用上面得到的位点
CHANGE MASTER TO
master_host = 'localhost',
master_user = 'root',
master_password = 'my-secret-pw',
master_log_file = 'mysql-bin.000003',
master_log_pos = 2433;
# 启动同步
START SLAVE;
# 查看从库相关状态，
SHOW SLAVE status;
```
当使用 `SHOW SLAVE status;` 查看从库状态的时候，字段 `Slave_IO_Running` 和 `Slave_SQL_Running` 都为 Yes 的时候，则说明成功了。
如果进行上述配置之前从库有主库指向的话，需要先执行清空指令。
```sql
STOP REPLICA FOR CHANNEL '';  
RESET SLAVE all;
```
如果想创建额外的账户来配置主从的话，可以参考如下的命令：
```sql
-- 查看用户配置
SELECT * FROM mysql.user;

-- 创建用户
CREATE USER '123'@'%' IDENTIFIED BY '123456';

-- 赋权语句
GRANT ALL ON *.* TO 'test_user'@'%';

-- 删除用户
DROP USER '123'@'%', 'test_user'@'%';
```