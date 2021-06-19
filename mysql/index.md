## 索引

二叉树是搜索效率最高的，但是随着数据量的增加，会导致二叉树的树高逐步增加，导致在单次搜索中需要访问的节点(数据块)数量过多，增大了搜索时间。因此，MySQL使用"N叉树"。

B+树能够减少单次查询的磁盘访问次数。

### 主键索引和普通(非主键)索引

主键索引B+树的叶子节点是整行数据，普通索引B+树的叶子节点是主键；因此，使用普通索引查询的时候，得到的是主键，需要通过主键"回表"查询以获得整行数据。

### 索引维护

B+树的索引模型，插入一条新数据，会导致B+树的更新，从而可能带来数据的搬移，而MySQL中的数据是分页存储的，数据搬移的过程中，可能导致页分裂或者页合并，这些都是会影响性能。

> 自增主键
>
> 在B+树中，如果节点一直是有序增加的，就一直是追加操作，不会出现搬移的情况，减少了因为页分裂导致的性能问题。

#### B+树对比B树和哈希

哈希：处理范围查询和排序性能差

数据加载方面：B+树的树高较低，减少io

### 覆盖索引

由于普通索引节点存储的是主键，如果需要查询整表的信息，就会涉及到回表的操作：先通过普通索引获得主键，再通过主键索引获得整表数据。

通过对高频查询字段建立联合索引，减少回表操作，可以显著提升查询性能。

### 最左前缀原则

例如联合索引`(a,b,c)`，会按照`a`，`b`，`c`的顺序对索引排序，同时单个字符串索引也会按照索引值的字符串从左向右排序。因此，索引`(a,b,c)`覆盖了`(a,b,c)`，`(a,b)`，`a`三个索引的功能。所以，在建立索引时，要合理安排索引字段的顺序。

### 索引下推

MySQL5.6引入的优化，本质上是为了减少回表的次数，例如有一个联合索引`(name,age)`，当搜索条件为：

~~~mysql
select * from tuser where name like '张%' and age > 10
~~~

搜索过程应该为：根据最左前缀原则，利用联合索引`(name,age)`查找`张`开头的`name`，得到对应的`age`值，直接判断`age > 10`，如果不符合，则丢弃，进入下一次搜索，否则，从该联合索引中获取主键，再回表获取对应的记录。

这样，在联合索引阶段进行判断，减少了不必要的回表操作。
