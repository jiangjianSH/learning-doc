# NamedParameterJdbcTemplate批量插入
`NamedParameterJdbcTemplate`类封装了`JdbcTemplate`，在此基础上提供了基于parameter的statement,

我们先了解一下`NamedParameterJdbcTemplate`提供有别于`JdbcTemplate`的批量操作接口(其实是实现了`NamedParameterJdbcOperations`接口)，下面是方法的定义

```java
/**
 * Executes a batch using the supplied SQL statement with the batch of supplied arguments.
 * @param sql the SQL statement to execute
 * @param batchValues the array of Maps containing the batch of arguments for the query
 * @return an array containing the numbers of rows affected by each update in the batch
 */
int[] batchUpdate(String sql, Map<String, ?>[] batchValues);

/**
 * Execute a batch using the supplied SQL statement with the batch of supplied arguments.
 * @param sql the SQL statement to execute
 * @param batchArgs the array of {@link SqlParameterSource} containing the batch of arguments for the query
 * @return an array containing the numbers of rows affected by each update in the batch
 */
int[] batchUpdate(String sql, SqlParameterSource[] batchArgs);


上面的方法提供两种方式来给定parameters,这里我们演示第二个方法，对于`SqlParameterSource`对象的构造，我们可以用`SqlParameterSourceUtils`这个工具类来辅助生成对应的数组对象.

示例代码如下:

1 Config.java

```java

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.PropertySource;
import org.springframework.core.env.Environment;
import org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate;
import org.springframework.jdbc.datasource.DriverManagerDataSource;

import javax.sql.DataSource;

/**
 * @author jiangjian
 */
@Configuration
@ComponentScan
@PropertySource("classpath:jdbc.properties")
public class Config {
    @Autowired
    private Environment env;

    @Bean
    public DataSource dataSource() {
        DriverManagerDataSource dataSource = new DriverManagerDataSource();
        dataSource.setDriverClassName(env.getProperty("spring.datasource.driver-class-name"));
        dataSource.setUrl(env.getProperty("spring.datasource.url"));
        dataSource.setUsername(env.getProperty("spring.datasource.username"));
        dataSource.setPassword(env.getProperty("spring.datasource.password"));
        return dataSource;
    }

    /**
     * NOTE: 注意这里我们配置的是NamedParameterJdbcTemplate,而不是JdbcTemplate
     * */
    @Bean
    public NamedParameterJdbcTemplate namedParameterJdbcTemplate() {
        return new NamedParameterJdbcTemplate(dataSource());
    }
}

```
2 jdbc.properties文件

```properties
spring.datasource.url=jdbc:mysql://localhost/test?useUnicode=true&characterEncoding=UTF-8&allowMultiQueries=true&autoReconnect=true&failOverReadOnly=false&autoReconnectForPools=true&serverTimezone=UTC
spring.datasource.username=root
spring.datasource.password=root
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

3 User类
```java

public class User {
    private Long id;
    private String name;

    public User() {
    }

    public User(Long id, String name) {
        this.id = id;
        this.name = name;
    }

    //..省略get set方法
     
    
    @Override
    public String toString() {
        return "User{" +
                "id=" + id +
                ", name='" + name + '\'' +
                '}';
    }
}
```

4 主类 NamedParameterJdbcTemplateBatchOperationSample
```java
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate;
import org.springframework.jdbc.core.namedparam.SqlParameterSourceUtils;

import java.util.Arrays;
import java.util.List;

/**
 * @author jiangjian
 */
public class NamedParameterJdbcTemplateBatchOperationSample {
    public static void main(String[] args) {
        ApplicationContext ac = new AnnotationConfigApplicationContext(Config.class);

        NamedParameterJdbcTemplate namedParameterJdbcTemplate = ac.getBean(NamedParameterJdbcTemplate.class);

        namedParameterJdbcTemplate.getJdbcOperations().execute("drop table if exists user  ");
        namedParameterJdbcTemplate.getJdbcOperations().execute("create table user(id int auto_increment primary key, name varchar(40))");
        List<User> users = Arrays.asList(new User("Alice"), new User("Bob"));

        namedParameterJdbcTemplate.batchUpdate("insert into user(name) values (:name)", SqlParameterSourceUtils.createBatch(users));

        List<User> findUsers = namedParameterJdbcTemplate.query("select * from user", (rs, rowNum) -> new User(rs.getLong(1), rs.getString(2)));
        findUsers.forEach(System.out::println);

        namedParameterJdbcTemplate.getJdbcOperations().execute("drop table user");
    }
}
```