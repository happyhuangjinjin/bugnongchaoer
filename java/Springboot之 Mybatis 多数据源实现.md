#### 简介

上篇讲解了 **JPA 多数据源实现**；这篇讲解一下 **Mybatis 多数据源实现** 。主要采用将不同数据库的 Mapper 接口分别存放到不同的 package，Spring 去扫描不同的包，注入不同的数据源来实现多数据源。原理跟 JPA 多数据源实现基本一致。

#### 创建 mybatis-multip-datasource 项目

##### 数据库脚本参考：

##### pom.xml文件引入如下依赖

```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>com.olive</groupId>
	<artifactId>mybatis-multip-datasource</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>jpa-multip-datasource</name>
	<url>http://maven.apache.org</url>

	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.5.14</version>
		<relativePath /> <!-- lookup parent from repository -->
	</parent>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<maven.compiler.source>8</maven.compiler.source>
		<maven.compiler.target>8</maven.compiler.target>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>2.1.3</version>
        </dependency>
		
		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
		</dependency>

		<dependency>
			<groupId>org.projectlombok</groupId>
			<artifactId>lombok</artifactId>
		</dependency>
		
		<dependency>
		    <groupId>org.springframework.boot</groupId>
		    <artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		
	</dependencies>
</project>
```
##### 配置两个数据源
分别为第一个主数据源(primary)，第二数据源(second)，具体配置如下：
```
# 基本配置
server:
  port: 8080

# 数据库
spring:
  datasource:
    primary:
      driver-class-name: com.mysql.jdbc.Driver
      jdbc-url: jdbc:mysql://127.0.0.1:3306/db01?characterEncoding=utf-8&allowMultiQueries=true&autoReconnect=true
      username: root
      password: root
    second:
      driver-class-name: com.mysql.cj.jdbc.Driver
      jdbc-url: jdbc:mysql://127.0.0.1:3306/crm72?characterEncoding=utf-8&allowMultiQueries=true&autoReconnect=true
      username: root
      password: root

  jackson:
    serialization:
      indent-output: true
```
##### 配置数据源
DataSourceConfig配置
```
package com.olive.config;

import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.jdbc.DataSourceBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

import javax.sql.DataSource;

/**
 * @Description: 数据源配置
 */
@Configuration
public class DataSourceConfig {

    @Bean(name = "primaryDataSource")
    @Qualifier("primaryDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.primary")
    @Primary
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean(name = "secondDataSource")
    @Qualifier("secondDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.second")
    public DataSource secondDataSource() {
        return DataSourceBuilder.create().build();
    }
}
```
PrimaryConfig数据源
```
/**
 * @Description: 主数据源配置
 */
@Configuration
@MapperScan(basePackages = "com.olive.mapper.primary",
            sqlSessionTemplateRef = "primarySqlSessionTemplate")
public class PrimaryConfig {

    @Autowired
    @Qualifier("primaryDataSource")
    private DataSource primaryDataSource;

    @Bean(name = "primarySqlSessionFactory")
    @Primary
    public SqlSessionFactory primarySqlSessionFacotory() throws Exception {
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(primaryDataSource);
        sqlSessionFactoryBean.setMapperLocations(new PathMatchingResourcePatternResolver()
                .getResources("classpath:mapper/primary/*.xml"));
        return sqlSessionFactoryBean.getObject();
    }

    @Bean(name = "primaryTransactionManager")
    @Primary
    public DataSourceTransactionManager primaryTransactionManager() {
        return new DataSourceTransactionManager(primaryDataSource);
    }

    @Bean(name = "primarySqlSessionTemplate")
    @Primary
    public SqlSessionTemplate primarySqlSessionTemplate(@Qualifier("primarySqlSessionFactory") SqlSessionFactory sqlSessionFactory) {
        return new SqlSessionTemplate(sqlSessionFactory);
    }

}
```
SecondConfig数据源
```
/**
 * @Description: 主数据源配置
 */
@Configuration
@MapperScan(basePackages = "com.olive.mapper.second",
        sqlSessionTemplateRef = "secondSqlSessionTemplate")
public class SecondConfig {

    @Autowired
    @Qualifier("secondDataSource")
    private DataSource secondDataSource;

    @Bean(name = "secondSqlSessionFactory")
    public SqlSessionFactory secondSqlSessionFacotory() throws Exception {
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(secondDataSource);
        sqlSessionFactoryBean.setMapperLocations(new PathMatchingResourcePatternResolver()
                .getResources("classpath:mapper/second/*.xml"));
        return sqlSessionFactoryBean.getObject();
    }

    @Bean(name = "secondTransactionManager")
    public DataSourceTransactionManager secondTransactionManager() {
        return new DataSourceTransactionManager(secondDataSource);
    }

    @Bean(name = "secondSqlSessionTemplate")
    public SqlSessionTemplate secondSqlSessionTemplate(@Qualifier("secondSqlSessionFactory") SqlSessionFactory sqlSessionFactory) {
        return new SqlSessionTemplate(sqlSessionFactory);
    }
}
```
##### 创建学生与老师实体类
Student实体类
```
package com.olive.entity.primary;

import java.io.Serializable;
import lombok.Data;

@Data
public class StudentDO implements Serializable{

    private Long id;

    private String name;

    private int sex;
    
    private String grade;
}
```
Teacher实体类
```
package com.olive.entity.second;

import java.io.Serializable;
import lombok.Data;

@Data
public class TeacherDO implements Serializable {

	private Long id;

	private String name;

	private int sex;
	
	private String office;
}
```

##### 数据库持久类

StudentMapper类
```
package com.olive.mapper.primary;

import com.olive.entity.primary.StudentDO;
import org.apache.ibatis.annotations.Param;


public interface StudentMapper {

    int save(@Param("studentDO") StudentDO studentDO);
}
```
TeacherMapper类
```
package com.olive.mapper.second;

import com.olive.entity.second.TeacherDO;
import org.apache.ibatis.annotations.Param;

public interface TeacherMapper {

    int save(@Param("teacherDO") TeacherDO teacherDO);
}
```
##### Mybatis xml映射
StudentMapper.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.olive.mapper.primary.StudentMapper">
  <resultMap id="BaseResultMap" type="com.olive.entity.primary.StudentDO">
    <id column="id" jdbcType="BIGINT" property="id" />
    <result column="name" jdbcType="VARCHAR" property="name" />
    <result column="sex" jdbcType="INTEGER" property="sex" />
    <result column="grade" jdbcType="VARCHAR" property="grade" />
  </resultMap>

  <insert id="save">
    INSERT INTO t_student (user_name, sex, grade) VALUES (#{studentDO.name}, #{studentDO.sex}, #{studentDO.grade});
  </insert>
</mapper>
```
TeacherMapper.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.olive.mapper.second.TeacherMapper">
  <resultMap id="BaseResultMap" type="com.olive.entity.second.TeacherDO">
    <id column="id" jdbcType="BIGINT" property="id" />
    <result column="name" jdbcType="VARCHAR" property="name" />
    <result column="sex" jdbcType="INTEGER" property="sex" />
    <result column="office" jdbcType="VARCHAR" property="office" />
  </resultMap>

  <insert id="save">
    INSERT INTO t_teacher ( user_name, sex, office) VALUES (#{teacherDO.name}, #{teacherDO.sex}, #{teacherDO.office});
  </insert>
</mapper>
```
主要注意这两个xml文件需要放到不同的目录，如下图

![](https://files.mdnice.com/user/34714/1afe8b18-8cc9-4f89-ae50-b5954a5afdeb.png)

#### 创建springboot引导类
```
package com.olive;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class);
    }

}
```
#### 测试

```
package com.olive;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import com.olive.entity.primary.StudentDO;
import com.olive.entity.second.TeacherDO;
import com.olive.mapper.primary.StudentMapper;
import com.olive.mapper.second.TeacherMapper;

@SpringBootTest
public class MybatisTest {

	@Autowired
	StudentMapper studentMapper;

	@Autowired
	TeacherMapper teacherMapper;

	@Test
	public void userSave() {
		StudentDO studentDO = new StudentDO();
		studentDO.setName("BUG弄潮儿");
		studentDO.setSex(1);
		studentDO.setGrade("一年级");
		studentMapper.save(studentDO);

		TeacherDO teacherDO = new TeacherDO();
		teacherDO.setName("Java乐园");
		teacherDO.setSex(2);
		teacherDO.setOffice("语文");
		teacherMapper.save(teacherDO);
	}
}
```
