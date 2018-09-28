# SimpleJdbcCall 调用存储过程
`SimpleJdbcCall`主要用来进行调用存储过程，这个方法使用起来也比较简单，先通过一个比较简单的示例来熟悉一下api

```java
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.namedparam.MapSqlParameterSource;
import org.springframework.jdbc.core.namedparam.SqlParameterSource;
import org.springframework.jdbc.core.simple.SimpleJdbcCall;

import javax.sql.DataSource;
import java.util.Map;

/**
 * @author jiangjian
 */
public class SimpleJdbcCallSample {
    public static void main(String[] args) {
        ApplicationContext ac = new AnnotationConfigApplicationContext(Config.class);
        JdbcTemplate jdbcTemplate = ac.getBean(JdbcTemplate.class);
        //初始化数据库
        jdbcTemplate.execute("drop table if exists user  ");
        jdbcTemplate.execute("create table user(id int auto_increment primary key, name varchar(40), age int)");
        jdbcTemplate.execute("insert into user(name, age) values('jiangjian', 26)");
        jdbcTemplate.execute("DROP PROCEDURE IF EXISTS read_user;");
        jdbcTemplate.execute("CREATE PROCEDURE read_user (\n" +
                "    IN in_id BIGINT,\n" +
                "    OUT out_id BIGINT,\n" +
                "    OUT out_name VARCHAR(40),\n" +
                "    OUT out_age int)\n" +
                "BEGIN\n" +
                "    SELECT id, name, age\n" +
                "    INTO out_id, out_name, out_age\n" +
                "    FROM user where id = in_id;\n" +
                "END");

        DataSource dataSource = ac.getBean(DataSource.class);
        SimpleJdbcCall simpleJdbcCall = new SimpleJdbcCall(dataSource).withProcedureName("read_user");
        SqlParameterSource in = new MapSqlParameterSource()
                .addValue("in_id", "1");
        Map<String, Object> result = simpleJdbcCall.execute(in);

        User user = new User();
        user.setId((Long) result.get("out_id"));
        user.setName((String) result.get("out_name"));
        user.setAge((Integer) result.get("out_age"));
        System.out.println(user);

        //清理环境
        jdbcTemplate.execute("drop table user");
    }
}

```

上面例子中，我们定义了一个存储过程:
```sql
CREATE PROCEDURE read_user (
    IN in_id BIGINT,
    OUT out_id BIGINT,
    OUT out_name VARCHAR(40),
    OUT out_age int)
BEGIN
    SELECT id, name, age
    INTO out_id, out_name, out_age
    FROM user where id = in_id;
END
```

从这个定义来看，该存储过程的入参是: `in_id`, 输出的结果是: `out_id`, `out_name`, `out_age`;

在实例化SimpleJdbcCall的时候，我们得配置`DataSource`以及对应存储过程名称，如上面示例:
`new SimpleJdbcCall(dataSource).withProcedureName("read_user")`


通过SimpleJdbcCall#execute方法可以触发执行，这个方法接受接受多种形式的入参（通过重载），这里我们使用`SqlParameterSource`作为入参类型，这个主要用来配置存储过程入参的信息，基本上也是Map的形式。
这个方法的返回类型是`Map<String, Object>`,map当中的key就是我们存储过程中定义的输出值的名称.

另外一个需要注意的是，不同的数据库产品可能对存储过程的输出的名称做了不同的处理，有些可能统一转化成小写形式，或者大写形式，所以上面的`out_id`可能在其他数据产品就是`OUT_ID`,为了兼容这种情况，我们可以使用JdbcTemplate作为SimleJdbcCall的构造参数，具体的方式如下:
```java
SimpleJdbcCall simpleJdbcCall = new SimpleJdbcCall(jdbcTemplate).withProcedureName("read_user");
   jdbcTemplate.setResultsMapCaseInsensitive(false);
```

对应`SimpleJdbcCall`我们也可以显示的配置入参的名称(通过`useInParameterNames`方法)，你可以配置全部的入参名称，也可以配置一部分，如果提供的信息不全的时候，SimpleJdbcCall会去查询相关的procedure metadata去获取全部必须的信息，当然这种方式也是比较浪费性能的，我们在配置好全部的入参名称后，可以显示的设置不要通过查询metadata来获取全部信息了（因为信息我们已经自己给全了<_<)， 配置的代码如下:
```java
JdbcTemplate jdbcTemplate = new JdbcTemplate(dataSource);
jdbcTemplate.setResultsMapCaseInsensitive(true);
SimpleJdbcCall simpleJdbcCall = new SimpleJdbcCall(jdbcTemplate)
              .withProcedureName("read_user")
              .withoutProcedureColumnMetaDataAccess()
              .useInParameterNames("in_id");
```

当然你可以配置存储过程的入参和出参的信息，主要是通过`declareParameters`方法来进行设置,对于当前示例，可以使用如下配置:
```java
SimpleJdbcCall simpleJdbcCall = new SimpleJdbcCall(jdbcTemplate)
         .withProcedureName("read_user")
         .useInParameterNames("in_id")
         .declareParameters(new SqlParameter("in_id", Types.BIGINT),
                 new SqlOutParameter("out_id", Types.BIGINT),
                 new SqlOutParameter("out_name", Types.VARCHAR),
                 new SqlOutParameter("out_age", Types.INTEGER));
```

SqlParameter和SqlOutParameter也比较容易理解其用途，用来指定名称和类型。

附:
下面是上面示例关联类的定义:
1  Config.java
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
