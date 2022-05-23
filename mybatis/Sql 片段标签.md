# Sql 片段标签
#Java #MyBatis 

MyBatis Mapper 文件中可以使用 sql 标签定义可重用的 SQL 代码片段，以便在其他语句中使用，且 sql 标签中可以定义参数：

```xml
<sql id="userColumns"> ${alias}.id,${alias}.username,${alias}.password </sql>
```

在其他标签使用 include 子标签导入时传入参数：

```xml
<select id="selectUsers" resultType="map">
  select
    <include refid="userColumns"><property name="alias" value="t1"/></include>,
    <include refid="userColumns"><property name="alias" value="t2"/></include>
  from some_table t1
    cross join some_table t2
</select>
```

