# SSM

## 1 Mybatis

[Mybaits系列文章](https://juejin.cn/post/6844903876617994248)

### 1.1 原生JDBC

### 1.2 Mybaits配置文件

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
  PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
  <properties resource="org/mybatis/example/config.properties">
  </properties>
  <environments default="development">
    <environment id="development">
      <transactionManager type="JDBC"/>
      <dataSource type="POOLED">
        <property name="driver" value="${driver}"/>
        <property name="url" value="${url}"/>
        <property name="username" value="${username}"/>
        <property name="password" value="${password}"/>
      </dataSource>
    </environment>
  </environments>
  <mappers>
    <mapper resource="org/mybatis/example/BlogMapper.xml"/>
  </mappers>
</configuration>
```

> 配置文件解析

http://mybatis.org/dtd/mybatis-3-config.dtd表示对xml配置文件的标签进行约束，以及标签提示

#### 1.2.1 properties

引入外部的配置文件，并且可以通过${}表达式在整个xml配置文件中引用properties文件中的属性

```xml
<dataSource type="POOLED">
  <property name="driver" value="${driver}"/>
  <property name="url" value="${url}"/>
  <property name="username" value="${username}"/>
  <property name="password" value="${password}"/>
</dataSource>
```

这里就是在外部定义了关于数据库连接的properties配置文件

#### 1.2.2 settings

可以修改Mybatis运行时行为

| 设置名                   | 描述                                                         | 有效值        |
| ------------------------ | ------------------------------------------------------------ | ------------- |
| cacheEnabled             | 全局性地开启或关闭所有mapper配置文件中已配置的任何缓存       | true \| false |
| lazyLoadingEnabled       | 延迟加载的全局开关，当开启时，所有关联对象都会延迟加载， 特定关联关系中可通过设置 `fetchType` 属性来覆盖该项的开关状态 | true \| false |
| useGeneratedKeys         | 允许 JDBC 支持自动生成主键，需要数据库驱动支持；如果设置为 true，将强制使用自动生成主键 | true \| false |
| mapUnderscoreToCamelCase | 是否开启驼峰命名自动映射，即从经典数据库列名 A_COLUMN 映射到经典 Java 属性名 aColumn。 | true \| false |

对settings标签进行设置

```xml
<settings>
  <setting name="cacheEnabled" value="true"/>
  <setting name="lazyLoadingEnabled" value="true"/>
  <setting name="multipleResultSetsEnabled" value="true"/>
  <setting name="useColumnLabel" value="true"/>
  <setting name="useGeneratedKeys" value="false"/>
  <setting name="autoMappingBehavior" value="PARTIAL"/>
</settings>
```

#### 1.2.3 environments

通过配置不同的environment实现不同的数据库支持；例如开发和测试环境的数据库不同，可以通过配置不同的environment并通过environments进行选择

```xml
<environments default="development">
  <environment id="development">
    <transactionManager type="JDBC">
      <property name="..." value="..."/>
    </transactionManager>
    <dataSource type="POOLED">
      <property name="driver" value="${driver}"/>
      <property name="url" value="${url}"/>
      <property name="username" value="${username}"/>
      <property name="password" value="${password}"/>
    </dataSource>
  </environment>
    <environment id="test">
        <transactionManager type="JDBC"/>
        <dataSource type="POOLED">
            <property name="driver" value="${driver}"/>
            <property name="url" value="${url}"/>
            <property name="username" value="${username}"/>
            <property name="password" value="${password}"/>
        </dataSource>
    </environment>
</environments>
```

> 可以看出每个environment都有一个唯一的id，并且environments通过default属性指定选择哪个数据库配置

#### 1.2.4 mappers

告诉mybatis去哪里寻找mapper映射配置文件

```xml
<!-- 使用相对于类路径的资源引用 -->
<mappers>
  <mapper resource="org/mybatis/builder/AuthorMapper.xml"/>
  <mapper resource="org/mybatis/builder/BlogMapper.xml"/>
  <mapper resource="org/mybatis/builder/PostMapper.xml"/>
</mappers>
<!-- 将包内的映射器接口实现全部注册为映射器 -->
<mappers>
  <package name="org.mybatis.builder"/>
</mappers>
```

> 可以通过mapper标签分别将每个映射文件注册到全局配置中
>
> 也可以通过package将某个包中的所有配置文件注册到全局配置中

### 1.3 mapper映射配置文件

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="org.jiang.admin.dao.UwdRoleDao">
	<select id="selectPerson" parameterType="int" resultType="hashmap">
  		SELECT * FROM PERSON WHERE ID = #{id}
	</select>
</mapper>
```

#### 1.3.1 mapper标签

定义该xml配置文件对应的属于哪个mapper接口，namespace属性也就是指定接口的全路径

#### 1.3.2 resultMap

自定义结果集的封装规则，本质上就是定义JavaBean和sql语句查询结果的列名匹配

常用于联合查询(比如左连接、子查询等)

##### 1.3.2.1 一般resultMap

mapper接口中方法定义

```java
Payment getPaymentById(Integer id);
```

定义resultMap

```xml
<resultMap id="BaseResultMap" type="org.jiang.model.Payment">
    <id property="id" column="id"></id>
    <result property="serial" column="serial"></result>
</resultMap>
```

> 定义resultMap之后即使Bean的属性名和数据库的列名既不一致，也没有使用驼峰命名，依然可以通过指定Bean属性名和数据库的列名的匹配规则匹配
>
> type属性指定为哪个JavaBean封装规则
>
> id标签指定数据库主键列和Bean属性名的匹配规则
>
> 而result标签指定数据库其他列和Bean属性名的匹配规则
>
> id和result标签的property属性指定JavaBean的属性名，而column属性指定数据库列名

select标签

```xml
<select id="getPaymentById" resultMap="BaseResultMap" >
    select * from payment where id = #{id,jdbcType=INTEGER}
</select>
```

> 可以看出这里select标签通过指定resultMap为刚才定义的resultMap

结果：

```bash
Payment(id=31, serial=jiang)
```

> 可以看出依然可以查询出结果

##### 1.3.2.2 进阶resultMap association

当一个model组合了另一个model，且关系为一对一时，就需要在resultMap标签内增加了association标签

![image-20210112212315745](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210112212315745.png)

> 可以看出在Payment对象中组合了person对象

那么如何定义Payment的resultMap呢？

```xml
<resultMap id="BaseResultMap" type="org.jiang.model.Payment">
    <id property="id" column="id"></id>
    <result property="serial" column="serial"></result>
    <association property="person" javaType="org.jiang.model.Person">
        <id property="id" column="pid"></id>
        <result property="name" column="name"></result>
        <result property="age" column="age"></result>
    </association>
</resultMap>
```

> 可以看出person属性定义在association标签中，且指定了javaType就是`org.jiang.model.Person`
>
> 而且需要注意在sql语句中定义的别名需要和resultMap中的column属性进行匹配
>
> 而在association标签内部也是通过id标签和result标签进行定义

那么对应的select标签应该是什么呢？

```xml
<select id="getPaymentById" resultMap="BaseResultMap" >
    SELECT pay.id id,pay.serial serial,per.id pid,per.age age,per.`name` name
        FROM payment pay
        LEFT JOIN person per ON pay.person_id = per.id
        WHERE pay.id = #{id,jdbcType=INTEGER}
</select>
```

> 可以看出一般组合其他对象时sql语句需要使用联合查询

##### 1.3.2.3 进阶resultMap collection

当一个model组合另外一个model，且关系为1-n或n-n时，需要在resultMap标签内使用collection标签

> 上面例子中多个payment对象对应一个person对象，那么反过来一个person对象对应多个payment对象

![image-20210112215101076](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210112215101076.png)

那么resultMap应该如何定义呢？

```xml
<resultMap id="BaseResultMap" type="org.jiang.model.Person">
    <id property="id" column="pid"></id>
    <result property="age" column="age"></result>
    <result property="name" column="name"></result>
    <collection property="payments" ofType="org.jiang.model.Payment">
        <id property="id" column="id"></id>
        <result property="serial" column="serial"></result>
    </collection>
</resultMap>
```

> 可以看出这里collection标签中使用的不是javaType属性，而是ofType属性指定payments的类型
>
> 而collection标签内部依然使用的是id和result标签

对应的select标签：

```xml
<select id="getPersonById" resultMap="BaseResultMap">
    SELECT pay.id id,pay.serial serial,per.id pid,per.age age,per.`name` name
        FROM payment pay
        LEFT JOIN person per ON pay.person_id = per.id
        WHERE per.id = 1
</select>
```

> 注意resultMap中的cloumn属性和sql语句中的别名进行匹配

结果：

```bash
22:05:11.922 [main] DEBUG org.jiang.mapper.PersonMapper.getPersonById - ==>  Preparing: SELECT pay.id id,pay.serial serial,per.id pid,per.age age,per.`name` name FROM payment pay LEFT JOIN person per ON pay.person_id = per.id WHERE per.id = ?
22:05:11.948 [main] DEBUG org.jiang.mapper.PersonMapper.getPersonById - ==> Parameters: 1(Integer)
22:05:11.967 [main] DEBUG org.jiang.mapper.PersonMapper.getPersonById - <==      Total: 3
Person(id=1, name=jiang, age=11, payments=[Payment(id=31, serial=jiang), Payment(id=32, serial=1111), Payment(id=35, serial=nixiaodi)])
```

> 可以看出将查询出的多个结果封装到一个list中

#### 1.3.3 select

定义一个查询操作，id指定接口中的方法

```xml
<select id="getPaymentById" resultType="org.jiang.model.Payment">
    select * from payment where id = #{id,jdbcType=INTEGER}
</select>
```

> 标签id表示指定的哪个方法，resultType指定获取的结果封装为哪个类型的对象
>
> sql语句中的#{}获取方法中的参数，而jdbcType就是指定javaType和jdbcType的匹配

#### 1.3.4 insert、delete、update

```xml
<insert id="insert">
    insert into payment(serial) values (#{serial,jdbcType=VARCHAR})
</insert>
<delete id="deleteById">
    delete from payment where id = #{id,jdbcType=INTEGER}
</delete>
<update id="update">
    update payment set serial = #{serial,jdbcType=VARCHAR} where id = #{id,jdbcType=INTEGER}
</update>
```

> 可以看出insert和update标签：原本方法中的参数为Payment对象，在sql语句中取出参数属性时并不是通过#{payment.id}，而是直接通过#{id}

##### 1.3.4.1 insert、delete、update标签具有的属性

![image-20210110222756124](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210110222756124.png)

> useGeneratedKeys和keyProperty配合使用获取数据库的自增主键

##### 1.3.4.2 测试自增主键的获取

映射文件中通过useGeneratedKeys属性设置使用自增主键

通过keyProperty属性指定自定主键

```xml
<insert id="insert" useGeneratedKeys="true" keyProperty="id">
    insert into payment(serial) values (#{serial,jdbcType=VARCHAR})
</insert>
```

获取插入数据的主键

```java
@Slf4j
public class Test01 {
    @Test
    public void test01() throws IOException {
        String resource = "mybatis-config.xml";
        InputStream is = Resources.getResourceAsStream(resource);
        SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(is);

        SqlSession sqlSession = factory.openSession(true);
        PaymentMapper mapper = sqlSession.getMapper(PaymentMapper.class);
        Payment payment = new Payment(null, "nixiaodi");
        int i = mapper.insert(payment);
        log.info(payment.getId().toString());
    }
}
```

结果：

```bash
22:35:35.603 [main] INFO org.jiang.Test01 - 35
```

> 可以看出插入的数据虽然没有指定id，但是id由于在数据库中属于自增主键，所以当再次获取插入对象的id时可以获取到
>
> 实际上就是mybatis在指定自增主键的获取时，获取到数据库被插入的记录的主键id并通过set方法赋值给payment对象的id属性

#### 1.3.5 mapper接口方法中的参数和mapper映射文件获取参数问题

首先在mapper接口方法中定义了两个参数

```java
public interface PaymentMapper {
	Payment getPaymentByIdAndSerial(Integer id,String serial);
}
```

然后在映射文件sql语句中获取参数

```xml
<select id="getPaymentByIdAndSerial" resultType="org.jiang.model.Payment">
    select * from payment where id = #{id} and serial = #{serial}
</select>
```

调用抽象方法获取结果

```java
@Slf4j
public class Test01 {
    @Test
    public void test01() throws IOException {
        String resource = "mybatis-config.xml";
        InputStream is = Resources.getResourceAsStream(resource);
        SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(is);

        SqlSession sqlSession = factory.openSession(true);
        PaymentMapper mapper = sqlSession.getMapper(PaymentMapper.class);
        Payment payment = mapper.getPaymentByIdAndSerial(31, "jiang");
        System.out.println(payment);
    }
}
```

结果：

```bash
Cause: org.apache.ibatis.binding.BindingException: Parameter 'id' not found. Available parameters are [arg1, arg0, param1, param2]
```

> 可以看出：
>
> 错误信息为没有找到参数id，可用的参数为[arg1, arg0, param1, param2]

为什么我们在方法中定义的参数就是id和serial，然而在sql语句中却获取不到id呢？

那么应该进行怎么处理呢？

![image-20210110225548349](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210110225548349.png)

> 可以看出实际上是通过param1和param2来指定到底是参数中的哪个参数

这种方式虽然可以执行成功，然而没有任何可读性，有没有更好的办法呢？

首先在mapper接口定义方法中通过@param注解指定参数名

```java
public interface PaymentMapper {
    Payment getPaymentByIdAndSerial(@Param("id") Integer id, @Param("serial") String serial);
}
```

这样就可以在映射文件的sql中获取对应的参数

```xml
<select id="getPaymentByIdAndSerial" resultType="org.jiang.model.Payment">
    select * from payment where id = #{id,jdbcType=INTEGER} and serial = #{serial,jdbcType=VARCHAR}
</select>
```

结果：

![image-20210110230041718](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/image-20210110230041718.png)

可以看到可以获取到对应的结果

#### 1.3.6 #{}和${}之间的区别

映射文件sql语句中分别使用#{}和${}

```xml
<select id="getPaymentByIdAndSerial" resultType="org.jiang.model.Payment">
    select * from payment where id = ${id} and serial = #{serial,jdbcType=VARCHAR}
</select>
```

日志输出结果：

```bash
23:04:00.339 [main] DEBUG org.apache.ibatis.transaction.jdbc.JdbcTransaction - Opening JDBC Connection
23:04:01.127 [main] DEBUG org.apache.ibatis.datasource.pooled.PooledDataSource - Created connection 1682681674.
23:04:01.129 [main] DEBUG org.jiang.mapper.PaymentMapper.getPaymentByIdAndSerial - ==>  Preparing: select * from payment where id = 31 and serial = ?
23:04:01.153 [main] DEBUG org.jiang.mapper.PaymentMapper.getPaymentByIdAndSerial - ==> Parameters: jiang(String)
23:04:01.170 [main] DEBUG org.jiang.mapper.PaymentMapper.getPaymentByIdAndSerial - <==      Total: 1
```

> 可以看出sql语句中${}取值是直接将参数封装到语句中
>
> 而#{}是通过预编译参数封装到语句中
>
> 类似于JDBC的statement和prepareStatement之间的区别
>
> 很明显通过${}取值可以防止sql注入问题

#### 1.3.7 select返回结果的封装

##### 1.3.7.1 返回list

mapper接口方法

```java
List<Payment> getAllPayment();
```

映射文件的select标签

```xml
<select id="getAllPayment" resultType="org.jiang.model.Payment">
    select * from payment
</select>
```

可以看出resultType依然为List结合中的对象类型

##### 1.3.7.2 返回map

mapper接口方法

```java
Map<String,Object> getMapPaymentById(Integer id);
```

映射文件的select标签

```xml
<select id="getMapPaymentById" resultType="java.util.Map">
    select * from payment where id = #{id,jdbcType=INTEGER}
</select>
```

> 可以看出这里的resultType是java.util.Map

结果：

```bash
{serial=jiang, id=31}
```

> 可以看出结果中将数据库中的列名作为map中的key，而查询结果作为value

### 1.3 sqlSession

#### 1.3.1 sqlSessionFactory

sqlSession创建工厂，负责创建sqlSession对象(工厂模式)

```java
import org.apache.ibatis.io.Resources;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;

public class Test01 {
    @Test
    public void test01() throws IOException {
        String resource = "mybatis-config.xml";
        InputStream is = Resources.getResourceAsStream(resource);
        SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(is);
    }
}
```

> 注意Resources类属于org.apache.ibatis.io.Resources
>
> 通过读取mybatis全局配置文件作为输入流，并通过SqlSessionFactoryBuilder(建造者模式)构建sqlSessionFactory

#### 1.3.2 sqlSession

代表和数据库的一次会话

```java
@Slf4j
public class Test01 {
    @Test
    public void test01() throws IOException {
        String resource = "mybatis-config.xml";
        InputStream is = Resources.getResourceAsStream(resource);
        SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(is);

        SqlSession sqlSession = factory.openSession(true);
    }
}
```

> 通过sqlSessionFactory调用openSession方法获取一次和数据库的会话
>
> 调用openSession方法可以传入一个布尔值来创建是否自动提交的会话

##### 通过sqlSession获取mapper接口的实例

```java
@Slf4j
public class Test01 {
    @Test
    public void test01() throws IOException {
        String resource = "mybatis-config.xml";
        InputStream is = Resources.getResourceAsStream(resource);
        SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(is);

        SqlSession sqlSession = factory.openSession(true);
        PaymentMapper mapper = sqlSession.getMapper(PaymentMapper.class);
        Payment payment = mapper.getPaymentById(31);
        log.info(JSONUtil.toJsonStr(payment));
    }
}
```

> 通过sqlSession调用getMapper传入某个mapper的class对象获取mapper接口的实例
>
> 该实例可以调用mapper接口中的抽象方法获取到执行数据库的结果

结果：

```bash
21:59:43.270 [main] DEBUG org.apache.ibatis.transaction.jdbc.JdbcTransaction - Opening JDBC Connection
21:59:43.941 [main] DEBUG org.apache.ibatis.datasource.pooled.PooledDataSource - Created connection 303240439.
21:59:43.943 [main] DEBUG org.jiang.mapper.PaymentMapper.getPaymentById - ==>  Preparing: select * from payment where id = ?
21:59:43.968 [main] DEBUG org.jiang.mapper.PaymentMapper.getPaymentById - ==> Parameters: 31(Integer)
21:59:43.986 [main] DEBUG org.jiang.mapper.PaymentMapper.getPaymentById - <==      Total: 1
21:59:44.011 [main] INFO org.jiang.Test01 - {"serial":"jiang","id":31}
```

> 可以看出首先创建了一条connection连接(也就是jdbc连接)
>
> 连接执行prepareStatement对象并获取执行的结果封装为Payment对象

##### 通过sqlSession获取到的接口实例到底是什么

通过日志中输出`PaymentMapper mapper`对象得到

```
22:11:30.247 [main] INFO org.jiang.Test01 - org.apache.ibatis.binding.MapperProxy@799d4f69
```

> 可以看出mapper是一个代理对象，本质上就是使用的动态代理

### 1.4 动态sql

#### 1.4.1 if标签

当对payment进行查询时，可以传入一个payment对象作为参数，判断对象中的某些属性是否存在来组合sql语句

mapper接口中方法的定义

```java
Payment getPaymentCondition(Payment payment);
```

映射文件中方法的定义

```xml
<select id="getPaymentCondition" resultMap="BaseResultMap">
    select * from payment where
    <if test="id != null ">
        id = #{id} and
    </if>
    <if test="serial != null and !serial != ''">
        serial like #{serial,jdbcType=VARCHAR} and
    </if>
</select>
```

> 可以看出这里通过if标签来判断对象参数中属性是否存在，且由于serial为字符串，需要判断是否为空字符串
>
> if标签中需要添加test属性，也就是判断条件，而判断条件为`OGNL表达式`
>
> 当test属性中需要添加多个条件时，不能使用&&，而是使用and
>
> 但是这里存在一个问题，就是无法判断哪里作为结尾合适，因为结尾肯定没有and？

#### 1.4.2 trim标签

由于上面在进行拼接时无法判断哪个条件作为结尾，所以最后的and成为问题，这里就可以使用trim标签进行补充

```xml
<select id="getPaymentCondition" resultMap="BaseResultMap">
    select * from payment
    <trim prefix="where" suffixOverrides="and">
        <if test="id != null ">
            id = #{id} and
        </if>
        <if test="serial != null and !serial != ''">
            serial like #{serial,jdbcType=VARCHAR} and
        </if>
    </trim>
</select>
```

> 在if标签外部添加了trim标签，trim标签中的prefix属性代表为下面的动态sql添加什么前缀，而suffixOverrides属性代表末尾如果存在对应的后缀时就去除

#### 1.4.3 foreach标签

mapper接口的方法

```java
Payment getPaymentCondition(@Param("ids") List<Integer> ids);
```

> 可以看出这里是传入了一个list作为参数，也就是对list中所有的数据进行查询

映射文件中的方法

```xml
<select id="getPaymentCondition" resultMap="BaseResultMap">
    select * from id in
    <foreach collection="ids" item="id" separator="," open="(" close=")">
        #{id,jdbcType=INTEGER}
    </foreach>
</select>
```

> 可以看出由于mapper方法中是对list集合中的数据进行遍历查询，所以这里一定是用where in
>
> 使用foreach标签进行遍历，这里的collection属性就是mapper方法中定义的参数，而方法中使用@Param注解对参数进行了命名
>
> item属性就是遍历出来的每一个参数值
>
> 由于sql语句中进行遍历格式为id in (  , , ,)，而又不能确定list中一共有多少个，所以使用separator属性表示遍历出来的每个参数之间的分隔符，而open属性表示sql语句的开头，close属性代表sql语句的结尾

#### 1.4.4 choose、when、otherwise标签

 由于if标签是直接判断条件是否成立，如果不成立就不执行，而如果类似于(if else)时就必须使用choose标签

```xml
<select id="findActiveBlogLike"
     resultType="Blog">
  SELECT * FROM BLOG WHERE state = ‘ACTIVE’
  <choose>
    <when test="title != null">
      AND title like #{title}
    </when>
    <when test="author != null and author.name != null">
      AND author_name like #{author.name}
    </when>
    <otherwise>
      AND featured = 1
    </otherwise>
  </choose>
</select>
```

### 1.5 代码生成器MBG

引入相关依赖

```xml
<dependency>
    <groupId>org.mybatis.generator</groupId>
    <artifactId>mybatis-generator-core</artifactId>
    <version>1.4.0</version>
</dependency>
```

#### 1.5.1 XML配置文件

[XML配置文件配置](https://juejin.cn/post/6844903982582743048#heading-3)

MBG的核心用于控制代码生成的所有行为，所有非标签独有的公共配置的`Key`可以在`mybatis-generator-core`的`PropertyRegistry`类中找到

```xml
<generatorConfiguration>
    <properties resource="generator.properties"/>
    <context id="MySqlContext" targetRuntime="MyBatis3" defaultModelType="flat">
        <property name="beginningDelimiter" value="`"/>
        <property name="endingDelimiter" value="`"/>
        <property name="javaFileEncoding" value="UTF-8"/>
        <!-- 为模型生成序列化方法-->
        <plugin type="org.mybatis.generator.plugins.SerializablePlugin"/>
        <!-- 为生成的Java模型创建一个toString方法 -->
        <plugin type="org.mybatis.generator.plugins.ToStringPlugin"/>
        <!--生成mapper.xml时覆盖原文件-->
        <plugin type="org.mybatis.generator.plugins.UnmergeableXmlMappersPlugin" />
        <commentGenerator type="com.macro.mall.CommentGenerator">
            <!-- 是否去除自动生成的注释 true：是 ： false:否 -->
            <property name="suppressAllComments" value="true"/>
            <property name="suppressDate" value="true"/>
            <property name="addRemarkComments" value="true"/>
        </commentGenerator>

        <jdbcConnection driverClass="${jdbc.driverClass}"
                        connectionURL="${jdbc.connectionURL}"
                        userId="${jdbc.userId}"
                        password="${jdbc.password}">
            <!--解决mysql驱动升级到8.0后不生成指定数据库代码的问题-->
            <property name="nullCatalogMeansCurrent" value="true" />
        </jdbcConnection>

        <javaModelGenerator targetPackage="com.macro.mall.model" targetProject="mall-mbg\src\main\java"/>

        <sqlMapGenerator targetPackage="com.macro.mall.mapper" targetProject="mall-mbg\src\main\resources"/>

        <javaClientGenerator type="XMLMAPPER" targetPackage="com.macro.mall.mapper"
                             targetProject="mall-mbg\src\main\java"/>
        <!--生成全部表tableName设为%-->
        <table tableName="%">
            <generatedKey column="id" sqlStatement="MySql" identity="true"/>
        </table>
    </context>
</generatorConfiguration>
```

最外部的标签为`<generatorConfiguration>`，它的子标签包括：

- 0或者1个`<properties>`标签，用于指定全局配置文件，下面可以通过占位符的形式读取`<properties>`指定文件中的值
- 1或者N个`<context>`标签，用于运行时的解析模式和具体的代码生成行为，所以这个标签里面的配置是最重要的

##### 1.5.1.1 context标签

`<context>`标签在`mybatis-generator-core`中对应的实现类为`org.mybatis.generator.config.Context`，它除了大量的子标签配置之外，比较主要的属性是：

- `id`：`Context`示例的唯一`ID`，用于输出错误信息时候作为唯一标记
- `targetRuntime`：用于执行代码生成模式
- `defaultModelType`：控制`Domain`类的生成行为。执行引擎为`MyBatis3DynamicSql`或者`MyBatis3Kotlin`时忽略此配置，可选值：
  - `conditional`：默认值，类似`hierarchical`，但是只有一个主键的时候会合并所有属性生成在同一个类
  - `flat`：所有内容全部生成在一个对象中
  - `hierarchical`：键生成一个XXKey对象，Blob等单独生成一个对象，其他简单属性在一个对象中

`targetRuntime`属性的可选值比较多，这里做个简单的小结：

|          属性          |                           功能描述                           |
| :--------------------: | :----------------------------------------------------------: |
|  `MyBatis3DynamicSql`  | 默认值，兼容`JDK8+`和`MyBatis 3.4.2+`，不会生成`XML`映射文件，忽略`<sqlMapGenerator>`的配置项，也就是`Mapper`全部注解化，依赖于`MyBatis Dynamic SQL`类库 |
|    `MyBatis3Kotlin`    |  行为类似于`MyBatis3DynamicSql`，不过兼容`Kotlin`的代码生成  |
|       `MyBatis3`       | 提供基本的基于动态`SQL`的`CRUD`方法和`XXXByExample`方法，会生成`XML`映射文件 |
|    `MyBatis3Simple`    |   提供基本的基于动态`SQL`的`CRUD`方法，会生成`XML`映射文件   |
| `MyBatis3DynamicSqlV1` |                     已经过时，不推荐使用                     |

一般需要把`SQL`文件和代码分离，所以选用`MyBatis3`或者`MyBatis3Simple`

`<context>`标签支持0或N个`<property>`标签，`<property>`的可选属性有：

|     property属性      |             功能描述             |          默认值          |                     备注                      |
| :-------------------: | :------------------------------: | :----------------------: | :-------------------------------------------: |
| `autoDelimitKeywords` | 是否使用分隔符号括住数据库关键字 |         `false`          |      例如`MySQL`中会使用反引号括住关键字      |
| `beginningDelimiter`  |        分隔符号的开始符号        |           `"`            |                                               |
|   `endingDelimiter`   |         分隔符号的结束号         |           `"`            |                                               |
|  `javaFileEncoding`   |            文件的编码            |       `系统默认值`       |       来源于`java.nio.charset.Charset`        |
|    `javaFormatter`    |        类名和文件格式化器        |  `DefaultJavaFormatter`  |   见`JavaFormatter`和`DefaultJavaFormatter`   |
|     `targetJava8`     |       是否JDK8和启动其特性       |          `true`          |                                               |
| `kotlinFileEncoding`  |         `Kotlin`文件编码         |       `系统默认值`       |       来源于`java.nio.charset.Charset`        |
|   `kotlinFormatter`   |    `Kotlin`类名和文件格式化器    | `DefaultKotlinFormatter` | 见`KotlinFormatter`和`DefaultKotlinFormatter` |
|    `xmlFormatter`     |        `XML`文件格式化器         |  `DefaultXmlFormatter`   |    见`XmlFormatter`和`DefaultXmlFormatter`    |

##### 1.5.1.2 jdbcConnection标签

`<jdbcConnection>`标签用于**指定数据源的连接信息**，它在`mybatis-generator-core`中对应的实现类为`org.mybatis.generator.config.JDBCConnectionConfiguration`，主要属性包括：

|      属性       |       功能描述       | 是否必须 |
| :-------------: | :------------------: | :------: |
|  `driverClass`  |  数据源驱动的全类名  |   `Y`    |
| `connectionURL` |  `JDBC`的连接`URL`   |   `Y`    |
|    `userId`     | 连接到数据源的用户名 |   `N`    |
|   `password`    |  连接到数据源的密码  |   `N`    |

> 可以在`<jdbcConnection>`标签添加properties标签用于解决mysql驱动升级到8.0后不生成指定数据库代码的问题

```xml
<jdbcConnection driverClass="${jdbc.driverClass}"
                connectionURL="${jdbc.connectionURL}"
                userId="${jdbc.userId}"
                password="${jdbc.password}">
    <!--解决mysql驱动升级到8.0后不生成指定数据库代码的问题-->
    <property name="nullCatalogMeansCurrent" value="true" />
</jdbcConnection>
```

##### 1.5.1.3 commentGenerator标签(可以自定义注释的实现)

`<commentGenerator>`标签是可选的，用于**控制生成的实体的注释内容**。它在`mybatis-generator-core`中对应的实现类为`org.mybatis.generator.internal.DefaultCommentGenerator`，可以通过可选的`type`属性指定一个自定义的`CommentGenerator`实现。`<commentGenerator>`标签支持0或N个`<property>`标签，`<property>`的可选属性有：

|     property属性      |                   功能描述                   |           默认值            |
| :-------------------: | :------------------------------------------: | :-------------------------: |
| `suppressAllComments` |               是否去掉生成注释               |           `false`           |
|    `suppressDate`     |         是否在注释中去掉生成的时间戳         |           `false`           |
|     `dateFormat`      | 配合`suppressDate`使用，指定输出时间戳的格式 | `java.util.Date#toString()` |
|  `addRemarkComments`  |      是否去掉输出表和列的`Comment`信息       |           `false`           |

##### 1.5.1.4 javaTypeResolver标签

`<javaTypeResolver>`标签是`<context>`的子标签，用于解析和计算数据库列类型和`Java`类型的映射关系，该标签只包含一个`type`属性，用于指定`org.mybatis.generator.api.JavaTypeResolver`接口的实现类。`<javaTypeResolver>`标签支持0或N个`<property>`标签，`<property>`的可选属性有：

|    property属性    |                           功能描述                           | 默认值  |
| :----------------: | :----------------------------------------------------------: | :-----: |
| `forceBigDecimals` | 是否强制把所有的数字类型强制使用`java.math.BigDecimal`类型表示 | `false` |
|  `useJSR310Types`  |         是否支持`JSR310`，主要是`JSR310`的新日期类型         | `false` |

如果`useJSR310Types`属性设置为`true`，那么生成代码的时候类型映射关系如下（主要针对日期时间类型）：

|    数据库（JDBC）类型     |          Java类型          |
| :-----------------------: | :------------------------: |
|          `DATE`           |   `java.time.LocalDate`    |
|          `TIME`           |   `java.time.LocalTime`    |
|        `TIMESTAMP`        | `java.time.LocalDateTime`  |
|   `TIME_WITH_TIMEZONE`    |   `java.time.OffsetTime`   |
| `TIMESTAMP_WITH_TIMEZONE` | `java.time.OffsetDateTime` |

有些时候，我们希望`INTEGER`、`SMALLINT`和`TINYINT`都映射为`Integer`，那么我们需要覆盖`JavaTypeResolverDefaultImpl`的构造方法：

```java
public class DefaultJavaTypeResolver extends JavaTypeResolverDefaultImpl {

    public DefaultJavaTypeResolver() {
        super();
        typeMap.put(Types.SMALLINT, new JdbcTypeInformation("SMALLINT",
                new FullyQualifiedJavaType(Integer.class.getName())));
        typeMap.put(Types.TINYINT, new JdbcTypeInformation("TINYINT",
                new FullyQualifiedJavaType(Integer.class.getName())));
    }
}
```

##### 1.5.1.5 javaModelGenerator标签

`<javaModelGenerator标签>`标签是`<context>`的子标签，主要用于控制实体（`Model`）类的代码生成行为。它支持的属性如下：

|      属性       |                  功能描述                  | 是否必须 |            备注            |
| :-------------: | :----------------------------------------: | :------: | :------------------------: |
| `targetPackage` |             生成的实体类的包名             |   `Y`    | 例如`club.throwable.model` |
| `targetProject` | 生成的实体类文件相对于项目（根目录）的位置 |   `Y`    |    例如`src/main/java`     |

`<javaModelGenerator标签>`标签支持0或N个`<property>`标签，`<property>`的可选属性有：

|      property属性      |                          功能描述                           | 默认值  |                             备注                             |
| :--------------------: | :---------------------------------------------------------: | :-----: | :----------------------------------------------------------: |
|   `constructorBased`   |           是否生成一个带有所有字段属性的构造函数            | `false` |             `MyBatis3Kotlin`模式下忽略此属性配置             |
|  `enableSubPackages`   |                是否允许通过`Schema`生成子包                 | `false` | 如果为`true`，例如包名为`club.throwable`，如果`Schema`为`xyz`，那么实体类文件最终会生成在`club.throwable.xyz`目录 |
| `exampleTargetPackage` |             生成的伴随实体类的`Example`类的包名             |    -    |                              -                               |
| `exampleTargetProject` | 生成的伴随实体类的`Example`类文件相对于项目（根目录）的位置 |    -    |                              -                               |
|      `immutable`       |                         是否不可变                          | `false` | 如果为`true`，则不会生成`Setter`方法，所有字段都使用`final`修饰，提供一个带有所有字段属性的构造函数 |
|      `rootClass`       |                   为生成的实体类添加父类                    |    -    |               通过`value`指定父类的全类名即可                |
|     `trimStrings`      |       `Setter`方法是否对字符串类型进行一次`trim`操作        | `false` |                              -                               |

##### 1.5.1.6 javaClientGenerator标签

`<javaClientGenerator>`标签是`<context>`的子标签，主要用于控制`Mapper`接口的代码生成行为。它支持的属性如下：

|      属性       |                     功能描述                     | 是否必须 |                             备注                             |
| :-------------: | :----------------------------------------------: | :------: | :----------------------------------------------------------: |
|     `type`      |               `Mapper`接口生成策略               |   `Y`    | `<context>`标签的`targetRuntime`属性为`MyBatis3DynamicSql`或者`MyBatis3Kotlin`时此属性配置忽略 |
| `targetPackage` |             生成的`Mapper`接口的包名             |   `Y`    |                 例如`club.throwable.mapper`                  |
| `targetProject` | 生成的`Mapper`接口文件相对于项目（根目录）的位置 |   `Y`    |                     例如`src/main/java`                      |

`type`属性的可选值如下：

- `ANNOTATEDMAPPER`：`Mapper`接口生成的时候依赖于注解和`SqlProviders`（也就是纯注解实现），不会生成`XML`映射文件。
- `XMLMAPPER`：`Mapper`接口生成接口方法，对应的实现代码生成在`XML`映射文件中（也就是纯映射文件实现）。
- `MIXEDMAPPER`：`Mapper`接口生成的时候复杂的方法实现生成在`XML`映射文件中，而简单的实现通过注解和`SqlProviders`实现（也就是注解和映射文件混合实现）

注意两点：

- `<context>`标签的`targetRuntime`属性指定为`MyBatis3Simple`的时候，`type`只能选用`ANNOTATEDMAPPER`或者`XMLMAPPER`
- `<context>`标签的`targetRuntime`属性指定为`MyBatis3`的时候，`type`可以选用`ANNOTATEDMAPPER`、`XMLMAPPER`或者`MIXEDMAPPER`

`<javaClientGenerator>`标签支持0或N个`<property>`标签，`<property>`的可选属性有：

|    property属性     |              功能描述              | 默认值  |                             备注                             |
| :-----------------: | :--------------------------------: | :-----: | :----------------------------------------------------------: |
| `enableSubPackages` |    是否允许通过`Schema`生成子包    | `false` | 如果为`true`，例如包名为`club.throwable`，如果`Schema`为`xyz`，那么`Mapper`接口文件最终会生成在`club.throwable.xyz`目录 |
| `useLegacyBuilder`  | 是否通过`SQL Builder`生成动态`SQL` | `false` |                                                              |
|   `rootInterface`   |   为生成的`Mapper`接口添加父接口   |    -    |              通过`value`指定父接口的全类名即可               |

##### 1.5.1.7 sqlMapGenerator标签

`<sqlMapGenerator>`标签是`<context>`的子标签，主要用于控制`XML`映射文件的代码生成行为。它支持的属性如下：

|      属性       |                   功能描述                    | 是否必须 |           备注           |
| :-------------: | :-------------------------------------------: | :------: | :----------------------: |
| `targetPackage` |           生成的`XML`映射文件的包名           |   `Y`    |      例如`mappings`      |
| `targetProject` | 生成的`XML`映射文件相对于项目（根目录）的位置 |   `Y`    | 例如`src/main/resources` |

`<sqlMapGenerator>`标签支持0或N个`<property>`标签，`<property>`的可选属性有：

|    property属性     |           功能描述           | 默认值  | 备注 |
| :-----------------: | :--------------------------: | :-----: | :--: |
| `enableSubPackages` | 是否允许通过`Schema`生成子包 | `false` |  -   |

##### 1.5.1.8 plugin标签

`<plugin>`标签是`<context>`的子标签，用于引入一些插件对代码生成的一些特性进行扩展，该标签只包含一个`type`属性，用于指定`org.mybatis.generator.api.Plugin`接口的实现类。内置的插件实现见[Supplied Plugins](http://mybatis.org/generator/reference/plugins.html)

例如：引入`org.mybatis.generator.plugins.SerializablePlugin`插件会让生成的实体类自动实现`java.io.Serializable`接口并且添加`serialVersionUID`属性

> 常用的插件

```xml
<!-- 为模型生成序列化方法-->
<plugin type="org.mybatis.generator.plugins.SerializablePlugin"/>
<!-- 为生成的Java模型创建一个toString方法 -->
<plugin type="org.mybatis.generator.plugins.ToStringPlugin"/>
<!--生成mapper.xml时覆盖原文件-->
<plugin type="org.mybatis.generator.plugins.UnmergeableXmlMappersPlugin" />
```

##### 1.5.1.9 table标签

<table>标签是<context>的子标签，主要用于配置要生成代码的数据库表格，定制一些代码生成行为等等。它支持的属性众多，列举如下：

|            属性             |                       功能描述                        | 是否必须 |                             备注                             |
| :-------------------------: | :---------------------------------------------------: | :------: | :----------------------------------------------------------: |
|         `tableName`         |                     数据库表名称                      |   `Y`    |                        例如`t_order`                         |
|          `schema`           |                    数据库`Schema`                     |   `N`    |                              -                               |
|          `catalog`          |                    数据库`Catalog`                    |   `N`    |                              -                               |
|           `alias`           |                      表名称标签                       |   `N`    |    如果指定了此值，则查询列的时候结果格式为`alias_column`    |
|     `domainObjectName`      |       表对应的实体类名称，可以通过`.`指定包路径       |   `N`    |   如果指定了`bar.User`，则包名为`bar`，实体类名称为`User`    |
|        `mapperName`         |   表对应的`Mapper`接口类名称，可以通过`.`指定包路径   |   `N`    | 如果指定了`bar.UserMapper`，则包名为`bar`，`Mapper`接口类名称为`UserMapper` |
|      `sqlProviderName`      |         动态`SQL`提供类`SqlProvider`的类名称          |   `N`    |                              -                               |
|       `enableInsert`        |               是否允许生成`insert`方法                |   `N`    | 默认值为`true`，执行引擎为`MyBatis3DynamicSql`或者`MyBatis3Kotlin`时忽略此配置 |
| `enableSelectByPrimaryKey`  |         是否允许生成`selectByPrimaryKey`方法          |   `N`    | 默认值为`true`，执行引擎为`MyBatis3DynamicSql`或者`MyBatis3Kotlin`时忽略此配置 |
|   `enableSelectByExample`   |           是否允许生成`selectByExample`方法           |   `N`    | 默认值为`true`，执行引擎为`MyBatis3DynamicSql`或者`MyBatis3Kotlin`时忽略此配置 |
| `enableUpdateByPrimaryKey`  |         是否允许生成`updateByPrimaryKey`方法          |   `N`    | 默认值为`true`，执行引擎为`MyBatis3DynamicSql`或者`MyBatis3Kotlin`时忽略此配置 |
| `enableDeleteByPrimaryKey`  |         是否允许生成`deleteByPrimaryKey`方法          |   `N`    | 默认值为`true`，执行引擎为`MyBatis3DynamicSql`或者`MyBatis3Kotlin`时忽略此配置 |
|   `enableDeleteByExample`   |           是否允许生成`deleteByExample`方法           |   `N`    | 默认值为`true`，执行引擎为`MyBatis3DynamicSql`或者`MyBatis3Kotlin`时忽略此配置 |
|   `enableCountByExample`    |           是否允许生成`countByExample`方法            |   `N`    | 默认值为`true`，执行引擎为`MyBatis3DynamicSql`或者`MyBatis3Kotlin`时忽略此配置 |
|   `enableUpdateByExample`   |           是否允许生成`updateByExample`方法           |   `N`    | 默认值为`true`，执行引擎为`MyBatis3DynamicSql`或者`MyBatis3Kotlin`时忽略此配置 |
| `selectByPrimaryKeyQueryId` |        `value`指定对应的主键列提供列表查询功能        |   `N`    | 执行引擎为`MyBatis3DynamicSql`或者`MyBatis3Kotlin`时忽略此配置 |
|  `selectByExampleQueryId`   |       `value`指定对应的查询`ID`提供列表查询功能       |   `N`    | 执行引擎为`MyBatis3DynamicSql`或者`MyBatis3Kotlin`时忽略此配置 |
|         `modelType`         |        覆盖`<context>`的`defaultModelType`属性        |   `N`    |            见`<context>`的`defaultModelType`属性             |
|      `escapeWildcards`      |                 是否对通配符进行转义                  |   `N`    |                              -                               |
|    `delimitIdentifiers`     | 标记匹配表名称的时候是否需要使用分隔符去标记生成的SQL |   `N`    |                              -                               |
|     `delimitAllColumns`     |               是否所有的列都添加分隔符                |   `N`    | 默认值为`false`，如果设置为`true`，所有列名会添加起始和结束分隔符 |

`<table>`标签支持0或N个`<property>`标签，`<property>`的可选属性有：

|        property属性         |                          功能描述                          | 默认值  |                             备注                             |
| :-------------------------: | :--------------------------------------------------------: | :-----: | :----------------------------------------------------------: |
|     `constructorBased`      |         是否为实体类生成一个带有所有字段的构造函数         | `false` |          执行引擎为`MyBatis3Kotlin`的时候此属性忽略          |
| `ignoreQualifiersAtRuntime` |                    是否在运行时忽略别名                    | `false` | 如果为`true`，则不会在生成表的时候把`schema`和`catalog`作为表的前缀 |
|         `immutable`         |                      实体类是否不可变                      | `false` |          执行引擎为`MyBatis3Kotlin`的时候此属性忽略          |
|         `modelOnly`         |                     是否仅仅生成实体类                     | `false` |                              -                               |
|         `rootClass`         |         如果配置此属性，则实体类会继承此指定的超类         |   `-`   |             如果有主键属性会把主键属性在超类生成             |
|       `rootInterface`       |         如果配置此属性，则实体类会实现此指定的接口         |   `-`   | 执行引擎为`MyBatis3Kotlin`或者`MyBatis3DynamicSql`的时候此属性忽略 |
|      `runtimeCatalog`       |                   指定运行时的`Catalog`                    |   `-`   | 当生成表和运行时的表的`Catalog`不一样的时候可以使用该属性进行配置 |
|       `runtimeSchema`       |                    指定运行时的`Schema`                    |   `-`   | 当生成表和运行时的表的`Schema`不一样的时候可以使用该属性进行配置 |
|     `runtimeTableName`      |                     指定运行时的表名称                     |   `-`   | 当生成表和运行时的表的表名称不一样的时候可以使用该属性进行配置 |
|  `selectAllOrderByClause`   |  指定字句内容添加到`selectAll()`方法的`order by`子句之中   |   `-`   |         执行引擎为`MyBatis3Simple`的时候此属性才适用         |
|        `trimStrings`        |            实体类的字符串类型属性会做`trim`处理            |   `-`   |          执行引擎为`MyBatis3Kotlin`的时候此属性忽略          |
|   `useActualColumnNames`    |               是否使用列名作为实体类的属性名               | `false` |                              -                               |
|     `useColumnIndexes`      | `XML`映射文件中生成的`ResultMap`使用列索引定义而不是列名称 | `false` | 执行引擎为`MyBatis3Kotlin`或者`MyBatis3DynamicSql`的时候此属性忽略 |
| `useCompoundPropertyNames`  |         是否把列名和列备注拼接起来生成实体类属性名         | `false` |                              -                               |

`<table>`标签还支持众多的**非**`property`的子标签：

- 0或1个`<generatedKey>`用于指定主键生成的规则，指定此标签后会生成一个`<selectKey>`标签：

  ```xml
  <!-- column：指定主键列 -->
  <!-- sqlStatement：查询主键的SQL语句，例如填写了MySql，则使用SELECT LAST_INSERT_ID() -->
  <!-- type：可选值为pre或者post，pre指定selectKey标签的order为BEFORE，post指定selectKey标签的order为AFTER -->
  <!-- identity：true的时候，指定selectKey标签的order为AFTER -->
  <generatedKey column="id" sqlStatement="MySql" type="post" identity="true" />
  ```

- 0或1个`<domainObjectRenamingRule>`用于指定实体类重命名规则：

  ```xml
  <!-- searchString中正则命中的实体类名部分会替换为replaceString -->
  <domainObjectRenamingRule searchString="^Sys" replaceString=""/>
  <!-- 例如 SysUser会变成User -->
  <!-- 例如 SysUserMapper会变成UserMapper -->
  ```

- 0或N个`<columnOverride>`用于指定具体列的覆盖映射规则：

  ```xml
  <!-- column：指定要覆盖配置的列 -->
  <!-- property：指定要覆盖配置的属性 -->
  <!-- delimitedColumnName：是否为列名添加定界符，例如`{column}` -->
  <!-- isGeneratedAlways：是否一定生成此列 -->
  <columnOverride column="customer_name" property="customerName" javaType="" jdbcType="" typeHandler="" delimitedColumnName="" isGeneratedAlways="">
     <!-- 覆盖table或者javaModelGenerator级别的trimStrings属性配置 -->
     <property name="trimStrings" value="true"/>
  <columnOverride/>
  ```

- 0或N个`<ignoreColumn>`用于指定忽略生成的列：

  ```xml
  <ignoreColumn column="version" delimitedColumnName="false"/>
  ```

##### 1.5.1.10 context子元素配置顺序

- **property** (0..N)
- **plugin** (0..N)
- **commentGenerator** (0 or 1)
- **jdbcConnection** (需要connectionFactory 或 jdbcConnection)
- **javaTypeResolver** (0 or 1)
- **javaModelGenerator** (至少1个)
- **sqlMapGenerator** (0 or 1)
- **javaClientGenerator** (0 or 1)
- **table** (1..N)

#### 1.5.2 运行Mybatis Generator

##### 1.5.2.1 使用Java运行MBG

```java
public class Generator {
    public static void main(String[] args) throws Exception {
        //MBG 执行过程中的警告信息
        List<String> warnings = new ArrayList<String>();
        //当生成的代码重复时，覆盖原代码
        boolean overwrite = true;
        //读取我们的 MBG 配置文件
        InputStream is = Generator.class.getResourceAsStream("/generatorConfig.xml");
        ConfigurationParser cp = new ConfigurationParser(warnings);
        Configuration config = cp.parseConfiguration(is);
        is.close();

        DefaultShellCallback callback = new DefaultShellCallback(overwrite);
        //创建 MBG
        MyBatisGenerator myBatisGenerator = new MyBatisGenerator(config, callback, warnings);
        //执行生成代码
        myBatisGenerator.generate(null);
        //输出警告信息
        for (String warning : warnings) {
            System.out.println(warning);
        }
    }
}
```

##### 1.5.2.2 使用maven插件生成

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.mybatis.generator</groupId>
            <artifactId>mybatis-generator-maven-plugin</artifactId>
            <version>1.3.7</version>
        </plugin>
    <plugins>    
</build>
```

通过maven插件生成代码

![16e12ab5728f79ce](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/16e12ab5728f79ce.jpg)

#### 1.5.3 扩展Example类

### 1.6 PageHelper分页插件



### 1.7 SpringBoot整合Mybatis

## 2 Spring

### 2.1 Spring概述

#### 2.1.1 模块划分                       

![1](https://learning-pool.oss-cn-chengdu.aliyuncs.com/netty/1.png)

> 核心容器：主要是容器IOC管理(包括SpEL表达式)
>
> AOP和Aspects：主要是面向切面编程
>
> Data Access：主要是和数据库对接，比如JDBC、ORM和事务，但是一般使用JDBC
>
> Web：主要是做Web应用，包括Servlet；就是Spring MVC部分

#### 2.1.2 IOC和DI

IOC：控制反转，本质上就是由主动获取资源转变为被动获取资源

所有组件的创建、管理交给IOC容器

DI:IOC属于一种思想，而DI就是对其实现，依赖注入，本质上就是通过反射创建对象，比如通过构造器或者Setter

### 2.2 XML配置驱动

#### 2.2.1 简单测试

创建Person类用于测试

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
@ToString
public class Person {
    private Long id;
    private String name;
    private Integer age;
}
```

创建Spring配置文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean class="org.jiang.model.Person" id="person">
        <property name="id" value="1"></property>
        <property name="name" value="jiang"></property>
        <property name="age" value="12"></property>
    </bean>
</beans>
```

> 可以看出配置文件最外层是一个beans标签，内部可以定义不同的bean标签，而一个bean标签就对应容器中的一个实例
>
> bean标签需要指定class就是组件的类，id就是该组件的名称
>
> bean标签内部可以通过property标签对组件的属性赋值，这种方式是通过setter进行赋值

获取容器以及组件

```java
public class Test01 {
    @Test
    public void test01() {
        ClassPathXmlApplicationContext ctx = new ClassPathXmlApplicationContext("SpringConfig.xml");
        Person person = (Person) ctx.getBean("person");
        System.out.println(person);
    }
}
```

> 可以看出这里通过配置文件的路径创建了IOC容器，从IOC容器中通过组件id获取了对应的组件

结果：

```bash
Person(id=1, name=jiang, age=12)
```

**测试通过property标签赋值实际上是通过反射调用setter方法进行赋值**

修改Person类id属性的setter方法

```java
public void setId(Long id) {
    this.id = id;
    System.out.println("这里id被赋值!");
}
```

重新通过IOC容器获取对象结果：

```bash
这里id被赋值!
Person(id=1, name=jiang, age=12)
```

可以看出这里先是调用id属性的setter方法

#### 2.2.2 通过有参构造器为属性赋值

Spring配置文件配置bean

```xml
<bean class="org.jiang.model.Person" id="person">
    <constructor-arg name="id" value="1"></constructor-arg>
    <constructor-arg name="name" value="jiang"></constructor-arg>
    <constructor-arg name="age" value="12"></constructor-arg>
</bean>
```

> 可以看出本质上就是通过全参构造器反射创建组件，依然可以从IOC容器中获取组件

#### 2.2.3 model类中组合其他model类

```java
@NoArgsConstructor
@AllArgsConstructor
@ToString
@Data
public class Person {
    private Long id;
    private String name;
    private Integer age;
    
    private Car car;
}
```

> 这里Person类中组合了Car类

##### 2.2.3.1 方式一：引用外部bean

Spring配置文件配置bean

```xml
<bean class="org.jiang.model.Car" id="car">
    <property name="name" value="宝马"></property>
    <property name="color" value="blue"></property>
</bean>

<bean class="org.jiang.model.Person" id="person">
    <property name="id" value="1"></property>
    <property name="name" value="jiang"></property>
    <property name="age" value="12"></property>
    <property name="car" ref="car"></property>
</bean>
```

> 可以看出在配置文件中定义了两个组件，而person组件在设置car属性时通过ref指定了引用的bean(id)

**如果修改了IOC容器中的car组件，那么person组件中的car属性是否会随之改变呢？**

```java
public class Test01 {
    @Test
    public void test01() {
        ClassPathXmlApplicationContext ctx = new ClassPathXmlApplicationContext("SpringConfig.xml");
        Car car = (Car) ctx.getBean("car");
        System.out.println(car);
        car.setColor("green");
        Person person = (Person) ctx.getBean("person");
        System.out.println(person.getCar());

    }
```

结果：

```bash
Car(name=宝马, color=blue)
Car(name=宝马, color=green)
```

> 可以看出person组件中的car属性随之改变，因为car属性只是对car组件的引用

##### 2.2.3.2 方式二：内部配置bean

Spring配置文件配置bean

```xml
<bean class="org.jiang.model.Person" id="person">
    <property name="id" value="1"></property>
    <property name="name" value="jiang"></property>
    <property name="age" value="12"></property>
    <property name="car">
        <bean class="org.jiang.model.Car" id="car2">
            <property name="name" value="奔驰"></property>
            <property name="color" value="green"></property>
        </bean>
    </property>
</bean>
```

> 可以看出这里就是在property标签内部又定义了一个bean标签，而bean标签重新配置了一个car2组件

**那么这里的car2组件是否可以直接获取到呢？**

```java
public class Test01 {
    @Test
    public void test01() {
        ClassPathXmlApplicationContext ctx = new ClassPathXmlApplicationContext("SpringConfig.xml");
        Car car2 = (Car) ctx.getBean("car2");
        System.out.println(car2);
    }
}
```

结果：

```bash
org.springframework.beans.factory.NoSuchBeanDefinitionException: No bean named 'car2' available
```

> 可以看出内部创建的bean组件并不能直接获取