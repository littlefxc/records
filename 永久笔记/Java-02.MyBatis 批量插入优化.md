---
title: MyBatis 批量插入优化
status: Done
Tags:
  - Java
  - MyBatis
  - 批处理
---
# MyBatis 批量插入优化

> 转载自 https://mp.weixin.qq.com/s/bo_Agw4dP7ZXLEl1fcKtIA

# 正文

近日，项目中有一个耗时较长的Job存在CPU占用过高的问题，经排查发现，主要时间消耗在往MyBatis中批量插入数据。

mapper configuration是用foreach循环做的，差不多是这样。（由于项目保密，以下代码均为自己手写的demo代码）

```xml
<insert id="batchInsert" parameterType="java.util.List">
    insert into USER (id, name) values
    <foreach collection="list" item="model" index="index" separator=",">
        (#{model.id}, #{model.name})
    </foreach>
</insert>

```

这个方法提升批量插入速度的原理是，将传统的：

```sql
INSERT INTO `table1` (`field1`, `field2`) VALUES ("data1", "data2");
INSERT INTO `table1` (`field1`, `field2`) VALUES ("data1", "data2");
INSERT INTO `table1` (`field1`, `field2`) VALUES ("data1", "data2");
INSERT INTO `table1` (`field1`, `field2`) VALUES ("data1", "data2");
INSERT INTO `table1` (`field1`, `field2`) VALUES ("data1", "data2");

```

转化为：

```sql
INSERT INTO `table1` (`field1`, `field2`)
VALUES ("data1", "data2"),
("data1", "data2"),
("data1", "data2"),
("data1", "data2"),
("data1", "data2");

```

在MySql Docs中也提到过这个trick，如果要优化插入速度时，可以将许多小型操作组合到一个大型操作中。

理想情况下，这样可以在单个连接中一次性发送许多新行的数据，并将所有索引更新和一致性检查延迟到最后才进行。

**乍看上去这个foreach没有问题，但是经过项目实践发现，当表的列数较多（20+），以及一次性插入的行数较多（5000+）时，整个插入的耗时十分漫长，达到了14分钟，这是不能忍的。**

在资料中也提到了一句话：

> Of course don't combine ALL of them, if the amount is HUGE. Say you have 1000 rows you need to insert, then don't do it one at a time. You shouldn't equally try to have all 1000 rows in a single query. Instead break it into smaller sizes.
> 

它强调，当插入数量很多时，不能一次性全放在一条语句里。可是为什么不能放在同一条语句里呢？这条语句为什么会耗时这么久呢？

**我查阅了资料发现：**

Insert inside Mybatis foreach is not batch, this is a single (could become giant) SQL statement and that brings drawbacks:

- **some database such as Oracle here does not support.**
- **in relevant cases: there will be a large number of records to insert and the database configured limit (by default around 2000 parameters per statement) will be hit, and eventually possibly DB stack error if the statement itself become too large.**

Iteration over the collection must not be done in the mybatis XML. Just execute a simple Insertstatement in a Java Foreach loop. The most important thing is the session Executor type.

```java
SqlSession session = sessionFactory.openSession(ExecutorType.BATCH);
for (Model model : list) {
    session.insert("insertStatement", model);
}
session.flushStatements();

```

Unlike default ExecutorType.SIMPLE, the statement will be prepared once and executed for each record to insert.

从资料中可知，默认执行器类型为Simple，会为每个语句创建一个新的预处理语句，也就是创建一个PreparedStatement对象。

在我们的项目中，会不停地使用批量插入这个方法，而因为MyBatis对于含有`<foreach>`的语句，无法采用缓存，那么在每次调用方法时，都会重新解析sql语句。

> Internally, it still generates the same single insert statement with many placeholders as the JDBC code above.
> 
> 
> MyBatis has an ability to cache PreparedStatement, but this statement cannot be cached because it contains `<foreach />` element and the statement varies depending on the parameters. As a result, MyBatis has to 1) evaluate the foreach part and 2) parse the statement string to build parameter mapping [1] on every execution of this statement.
> 
> And these steps are relatively costly process when the statement string is big and contains many placeholders.
> 
> [1] simply put, it is a mapping between placeholders and the parameters.
> 

从上述资料可知，耗时就耗在，由于我foreach后有5000+个values，所以这个PreparedStatement特别长，包含了很多占位符，对于占位符和参数的映射尤其耗时。并且，查阅相关资料可知，values的增长与所需的解析时间，是呈指数型增长的。

![Number_of_rows_in_a_single_insert](Number_of_rows_in_a_single_insert.png)

所以，如果非要使用 foreach 的方式来进行批量插入的话，可以考虑减少一条 insert 语句中 values 的个数，最好能达到上面曲线的最底部的值，使速度最快。一般按经验来说，一次性插20~50行数量是比较合适的，时间消耗也能接受。

重点来了。上面讲的是，如果非要用`<foreach>`的方式来插入，可以提升性能的方式。而实际上，MyBatis文档中写批量插入的时候，是推荐使用另外一种方法。（可以看 http://www.mybatis.org/mybatis-dynamic-sql/docs/insert.html 中 Batch Insert Support 标题里的内容）

```java
SqlSession session = sqlSessionFactory.openSession(ExecutorType.BATCH);
try {
    SimpleTableMapper mapper = session.getMapper(SimpleTableMapper.class);
    List<SimpleTableRecord> records = getRecordsToInsert(); // not shown

    BatchInsert<SimpleTableRecord> batchInsert = insert(records)
            .into(simpleTable)
            .map(id).toProperty("id")
            .map(firstName).toProperty("firstName")
            .map(lastName).toProperty("lastName")
            .map(birthDate).toProperty("birthDate")
            .map(employed).toProperty("employed")
            .map(occupation).toProperty("occupation")
            .build()
            .render(RenderingStrategy.MYBATIS3);

    batchInsert.insertStatements().stream().forEach(mapper::insert);

    session.commit();
} finally {
    session.close();
}

```

即基本思想是将 MyBatis session 的 executor type 设为 Batch ，然后多次执行插入语句。就类似于JDBC的下面语句一样。

```java
Connection connection = DriverManager.getConnection("jdbc:mysql://127.0.0.1:3306/mydb?useUnicode=true&characterEncoding=UTF-8&useServerPrepStmts=false&rewriteBatchedStatements=true","root","root");
connection.setAutoCommit(false);
PreparedStatement ps = connection.prepareStatement(
        "insert into tb_user (name) values(?)");
for (int i = 0; i < stuNum; i++) {
    ps.setString(1,name);
    ps.addBatch();
}
ps.executeBatch();
connection.commit();
connection.close();

```

经过试验，使用了 `ExecutorType.BATCH` 的插入方式，性能显著提升，不到 2s 便能全部插入完成。

总结一下，如果MyBatis需要进行批量插入，推荐使用 `ExecutorType.BATCH` 的插入方式，如果非要使用 `<foreach>`的插入的话，需要将每次插入的记录控制在 20~50 左右。

# ****Mybatis 批处理数据优化注意点****

<aside>
💡 每次向数据库发送的 SQL 语句的条数是有上限的，如果批量执行的时候超过这个上限值，数据库就会抛出异常，拒绝执行这一批 SQL 语句，

</aside>

<aside>
💡 所以我们需要控制批量发送 SQL 语句的条数和频率。

</aside>

# MyBatis源码分析

我们知道每个`SqlSession`都会拥有一个`Executor`对象，这个对象才是执行 SQL 语句的幕后黑手，

我们也知道Spring跟Mybatis整合的时候使用的`SqlSession`是`SqlSessionTemplate`，默认用的是`ExecutorType.SIMPLE`，这个时候你通过自动注入获得的Mapper对象其实是没有开启批处理的；

```java
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
```

那么我们实际上是需要通过`sqlSessionFactory.openSession(ExecutorType.BATCH)`得到的`sqlSession`对象（此时里面的`Executor`是`BatchExecutor`）去获得一个新的Mapper对象才能生效！！！

# 示例

```java
package com.new3s.gasanalysis.common.util;

import com.new3s.gasanalysis.common.exception.CoreServiceException;
import org.apache.ibatis.session.ExecutorType;
import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactory;
import org.springframework.stereotype.Component;
import org.springframework.transaction.support.TransactionSynchronizationManager;

import javax.annotation.Resource;
import java.util.List;
import java.util.function.BiFunction;

/**
 * @Description mybatis批处理数据工具类
 * @Date 2022-06-09 14:37
 * @Author xie
 */
@Component
public class MybatisBatchUtils {

    /**
     * 每次处理1000条
     */
    private static final int BATCH_SIZE = 1000;

    @Resource
    private SqlSessionFactory sqlSessionFactory;

    /**
     * 批量处理修改或者插入
     *
     * @param data     需要被处理的数据
     * @param mapperClass  Mybatis的Mapper类
     * @param function 自定义处理逻辑
     * @return int 影响的总行数
     */
    public <T,U,R> int batchUpdateOrInsert(List<T> data, Class<U> mapperClass, BiFunction<T, U, R> function) {
        int i = 1;
        // SqlSession 修改为批处理类型（获得一个新的Mapper对象才能生效）
        SqlSession batchSqlSession = sqlSessionFactory.openSession(ExecutorType.BATCH);
        try {
            U mapper = batchSqlSession.getMapper(mapperClass);
            int size = data.size();
            for (T element : data) {
                function.apply(element,mapper);
                if ((i % BATCH_SIZE == 0) || i == size) {
                    batchSqlSession.flushStatements();
                }
                i++;
            }
            // 非事务环境下强制commit，事务情况下该commit相当于无效
            batchSqlSession.commit(!TransactionSynchronizationManager.isSynchronizationActive());
        } catch (Exception e) {
            batchSqlSession.rollback();
            throw new CoreServiceException();
        } finally {
            batchSqlSession.close();
        }
        return i - 1;
    }

}
```

**调用（单纯为了演示）**

```java
@Autowired
private MybatisBatchUtils mybatisBatchUtils;

@Override
@Transactional(rollbackFor = {Exception.class, RuntimeException.class})
public ResMsg<StationVo> addStation(StationVo stationVo) throws IOException, InterruptedException {

    // 测试数据，实际情况可能会从文本中读取
    List<StationVo> stationList = getStationList(StationDto.builder().stationName(stationVo.getStationName()).build()).getData();

    mybatisBatchUtils.batchUpdateOrInsert(stationList, StationMapper.class, (item, stationMapper) -> stationMapper.insert(item));
    
    return ResMsg.success();
}
```

****附：Oracle批量插入优化****

我们都知道Oracle主键序列生成策略跟MySQL不一样，我们需要弄一个序列生成器，这里就不详细展开描述了，然后Mybatis Generator生成的模板代码中，insert的id是这样获取的

```xml
<selectKey keyProperty="id" order="BEFORE" resultType="java.lang.Long">
  select XXX.nextval from dual
</selectKey>
```

如此，就相当于你插入1万条数据，其实就是insert和查询序列合计预计2万次交互，耗时竟然达到10s多。我们改为用原生的Batch插入，这样子的话，只要500多毫秒，也就是0.5秒的样子

```xml
<insert id="insert" parameterType="user">
    insert into table_name(id, username, password)
    values(SEQ_USER.NEXTVAL,#{username},#{password})
</insert>
```

# **参考资料**

1. **https://dev.mysql.com/doc/refman/5.6/en/insert-optimization.html**
2. **https://stackoverflow.com/questions/19682414/how-can-mysql-insert-millions-records-fast**
3. **https://stackoverflow.com/questions/32649759/using-foreach-to-do-batch-insert-with-mybatis/40608353**
4. **https://blog.csdn.net/wlwlwlwl015/article/details/50246717**
5. **http://blog.harawata.net/2016/04/bulk-insert-multi-row-vs-batch-using.html**
6. **https://www.red-gate.com/simple-talk/sql/performance/comparing-multiple-rows-insert-vs-single-row-insert-with-three-data-load-methods/**
7. **https://stackoverflow.com/questions/7004390/java-batch-insert-into-mysql-very-slow**
8. **http://www.mybatis.org/mybatis-dynamic-sql/docs/insert.html**
9. [Mybatis 批处理数据优化 - xiexie0812 - 博客园 (cnblogs.com)](https://www.cnblogs.com/mask-xiexie/p/16359359.html)