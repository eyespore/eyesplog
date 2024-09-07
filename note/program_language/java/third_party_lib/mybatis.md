# Mybatis

中文官方网站：https://mybatis.net.cn/，点评是十分易读好懂

## 快速开始

MVC框架中作为持久层存在，也可以单独用于操作数据库，与传统ORM持久层框架不同，Mybatis采用基于Mapper映射器的关系，简化对数据库的CRUD操作

在项目中引入依赖：

```xml
<dependency>  <!-- 持久层 -->
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.5.14</version>
</dependency>

<dependency>  <!-- mysql驱动，取决于使用什么数据库 -->
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.30</version>
</dependency>

<dependency>  <!-- 简化pojo书写 -->
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.26</version>
</dependency>
```

`mybatis`需要进行正确的配置才能使用，其中包括：

- 数据库连接信息
- 事务管理器
- 指定包扫描，从而构建mapper
- 返回对象的别名等信息

习惯性地，一般直接在`resources`目录下创建一份名为`mybatis-config.xml`的配置文件和一份名为`jdbc.properties`的数据库连接信息配置文件

> 也可以直接将连接信息写在`mybatis-config.xml`当中，只是这种分文件的写法要更灵活一些

`mybatis-config-xml`：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <properties resource="jdbc.properties"/>
    <!-- 字段小驼峰转下划线支持 -->
    <settings>
        <setting name="mapUnderscoreToCamelCase" value="true"/>
    </settings>
    <typeAliases>
        <package name="club.pineclone.pojo"/>
    </typeAliases>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="${jdbc.driver}"/>
                <property name="url" value="${jdbc.url}"/>
                <property name="username" value="${jdbc.username}"/>
                <property name="password" value="${jdbc.password}"/>
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <package name="club.pineclone.mapper"/>
    </mappers>
</configuration>
```

`jdbc.properties`：

```properties
jdbc.url=jdbc:mysql://localhost:3306/database1?useSSL=false
jdbc.username=root
jdbc.password=1234
jdbc.driver=com.mysql.cj.jdbc.Driver
```

编写pojo：以下仅仅为示例，按照项目需求编写即可
`club.pineclone.pojo.User.java`

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class User {
    private Integer id;
    private String username;
    private String password;
}
```

编写mapper映射器：sql可以是**配置文件形式**、也可以是**注解形式**，通常前者用于复杂场景，后者用于简单场景：

配置形式：

`club.pineclone.mapper.UserMapper.java`：

```java
public interface UserMapper {
    List<User> selectAll();
}
```

`club/pineclone/mapper/UserMapper.xml`
（注：mapper配置文件需要和类有同样的层级结构才能构成映射关系）

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="club.pineclone.mapper.UserMapper">
    <select id="selectAll" resultType="User">
        select * from user
    </select>
</mapper>
```

注解形式：

```java
public interface UserMapper {
    @Select("select * from user")
    List<User> selectAll();
}
```

mybaits的核心在于`SqlSessionFactory`提供的API，因此首先需要构建`SqlSessionFactory`：

```java
String resource = "mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```

通过`sqlSessionFactory`获得mapper对象来执行方法操作数据库

```java
SqlSession session = sqlSessionFactory.openSession();
UserMapper mapper = session.getMapper(UserMapper.class);
System.out.println(mapper.selectAll());
session.commit();  // 默认开启事务
session.close();  // 关闭资源
```

## CRUD

### 简单SQL

简单SQL可以直接使用注解形式书写：

```java
@Insert("insert into users(name, age) VALUES (#{user.name}, #{user.age}")
int insertUser(Users user);

@Delete("delete from users where id = #{id}")
int deleteById(Long id);

@Select("select name, age from users where id = #{id}")
User selectById(Long id);

@Update("update users set name = #{name} where id = #{id}")
int updateById(User user);
```

### 复杂SQL

复杂SQL建议使用`mapper.xml`的形式进行配置，合理利用MyBatis提供的动态 SQL（Dynamic SQL）来编写复杂的SQL指令

**插入**

批量插入：

```xml
<insert id="insertBatch">
    insert into dish_flavor(dish_id, name, value)
    values
    <foreach collection="dishFlavors" item="flavor" separator=",">
        (#{flavor.dishId}, #{flavor.name}, #{flavor.value})
    </foreach>
</insert>
```

插入后返回主键：

```xml
<insert id="insert" useGeneratedKeys="true" keyProperty="id">
    insert into users (name, age)
    values (#{name}, #{age})
</insert>
```

***

**删除**

批量删除：

```xml
<delete id="deleteBatchByDishIds">
    delete
    from dish_flavor
    where dish_id in
    <foreach collection="dishIds" item="dishId" open="(" close=")" separator=",">
        #{dishId}
    </foreach>
</delete>
```

***

**修改**

条件插入：

```xml
<update id="update" parameterType="Category">
    update category
    <set>
        <if test="type != null">type = #{type},</if>
        <if test="name != null">name = #{name},</if>
        <if test="sort != null">sort = #{sort},</if>
        <if test="status != null">status = #{status},</if>
        <if test="updateTime != null">update_time = #{updateTime},</if>
        <if test="updateUser != null">update_user = #{updateUser}</if>
    </set>
    where id = #{id}
</update>
```

> 仅仅在值存在的情况下执行更新操作

***

**查询**

条件查询：

```xml
<select id="pageQuery" resultType="com.sky.entity.Category">
    select * from category
    <where>
        <if test="name != null and name != ''">and name like concat('%',#{name},'%')</if>
        <if test="type != null">and type = #{type}</if>
    </where>
    order by sort asc , create_time desc
</select>
```

> 按照`sort`列进行升序排列，若`sort`字段值相同，则按`create_time`的值降序排列

连接查询：

```xml
<select id="queryPage" resultType="com.sky.vo.DishVO">
    select d.*, c.`name` as `categoryName`
    from dish d
    left join category c on d.category_id = c.id
    <where>
        <if test="name != null and name != ''">
            and d.name like concat('%', #{name} ,'%')
        </if>
        <if test="categoryId != null and categoryId != ''">
            and d.category_id = #{categoryId}
        </if>
        <if test="status != null">
                and d.status = #{status}
        </if>
    </where>
    order by d.create_time desc
</select>
```

### 注意事项

1. 关于`#{}`和`${}`占位符

   **`${}`**：用于直接文本替换，不会进行参数的转义或类型检查，存在 SQL 注入风险，适用于动态 SQL 构建场景，如表名或列名的动态设置，例如：

   ```sql
   SELECT * FROM ${tableName} WHERE ${columnName} = 'some_value';
   ```

   ***

   **`#{}`**：用于安全地传递参数，MyBatis 会对参数进行预编译处理，避免 SQL 注入，适用于大多数传递参数的场景，如查询条件或插入数据，例如：

   ```sql
   SELECT * FROM users WHERE username = #{username};
   ```



## 更换数据源

作为ORM框架，mybatis通过与数据库建立连接来工作，通常情况下，当`dataSource`标签的type属性值为`POOLED`时，mybatis使用其**内置**的连接池来管理连接

在实际中，我们会倾向于选择提供了更好的性能优化、监控管理能力、多数据库支持的数据源，而mybaits内置的数据源往往不能满足需求，此时需要更换为第三方数据源，常用第三方数据源如下：

- DBCP
- C3P0
- **Druid**：性能也比较好，提供了比较便捷的监控系统
- Hikari：性能最好

### druid

将mybaits数据源切换为druid，为项目引入druid依赖：

```xml
<dependency>
	<groupId>com.alibaba</groupId>
	<artifactId>druid</artifactId>
	<version>1.2.1</version>
</dependency>  
```

参考mybaits的配置文件中对`datasource`的配置：

```xml
<dataSource type="POOLED">
    <property name="driver" value="${jdbc.driver}"/>
    <property name="url" value="${jdbc.url}"/>
    <property name="username" value="${jdbc.username}"/>
    <property name="password" value="${jdbc.password}"/>
</dataSource>
```

我们通过`type`来指定类名，但是由于mybatis要求这里必须是`DataSourceFacotry`的实现类，因此这里不能直接指定`com.alibaba.pool.DruidDataSource`

可以手动创建一个实现类，让它返回`DruidDataSource`即可：

```java
public class DruidDataSourceFactory implements DataSourceFactory {
    @Override
    public void setProperties(Properties props) {}

    @Override
    public DataSource getDataSource() {
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setUrl("jdbc:mysql://127.0.0.1:3306/pineclone?useSSL=false");
        dataSource.setUsername("root");
        dataSource.setPassword("1234");
        dataSource.setDriverClassName("com.mysql.jdbc.Driver");
        return dataSource;
    }
}
```

当然，和数据源相关的配置也可以一并在此处配置好，大可以将`mybatis-config.xml`当中的配置给注释掉：

```xml
            <dataSource type="club.pineclone.utils.DruidDataSourceFactory">
<!--                <property name="driver" value="${jdbc.driver}"/>-->
<!--                <property name="url" value="${jdbc.url}"/>-->
<!--                <property name="username" value="${jdbc.username}"/>-->
<!--                <property name="password" value="${jdbc.password}"/>-->
            </dataSource>
```

> 不过使用什么样的手法来配置DruidDataSource是自由的，并不拘泥于这种方式，您也许注意到了上面`DataSourceFactory`接口的`setProperties`方法，它同样能用于配置参数