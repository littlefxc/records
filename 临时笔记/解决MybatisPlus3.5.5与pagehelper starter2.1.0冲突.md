
解决MybatisPlus3.5.5与pagehelper starter2.1.0冲突
项目升级的时候，MybatisPlus与PageHelper又双叕打架了。

mybatis-spring-boot-starter，版本3.5.5
pagehelper-spring-boot-starter，版本2.1.0
原因是它们同时都引用了jsqlparser的依赖，然而，mybatis plus用的是jsqlparser4.6版本，而pagehelper用的是4.7。

那么会出现什么情况？

如果以jsqlparser4.7版本为准，启动项目都起不起来，原因是jsqlparser4.7版本中把版本4.6的一个类被干掉了
如果以jsqlparser4.6版本为准，启动可以成功，但是查询会有问题
原因下面跟大家分析，直接给解决方案：

```xml
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper-spring-boot-starter</artifactId>
    <version>2.1.0</version>
    <exclusions>
    	<exclusion>
    		<groupid>com.github.jsqlparser</groupid>
    		<artifactId>jsqlparser</artifactId>
    	</exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.5.5</version>
    <exclusions>
    	<exclusion>
    		<groupid>com.github.jsqlparser</groupid>
    		<artifactId>jsqlparser</artifactId>
    	</exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>sqlparser4.5</artifactId>
    <version>6.1.0</version>
</dependency>

```

上面将mybatis和pagehelper中的jsqlparser都排除掉，引入sqlparser4.5，这个是官方给的解决方案，可以在看下面官方的issue：

https://github.com/pagehelper/Mybatis-PageHelper/issues/802
https://github.com/pagehelper/Mybatis-PageHelper/releases/tag/v6.1.0
但是呢，issue里面只说了一半，还需要建立两个类：

第一个类：LocalMySqlDialect

```java
import com.github.pagehelper.dialect.helper.MySqlDialect;
import com.github.pagehelper.parser.CountJSqlParser45;
import com.github.pagehelper.parser.CountSqlParser;
import com.github.pagehelper.parser.OrderByJSqlParser45;
import com.github.pagehelper.parser.OrderBySqlParser;
import com.github.pagehelper.util.ClassUtil;

import java.util.Properties;

/**
 * 解决Mybatis Plus与PageHelper之间的冲突
 * 覆盖父类 {@link com.github.pagehelper.dialect.AbstractDialect} 中的setProperties方法，
 * 将CountJSqlParser45、OrderByJSqlParser45提供的两个类来替换掉Default类
 *
 * @Author xiuvee
 * @Date 2024/3/4 11:00
 **/
public class LocalMySqlDialect extends MySqlDialect {

    @Override
    public void setProperties(Properties properties) {
        this.countSqlParser = ClassUtil.newInstance(properties.getProperty("countSqlParser"), CountSqlParser.class, properties, CountJSqlParser45::new);
        this.orderBySqlParser = ClassUtil.newInstance(properties.getProperty("orderBySqlParser"), OrderBySqlParser.class, properties, OrderByJSqlParser45::new);
    }
}

```

- 第二个类：DialectInit

```java
import com.github.pagehelper.page.PageAutoDialect;
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.stereotype.Component;

/**
 * 在spring boot启动完成后将LocalMySqlDialect注册进pagehelper
 *
 * @Author xiuvee
 * @Date 2024/3/4 11:10
 **/
@Component
public class DialectInit implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) throws Exception {
        PageAutoDialect.registerDialectAlias("mysql", LocalMySqlDialect.class);
    }
}

```

说下大致原理，PageAutoDialect这个类是用来管理注册方言的，它在MySql的方言中默认使用了com.github.pagehelper.dialect.helper.MySqlDialect类，而MySqlDialect类继承自com.github.pagehelper.dialect.AbstractDialect类，而AbstractDialect默认实现了setProperties方法，不兼容的地方就在这里：

```java
    @Override
    public void setProperties(Properties properties) {
        this.countSqlParser = ClassUtil.newInstance(properties.getProperty("countSqlParser"), CountSqlParser.class, properties, DefaultCountSqlParser::new);
        this.orderBySqlParser = ClassUtil.newInstance(properties.getProperty("orderBySqlParser"), OrderBySqlParser.class, properties, DefaultOrderBySqlParser::new);
    }

```

我们覆盖掉这个方法，使用官方提供的4.5兼容包，并重新注册即可。

PS：官方没有给4.6的兼容包，只能使用4.5的了（可能4.5和4.6变化不大的原因）