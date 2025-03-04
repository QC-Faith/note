# mybatis


1. `Mybatis `是一个半 `ORM`（对象关系映射）框架，它内部封装了 `JDBC`，开发时只需要关注 `SQL `语句本身，不需要花费精力去处理加载驱动、创建连接、创建`statement `等繁杂的过程。程序员直接编写原生态 `sql`，可以严格控制 `sql `执行性能，灵活度高。

2. `MyBatis `可以使用 `XML `或注解来配置和映射原生信息，将 `POJO `映射成数据库中的记录，避免了几乎所有的 `JDBC `代码和手动设置参数以及获取结果集。

3. 通过 `xml `文件或注解的方式将要执行的各种 `statement `配置起来，并通过`java `对象和 `statement `中 `sql `的动态参数进行映射生成最终执行的 `sql `语句，最后由 `mybatis `框架执行 `sql `并将结果映射为 `java `对象并返回。（从执行 `sql `到返回 `result `的过程）。

## 工作原理

1.  **配置文件**：MyBatis需要一个XML配置文件，叫做`mybatis-config.xml`，用于定义数据源、事务管理以及其他一些设置。 
2.  **SQL映射文件**：为了告诉MyBatis如何映射SQL查询到我们的对象或Java Beans，我们需要定义另一些XML文件。这些文件里，我们会写SQL语句，并定义输入和输出。 
3.  **SqlSessionFactory**：当MyBatis初始化时，它会根据上面提到的配置文件创建一个SqlSessionFactory。这个工厂只会被创建一次，然后被用来生产SqlSession，这些SqlSession是应用中真正做数据库操作的对象。 
4.  **SqlSession**：这是MyBatis的一个关键组件。每当我们想和数据库进行交互时，我们就从SqlSessionFactory那里拿到一个SqlSession。这个会话包含了所有执行SQL的方法，比如`insert`, `update`, `delete`, `select`等。 
5.  **映射器**：为了使代码更整洁，我们经常使用接口来代表SQL映射。这些接口的方法对应了之前在XML映射文件中定义的SQL语句。这样，我们就可以像调用普通的Java方法那样执行SQL语句了。 

## 优点

1. 用 `SQL `语句编程，相当灵活，不会对应用程序或者数据库的现有设计造成任何影响，`SQL `写在 `XML `里，解除 `sql `与程序代码的耦合，便于统一管理；提供 `XML`标签，支持编写动态 `SQL `语句，并可重用。

2. 与 `JDBC `相比，减少了 `50%`以上的代码量，消除了 `JDBC` 大量冗余的代码，不需要手动开关连接；

3. 很好的与各种数据库兼容（因为 `MyBatis` 使用 `JDBC` 来连接数据库，所以只要`JDBC `支持的数据库 `MyBatis `都支持）。

4. 能够与 `Spring `很好的集成；

5. 提供映射标签，支持对象与数据库的 `ORM `字段关系映射；提供对象关系映射标签，支持对象关系组件维护。

## 缺点

1. `SQL `语句的编写工作量较大，尤其当字段多、关联表多时，对开发人员编写`SQL `语句的功底有一定要求。

2. `SQL `语句依赖于数据库，导致数据库移植性差，不能随意更换数据库。

## 与Hibernant

1. `Mybatis `和 `hibernate `不同，它不完全是一个 `ORM `框架，因为 `MyBatis `需要程序员自己编写 Sql 语句。
2. `Mybatis `直接编写原生态 `sql`，可以严格控制 `sql `执行性能，灵活度高，非常适合对关系数据模型要求不高的软件开发，因为这类软件需求变化频繁，一但需求变化要求迅速输出成果。但是灵活的前提是 `mybatis `无法做到数据库无关性，如果需要实现支持多种数据库的软件，则需要自定义多套 `sql `映射文件，工作量大。
3. `Hibernate `对象/关系映射能力强，数据库无关性好，对于关系模型要求高的软件，如果用`hibernate`开发可以节省很多代码，提高效率。

## 分页方式

1. **Limit分页**

   实现：在`sql`语句后面添加`limit`分页语句。

2. **Mybatis_PageHelper分页插件**（物理分页）

   自定义拦截器实现了拦截所有以`ByPage`结尾的查询语句，并且利用获取到的分页相关参数统一在`sql`语句后面加上`limit`分页的相关语句，一劳永逸。不再需要在每个语句中单独去配置分页相关的参数了。

3. **RowBounds分页（不推荐使用）**（逻辑分页）

   原理：一次获取所有符合条件的数据，然后在内存中对大数据进行操作，实现分页效果。只是数组分页需要我们自己去实现分页逻辑，这里更加简化而已。

   存在问题：一次性从数据库获取的数据可能会很多，对内存的消耗很大，可能导致性能变差，甚至引发内存溢出。

   适用场景：在数据量很大的情况下，建议还是适用拦截器实现分页效果。`RowBounds`建议在数据量相对较小的情况下使用。

   ~~~java
   @Test
   public void selectUserRowBounds() {
       SqlSession session = MybatisUtils.getSession();
       UserMapper mapper = session.getMapper(UserMapper.class);
       // List<User> users = session.selectList("com.dy.mapper.UserMapper.getUserInfoRowBounds",null,new RowBounds(0, 5));
       List<User> users = mapper.getUserInfoRowBounds(new RowBounds(0,5));
       for (User map: users){
           System.out.println(map);
       }
       session.close();
   }
   
   List<User> getUserInfoRowBounds(RowBounds rowBounds);
   
   <select id="getUserInfoRowBounds" resultType="dayu">
      select * from user
   </select>
   ~~~

   总结：
   
   从上面三种`sql`分页的实现方式可以看出，通过`RowBounds`实现是最简便的，但是通过拦截器的实现方式是最优的方案。只需一次编写，所有的分页方法共同使用，还可以避免多次配置时的出错机率，需要修改时也只需要修改这一个文件，一劳永逸。而且是我们自己实现的，便于我们去控制和增加一些逻辑处理，使我们在外层更简单的使用。同时也不会出现数组分页和`RowBounds`分页导致的性能问题。数据量小时，`RowBounds`为一种好办法。但数据量大时，实现拦截器更好。

## 缓存

### 一级缓存（本地缓存）：

1. **作用范围：** 一级缓存是在`SqlSession`的生命周期内有效，也就是说，每个`SqlSession`拥有独立的一级缓存。 

2. **默认开启：** 一级缓存在MyBatis中默认是开启的，无需额外配置。 

3. **特点：** 当执行查询操作时，查询的结果会被缓存在当前`SqlSession`中。如果再次执行相同的查询，MyBatis会首先尝试从缓存中获取数据，而不再访问数据库。 

4. **自动刷新：** `MyBatis`会在执行insert、update、delete等写操作时自动清空一级缓存，以保持数据的一致性。

5. 在与Spring整合后，由于Spring管理`SqlSession`的方式改变，一级缓存的作用范围实际上变成了'当前事务'范围，而非单个方法调用。所以在跨事务的情况下，一级缓存会失效。

   > 当MyBatis与Spring整合时：
   >
   > 1. **SqlSession的管理方式改变**：
   >    - Spring使用`SqlSessionTemplate`代理SqlSession
   >    - 事务由Spring的事务管理器控制
   >    - SqlSession与Spring事务绑定
   > 2. **缓存的生命周期变化**：
   >    - 在**同一个Spring事务内**的多个数据库操作共享同一个SqlSession
   >    - 一级缓存在当前事务范围内有效
   >    - 事务提交或回滚时，SqlSession会被关闭
   > 3. **实际效果**：
   >    - 同一Service方法内多次调用相同Mapper方法，一级缓存生效
   >    - 不同Service方法间如果在同一事务中，一级缓存同样生效
   >    - 不同事务间的调用，一级缓存不共享

![image-20211113140004897](https://gitee.com/qc_faith/picture/raw/master/image/image-20211113140004897.png)

### 二级缓存（全局缓存）：

二级缓存是`MyBatis`提供的命名空间(namespace)级别的缓存机制，作用于mapper级别，可以跨SqlSession共享数据，本质上是一种空间换时间的优化策略。

1.  **作用范围：** 二级缓存是在多个`SqlSession`之间共享的，即多个`SqlSession`可以共享同一个二级缓存。 
2.  **配置开启：** 二级缓存需要手动配置开启，需要在映射文件的`<mapper>`标签下添加`<cache>`元素。 

```xml
<cache eviction="LRU" flushInterval="60000" size="1024" readOnly="true"/>
```

3. **特点：** 二级缓存能够跨`SqlSession`共享查询结果，有效减少数据库访问次数。它的数据存储在全局范围的缓存中，可以由多个`SqlSession`访问。 

4. **缓存策略：** 可以根据需求选择不同的缓存策略（例如`LRU`、`FIFO`等），以及配置缓存的大小、刷新间隔等参数。 

5. **注意事项：** 

   > - 二级缓存可以缓存的对象需要是可序列化的，当`readOnly="false"`(默认值)时，对象必须实现`Serializable`接口。当`readOnly="true"`时，返回对象的引用，无需序列化，但线程不安全
   > - 对namespace内的表执行INSERT/UPDATE/DELETE时，该namespace的缓存会自动清空；关联表更新时，需要手动调用清除方法或者通过`@CacheNamespaceRef`

6. **使用场景：**二级缓存是提升读取性能的有效工具，适用于读多写少、数据变化不频繁的场景，但在数据一致性要求高或写操作频繁的系统中需谨慎使用

##`#{}`和`${}`的区别

`#{}`是占位符，预编译处理；`${}`是拼接符，字符串替换，没有预编译处理。

`Mybatis`在处理`#{}`时，`#{}`传入参数是以字符串传入，会将`SQL`中的`#{}`替换为`?`号，调用`PreparedStatement`的`set`方法来赋值。

`Mybatis`在处理时是原值传入 ， 就是把`{}`替换成变量的值，相当于`JDBC`中的`Statement`编译

变量替换后，`#{} `对应的变量自动加上单引号 `‘’`，`${}` 对应的变量不会加上单引号 `‘’`

`#{} `可以有效的防止`SQL`注入，提高系统安全性；`${} `不能防止`SQL` 注入

`#{}` 的变量替换是在`DBMS` 中；`${}` 的变量替换是在 `DBMS` 外

## SqlSessionFactory

SqlSessionFactory是MyBatis的核心接口，主要负责创建SqlSession实例。它在应用中通常是单例的，一旦创建就应当在应用的运行期间一直存在。

## SqlSession

SqlSession代表一个数据库会话，是执行SQL操作的主要接口。

1. **生命周期**：非永久性，每次数据库访问都需要创建，使用后应关闭
2. **核心功能**：提供执行SQL的各种方法（selectOne、selectList、insert、update、delete等），`Mybatis`中所有的数据库交互都由`SqlSession`来完成
3. **事务管理**：提供commit()、rollback()方法进行事务控制
4. **实现方式**：默认实现是DefaultSqlSession，与Spring集成时使用SqlSessionTemplate

<img src="https://gitee.com/qc_faith/picture/raw/master/image/20250302200923.jpeg" alt="img" style="zoom:67%;" />

### SqlSession创建步骤

- 从配置中获取Environment；
- 从Environment中取得DataSource；
- 从Environment中取得TransactionFactory；
- 从DataSource里获取数据库连接对象Connection；
- 在取得的数据库连接上创建事务对象Transaction；
- 创建Executor对象（该对象非常重要，事实上SqlSession的所有操作都是通过它完成的）；
- 创建SqlSession对象。

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

## Interceptor插件

`MyBatis`的插件机制允许在`MyBatis`的核心组件执行过程中插入自定义逻辑，以扩展或修改其行为。插件可以在SQL执行、结果映射、参数处理等阶段进行干预。插件运行原理是基于Java的动态代理，它可以包装`MyBatis`的核心组件，拦截方法调用，并在方法执行前后执行自定义逻辑。

> `MyBatis`拦截器可拦截`Executor`、`ParameterHandler`、`ResultSetHandler`和`StatementHandler`

插件机制的核心是`Interceptor`接口，可以实现这个接口，编写自己的插件逻辑。一个插件主要包括以下几个步骤：

1.  **实现Interceptor接口：** 创建一个类，实现`MyBatis`提供的`Interceptor`接口，该接口包含了`intercept`和`plugin`两个方法。 
2.  **实现intercept方法：** `intercept`方法是插件的核心，它会在方法执行前后进行拦截。你可以在这个方法中编写自己的逻辑。 
3.  **实现plugin方法：** `plugin`方法用于创建代理对象，将插件包装在目标对象上，使得插件逻辑能够被执行。 
4.  **配置插件：** 类上使用`@Intercepts`注解或者在`MyBatis`的配置文件中，通过`<plugins>`标签配置。通常需要指定插件类和一些参数。 

例如：拦截器与SQL时间统计

~~~java
@Intercepts({
        @Signature(
                type = Executor.class,
                method = "update",
                args = {MappedStatement.class, Object.class}),
        @Signature(
                type = Executor.class,
                method = "query",
                args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class})
})
public class MyPlugin implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        // 记录开始时间戳
        long startTime = System.currentTimeMillis();

        // 调用原始方法(SQL执行)
        Object result = invocation.proceed();

        // 计算执行时间
        long endTime = System.currentTimeMillis();
        long executionTime = endTime - startTime;

        // 获取相关SQL信息用于日志
        String methodName = invocation.getMethod().getName();
        Object target = invocation.getTarget();
        String sqlId = getSqlId(invocation); // 辅助方法，见下文

        // 打印SQL执行时间
        System.out.println("SQL执行ID: " + sqlId + ", 执行时间: " + executionTime + "ms");

        return result;
    }

    // 辅助方法：提取SQL ID
    private String getSqlId(Invocation invocation) {
        Object[] args = invocation.getArgs();
        if (args != null && args.length > 0 && args[0] instanceof MappedStatement) {
            MappedStatement ms = (MappedStatement) args[0];
            return ms.getId();
        }
        return "Unknown";
    }
}
~~~

### 插件拦截点

插件拦截关键点：

1. **Executor（执行器）层面的拦截：** 这是SQL语句的执行层面，插件可以在SQL语句执行前后进行拦截。这包括了SQL的预处理、参数设置、查询结果的映射等。 

2. **StatementHandler（语句处理器）层面的拦截：** 这是对SQL语句的处理层面，插件可以在SQL语句被执行之前进行拦截，可以在这里修改、替换、生成SQL语句。 

3. **ParameterHandler（参数处理器）层面的拦截：** 这是处理参数的层面，插件可以在参数传递给SQL语句之前进行拦截，可以在这里修改参数值。 

4. **ResultSetHandler（结果集处理器）层面的拦截：** 这是处理查询结果的层面，插件可以在查询结果返回给调用方之前进行拦截，可以在这里对查询结果进行修改、处理。 

   > 插件使用场景示例：
   >
   > 1.  **日志记录：** 创建一个插件，拦截Executor层的SQL执行，记录每个SQL语句的执行时间、执行情况等，以便于性能分析和故障排查。 
   > 2.  **分页支持：** 编写一个拦截器，在StatementHandler层拦截SQL语句，根据传入的分页参数，动态修改SQL语句以实现数据库分页查询。 
   > 3.  **权限控制：** 开发一个插件，拦截StatementHandler层的SQL执行，根据当前用户的权限动态添加查询条件，确保用户只能访问其有权限的数据。 
   > 4.  **二级缓存扩展：** 创建一个插件，在ResultSetHandler层拦截查询结果，对结果进行加工处理，然后再将处理后的结果放入二级缓存，提供更加定制化的缓存机制。 
   > 5.  **自动填充字段：** 编写一个拦截器，在ParameterHandler层拦截参数设置，根据需要自动填充一些字段，比如创建时间、更新时间等。



## 问题

1. **如何获取生成的主键？**

   `insert `方法总是返回一个 `int `值 ，这个值代表的是插入的行数。

   如果采用自增长策略，自动生成的键值在 `insert `方法执行完后可以被设置到传入的参数对象中。

2. **当实体类中的属性名和表中的字段名不一样 ，怎么办**

   1. 通过在查询的`SQL`语句中定义字段名的别名，让字段名的别名和实体类的属性名一致。
   2. 通过`<resultMap>`来映射字段名和实体类属性名的一一对应的关系。


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

