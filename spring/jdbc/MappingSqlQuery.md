# MappingSqlQuery 的使用
spring jdbc模块为了简化查询工作，增加MappingSqlQuery来简化开发，在继承`MappingSqlQuery`类的时候，用户需要实现`mapRow`抽象方法，这个方法主要用来完成Resultset和实体之间的映射。

下面给出一个简单的实现:
```java
import org.springframework.jdbc.core.SqlParameter;
import org.springframework.jdbc.object.MappingSqlQuery;

import javax.sql.DataSource;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Types;

/**
 * @author jiangjian
 */
public class UserMappingSqlQuery extends MappingSqlQuery<User> {
    public UserMappingSqlQuery(DataSource dataSource) {
        super(dataSource, "select id, name, age from user where id = ?");
        declareParameter(new SqlParameter("id", Types.BIGINT));
        compile();
    }

    @Override
        protected User mapRow(ResultSet rs, int rowNum) throws SQLException {
        User user = new User();
        user.setId(rs.getLong("id"));
        user.setName(rs.getString("name"));
        user.setAge(rs.getInt("age"));
        return user;
    }
}

```

上面的实现也是比较简单的，但是要注意构造方法:
1. 调用super(datasource, sql)方法来完成对当前MappingSqlQuery的初始化工作
2. declareParameter用来配置当前query接受的参数；
3. 用来编译上面提供的sql,从而生成对应PreparedStatement;

【注意】MappingSqlQuery是线程安全的，所以应用当前没有必要实例化多次

下面的代码是给出如何使用实现的`UserMappingSqlQuery`类
```java
UserMappingSqlQuery userMappingSqlQuery = new UserMappingSqlQuery(dataSource);
User user = userMappingSqlQuery.findObject(1);
```

需要了解的是MappingSqlQuery的findObject针对的是最多仅返回一条记录的情况，下面对返回多条记录的情况进行探讨.

首先假设我们需要获取全部user的场景，对应的MappingSqlQuery应该怎么写呢？
```java
import org.springframework.jdbc.object.MappingSqlQuery;

import javax.sql.DataSource;
import java.sql.ResultSet;
import java.sql.SQLException;

/**
 * @author jiangjian
 */
public class UsersMappingSqlQuery extends MappingSqlQuery<User> {
    public UsersMappingSqlQuery(DataSource dataSource) {
        super(dataSource, "select id, name, age from user");
        compile();
    }

    @Override
    protected User mapRow(ResultSet rs, int rowNum) throws SQLException {
        User user = new User();
        user.setId(rs.getLong("id"));
        user.setName(rs.getString("name"));
        user.setAge(rs.getInt("age"));
        return user;
    }
}
```
上面的`UsersMappingSqlQuery`的使用方式如下:
```java
UsersMappingSqlQuery usersMappingSqlQuery = new UsersMappingSqlQuery(jdbcTemplate.getDataSource());
List<User> users = usersMappingSqlQuery.execute();
```

注意现在我们调用的是MappingSqlQuery的`execute`方法，这个方法返回的list。

下面是完整的示例代码:

1 MappingSqlQuerySample.java
```java
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.jdbc.core.JdbcTemplate;

import java.util.List;

/**
 * @author jiangjian
 */
public class MappingSqlQuerySample {
    public static void main(String[] args) {
        ApplicationContext ac = new AnnotationConfigApplicationContext(Config.class);
        JdbcTemplate jdbcTemplate = ac.getBean(JdbcTemplate.class);
        //初始化数据库
        jdbcTemplate.execute("drop table if exists user  ");
        jdbcTemplate.execute("create table user(id int auto_increment primary key, name varchar(40), age int)");
        jdbcTemplate.execute("insert into user(name, age) values('alice', 20),('Bob', 18)");

        UserMappingSqlQuery userMappingSqlQuery = new UserMappingSqlQuery(jdbcTemplate.getDataSource());
        User user = userMappingSqlQuery.findObject(1);
        System.out.println(user);

        UsersMappingSqlQuery usersMappingSqlQuery = new UsersMappingSqlQuery(jdbcTemplate.getDataSource());
        List<User> users = usersMappingSqlQuery.execute();
        users.forEach(System.out::println);

        //清理环境
        jdbcTemplate.execute("drop table user");
    }
}
```

2 Config.java
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.PropertySource;
import org.springframework.core.env.Environment;
import org.springframework.jdbc.core.JdbcTemplate;
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
     *  用来对数据库进行相关的设置，和MappingSqlQuery使用没有关联
     * **/
    @Bean
    public JdbcTemplate jdbcTemplate() {
        return new JdbcTemplate(dataSource());
    }
}
```

3 jdbc.properties
```properties
spring.datasource.url=jdbc:mysql://localhost/test?useUnicode=true&characterEncoding=UTF-8&allowMultiQueries=true&autoReconnect=true&failOverReadOnly=false&autoReconnectForPools=true&serverTimezone=UTC
spring.datasource.username=root
spring.datasource.password=root
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

4  User.java
```java
public class User {
    private Long id;
    private String name;
    private int age;

    public User() {
    }

    public User(Long id, String name) {
        this.id = id;
        this.name = name;
    }

    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "User{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}

```