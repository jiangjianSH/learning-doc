# SimpleJdbcCall 调用返回ResultSet的存储过程
问题:  存储过程返回ResultSet, 此时结果应该如何匹配？

SimpleJdbcCall提供了相关的配置方式，下面我们通过例子来了解具体的api使用:

```java
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.jdbc.core.BeanPropertyRowMapper;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.simple.SimpleJdbcCall;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * @author jiangjian
 */
public class SimpleJdbcCallOnProcedureResultSetSample {
    public static void main(String[] args) {
        ApplicationContext ac = new AnnotationConfigApplicationContext(Config.class);
        JdbcTemplate jdbcTemplate = ac.getBean(JdbcTemplate.class);
        //初始化数据库
        jdbcTemplate.execute("drop table if exists user  ");
        jdbcTemplate.execute("create table user(id int auto_increment primary key, name varchar(40), age int)");
        jdbcTemplate.execute("insert into user(name, age) values('alice', 20),('Bob', 18)");
        jdbcTemplate.execute("DROP PROCEDURE IF EXISTS list_users;");
        jdbcTemplate.execute("CREATE PROCEDURE list_users()\n" +
                "BEGIN\n" +
                " SELECT id, name, age FROM user;\n" +
                "END;");

        jdbcTemplate.setResultsMapCaseInsensitive(false);
        SimpleJdbcCall simpleJdbcCall = new SimpleJdbcCall(jdbcTemplate)
                .withProcedureName("list_users")
                .returningResultSet("users", BeanPropertyRowMapper.newInstance(User.class));

        Map result = simpleJdbcCall.execute(new HashMap<>(0));
        List<User> users = (List<User>) result.get("users");
        users.forEach(System.out::println);

        //清理环境
        jdbcTemplate.execute("drop table user");
    }
}

```

上面的代码中，我们创建了一个存储过程--list_users, 它会得到所有的user信息.

```sql
CREATE PROCEDURE list_users()
BEGIN
 SELECT id, name, age FROM user;
END;
```

为了映射ResultSet里面的结果，我们需要对`SimpleJdbcCall`实例进行适当的配置，当前需要重点关注的是`returningResultSet`这个方法的作用，下面是这个方法的定义信息：

```java
/**
 * Used to specify when a ResultSet is returned by the stored procedure and you want it
 * mapped by a {@link RowMapper}. The results will be returned using the parameter name
 * specified. Multiple ResultSets must be declared in the correct order.
 * <p>If the database you are using uses ref cursors then the name specified must match
 * the name of the parameter declared for the procedure in the database.
 * @param parameterName the name of the returned results and/or the name of the ref cursor parameter
 * @param rowMapper the RowMapper implementation that will map the data returned for each row
 * */
SimpleJdbcCallOperations returningResultSet(String parameterName, RowMapper<?> rowMapper);
```
主要的意思就是：用来指定procedure执行生成的resultset和rowmapper的匹配，注意这里的`parameterName`是给返回的resultset指定的名称，如果你的数据库使用是ref cursor parameter, 请和数据库里面保持一致.

上面的示例中我们使用`BeanPropertyRowMapper`来生成RowMapper实现类，主要就是简化操作（主要原因User里面的属性名称和resultset 里面的返回名称一致， 如果不同，则需要自定义匹配的过程）

上面尤其要注意的是`SimpleJdbcCall.execute`返回的是Map<String, Object>, 我们的结果是保存在对应keys='users'这个entry下面（为啥是'users'?，因为我们通过`returningResultSet`方法配置的.

附:
下面是上面示例关联类的定义
1 Config.java
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

    @Bean
    public JdbcTemplate jdbcTemplate() {
        return new JdbcTemplate(dataSource());
    }
}
```

2 jdbc.properties
```properties
spring.datasource.url=jdbc:mysql://localhost/test?useUnicode=true&characterEncoding=UTF-8&allowMultiQueries=true&autoReconnect=true&failOverReadOnly=false&autoReconnectForPools=true&serverTimezone=UTC
spring.datasource.username=root
spring.datasource.password=root
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

3 User.java
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
