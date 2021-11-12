# JD_Internship_Knowledge_record

## 数据仓库

### 数据仓库位置

![数据仓库位置](/Users/chenli75/Desktop/MyFile/github/MyKnowledgeRepository/internship_picture/数据仓库位置.png)

### 数据仓库分层架构

数据流向是：业务系统日志数据 -> BDM -> FDM -> GDM -> ADM -> (DIM层、TMP层) -> APP层 -> BI 

为什么数据仓库要分层设计？利于回溯查询。比如在上层应用中缺少某个维度的数据，就可以回溯到最原始的数据层进行提取。

![分层架构](/Users/chenli75/Desktop/MyFile/github/MyKnowledgeRepository/internship_picture/分层架构.png)



**分层说明**

BDM（缓冲数据层）：源业务系统的快照、会保留一段时间。一个BDM对应业务系统的一个表或者一个日志文件。处理成 dt(日期) dh(小时) data(json格式数据)。以dt和ht为分区

FDM（基础数据层）：从BDM层中提取出基础的字段信息，以dt为分区。

GDM（通用数据层）：更详细的字段信息，扩展FDM层的数据维度，是一个宽表。其中每一列代表一个纬度

ADM（聚合数据层）：对GDM中的表做聚合，即进行join，聚合后的每一列数据代表一个指标，或者度量

![分层说明](/Users/chenli75/Desktop/MyFile/github/MyKnowledgeRepository/internship_picture/分层说明.png)



### 数据仓库设计流程

![数据仓库设计流程](/Users/chenli75/Desktop/MyFile/github/MyKnowledgeRepository/internship_picture/数据仓库设计流程.png)



### 数据仓库设计-FDM层

设计原则：保留历史数据的同时，尽可能少的占用存储空间快速、高效的获取历史上任意一天的快照数据 

设计方法：通过拉链表实现，经比较记录数据的全生命周期，可快速还原任意天的历史快照，同时极大的节省空间，其中模型带“_chain”为拉链表（归档）、不带为流水表



**概念解析**

拉链表：所谓拉链就是记录历史，记录一个事物从开始一直到当前状态的所有变化的信息。链表中有开始时间（start_time）和结束时间（end_time）两个字段，同时有dt和dp两个分区字段，end_time也可做分区；

 start_time:数据开始的时间

end_time:数据结束的时间

dp:一般有ACTIVE（线上）和EXPIRED（过期）两个分区。ACTIVE表示数据当前在线上使用，所以其end_time为4712-12-31（系统能处理的最大时间，表示一个达不到的无限向后延伸的时间，意味着该数据在线上永久有效）；EXPIRED表示数据过期，已变更，为历史状态，其end_time是数据变更时具体的时间。

dt:数据所在的时间分区，记录数据从ACTIVE转移到EXPIRED的日期，即数据发生变更的时间，大部分与end_time一致。拉链表不建议使用dt分区， 标黄为变更前数据，标红为变更后数据

标黄为变更前数据，标红为变更后数据

![拉链表](/Users/chenli75/Desktop/MyFile/github/MyKnowledgeRepository/internship_picture/拉链表.png)

查询问题：如还原2020-06-02的历史快照，使用end_time> ‘2020-06-02’ and start_time<= ‘2020-06-02’ 查询，end_time过滤2020-06-02之前的旧数据，start_time过滤2020-06-02之后的新数据。 

sysdate(-1)表示昨天的日期

拉链表查询方式：

![拉链表查询方式](/Users/chenli75/Desktop/MyFile/github/MyKnowledgeRepository/internship_picture/拉链表查询方式.png)



流水表：流水表存放的是一个用户的变更记录，比如在一个张流水表中，一天的数据，会存放一个用户的每条修改记录，但是在拉链表中只有一条记录

![流水表](/Users/chenli75/Desktop/MyFile/github/MyKnowledgeRepository/internship_picture/流水表示意图.png)



### 数据仓库设计-GDM层

GDM层的表数据是一个宽表，就是字段比较多的数据库表。宽表已经不符合三范式的规范，存在大量数据冗余，好处就是查询性能提高了。窄表是按照数据库三范式，减少了数据冗余，但缺点就是修改一个数据可能需要修改多张表。 

**设计原则**

数据通用性：模型设计的时候，尽可能的将各业务部常用、通用行较高的字段进行统一处理

主题建模：按照主题进行建模

统计周期：模型可区分全量表、增量表，其中模型带“_da”为全量表，否则为增量表规范易懂：命名规范 设计方法：业务分析：分析各个部门的需求

 **概念解析**

增量表：记录更新周期内的数据，即在原表的基础上新增本周期内产生的数据。比如按天更新的流量表，每次更新只新增一天内产生的新数据。注意，每次新产生的数据是以最新分区增加到表中，原先数据依然存在表中。

![增量表](/Users/chenli75/Desktop/MyFile/github/MyKnowledgeRepository/internship_picture/增量表.png)

![增量表2](/Users/chenli75/Desktop/MyFile/github/MyKnowledgeRepository/internship_picture/增量表2.png)

全量表：每次记录都会记录全量数据。包括原全量数据和本次新增数据。注意，全量表中每个分区内都是截至分区时间的全量数据，原先分区的数据依然存在于表中，只是每次更新会在最新分区内再更新一遍全量数据。

![全量表](/Users/chenli75/Desktop/MyFile/github/MyKnowledgeRepository/internship_picture/全量表.png)

![全量表2](/Users/chenli75/Desktop/MyFile/github/MyKnowledgeRepository/internship_picture/全量表2.png)



### 用户主题模型设计

![用户主题模型设计](/Users/chenli75/Desktop/MyFile/github/MyKnowledgeRepository/internship_picture/用户主题模型设计.png)



## Hive优化

### 优化的根本思想：

1、尽早过滤数据，减少每个阶段的数据量。数据量少了，减少了网络传输

2、减少job数。所谓的job就是你提交一个复杂的SQL后，hive会把复杂的sql转换成若干个mapreduce的job任务，每个job对应你sql中的部分逻辑，每个job之间任务是独立的。任务是独立的，但数据是依赖的。就是下一个job依赖上一个job的数据文件，上一个job任务执行完会落盘到hdfs上，你job越多，落盘次数也越多，磁盘IO也越多。

3、解决数据倾斜

### 优化方法

1、列裁剪。hive在读取数据的时候，只读取查询中所需要的列，而忽略其他列。

2、分区裁剪。在查询过程中减少不必要的分区。如果所需要的数据在某一个分区，就指定那个分区进行查询。那怎么知道查询的数据在哪个分区？通过Explain dependency获取。如Explain dependency select count(*) from table where dt>="2014-06-01"

3、利用hive的优化机制减少job数。不论是外关联outer join还是内关联inner join，如果join的key相同，不管有多少个表，都会合并为一个MapReduce任务。

```sql
1个job
select a.val,b.val,c.val 
from a join b on (a.key=b.key1) 
join c on (c.key=b.key1)

2个job
select a.val,b.val,c.val 
from a join b on (a.key=b.key1) 
join c on (c.key=b.key2)

```

4、善用muti-insert、unoin all，不同表的union all相当于多次输入，同一个表的union all，相当于map一次输出多条

```sql
1、相当于查询了表a两次
insert overwriter table tmp1
  select ... from a where 条件1
insert overwriter table tmp2
  select ... from a where 条件2
  
2、相当于查询了表a一次
from a
  insert overwriter table tmp1
    select ...  where 条件1
  insert overwriter table tmp2
    select ...  where 条件2

```

5、避免笛卡尔积。在两个表进行关联时，写上on关联条件

6、在join前过滤掉不需要的数据。

```sql
1、优化前
select a.val,b.val from a left outer join b on (a.key=b.key)
where a.ds='2009-07-07'and b.ds='2009-07-07'

2、优化后
select x.val,y.val from
(select key,val from a where a.ds='2009-07-07') x
left outer join
(select key,val from b where b.ds='2009-07-07') y
on x.key=y.key

```

7、小表放前大表放后原则。在编写带有join操作的代码语句时，应该将条目少的表放在join操作符的左边。因为Reduce阶段，位于join操作符左边的表的内容会被加载进内存，载入数据量较小的表可以有效减少OOM即内存溢出。

8、mapjoin()。当小表与大表join时，采用mapjoin，即在map端完成。同时可以避免小表与大表join产生的数据倾斜

```sql
select /*+ mapjoin(b)*/ a.key,a.value
from a join b
on a.key=b.key

b是小表

```

9、去重优化。尽量避免使用distinct进行去重，特别是大表操作，使用group by代替。

```sql
1、select distinct key from a
如果a表数据特别庞大的话，distinct key很容易造成数据倾斜，就是key一样的话他会放到
同一个mapreduce进行处理

2、select key from a group by key
效果是等价的，但效率比较高。因为group by可以避免数据倾斜
```

10、排序优化。

只有order by产生的结果是全局有序的，可以根据实际场景进行选择排序

- order by 实现全局排序，一个reduce实现，由于不能并发执行，所以效率偏低

- sort by 实现部分有序，单个reduce输出的结果是有序的，效率高。通过和distribute by关键字一起使用（distribute by关键字可以指定map到reduce端的分发key）

- cluster by col1等价于distribute by col1 sort by col1，但不能指定排序规则 如何判断出现数据倾斜：任务进度长时间维持在99%，查看任务监控画面，发现只有少量reduce子任务未完成。因为其处理的数据量和其他reduce差异过大





## Hive函数相应用法

Hive常用数据类型：String、Int、Double

类型转换操作：cast(字段名 or 具体值 as 指定类型)如：select cast(1 as double)from table; 

explode(myCol) 行转列函数，把一行转化为一列



### Hive分区表

所谓的分区表就是将数据以一种方式进行组织，比如分层存储。简单理解就是分区就是目录，建立一个目录进行存储。

分区表分为静态分区和动态分区。 

所谓静态分区就是自己手动设置分区字段。什么时候需要动态分区？加入中国有50个省份，每个省有50个市，每个市都有100个区，如果使用静态分区就不知道什么时候搞完，工作量也大，此时就需要动态分区。

 静态分区的值必须在动态分区值的前面



## 知识盲区

#### 参数设置

##### spark

spark.rdd.compress

这个参数决定了RDD Cache过程中，RDD数据在序列化之后是否进一步进行压缩再存储到内存或磁盘上。

##### hive

set hive. exec.dynamic.partition =true.   开启hive动态分区表

set hive.exec.dynamic.partition.mode = nonstrict.   开启允许所有分区都是动态的，否则必须要有静态分区才能用

#### Hive SQL

from_unixtime(timestamp,format)：即将时间戳转换为日期类型进行显示  

cast(expression AS data_type)：将某种数据类型的表达式转换为另一种数据类型 

nvl(表达式1，表达式2)：空值转换函数。其表达式的值可以是数字型、字符型和日期型。但是表达式1和表达式2的数据类型必须为同一个类型。对数字型： NVL（ comm,0);对字符型 NVL( TO_CHAR(comm), 'No Commission')对日期型 NVL（hiredate,' 31-DEC-99')

NVL2(表达式1，表达式2，表达式3）：如果表达式1为空，返回值为表达式3的值。如果表达式1不为空，返回值为表达式2的值

 lag() over()与lead() over()函数是跟偏移量相关的两个分析函数

lag()：可以在一次查询中取出同一字段的前N行数据

lead()：可以在一次查询中取出同一字段的后N行数据

例如：这是原始数据表

![原始数据表](/Users/chenli75/Desktop/MyFile/github/MyKnowledgeRepository/internship_picture/lag()前原始数据表.png)

获取当前记录的ID以及下一条记录的ID

```sql
select t.id id ,
       lead(t.id, 1, null) over (order by t.id)  next_record_id, 
       t.cphm
from tb_test t       
order by t.id asc

```

![lead()函数数据](/Users/chenli75/Desktop/MyFile/github/MyKnowledgeRepository/internship_picture/lead()函数数据.png)

获取当前记录的ID以及上一条记录的ID

```sql
select t.id id ,
       lag(t.id, 1, null) over (order by t.id)  next_record_id,
       t.cphm
from tb_test t       
order by t.id asc

```

![lag()函数数据](/Users/chenli75/Desktop/MyFile/github/MyKnowledgeRepository/internship_picture/lag()函数数据.png)

获取号码牌号码相同，获取当前记录的ID以及下一条记录的ID（使用partition by）

```sql
select t.id id, 
       lead(t.id, 1, null) over(partition by cphm order by t.id) next_same_cphm_id, 
       t.cphm
from tb_test t
     order by t.id asc 

```

![lead()partition by](/Users/chenli75/Desktop/MyFile/github/MyKnowledgeRepository/internship_picture/lead()partition by数据.png)

insert into 与 insert overwriter的异同

insert into与insert overwriter都可以向hive表中插入数据，但是insert into直接追加到表中数据的尾部；insert overwriter会重写数据。



like与rlike的区别

like的内容不是正则，而是通配符。

两个模式：_和%_

：表示单个字符，用来查询定长的数据

%：表示0个或者多个任意字符

rlike是正则，正则写法和java一样。'\'需要使用'\\'

.：匹配任何单字符

*：匹配前面表达式零次或多次

+：匹配前面表达式一次或多次

？：匹配前面表达式零次获一次



union与union all

union用于把来自多个select语句结果组合到一个结果集合中，语法为：

```sql
select  column,......from table1
union [all]
select  column,...... from table2
```



With table as :创建一个临时表，放在select 语句前面

语法：

```sql
with temptable as(select ...)
select ...
```

这个临时表只能用于查询，不能用于更新



Get_json_object(string json_string,string path):用于解析json字符串，支持多层嵌套解析

第一个参数是要解析的字符串

第二个参数是要获取的key,第二个参数使用$表示json变量标识，然后用.或者[]读取对象或数组



Case when then else end多条件判断

```sql
SELECT
    STUDENT_NAME,
    (CASE WHEN score < 60 THEN '不及格'
        WHEN score >= 60 AND score < 80 THEN '及格'
        WHEN score >= 80 THEN '优秀'
        ELSE '异常' END) AS REMARK
FROM
    TABLE
```



count(常量)、count(*)、count(列名)的区别

count(常量)、count(*)表示直接查询符合条件的数据库表的行数

count(列名)表示查询符合条件的并且列的值不为null的行数

count( * )和count(1)是实现上没有任何区别，效率一样，但还是推荐使用count(*)

Count(列名)需要对字段进行非null判断，效率会低一些



group by、order by后面跟数字指的是 select后面选择的列，1代表第一个列



truncate与drop，delete对比

truncate的作用是清空表，只能作用于表。它会清空表中的所有行，但表结构及其约束



#### Java

Calendar类

作用：为操作日历字段提供一些方法

构造方法：protected Calendar() :由于修饰符是protected，所以无法直接创建该对象。需要通过别的途径生成该对象。通过Calendar.getInstance()生成实例对象

方法：

setTime(Date date):使用给定的Date设置此日历的时间

getTime():返回一个Date表示此日历的时间

addTime():按照日历规则，给指定字段添加或减少时间



String.format()字符串格式化



#### Scala

scala中使用"""创建多行字符串，这样的话，多行的字符前面都会有空格。可以在多行字符的末尾stripMargin，并且在第二行开始的每一行前面添加上｜。

Scala 字符串中取变量的值，在变量字符前面加$，${}可以嵌入表达式。



List/ListBuffer

不可变List：长度内容不可变

可变ListBuffer：长度内容都可变，必须导入包



#### Spark

Row的数值获取

Row.getAs [T] ("字段" )

Row.getString(index)



spark.sql(sql)返回的是RDD类型的数据









## 埋点





## Presto

### 定义

Presto是一个开源的分布式查询引擎，适用于交互式分析查询，数据量支持GB到PB级别。它本身不存储数据，但他可以接入多种数据源，比如Hadoop、hive、kafka、redis、mysql等。

它的作用：用于交互式分布式查询

### 工作原理

Presto是在Hadoop上运行的分布式系统，也是采用大规模并行处理相似的架构。它有一个协调节点，与多个工作线程节点同步工作。用户将SQL提交给协调器，由其查询和执行引擎进行解析、计划并将分布式查询计划安排到工作线程节点之间。

所有处理都在内存进行，添加更多工作线程节点可提高并行能力，并加快处理速度。

为了使它可扩展到任何数据源，它的设计采用了抽象存储化，可轻松构建连接器。

数据在其存储位置接受查询，无需将其移动到独立的分析系统中。

### 使用场景

1、mysql跨数据库查询。Presto支持扩展任何数据源

2、数据仓库表的查询。它是基于内存的查询，速度快

### 为什么Presto查询速度比Hive快？

Presto是常驻任务，接受请求立即执行，全内存计算并行计算。

Hive需要使用yarn做资源调度，查询前需要先申请资源，启动进程，并且中间结果会经过磁盘。



## 沟通

如何进行需求沟通？5W2H

5W

1、why-为什么？为什么要这么做？原因是什么？

2、what-是什么？目的是什么？指标是什么？

3、where-何处？在哪里做？从哪里下手？

4、when-何时？什么时间完成？

5、who-谁？谁来完成？谁负责？

2H

1、how-怎么做？如何实施？方法是怎样的？

2、how much-多少？做到什么程度？想要的效果是怎么样的？



## 总结模板

1、背景（为什么做）=》往业务挂靠（解决了什么问题，为什么需要）

2、目标（比如几号上线）

3、进展。比如目前完成了代码评审、代码开发到了哪一步；遇到了什么问题，怎么解决的

4、思考。=》避免出现相同的问题/提升效率

## 各种表的概念

### 一维表和二维表

需要行和列来定位数值的就是二维表，仅靠单行就能锁定全部信息的就是一维表

### 拉链表和流水表

拉链表：所谓拉链就是记录历史，记录一个事物从开始一直到当前状态的所有变化的信息。链表中有开始时间（start_time）和结束时间（end_time）两个字段，同时有dt和dp两个分区字段，end_time也可做分区；

 start_time:数据开始的时间

end_time:数据结束的时间

dp:一般有ACTIVE（线上）和EXPIRED（过期）两个分区。ACTIVE表示数据当前在线上使用，所以其end_time为4712-12-31（系统能处理的最大时间，表示一个达不到的无限向后延伸的时间，意味着该数据在线上永久有效）；EXPIRED表示数据过期，已变更，为历史状态，其end_time是数据变更时具体的时间。

dt:数据所在的时间分区，记录数据从ACTIVE转移到EXPIRED的日期，即数据发生变更的时间，大部分与end_time一致。拉链表不建议使用dt分区



流水表：流水表存放的是一个用户的变更记录，比如在一个张流水表中，一天的数据，会存放一个用户的每条修改记录，但是在拉链表中只有一条记录



### 全量表和增量表

前面有

### 事实表和维度表

维度表：表格里存放了具有独立属性和层次结构的数据，一般有维度编码金和对应的维度说明（标签）。

事实表：表格里存储了能体现实际数据或详细数值，一般由维度编码和事实数据组成。

![魔方-维度表和事实表](/Users/chenli75/Desktop/MyFile/github/MyKnowledgeRepository/internship_picture/魔方-维度表和事实表.jpg)

![维度表和事实表](/Users/chenli75/Desktop/MyFile/github/MyKnowledgeRepository/internship_picture/维度表和事实表.jpg)



## 电商概念

### SPU和SKU

SPU(Standard Product Unit)，标准化产品单元，简称商品。

SKU(Stack Keeping Unit)，库存单位，简称单品。

比如iPhone 8为例，他是一个SPU，是一个商品。但它规格有很多种，比如颜色包含金色、黑色、白色这3种，内存包含32G、64G、128 G这3种，对应9种SKU(3*3)。比如”iPhone 8 白色 128G“，这是一个SKU，它具体到了一个实物。

SPU和SkU是包含关系，一个SPU可以包含若干个SKU，SKU从属于SPU



## 项目

需求简述：小程序上展示屏前客流人次指标，需要大数据将租户设备的屏前客流回写给业务系统。

需求收益：对服务端反向输出屏前客流数据，使数据更准确。



负责内容：

1、屏前客流统计的优化，极大地提高了程序运行效率，加快了统计的速度

2、数据仓库的建设和完善



项目难点：

1、设备id和场景id一直匹配不上。通过数据排查，发现有些设备会出现换绑，或者有些设备是已经弃用了的，所以导致把无用的设备和已经换绑的设备拿来一起计算，导致数据错误。最后的解决办法就是按照最新的日期数据去取设备id。本身屏前客流就是要最新日期的数据，也并不需要历史回刷。

2、统计不同停留时间段的客流人数，一直出现错误。最后通过回查源数据表发现，adm层统计的指标是按照设备、性别、年龄粒度进行聚合的，而需求是按照设备、durn停留时长的聚合数据，所以adm表不能使用，需要使用adm表的上一层数据表gdm表。

3、自定义UDF函数。positionid=2_0_1_6_20,通过split方法将positionid转换成数组，根据不同sourceType进行分别组装。



实现：

1、通过spark sql读取hive表中屏前客流数据和设备数据，读取mysql表中的场景数据，将他们进行join，注册为临时表，按照需求提取相应的曝光人数、触达人次、互动人数的数据。

2、不同时间段的durn进行分组，分组统计人数






