# SimpleJdbcCall 调用存储方法
`SimpleJdbcCall`可以用来调用存储方法，使用起来也比较简单，先通过一个比较简单的示例来熟悉一下api

```java
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.SqlOutParameter;
import org.springframework.jdbc.core.SqlParameter;
import org.springframework.jdbc.core.namedparam.MapSqlParameterSource;
import org.springframework.jdbc.core.namedparam.SqlParameterSource;
import org.springframework.jdbc.core.simple.SimpleJdbcCall;

import java.sql.Types;
import java.util.Map;

/**
 * @author jiangjian
 */
public class SimpleJdbcCallOnFunctionSample {
    public static void main(String[] args) {
        ApplicationContext ac = new AnnotationConfigApplicationContext(Config.class);
        JdbcTemplate jdbcTemplate = ac.getBean(JdbcTemplate.class);
        //初始化数据库
        jdbcTemplate.execute("drop table if exists user  ");
        jdbcTemplate.execute("create table user(id int auto_increment primary key, name varchar(40), age int)");
        jdbcTemplate.execute("insert into user(name, age) values('jiangjian', 26)");
        jdbcTemplate.execute("DROP FUNCTION IF EXISTS get_user_description;");
        jdbcTemplate.execute("CREATE FUNCTION get_user_description (in_id INTEGER)\n" +
                "RETURNS VARCHAR(200) READS SQL DATA\n" +
                "BEGIN\n" +
                "    DECLARE out_description VARCHAR(200);\n" +
                "    SELECT concat('name is ', name, ' and age is ', age)\n" +
                "        INTO out_description\n" +
                "        FROM user where id = in_id;\n" +
                "    RETURN out_description;\n" +
                "END");
        
        jdbcTemplate.setResultsMapCaseInsensitive(false);
        SimpleJdbcCall simpleJdbcCall = new SimpleJdbcCall(jdbcTemplate)
                .withFunctionName("get_user_description");

        SqlParameterSource in = new MapSqlParameterSource()
                .addValue("in_id", "1");
        String description = simpleJdbcCall.executeFunction(String.class, in);
        System.out.println(description);

        //清理环境
        jdbcTemplate.execute("drop table user");
    }
}
```

上面例子中，我们定义了一个存储方法:
```sql
CREATE FUNCTION get_user_description (in_id INTEGER)
RETURNS VARCHAR(200) READS SQL DATA
BEGIN
    DECLARE out_description VARCHAR(200);
    SELECT concat('name is ', name, ' and age is ', age)
        INTO out_description
        FROM user where id = in_id;
    RETURN out_description;
END
```

从这个定义来看，该存储方法的入参是: `in_id`, 输出的结果是: `out_description`;

在实例化SimpleJdbcCall的时候，我们得显示配置对应DataSource(或者通过JdbcTemplate),同时也要配置对应的function命名，上面的例子中可以了解一下api使用情况:
```java
jdbcTemplate.setResultsMapCaseInsensitive(false);
SimpleJdbcCall simpleJdbcCall = new SimpleJdbcCall(jdbcTemplate)
        .withFunctionName("get_user_description");
```

【注意】: 在执行function的时候，要注意和procedure的不同，执行function时调用`executeFunction`, 而执行procedure的时候调用`execute`方法

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
