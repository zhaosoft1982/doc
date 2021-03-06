#### 1、乐观锁和悲观锁的区别

- 乐观锁（Optimistic Lock）：顾名思义，就是很乐观，每次去拿数据的时候都会认为别人不会修改，所以不会上锁，但是在更新的时候会去判断在此期间别人有没有去更新这个数据，可以使用版本号等机制。
- 乐观锁适用于多读的应用类型，这样可以提高吞吐量。
- 悲观锁（Pessimistic Lock）：顾名思义，就是很悲观，每次去拿数据的时候都会认为别人会修改，所以在每次那数据的时候都会上锁，这样别人想拿到数据就会block知道它拿到锁。
- 传统的关系型数据库里面就用到了很多这种锁机制，比如行锁、表锁、读锁、写锁等，都是在操作之前先上锁。

#### 2、事务特性

- 原子性（Atomicity）：强调事务的不可分割
- 一致性（Consistency）：事务在执行的前后，数据的完整性保持一致
- 隔离性（Isolation）：多个事务并发执行的时候，一个事务的执行不应该受到其他事务的影响
- 持久性（Durability）：事务结束后，数据就永远保存到数据库中。

#### 3、不考虑事务的隔离性引发的一些列安全问题

- 脏读：一个事务读到另一个事务未提交的数据
- 不可重复读：一个事务读到了另一个事务已经提交的update数据，导致在一个事务中多次查询结果不一致。
- 虚读/幻读：一个事务读到了另一个事务已经提交的insert数据，导致在一个事务中多次查询结果不一致。

#### 4、数据库提供了隔离级别用于解决上面三个问题

- read uncommitted：未提交读。
  - 脏读、不可重复读、虚读都是可能发生的
- read committed：已提交读。
  - 避免脏读，但是不可重复读、虚读是有可能发生的
- repeatable read：可重复读
  - 避免脏读、不可重复读，但是虚读有可能发生
- serializable：串行化
  - 避免脏读、不可重复读、虚读的发生
- **安全性**
  - serializable > repeatable read > read committed > read uncommitted
- **效率性**
  - read uncommitted > read committed  > repeatable read > serializable

#### 5、mysql常用的数据库引擎

![img](https://img-blog.csdnimg.cn/20190221091906149.png)

- InnoDB：支持事务处理，支持外键，支持崩溃修复能力和并发控制。InnoDB是提供了提交、回滚和崩溃恢复能力的事务安全存储引擎，支持行锁定和外键，是mysql的默认存储引擎。
- MyISAM：插入数据块，空间和内存使用比较低。MyISAM不支持事务，插入和查询效率高，支持索引
- MEMORY：所有的数据都在内存中，数据的处理速度快，但是安全性不高。

#### 6、sql优化

- **insert优化**

  - 尽量使用批量插入操作
  - 多次单条数据插入，在事务中进行数据插入
  - rewriteBatchedStatements = true，mysql默认关闭了batch处理，通过此参数进行打开

- **order by 优化**

  - 尽量减少额外排序通过索引直接返回有序数据，通过建立合适的索引减少Filesort的出现
  - mysql两种排序算法
    - **两次扫描算法**：首先根据条件取出排序字段和指针信息，在排序区中排序，完成排序后，根据行指针回表读取记录。
    - **一次扫描算法**：一次性取出满足条件的所有字段在排序区排序后直接输出结果。
  - 可以提高**sort_buffer_size**和**max_length_for_sort_data**系统变量，增大排序区大小，调高排序效率

- **group by 优化**

  - group by 会进行额外的排序操作

- **查询优化**

  - 对查询进行优化，要尽量避免全部扫描，首先应考虑在where及order by涉及的列上建立索引

  - 应尽量避免在where子句中对字段进行null值判断，否则景导致引擎放弃使用索引而进行全表扫描，最好不要给数据留null，尽可能的使用NOT NULL 填充数据库。

  - 应尽量避免在where资金中使用！=或<>操作符，否则引擎将放弃使用索引而进行全表扫描

  - 应尽量避免在where子句中使用or来连接条件，如果一个字段有索引，一个字段没有索引，将导致引擎放弃使用索引而进行全表扫描。

  - in 和 not in 也要慎用，否则和导致全表扫描

  - 创建合适的索引

  - like语句优化

    - select id from a where name like '%abc%';   由于abc前面用了“%”，因此该查询必然走全表查询，除非必要，否则不要在关键词前面加%。

  - 尽量用内连接替换外连接，用外连接替换子查询

  - **分页优化**

    - **在索引上完成排序操作，在根据索引关联其他内容**

    - ```sql
      select * 
          from tb_item t,(select id from tb_item order by id limit 2000000,10) a
      where 
          t.id=a.id
      ```

#### 7、oracle 数据库分页

```sql
select t1.* from 
(select rownum rw, k.* from ks_collect_pushed k
where rownum < = 15) t1 where rw > 5
```

#### 8、mysql索引分类

- 普通索引：允许列中插入重复值和空值
- 唯一索引：索引列的值必须唯一，但允许有空值
- 主键索引：索引列的值必须是唯一，且不允许有空值
- 联合索引
  - 联合索引又叫复合索引，对于符合索引：Mysql从左到右的使用索引中的字段，一个查询可以只使用索引中的一部分，但只能是最左侧部分。例如索引是key index（a, b,c），可以支持a| a,b| a,b,c| 3中组合进行查找，但不支持b,c进行查找，当最左侧字段是常量引用时，索引就十分有效。
- 全文索引
  - 全文索引类型FULLTEXT，可以在CHAR、VARCHAR、TEXT上创建，只有MYISAM存储引擎支持
- 空间索引
  - 空间索引是对空间数据类型的字段建立的索引，mysql中空间数据类型有4种，分别是：GEOMETRY、POINT、LINESTRING、PLYGON。mysql使用SPATIAL创建索引，空间索引的列为NOTNULL

#### 9、MySQL存储引擎MyISAM和InnoDB区别

- 事务支持
  - MyISAM不支持事务，而InnoDB支持事务
- 存储结构
  - MyISAM：每个MyISAM在磁盘上存储成三个文件。第一个文件的名字以表的名字开始，扩展名指出文件类型。.frm文件存储表定义。数据文件的扩展名为.MYD(MYData)。索引文件的扩展名是.MYI(MYIndex)。
  - InnoDB：所有的表都保存在同一个数据文件中（也可能是多个文件，或者是独立的表空间文件），InnoDB表的大小只受限于操作系统文件的大小，一般为2GB
- 存储空间
  - MyISAM：可被压缩，存储空间较小。支持三种不同的存储格式：静态表（默认，但是注意数据表尾不能有空格，会被去掉）、动态表、压缩表。
  - InnoDB：需要更多的内存和存储，它会在主内存中建立其专用的缓存池用于高速缓冲数据和索引。
- 可移植性、备份及恢复
  - MyISAM：数据是以文件的形式存储，所以在跨平台的数据转移中会很方便。在备份和恢复时可单独针对某个表进行操作。
  - InnoDB：免费的方案可以是拷贝数据文案、备份binlog，或者用mysqldump，在数据量达到几十G的时候就相对痛苦了
- AUTO_INCREMENT
  - MyISAM：可以和其他字段一起建立联合索引。引擎的自动增长列必须是索引，如果是组合索引，自动增长可以不是第一列，他可以根据前面几列进行排序后递增。
  - InnoDB：InnoDB中必须包含只有该字段的索引。引擎的走动增长列必须是索引，如果是组合索引页必须是组合索引的第一列。
- 表锁差异
  - MySIAM：只支持表锁。用户在操作MySIAM表时，select、update、delete，都会给表自动加锁，如果加锁以后的表满足insert并发的情况下，可以在表的尾部插入新的数据。
  - InnoDB：支持事务和行级锁，是InnoDB的最大特色。行锁大幅度提高了多用户并发操作的性能，但是InnoDB的行锁，只是在WHERE的主键是有效的，非主键的WHERE都会锁全表。
- 全文索引
  - MyISAM：支持全文索引
  - InnoDB：不支持全文索引，但是InnoDB可以使用sphinx插件支持全文索引，并且效果更好。
- 表主键
  - MySIAM：允许没有任何索引和主键的表存在，索引都是保存行的地址。
  - InnoDB：如果么有设定主键或者非空唯一索引，机会自动生成一个6字节的主键（用户不可见），数据是主索引的一部分，附件索引保存的是主索引的值。InnoDB的主键范围更大，最大是MySIAM的2倍。
- 表的具体行数
  - MySIAM：保存有表的总行数
  - InnoDB：没有保存表的总行数
- 外键
  - MySIAM：不支持
  - InnoDB：支持

#### 10、索引

- 什么是索引？
  - 索引是一种数据结构，方便我们快速的进行数据的查找。
- 索引是个什么样的数据机构？
  - 索引的数据结构和具体的存储引擎实现有关，在MySQL中使用较多的索引又Hash索引和B+树索引等。InnoDB存储引擎默认索引实现为：B+树索引。
- Hash索引和B+树索引有什么区别？或者劣势？
  - 实现原理
    - hash索引底层就是hash表，进行查找时，调用一次hash函数就可以获取到相应的键值，之后进行会标查询获得实际数据。
    - B+树底层实现时多路平衡查找树，对于每一次的查询都是从根节点出发，查找到叶子节点方可以获得所查键值，然后根据查询判断是否需要回表查询数据。
  - 区别
    - hash索引进行等值查询更快（一般情况下），但是却无法进行范围查询
    - hash索引不支持使用索引进行排序
    - hash索引不支持模糊查询及多列索引的最左前缀匹配
    - hash索引任何时候都避免不了回表查询数据
    - B+树索引默认索引有序
    - B+树索引支持模糊查询和多列索引最左前缀匹配
    - B+树索引在符合某些条件（聚簇索引）时，可以不用回表查询数据。
    - B+树索引查询效率比较稳定。
- 聚簇索引和非聚簇索引
  - 在InnoDB中，只有主键索引是聚簇索引，如果没有主键，则挑选一个唯一索引建立聚簇索引，如果没有唯一键，则隐式的生成一个键来建立聚簇索引。
  - 在B+树的索引中，叶子节点可能存储了当前的key值，也可能存储了当前的key值以及整行的数据，这就是聚簇索引和非聚簇索引。
  - 聚簇索引的优点
    - 聚簇索引将索引和数据行保存在同一个B+树的叶子节点中，查询通过聚簇索引可以直接获取数据，相比非聚簇索引需要二次查询（非覆盖索引的情况下）效率要高。
    - 聚簇索引对范围查询的效率很高，因为其数据是按照大小排列的
  - 聚簇索引的缺点
    - 在插入新行和更新主键时，可能到值“页分裂”问题
    - 可能导致全表扫描速度变慢，因为可能需要加载物理上相隔较远的页到内存中（需要耗时的磁盘寻道操作）
- 联合索引是什么？为什么需要注意联合索引中的顺序？
  - Mysql可以使用多个字段同时建立一个索引，叫做联合索引。在联合索引中，如果想要命中索引，需要按照建立索引时的字段顺序挨个使用，否则无法命中索引。
  - 假设现在建立了“name,age,school”的联合索引，那么索引的排序为：先按照name排序，如果name相同，则按照age排序，如果age相同，则按照school进行排序。
- 创建的索引没有被使用到？或者说怎样可以知道这条语句运行很慢的原因？
  - MySQL提供了explain命令来查看语句的执行计划。MySQL在执行某个语句之前，会将该语句过一遍查询优化器之后会拿到对语句的分析，也就是执行计划，其中包含了许多信息。可以通过其中和索引有关的信息来分析是否命中索引。
- 在哪些情况下会发生针对该列创建了索引但是查询时没有使用呢？
  - 使用不等于查询
  - 列参与了数据运算或者函数
  - 在字符串like是左边的通配符，类似于“%aaa”
  - 当mysql分析全表扫描使用索引快的时候不适用索引
  - 当使用联合索引，前面一个条件为范围查询，后面的即使符合最左前缀原则，也无法使用索引。

#### 11、为什么MySQL的索引要使用B+树而不是其他树形结构？

- 因为B+树中只有叶子节点会保存数据，非叶子节点只会保存键值和指针。尽量减少查询过程中I/O的存取次数。

#### 12、InnoDB的一棵B+树可以存放多少行数据?

- 约2千万