# MyBatisBatchItemWriter 批量写入器
#Java #MyBatis #Spring 

MyBatisBatchItemWriter 是位于 mybatis-spring 包中类，该类用于批量插入数据都数据库中，通过使用 BatchExector 来提升批量插入的效率。

> 注意，MyBatisBatchItemWriter 需要引入 spring-batch 才能正常使用。

## 使用方式

与通常使用 foreach 批量插入不同，使用 MyBatisBatchItemWriter 时无需加入 foreach 标签：

```xml
<?xml version="1.0" encoding="UTF-8" ?>  
<!DOCTYPE mapper  
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"  
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">  
<mapper namespace="com.example.demo.mapper.UserMapper">  
    <insert id="insertBatch">  
        INSERT INTO user(name) VALUES (#{user.name})   
    </insert>  
</mapper>
```

同时构造对应的 MyBatisBatchItemWriter 对象进行调用：

```java
@Bean  
public MyBatisBatchItemWriter<User> writer(SqlSessionFactory factory) {  
    return new MyBatisBatchItemWriterBuilder<User>()  
            .sqlSessionFactory(factory)  
            .statementId("com.example.demo.mapper.UserMapper.insertBatch")  
            .itemToParameterConverter(item -> Map.of("user", item))  
            .build();  
}
```

需要注意的是，使用 MyBatisBatchItemWriter 需要调用 itemToParameterConverter 将传入的待插入元素映射到对应的参数名上。

在使用的时候，直接使用 MyBatisBatchItemWriter 对象进行插入，无需像通常一样使用 Mapper 接口：

```java
@Service  
public class UserService {  
  
    @Autowired  
    private MyBatisBatchItemWriter<User> writer;  
  
    @Transactional  
    public void batchWriter() {  
        User u1 = new User("lawyer");  
        User u2 = new User("better");  
  
        writer.write(List.of(u1, u2));  
    }  
}
```