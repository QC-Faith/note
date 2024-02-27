# mybatis

##**是什么**

1. `Mybatis `是一个半 `ORM`（对象关系映射）框架，它内部封装了 `JDBC`，开发时只需要关注 `SQL `语句本身，不需要花费精力去处理加载驱动、创建连接、创建`statement `等繁杂的过程。程序员直接编写原生态 `sql`，可以严格控制 `sql `执行性能，灵活度高。

2. `MyBatis `可以使用 `XML `或注解来配置和映射原生信息，将 `POJO `映射成数据库中的记录，避免了几乎所有的 `JDBC `代码和手动设置参数以及获取结果集。

3. 通过 `xml `文件或注解的方式将要执行的各种 `statement `配置起来，并通过`java `对象和 `statement `中 `sql `的动态参数进行映射生成最终执行的 `sql `语句，最后由 `mybatis `框架执行 `sql `并将结果映射为 `java `对象并返回。（从执行 `sql `到返回 `result `的过程）。

##**优点**

1. 于 `SQL `语句编程，相当灵活，不会对应用程序或者数据库的现有设计造成任何影响，`SQL `写在 `XML `里，解除 `sql `与程序代码的耦合，便于统一管理；提供 `XML`标签，支持编写动态 `SQL `语句，并可重用。

2. 与 `JDBC `相比，减少了 `50%`以上的代码量，消除了 `JDBC` 大量冗余的代码，不需要手动开关连接；

3. 很好的与各种数据库兼容（因为 `MyBatis` 使用 `JDBC` 来连接数据库，所以只要`JDBC `支持的数据库 `MyBatis `都支持）。

4. 能够与 `Spring `很好的集成；

5. 提供映射标签，支持对象与数据库的 `ORM `字段关系映射；提供对象关系映射标签，支持对象关系组件维护。

##**缺点**

1. `SQL `语句的编写工作量较大，尤其当字段多、关联表多时，对开发人员编写`SQL `语句的功底有一定要求。

2. `SQL `语句依赖于数据库，导致数据库移植性差，不能随意更换数据库。

##**与Hibernant**

1. `Mybatis `和 `hibernate `不同，它不完全是一个 `ORM `框架，因为 `MyBatis `需要程序员自己编写 Sql 语句。
2. `Mybatis `直接编写原生态 `sql`，可以严格控制 `sql `执行性能，灵活度高，非常适合对关系数据模型要求不高的软件开发，因为这类软件需求变化频繁，一但需求变化要求迅速输出成果。但是灵活的前提是 `mybatis `无法做到数据库无关性，如果需要实现支持多种数据库的软件，则需要自定义多套 `sql `映射文件，工作量大。
3. `Hibernate `对象/关系映射能力强，数据库无关性好，对于关系模型要求高的软件，如果用`hibernate`开发可以节省很多代码，提高效率。

## 四种分页方式

1. 数组分页

原理：进行数据库查询操作时，获取到数据库中所有满足条件的记录，保存在应用的临时数组中，再通过`List`的`subList`方法，获取到满足条件的所有记录。

缺点：数据库查询并返回所有的数据，而我们需要的只是极少数符合要求的数据。当数据量少时，还可以接受。当==数据库数据量过大时，每次查询对数据库和程序的性能都会产生极大的影响。==

2. 借助`Sql`语句进行分页

实现：在`sql`语句后面添加`limit`分页语句。

3. 拦截器分页

自定义拦截器实现了拦截所有以`ByPage`结尾的查询语句，并且利用获取到的分页相关参数统一在`sql`语句后面加上`limit`分页的相关语句，一劳永逸。不再需要在每个语句中单独去配置分页相关的参数了。

4. `RowBounds`实现分页

原理：通过`RowBounds`实现分页和通过数组方式分页原理差不多，都是一次获取所有符合条件的数据，然后在内存中对大数据进行操作，实现分页效果。只是数组分页需要我们自己去实现分页逻辑，这里更加简化而已。

存在问题：一次性从数据库获取的数据可能会很多，对内存的消耗很大，可能导致性能变差，甚至引发内存溢出。

适用场景：在数据量很大的情况下，建议还是适用拦截器实现分页效果。`RowBounds`建议在数据量相对较小的情况下使用。

总结：

从上面四种`sql`分页的实现方式可以看出，通过`RowBounds`实现是最简便的，但是通过拦截器的实现方式是最优的方案。只需一次编写，所有的分页方法共同使用，还可以避免多次配置时的出错机率，需要修改时也只需要修改这一个文件，一劳永逸。而且是我们自己实现的，便于我们去控制和增加一些逻辑处理，使我们在外层更简单的使用。同时也不会出现数组分页和`RowBounds`分页导致的性能问题。数据量小时，`RowBounds`为一种好办法。但数据量大时，实现拦截器更好。

## 缓存

缓存的意义

- 将用户==经常查询的数据放在缓存（内存）中==，用户去查询数据就不用从磁盘上(关系型数据库数据文件)查询，==从缓存中查询，从而提高查询效率==，解决了高并发系统的性能问题。

- **`mybatis`一级缓存是一个`SqlSession`级别，`Sqlsession`只能访问自己的一级缓存的数据**

`Mybatis`的一级缓存是`sqlSession`级别的。只能访问自己的`sqlSession`内的缓存。如果`Mybatis`与`Spring`整合了，`Spring`会自动关闭`sqlSession`的。所以一级缓存会失效的。

- **二级缓存是跨`SqlSession`，是`mapper`级别的缓存，对于`mapper`级别的缓存不同的`Sqlsession`是可以共享的。**

![image-20211113140004897](https://gitee.com/qc_faith/picture/raw/master/image/image-20211113140004897.png)

###一级缓存

==第一次发出一个查询`sql`，`sql`查询结果写入`sqlsession`的一级缓存中，缓存使用的数据结构是一个map==

- `key`：`hashcode+sql+sql`输入参数+输出参数（`sql`的唯一标识）
- `value`：用户信息

同一个`sqlsession`再次发出相同的`sql`，就从缓存中取，不走数据库。如果两次中间出现`commit`操作（修改、添加、删除），本`sqlsession`中的一级缓存区域全部清空，下次再去缓存中查询不到所以要从数据库查询，从数据库查询到再写入缓存。

**注意**

- `Mybatis`默认就是支持一级缓存的，并不需要我们配置.
- `mybatis`和`spring`整合后进行`mapper`代理开发，不支持一级缓存，`mybatis`和`spring`整合，**`spring`按照`mapper`的模板去生成`mapper`代理对象，模板中在最后统一关闭`sqlsession`。**

###二级缓存

==二级缓存的范围是`mapper`级别（`mapper`同一个命名空间），`mapper`以命名空间为单位创建缓存数据结构，结构是`map`。==

每次查询先看是否开启二级缓存，如果开启从二级缓存的数据结构中取缓存数据，如果从二级缓存中没有取到，再去一级缓存中找，如果一级缓存也没有，就去数据库查找。



**二级缓存配置**

在`Mybatis`的配置文件中配置二级缓存

```xml
    <!-- 全局配置参数 -->
    <settings>
        <!-- 开启二级缓存 -->
        <setting name="cacheEnabled" value="true"/>
    </settings>
```

二级缓存的范围是mapper级别的，因此我们的**Mapper如果要使用二级缓存，还需要在对应的映射文件中配置**

`mybatis`二级缓存需要将查询结果映射的`pojo`实现` java.io.serializable`接口，如果不实现则抛出异常

**二级缓存可以将内存的数据写到磁盘，存在对象的序列化和反序列化**，所以要实现`java.io.serializable`接口。如果结果映射的`pojo`中还包括了`pojo`，都要实现`java.io.serializable`接口。

==变化频率较高的`sql`，需要禁用二级缓存==：

在`statement`中设置`useCache=false`可以禁用当前`select`语句的二级缓存，即每次查询都会发出`sql`去查询，默认情况是`true`，即该`sql`使用二级缓存。

**应用场景**

==对查询频率高，变化频率低的数据建议使用二级缓存。==

对于==访问多的查询请求且用户对查询结果实时性要求不高==，可采用二级缓存降低数据库访问量，提高访问速度

比如：

- 耗时较高的统计分析`sql`、
- 电话账单查询`sql`等。

实现方法如下：==通过设置刷新间隔时间，由`mybatis`每隔一段时间自动清空缓存，根据数据变化频率设置缓存刷新间隔`flushInterval`==，比如设置为30分钟、60分钟、24小时等，根据需求而定。

**局限性**

`mybatis`二级缓存对==细粒度的数据级别==的缓存实现不好，比如：对商品信息进行缓存，由于商品信息查询访问量大，但是要求用户每次都能查询最新的商品信息，此时如果使用`mybatis`的二级缓存就无法实现当一个商品变化时只刷新该商品的缓存信息而不刷新其它商品的信息，**因为`mybaits`的二级缓存区域以`mapper`为单位划分，当一个商品信息变化会将所有商品信息的缓存数据全部清空**。解决此类问题需要在业务层根据需求对数据有针对性缓存。

##`#{}`和`${}`的区别

`#{}`是占位符，预编译处理；`${}`是拼接符，字符串替换，没有预编译处理。

`Mybatis`在处理`#{}`时，`#{}`传入参数是以字符串传入，会将`SQL`中的`#{}`替换为`?`号，调用`PreparedStatement`的`set`方法来赋值。

`Mybatis`在处理时是原值传入 ， 就是把`{}`替换成变量的值，相当于`JDBC`中的`Statement`编译

变量替换后，`#{} `对应的变量自动加上单引号 `‘’`，`${}` 对应的变量不会加上单引号 `‘’`

`#{} `可以有效的防止`SQL`注入，提高系统安全性；`${} `不能防止`SQL` 注入

`#{}` 的变量替换是在`DBMS` 中；`${}` 的变量替换是在 `DBMS` 外



## 问题

1. **如何获取生成的主键？**

   `insert `方法总是返回一个 `int `值 ，这个值代表的是插入的行数。

   如果采用自增长策略，自动生成的键值在 `insert `方法执行完后可以被设置到传入的参数对象中。

2. **当实体类中的属性名和表中的字段名不一样 ，怎么办**

第1种： 通过在查询的`SQL`语句中定义字段名的别名，让字段名的别名和实体类的属性名一致。

第2种： 通过`<resultMap>`来映射字段名和实体类属性名的一一对应的关系。

3. **SqlSession是啥**

`SqlSession`是一个会话，相当于`JDBC`中的一个`Connection`对象，是整个`Mybatis`运行的核心。`SqlSession`接口提供了查询，插入，更新，删除方法，`Mybatis`中所有的数据库交互都由`SqlSession`来完成

`SqlSession`接口有两个实现`SqlSessionManager，DefaultSqlSession`。

- `SqlSessionManager`：对`SqlSessionFactory `和`SqlSession `接口的实现，主要功能是对`SqlSessionFactory `和`SqlSession`的管理，是更高层次的封装
- `DefaultSqlSession`：`SqlSession`的实现，`Mybatis`工作时真正调用的类，所有调用都是通过此类来实现。

`DefaultSqlSession`实现主要由4大组件来完成：`Executor,StatementHandler,ParameterHandler,ResultHandler`:

- `Executor`(执行器)：由他来调度`StatementHandler`，`ParameterHandler`，`ResultHandler`等来执行对应的SQL;

  `Executor`接口主要有三个实现

  - `SimpleExecutor`：简单执行器，如果不配置就是`mybatis`默认的执行器
  - `ReuseExecutor`：一种重用预处理语句的执行器
  - `BatchExecutor`：针对批量处理的执行器，执行重用语句和批量更新。

- `StatementHandler`(数据库会话处理器 )：使用数据库的`statement`(`PreparedStatement`)来执行`SQL`;

  - `StatementHandler`执行了与数据库的交互工作。其接口的主要实现有3个`SimpleStetementHandler,CallableStatementHandler,PreparedStatementHandler`,从名称上看出`CallableStatementHandler,PreparedStatementHandler`分别对应作为`JDBC`的`CallbackStatement,PreparedStatement`的处理器，而默认使用的为`PreparedStatementHandler`.

- `ParameterHandler`(参数处理器)：用于对`SQL`语句中参数的处理；

  - `ParameterHandler`完成对`SQL`语句的配置，只有一个实现`DefaultParameterHandler`

- `ResultHandler`(结果集处理器)：进行最后结果集（`ResultSet`）的封装返回处理。
  - `ResultSetHandler`完成对对结果集的处理，返回需要的数据类型。也只有一个实现`DefaultReultSetHandler`.

# ParameterHandler

> https://blog.csdn.net/BryantLmm/article/details/78640632

```java
 public void setParameters(PreparedStatement ps) throws SQLException {
    ErrorContext.instance().activity("setting parameters").object(mappedStatement.getParameterMap().getId());
   // parameterMappings  是对#{} 里给出的参数信息的封装，即这个SQL是个参数化SQL（SQL语句中带占位符?）时会存在
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
   //如果是参数化SQL，便要设置参数
   if (parameterMappings != null) {
      for (int i = 0; i < parameterMappings.size(); i++) {
        ParameterMapping parameterMapping = parameterMappings.get(i);
        //如果参数的类型不是OUT类型，CallableStatementHandler会用到
        //因为存储过程才存在输出参数，所以当参数不是输出参数的时候，就需要设置
        if (parameterMapping.getMode() != ParameterMode.OUT) {
          Object value;
          //在本次查询中，对应的不是一个Bean，对应的是一个additionalParameters Map对象，所以这里的property不是Bean的属性，而是经过封装的Key（可以通过Key找到对应的Value,该Value就是我们要找的值）
         //如果是Bean 则通过属性的反射来得到Value
          String propertyName = parameterMapping.getProperty();
          //如果propertyName是additionalParameters中的key
          if (boundSql.hasAdditionalParameter(propertyName)) { // issue #448 ask first for additional params
            //通过Key来得到Map中的Value
            value = boundSql.getAdditionalParameter(propertyName);
          }//如果不是additionalParameters中的Key，而且传入参数是null，那value就为null
          else if (parameterObject == null) {
            value = null;
          } 
          //如果在typeHandlerRegistry中已经注册了这个参数的Class对象，即他是Primitive类型或者是String ByteArray 等等 26个类型 也就是说不是JavaBean 也不是Map List类型时。value直接等于Method传进来的参数
          else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
            value = parameterObject;
          }
          //否则就是Map（Method穿进行的原始参数是List类型或者是Array类型时，这时已经被封装成了Map类型了）类型或者是Bean 通过封装的MataObject对象，用propertyName得到相应的Value,Bean通过反射得到，Map类型则是通过Key得到Value
          else {
            MetaObject metaObject = configuration.newMetaObject(parameterObject);
            value = metaObject.getValue(propertyName);
          }
          //在通过SqlSource的parse方法得到paramterMappings的具体实现中，我们会得到parameterMapping的TypeHandler
        //本次查询中TypeHandler均为IntegerTypeHandler          
          TypeHandler typeHandler = parameterMapping.getTypeHandler();
          //得到相应parameterMapping的JdbcType,如果没有在#{}中显式的指定JdbcType，则为null
          JdbcType jdbcType = parameterMapping.getJdbcType();
          //如果得到的value为null并且jdbcType也是null的时候，jdbcType就会成为configuration.getJdbcTypeForNull();  即OTHER，所以我们在编写Mapper配置文件的Insert或者Update类型的SqlMapper时，如果某个#{propertyName}对应的Value是null的话，插入数据库是会报错的，
// 因为此时的JdbcType是Other,执行setNull时会报错，

       //java.sql.SQLException: 无效的列类型: 1111 
          //1111 就是OTHER对应的sqlType
         // 所以Value为null的情况下，我们需要指定JdbcType
          if (value == null && jdbcType == null) jdbcType = configuration.getJdbcTypeForNull();
          //这一行实现了具体的ps.set***
          typeHandler.setParameter(ps, i + 1, value, jdbcType);
        }
      }
    }
  }
```

