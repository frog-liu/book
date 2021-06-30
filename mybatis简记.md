## Mybatis

1. **介绍**

   ​		Mybatis是一个开源的、轻量级、持久化框架，能够自动完成java对象和数据表之间的映射关系。Mybatis封装了几乎所有JDBC的代码，通过xml配置文件可实现sql与业务逻辑的解耦，同时Mybatis减少了不必要的参数配置，并支持自定义sql、存储过程和高级映射。

2. **XML文件配置Mybatis**

   ```xml
   <configuration>
      <typeAliases>
         <typeAlias alias = "class_alias_Name" type = "absolute_clas_Name"/>
      </typeAliases>
   		
      <environments default = "environment_id">
         <environment id = "environment_id">
            <transactionManager type = "JDBC/MANAGED"/>  
               <dataSource type = "UNPOOLED/POOLED/JNDI">
                  <property name = "driver" value = "database_driver_class_name"/>
                  <property name = "url" value = "database_url"/>
                  <property name = "username" value = "database_user_name"/>
                  <property name = "password" value = "database_password"/>
               </dataSource>        
         </environment>
      </environments>
   	
      <mappers>
         <mapper resource = "path of the configuration XML file"/>
      </mappers>
   </configuration>
   ```

   ​		不同环境的database配置可能不同，**environments**标签里面包含多个**environment**标签，每个**environment**标签对应一个环境的database配置。**environments**标签可以通过**default**参数设置默认启用的**environment**配置，值与**environment**标签的id对应。

   ​		每个**environment**标签，包含一个**transactionManager**标签和一个**dataSource**标签，**transactionManager**标签类型分两种，**JDBC**和**MANAGED**，**dataSource**标签类型分为三种，**UNPOOLED**、**POOLED** 和 **JNDI**，**UNPOOLED**每次请求都会发起一个新的连接，结束后关闭；**POOLED**会创建一个连接池，每次请求会从连接池获取连接，可以减少连接初始化和验证的时间。**dataSource**标签里面可以包含多个**property**标签，用来配置数据库链接的driver、url、username、password。

   ​		其次，typeAliases标签，避免对象路径过长，我们可以使用该标签来替代该对象。例如

   ```xml
   <typeAliases>
      <typeAlias alias = "student" type = "com.spring.boot.demo.Student"/>
   </typeAliases>
   ```

3. **CURD**

   在mapper.xml文件中，使用select、update、insert、delete标签完成的jdbc的curd操作:

   ```xml
   <?xml version = "1.0" encoding = "UTF-8"?>
   
   <!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
   	
   <mapper namespace = "Student">	
      <resultMap id = "result" type = "Student">
      	  <result property = "id" column = "ID"/>
      		<result property = "name" column = "NAME"/>
      		<result property = "branch" column = "BRANCH"/>
      		<result property = "percentage" column = "PERCENTAGE"/>
      		<result property = "phone" column = "PHONE"/>
      		<result property = "email" column = "EMAIL"/>
   	 </resultMap>
   	
      <select id = "getAll" resultMap = "result">
         SELECT * FROM STUDENT; 
      </select>
       
      <select id = "getById" parameterType = "int" resultMap = "result">
         SELECT * FROM STUDENT WHERE ID = #{id};
      </select>
      
      <insert id="insert" useGeneratedKeys="true" keyProperty="id" keyColumn="ID">
   		  insert into student (NAME, BRANCH, PERCENTAGE, PHONE, EMAIL) values (#{name}, #{branch}, #{percentage}, #{phone}, #{email})
      </insert>
      
      <insert id="insert" parameterType="student">
   			insert into student (NAME, BRANCH, PERCENTAGE, PHONE, EMAIL) values (#{name}, #{branch}, #{percentage}, #{phone}, #{email});
         <selectKey keyProperty = "id" resultType = "int" order = "AFTER">
            select last_insert_id() as id
         </selectKey>
      </insert>
      
      <update id = "update" parameterType = "Student">
         UPDATE STUDENT SET NAME = #{name}, 
            BRANCH = #{branch}, 
            PERCENTAGE = #{percentage}, 
            PHONE = #{phone}, 
            EMAIL = #{email} 
         WHERE ID = #{id};
      </update>
      
      <delete id = "deleteById" parameterType = "int">
         DELETE from STUDENT WHERE ID = #{id};
      </delete>
   </mapper>
   ```

   ​		**resultMap**标签是Mybatis中一个非常重要的元素，在resultMap标签中，可通过**result**标签的**property**属性以及**column**属性来指定Java对象属性与表中字段的映射关系。如果对象属性和数据表的字段名称相同，则可以不用配置映射关系。

   ​		每个curd标签都一个id属性，Java程序中可通过指定标签id来执行某个sql语句。**parameterType**作用和字面意思一样，用来指定方法参数的类型，可以为基本类型如int/float/double等，也可以是Java中引用类型。**resultType**和**resultMap**都用来表示返回值类型，**resultType**用来指定具体的pojo。Mybatis返回结果都是map类型，当使用**resultType**时，Mybatis将map中的数据赋值给pojo，从而返回pojo，这就要求pojo中的属性要和map的key相同，如果不同我们也可以使用别名的方式进行重命名。如：select name1 name from student。相比**resultType**，**resultMap**更加灵活，我们可以在resultMaps标签来定义resultMap的映射关系。还有一种场景，就是Mybatis返回值不是任意一种pojo时，我们也需要用resultMap来代替，如联表查询等。

   ​		**insert**标签，如果**insert**需要返回主键，可以有以下两种方式：

   一种是使用**insert**标签的**useGeneratedKeys="true", keyProperty="id"**用来指定返回某个字段的值：

   ```xml
   <insert id="insert" useGeneratedKeys="true" keyProperty="id" keyColumn="ID">
   	insert into student (NAME, BRANCH, PERCENTAGE, PHONE, EMAIL) values (#{name}, #{branch}, #{percentage}, #{phone}, #{email})
   </insert>
   ```

   另一种可以在**insert**标签使用**selectKey**标签：

   ```xml
   <insert id="insert" parameterType="student">
   	insert into student (NAME, BRANCH, PERCENTAGE, PHONE, EMAIL) values (#{name}, #{branch}, #{percentage}, #{phone}, #{email});
         <selectKey keyProperty = "id" resultType = "int" order = "AFTER">
            select last_insert_id() as id
         </selectKey>
   </insert>
   ```

   ​		也可以使用注解的形式使用CRUD：

   ```java
   import java.util.List;
   
   import org.apache.ibatis.annotations.*;
   
   public interface StudentMapper {
   	
      final String getAll = "SELECT * FROM STUDENT"; 
      final String getById = "SELECT * FROM STUDENT WHERE ID = #{id}";
      final String deleteById = "DELETE from STUDENT WHERE ID = #{id}";
      final String insert = "INSERT INTO STUDENT (NAME, BRANCH, PERCENTAGE, PHONE, EMAIL ) VALUES (#{name}, #{branch}, #{percentage}, #{phone}, #{email})";
      final String update = "UPDATE STUDENT SET EMAIL = #{email}, NAME = #{name}, BRANCH = #{branch}, PERCENTAGE = #{percentage}, PHONE = #{phone} WHERE ID = #{id}";
      
      @Select(getAll)
      @Results(value = {
         @Result(property = "id", column = "ID"),
         @Result(property = "name", column = "NAME"),
         @Result(property = "branch", column = "BRANCH"),
         @Result(property = "percentage", column = "PERCENTAGE"),       
         @Result(property = "phone", column = "PHONE"),
         @Result(property = "email", column = "EMAIL")
      })
      
      List getAll();
     
      @Select(getById)
      @Results(value = {
         @Result(property = "id", column = "ID"),
         @Result(property = "name", column = "NAME"),
         @Result(property = "branch", column = "BRANCH"),
         @Result(property = "percentage", column = "PERCENTAGE"),       
         @Result(property = "phone", column = "PHONE"),
         @Result(property = "email", column = "EMAIL")
      })
      
      Student getById(int id);
   
      @Update(update)
      void update(Student student);
   
      @Delete(deleteById)
      void delete(int id);
     
      @Insert(insert)
      @Options(useGeneratedKeys = true, keyProperty = "id")
      void insert(Student student);
   }
   ```

4. **存储过程**

   Mysql服务器中创建如下存储过程：

   ```mysql
   DELIMITER //
      DROP PROCEDURE IF EXISTS details.read_recordById //
      CREATE PROCEDURE details.read_recordById (IN emp_id INT)
   	
      BEGIN 
         SELECT * FROM STUDENT WHERE ID = emp_id; 
      END// 
   DELIMITER ;
   ```

   Mybatis对这个存储过程的调用：

   ```xml
   <select id = "callById" resultMap = "result" parameterType = "Student" statementType = "CALLABLE">
       {call read_record_byid(#{id, jdbcType = INTEGER, mode = IN})}
   </select>   
   ```

5. **动态SQL**

   常见的动态SQL：

   ```xml
   <select id = "getRecByName" parameterType = "Student" resultType = "Student">
      SELECT * FROM STUDENT		 
      <if test = "name != null">
         WHERE name LIKE #{name}
      </if> 
   </select>
   ```

   ​		如果name不为null的时候，增加查询条件，name like #{name}，name字段应包含通用匹配符。还可以使用多个if语句拼接。

   ​		我们可以使用bind标签解决通配符的问题：

   ```xml
   <select id = "getRecByName" parameterType = "Student" resultType = "Student">
      <bind name="pattern" value="'%' + name + '%'" />
      SELECT * FROM STUDENT		 
      <if test = "name != null">
         WHERE name LIKE #{pattern}
      </if> 
   </select>
   ```

   ​		如果多个条件只需要满足一个，可以使用choose...when...otherwise:

   ```xml
   <select id = "getRecByName" parameterType = "Student" resultType = "Student">
      SELECT * FROM STUDENT WHERE ID = 5	 
      <choose>
      	  <when test="name != null">
         		AND name like #{name}
         </when>
         <when test="branch != null">
         		AND branch like #{branch}
         </when>
         <otherwise>
         		AND phone = 133xxxx1212
         </otherwise>
      </choose>
   </select>
   ```

   ​		也可以使用常用的where...if组合，使用where...if组合，如果前面条件不满足，会自动去除多余的AND或OR等关键字。

   ```xml
   <select id = "getRecByName" parameterType = "Student" resultType = "Student">
      SELECT * FROM STUDENT
      <where>
      	   <if test = "id != null">
             id = #{id}
          </if>
          <if test = "name != null">
             AND name LIKE #{name}
          </if> 
          <if test = "branch != null">
             AND branch LIKE #{branch}
          </if> 
      </where>
   </select>
   ```

   还可以使用set..if组合来选择性更新record：

   ```xml
   <update id="update" parameterType="student">
     update student
       <set>
         <if test="name != null">name=#{name},</if>
         <if test="branch != null">branch=#{branch},</if>
         <if test="phone != null">phone=#{phone},</if>
         <if test="email != null">email=#{email}</if>
       </set>
     where id=#{id}
   </update>
   ```

   我们还可以使用tirm元素来定义我们想要的元素，比如上面的WHERE和SET:

   ```xml
   <trim prefix="WHERE" prefixOverrides="AND|OR ">
     ...
   </trim>
   <!-- 去掉前缀AND或者OR，多个关键用|分隔 -->
   <trim prefix="SET" suffixOverrides=",">
     ...
   </trim>
   <!-- 去掉后缀,号 -->
   ```

   使用foreach实现批处理：

   ```xml
   <select id="getStudentById" resultType="com.demo.Student">
   		select * from student where id IN 
       <foreach item = "item" index = "index" collect="list"
       		open = "(" close = ")" separator = ",">
           #{item}
       </foreach>
   </select>
   ```

   script标签：如果注解需要使用动态SQL，可以使用script标签:

   ```xml
       @Update({"<script>",
         "update student",
         "  <set>",
         "    <if test='name != null'>name=#{name},</if>",
         "    <if test='branch != null'>branch=#{branch},</if>",
         "    <if test='phone != null'>phone=#{phone},</if>",
         "    <if test='email != null'>email=#{email}</if>",
         "  </set>",
         "where id=#{id}",
         "</script>"})
       void update(Student student);
   ```

6. **Mybatis执行流程**

   **（1）读取Mybatis全局配置文件mybatis-config.xml,获取数据库相关的配置信息;**

   ​			Reader reader = Resources.getResourceAsReader("mybatis-config.xml");

   **（2）根据mybatis-config.xml文件的配置加载多个配置文件，每个配置文件对应一个数据表;**

   **（3）构建会话工厂SqlSessionFactory**

   ​			SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder.build(reader);

   **（4）构建会话对象SqlSession:**

   ​			SqlSession sqlSession = sqlSessionFactory.openSession();

   **（5）执行对应语句：**

   ​		**注解形式：**Student_mapper mapper = session.getMapper(Student_mapper.class);

   mapper执行对应的方法

   ​		**xml配置形式：**sqlSession.selectList("Student.getAll");

   ​								  sqlSession.selectOne("getStudentById", 2);

   ​								  sqlSession.insert("Student.insert", student);

   ​                                  sqlSession.update("Student.update", student);

   ​                                  sqlSession.delete("Student.deleteById", 2);

   **（6）提交和关闭会话对象**

   ​                                  sqlSession.commit();

   ​                                  sqlSession.close();

7. **Mybatis的预编译机制**

   （1）“#{}”和“${}”的区别

   “#{}”预编译时会被替换成占位符“?”，在实际执行时，替换成实际传入值，并加上单引号，可以一定程度上防止sql注入；“${}”预编译时会被直接替换成实际传入的字符串，一般用于表名。

8. **Mybatis的执行器**

   常见的三种执行器：**SimpleExecutor**、**ReuseExecutor**、**BatchExecutor**

   **SimpleExecutor**：每执行一次update或select，就开启一个Statement对象，用完就关闭；

   **ReuseExecutor**：执行select或update，已sql为key查找Statement对象，没有就创建，用完不关闭，放到Map中，等到下次继续使用。

   **BatchExecutor**：执行update，将所有sql都添加到批处理中，等待统一执行

9. **延迟加载**

   ​		延迟加载是指需要的时候再加载，常见的如，联表查询的时候，先查询其中一个表信息，根据需要再查询另一张表的信息，而不是一次性查询完。

   ​		mybatis-config.xml文件修改配置，开启延迟加载：

   ```xml
   <settings>
   	 <!-- 开启延迟加载 -->
   	 <setting name="lazyLoadingEnabled" value="true"/>
      <!-- 关闭积极加载， -->
   	 <setting name="aggressiveLazyLoading" value="false"/>
   </settings>
   ```

   在xxMapper.xml文件中配置延迟加载信息；假如student中选课信息：

   ```xml
   <mapper namespace="com.demo.mybatis.mapper.ProjectMapper">
       <resultMap id="projectMap" type="project">
           <id property="id" column="id"></id>
         	<result property="sid" column="sid"></result>
           <result property="name" column="name"></result>
           <association property="student" column="sid" javaType="Student" select="com.demo.mybatis.mapper.StudentMapper.findById"></association>
       </resultMap>
       <select id="findAll" resultMap="projectMap">
           SELECT * FROM account
       </select>
   </mapper>
   ```

   ​		mybatis延迟加载的原理是：使用CGLIB创建目标对象的代理对象，当调用目标方法时，比如project的getStudent方法，进入拦截器，拦截器invoke()执行方法时如果发现值为null，就会执行关联Student获取student的sql，并使用set方法注入。

10. **Mybatis多个参数传入的几种方式**

     (1) 使用#{0}、#{1}......按顺序获取传入值

     (2) 使用@Param注解方法，指定参数名。#{}里面的名称应和@Param的名称相同。

     (3) Map传入参数，#{}应和Map里面的可以相同。

     (4) JavaBean传入参数

11. **Mapper接口的方法能否重载？**

    ​		Mapper接口的全限名，即为对应Mapper文件里面的namespace值，接口的方法名就是MapperStatement里面的id值。接口方法的参数，就是传递给sql的值。以namespace+MapperStatement Id 的方式来确定唯一的MapperStatement，所以同一个Mapper里面不能有相同的方法名，否则无法确认是哪个MapperStatement。Mapper接口的工作原理是JDK动态代理，生成代理，执行MapperStatement里面的sql，返回结果。

12. **Mybatis的一二级缓存**

    ​		一级缓存：基于HashMap的PerpetualCache，默认开启，与某个SqlSession绑定，使用本地内存；SqlSession进行flush或者close操作，都会清空缓存。可以在mybatis-config.xml配置一级缓存级别：

    ```xml
    <settings>
    	<setting name="localCacheScope" value="STATEMENT"/>
    </settings>
    ```

    ​		STATEMENT级别只对当前查询语句有效，相当于没有开启缓存；SESSION级别对某个SqlSession有效。

    二级缓存：基于HashMap的PerpetualCache，全局缓存，所有SqlSession共享，作用域为mapper。二级缓存配置分为三步：mybatis-config.xml配置

    ```xml
    <settings>
    	<setting name="cacheEnabled" value="true" />
    </settings>
    ```

    配置每个mapper文件缓存类型，不配置默认使用本地缓存:

    ```xml
    <cache type="org.mybatis.caches.ehcache.EhcacheCache" />
    ```

    配置每个select是否开启缓存，默认为true:

    ```xml
    <select ... useCache="false"></select>
    ```

    

