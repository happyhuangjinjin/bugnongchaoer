### 1 文章导读
Spring Security 是 Spring 家族中的一个安全管理框架，可以和Spring Boot项目很方便的集成。Spring Security框架的两大核心功能：认证和授权

认证： 验证当前访问系统的是不是本系统的用户，并且要确认具体是哪个用户。简单的理解就是登陆操作，如果可以登录成功就说明您是本系统的用户，如不能登录就说明不是本系

统的用户！而且登录成功以后需要记录当前登录用户的信息！

![](https://files.mdnice.com/user/34714/82a4014f-d5ae-4fdf-a37d-5093c28e4f9c.png)


授权：经过认证后判断当前用户是否有权限进行某个操作！

![](https://files.mdnice.com/user/34714/28953f31-bf58-454a-ad61-e702f9bb6c81.png)


如上图所示就是展示了当前登录用户可以操作的权限：用户管理、角色管理、菜单管理等，并且针对角色管理可以进行新增、修改、删除、导出等权限。

而现在前后端分离开发成为了主流的开发方式，那么在前后端分离开发方式下如何使用Spring Security就是本文章需要重点研究的内容。

### 2 Spring Security认证功能
#### 2.1 前端分离项目的认证流程
要想了解如果使用Spring Security进行认证，那么就需要先了解一下前后端分离项目中的认证流程，如下所示：
![](https://files.mdnice.com/user/34714/3e9ab4d0-f4dc-47e3-ba4d-b4baade90b87.png)

#### 2.2 Spring Security原理初探
要想使用Spring Security框架来实现上述的认证操作，就必须先要了解一个Spring Security框架的工作流程。

##### 2.2.1 过滤器链
Spring Security的原理其实就是一个过滤器链，内部包含了提供各种功能的过滤器。这里我们可以看看入门案例中的过滤器。

![](https://files.mdnice.com/user/34714/7e7d7da4-bc97-4aa3-b785-441a9e08a42f.png)


图中只展示了核心过滤器，其它的非核心过滤器并没有在图中展示。


UsernamePasswordAuthenticationFilter: 负责处理我们在登陆页面填写了用户名密码后的登陆请求。


ExceptionTranslationFilter：处理过滤器链中抛出的任何AccessDeniedException和AuthenticationException 。

FilterSecurityInterceptor：负责权限校验的过滤器。

##### 2.2.2 认证流程
Spring Security的认证流程大致如下所示：


![](https://files.mdnice.com/user/34714/99d553a4-c5a7-4c60-bac6-9446e8671ff3.png)



概念速查:

Authentication接口: 它的实现类，表示当前访问系统的用户，封装了用户相关信息。

AuthenticationManager接口：定义了认证Authentication的方法

UserDetailsService接口：加载用户特定数据的核心接口。里面定义了一个根据用户名查询用户信息的方法。

UserDetails接口：提供核心用户信息。通过UserDetailsService根据用户名获取处理的用户信息要封装成UserDetails对象返回。然后将这些信息封装到Authentication对象中。

#### 2.3 认证实现
在前后端分离项目中，前端请求的是我们自己定义的认证接口。因为在认证成功以后就需要针对当前用户生成token，Spring Security中提供的原始认证就无法实现了。在我们自定

义的认证接口中，需要调用Spring Security的API借助于Spring Security实现认证。

#### 2.3.1 思路分析
认证：

1、自定义认证接口

① 调用ProviderManager的方法进行认证 如果认证通过生成jwt

② 把用户信息存入redis中

2、自定义UserDetailsService

① 在这个实现类中去查询数据库

校验：

1、定义Jwt认证过滤器

① 获取token

② 解析token获取其中的userid

③ 从redis中获取用户信息

④ 存入SecurityContextHolder

#### 2.3.2 集成Redis
添加依赖
```
<!--redis依赖-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```
添加redis配置
在application.yml文件中添加Redis的相关配置
```
spring:
  redis:
    host: 127.0.0.1
    port: 6379
```    
#### 2.3.3 集成Mybatis Plus
添加依赖
```
<!-- 引入mybatis plus的依赖 -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.4.3</version>
</dependency>

<!-- 数据库驱动 -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>

<!-- lombok依赖包 -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
</dependency>
```
创建数据库表
```
CREATE TABLE `sys_user` (
  `id` BIGINT(20) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `user_name` VARCHAR(64) NOT NULL DEFAULT 'NULL' COMMENT '用户名',
  `nick_name` VARCHAR(64) NOT NULL DEFAULT 'NULL' COMMENT '昵称',
  `password` VARCHAR(64) NOT NULL DEFAULT 'NULL' COMMENT '密码',
  `status` CHAR(1) DEFAULT '0' COMMENT '账号状态（0正常 1停用）',
  `email` VARCHAR(64) DEFAULT NULL COMMENT '邮箱',
  `phone_number` VARCHAR(32) DEFAULT NULL COMMENT '手机号',
  `sex` CHAR(1) DEFAULT NULL COMMENT '用户性别（0男，1女，2未知）',
  `avatar` VARCHAR(128) DEFAULT NULL COMMENT '头像',
  `user_type` CHAR(1) NOT NULL DEFAULT '1' COMMENT '用户类型（0管理员，1普通用户）',
  `create_by` BIGINT(20) DEFAULT NULL COMMENT '创建人的用户id',
  `create_time` DATETIME DEFAULT NULL COMMENT '创建时间',
  `update_by` BIGINT(20) DEFAULT NULL COMMENT '更新人',
  `update_time` DATETIME DEFAULT NULL COMMENT '更新时间',
  `del_flag` INT(11) DEFAULT '0' COMMENT '删除标志（0代表未删除，1代表已删除）',
  PRIMARY KEY (`id`)
) ENGINE=INNODB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8mb4 COMMENT='用户表'

-- 插入数据
insert into security.sys_user (id, user_name, nick_name, password, status, email, phone_number, sex, avatar, user_type, create_by, create_time, update_by, update_time, del_flag) values (1501123580308578309, 'zhangsan', '张三', '1234', '0', 'hly@itcast.cn', '1312103105', '0', 'http://www.itcast.cn', '1', 1, '2022-03-08 09:12:06', 1, '2022-03-08 09:12:06', 0);
```
数据库相关配置
```
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/security?characterEncoding=utf-8&serverTimezone=UTC
    username: root
    password: 1234
    driver-class-name: com.mysql.cj.jdbc.Driver
# mybatis plus的配置
mybatis-plus:
  configuration:
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
  global-config:
    db-config:
      id-type: assign_id  
```     
User实体类
```
@Data
@TableName(value = "sys_user")
public class User {

    @TableId
    private Long id ;                         // 唯一标识
    private String userName ;                // 用户名
    private String nickName ;                // 昵称
    private String password ;                // 密码
    private String status ;                  // 状态 账号状态（0正常 1停用）
    private String email ;                   // 邮箱
    private String phoneNumber ;            // 电话号码
    private String sex ;                     // 性别  用户性别（0男，1女，2未知）
    private String avatar ;                  // 用户头像
    private String userType ;                // 用户类型 （0管理员，1普通用户）
    private Long createBy ;                  // 创建人
    private Date createTime ;                // 创建时间
    private Long updateBy ;                  // 更新人
    private Date updateTime ;                // 更新时间
    private Integer delFlag ;                // 是否删除  （0代表未删除，1代表已删除）
    
}
```
UserMapper接口

```
public interface UserMapper extends BaseMapper<User> { }
启动类
@SpringBootApplication
@MapperScan(basePackages = "com.itheima.security.mapper")
public class SecurityApplication {

    public static void main(String[] args) {
        SpringApplication.run(SecurityApplication.class , args) ;
    }

}
```
#### 2.3.4 集成Junit
添加依赖
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
</dependency>
```
编写测试类
```
@SpringBootTest(classes = SecurityApplication.class)
public class SecurityApplicationTest {

    @Autowired
    private UserMapper userMapper ;

    @Test
    public void findAll() {
        List<User> selectList = userMapper.selectList(new LambdaQueryWrapper<User>());
        selectList.forEach( s -> System.out.println(s) );
    }

}
```
#### 2.3.5 UserDetailsService
在Spring Security的整个认证流程中会调用会调用UserDetailsService中的loadUserByUsername方法根据用户名称查询用户数据。默认情况下调用的是


InMemoryUserDetailsManager中的方法，该UserDetailsService是从内存中获取用户的数据。现在我们需要从数据库中获取用户的数据，那么此时就需要自定义一个

UserDetailsService来覆盖默认的配置。

UserDetailsServiceImpl
```
@Service
public class UserDetailsServiceImpl implements UserDetailsService {

    @Autowired
    private UserMapper userMapper ;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {

        // 根据用户名查询用户数据
        LambdaQueryWrapper<User> lambdaQueryWrapper = Wrappers.<User>lambdaQuery().eq(User::getUserName ,username) ;
        User user = userMapper.selectOne(lambdaQueryWrapper);

        // 如果查询不到数据，说明用户名或者密码错误，直接抛出异常
        if(user == null) {
            throw new RuntimeException("用户名或者密码错误") ;
        }

        // 将查询到的对象转换成Spring Security所需要的UserDetails对象
        return new LoginUser(user);

    }

}
```
LoginUser
```
package com.itheima.security.domain;

import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import java.util.Collection;

// 用来封装数据库查询出来的用户数据
@Data
@NoArgsConstructor
@AllArgsConstructor
public class LoginUser implements UserDetails {

    private User user ;
    
    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return null;
    }

    @Override
    public String getPassword() {
        return user.getPassword();
    }

    @Override
    public String getUsername() {
        return user.getUserName();
    }

    @Override
    public boolean isAccountNonExpired() {          // 账号是否没有过期
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {           // 账号是否没有被锁定
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {      // 账号的凭证是否没有过期
        return true;
    }

    @Override
    public boolean isEnabled() {                    // 账号是否可用
        return true;
    }
}
```
测试认证
先通过Spring Security提供的默认登录接口进行认证的测试，需要启动Redis。此时控制台会输出如下错误：

![](https://files.mdnice.com/user/34714/26b155c6-089a-4ab9-b9d9-13ef4772fe57.png)


报错的原因：默认情况下Spring Security在获取到UserDetailsService返回的用户信息以后，会调用PasswordEncoder中的matches方法进行校验，但是此时在Spring容器中并不

存在任何的PasswordEncoder的对象，因此无法完成校验操作。

解决方案：

① 使用明文认证

要使用明文进行认证，就需要在密码字段值的前面添加{noop}字样！

![](https://files.mdnice.com/user/34714/0ae32048-9325-4119-a4c5-5b20bf61d884.png)


② 配置加密算法

2.3.6 配置加密算法
一般情况下关于密码在数据库中都是密文存储的，在进行认证的时候都是基于密文进行校验。具体的实现步骤：

1、使用指定的加密算法【BCrypt】对密码进行加密处理，将加密以后的密文存储到数据库中

2、在Spring容器中注入一个PasswordEncoder对象，一般情况下注入的就是：BCryptPasswordEncoder

我们可以定义一个Spring Security的配置类，Spring Security要求这个配置类要继承
WebSecurityConfigurerAdapter。
```
@Configuration
public class SpringSecurityConfigurer extends WebSecurityConfigurerAdapter {

    @Bean
    public BCryptPasswordEncoder bCryptPasswordEncoder() {
        return new BCryptPasswordEncoder() ;
    }

}
```
测试：将数据库的用户密码更改为使用BCryptPasswordEncoder加密以后的密文
```
@SpringBootTest(classes = SecurityApplication.class)
public class SecurityApplicationTest {

    @Autowired
    private PasswordEncoder passwordEncoder ;

    @Test
    public void testBcrypt() {
        // 加密测试
        String encode = passwordEncoder.encode("1234");
        System.out.println(encode);

        // 校验测试
        boolean matches = passwordEncoder.matches("1234", "$2a$10$ZqVB18PPA3P/MR9So/i8N.1UvVb.PblNl2sbj6pQJNDCgqiZqNQUm");
        System.out.println(matches);
    }
}
```
#### 2.3.7 登录接口
整体实现思路：

① 接下我们需要自定义登陆接口，然后让Spring Security对这个接口放行,让用户访问这个接口的时候不用登录也能访问。

② 在接口中我们通过AuthenticationManager的authenticate方法来进行用户认证,所以需要在Security Config中配置把AuthenticationManager注入容器。

③ 认证成功的话要生成一个jwt，将jwt令牌进行返回。并且为了让用户下回请求时能通过jwt识别出具体的是哪个用户，在返回之前，我们需要把用户信息存入redis，可以把用户id

作为key。

拦截规则配置
在SpringSecurityConfigurer中重写configure(HttpSecurity http)方法：
```
// 配置Spring Security的拦截规则
@Override
protected void configure(HttpSecurity http) throws Exception {
    http
            .csrf().disable()                                                               // 关闭csrf
            .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)     // 指定session的创建策略，不使用session
            .and()                                                                          // 再次获取到HttpSecurity对象
            .authorizeRequests()                                                            // 进行认证请求的配置
            .antMatchers("/user/login").anonymous()                         				// 对于登录接口，允许匿名访问
            .anyRequest().authenticated();                                                  // 除了上面的请求以外所有的请求全部需要认证
}
```
Spring容器注册AuthenticationManager
在SpringSecurityConfigurer中重写authenticationManagerBean方法：

![](https://files.mdnice.com/user/34714/9a02eaf4-283b-4f3e-833f-371c0e1f48f2.png)


登录接口定义
```
UserController
@RestController
@RequestMapping(value = "/user")
public class UserController {

    @Autowired
    private UserService userService ;

    @PostMapping(value = "/login")
    public ResponseResult<Map> login(@RequestBody User user) {
        return userService.login(user) ;
    }

}
ResponseResult
@Data
@NoArgsConstructor
@AllArgsConstructor
public class ResponseResult<T> {

    private Integer code ;
    private String msg ;
    private T data ;

}
UserService
@Service
public class UserServiceImpl implements UserService {

    @Autowired
    private AuthenticationManager authenticationManager ;

    @Autowired
    private RedisTemplate<String , String> redisTemplate ;

    @Override
    public ResponseResult<Map> login(User user) {

        // 创建Authentication对象
        UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(user.getUserName() , user.getPassword()) ;

        // 调用AuthenticationManager的authenticate方法进行认证
        Authentication authentication = authenticationManager.authenticate(authenticationToken);
        if(authentication == null) {
            throw new RuntimeException("用户名或密码错误");
        }

        // 将用户的数据存储到Redis中
        LoginUser loginUser = (LoginUser) authentication.getPrincipal();
        String userId = loginUser.getUser().getId().toString();
        redisTemplate.boundValueOps("login_user:" + userId).set(JSON.toJSONString(loginUser));

        // 生成JWT令牌并进行返回
        Map<String , String> params = new HashMap<>() ;
        params.put("userId" , userId) ;
        String token = JwtUtils.getToken(params);

        // 构建返回数据
        Map<String , String> result = new HashMap<>();
        result.put("token" , token) ;
        return new ResponseResult<Map>(200 , "操作成功" , result);

    }

}
```
##### 2.3.8 认证过滤器
当用户在访问我们受保护的资源的时候，就需要校验用户是否已经登录。我们需要自定义一个过滤器进行实现。

过滤器内部的逻辑：

1、获取请求头中的token，对token进行解析

2、取出其中的userid

3、使用userid去redis中获取对应的LoginUser对象。

4、然后封装Authentication对象存入SecurityContextHolder

5、放行

注意：这个过滤器需要将其加入到Spring Security的过滤器链中

认证过滤器：
```
@Component
public class JwtAuthenticationTokenFilter extends OncePerRequestFilter {

    @Autowired
    private RedisTemplate<String , String> redisTemplate ;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {

        // 1、从请求头中获取token，如果请求头中不存在token，直接放行即可！由Spring Security的过滤器进行校验！
        String token = request.getHeader("token");
        if(token == null || "".equals(token)) {
            filterChain.doFilter(request , response);
            return ;
        }

        // 2、对token进行解析，取出其中的userId
        String userId = null ;
        try {
            Claims claims = JwtUtils.getClaims(token);
            userId= claims.get("userId").toString();
        }catch (Exception e) {
            e.printStackTrace();
            throw new RuntimeException("token非法") ;
        }

        // 3、使用userId从redis中查询对应的LoginUser对象
        String loginUserJson = redisTemplate.boundValueOps("login_user:" + userId).get();
        LoginUser loginUser = JSON.parseObject(loginUserJson, LoginUser.class);
        if(loginUser != null) {
            // 4、然后将查询到的LoginUser对象的相关信息封装到UsernamePasswordAuthenticationToken对象中，然后将该对象存储到Security的上下文对象中
            UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(loginUser, null , null) ;
            SecurityContextHolder.getContext().setAuthentication(authenticationToken); 
        }
        
        // 5、放行
        filterChain.doFilter(request , response);
    }

}
```
配置过滤器：

![](https://files.mdnice.com/user/34714/8a6f2eeb-21d6-4ac4-bba5-abbbb1a05e85.png)


##### 2.3.9 退出登录
我们只需要定义一个退出接口，然后获取SecurityContextHolder中的认证信息，删除redis中对应的数据即可。

UserService添加退出登录接口：
```
@Override
public ResponseResult logout() {

    // 获取登录的用户信息
    LoginUser loginUser = (LoginUser) SecurityContextHolder.getContext().getAuthentication().getPrincipal();
    Long userId = loginUser.getUser().getId();

    // 删除Redis中的用户数据
    redisTemplate.delete("login_user:" + userId) ;

    // 返回
    return new ResponseResult(200 , "退出成功" , null) ;

}
```
### 3 Spring Security授权功能
#### 3.1 权限系统的作用
权限系统作用：保证系统的安全性

举例：例如一个学校图书馆的管理系统，如果是普通学生登录以后使用借书和还书的功能，不可能让他具有添加书籍信息，删除书籍信息等功能。但是如果是一个图书馆管理员的账

号登录了，应该就能看到并使用添加书籍信息，删除书籍信息等功能。总结起来就是不同的用户可以使用不同的功能，这就是权限系统要去实现的效果。

权限功能的实现我们不能只依赖前端去根据用户的权限来选择显示哪些菜单、哪些按钮。因为如果有人知道了对应功能的接口地址就可以不通过前端，直接去发送请求来实现相关功

能操作。所以我们还需要在后台进行用户权限的判断，判断当前用户是否有相应的权限，必须具有所需权限才能进行相应的操作。

#### 3.2 授权基本流程
在Spring Security中，会使用默认的FilterSecurityInterceptor来进行权限校验。在FilterSecurityInterceptor中会从SecurityContextHolder获取其中的Authentication，然后

获取其中的权限信息。当前用户是否拥有访问当前资源所需的权限。所以我们在项目中只需要把当前登录用户的权限信息也存入Authentication。然后设置我们的资源所需要的权限

即可。

#### 3.3 入门案例
##### 3.3.1 资源添加所需权限
Spring Security为我们提供了基于注解的权限控制方案，这也是我们项目中主要采用的方式。我们可以使用注解去指定访问对应的资源所需的权限。但是要使用它我们需要先开启

相关配置。

开启权限配置功能
在启动类上添加@
EnableGlobalMethodSecurity(prePostEnabled = true)

方法添加所需权限

![](https://files.mdnice.com/user/34714/3a4922fc-c461-4f3c-b2ce-d256ced4c8bb.png)

不给用户添加任何权限信息进行测试，返回信息为：
```
{
    "timestamp": "2022-07-04T06:31:47.821+00:00",
    "status": 403,
    "error": "Forbidden",
    "path": "/hello"
}
```
##### 3.3.2 用户添加所拥有的权限
UserDetailsServiceImpl
在UserDetailsServiceImpl中构建测试的权限数据，并将其设置给LoginUser对象：


![](https://files.mdnice.com/user/34714/93541380-eaeb-49ec-b7a9-3a0c058cc80d.png)

LoginUser
LoginUser接收权限数据，并且对getAuthorities方法进行改造，返回Spring Security所需要的权限对象：


![](https://files.mdnice.com/user/34714/ab3e62af-607d-4292-bdb9-c4af792dabb0.png)

JwtAuthenticationTokenFilter
在JWT过滤器中需要从Redis中获取LoginUser对象，在构建
UsernamePasswordAuthenticationToken对象的时候，为其设置权限数据：
![](https://files.mdnice.com/user/34714/b6d4cef8-dfcc-495c-a073-7a420433d27e.png)

#### 3.4 从数据库查询权限信息
##### 3.4.1 RBAC权限模型
RBAC权限模型（Role-Based Access Control）即：基于角色的权限控制。这是目前最常被开发者使用也是相对易用、通用权限模型。


![](https://files.mdnice.com/user/34714/f717e1dd-c62e-47b2-9d6a-14b21c34eb90.png)



##### 3.4.2 环境准备
数据库环境准备
权限表(菜单表)：
```
CREATE TABLE `sys_menu` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `menu_name` varchar(64) NOT NULL DEFAULT 'NULL' COMMENT '菜单名',
  `path` varchar(200) DEFAULT NULL COMMENT '路由地址',
  `component` varchar(255) DEFAULT NULL COMMENT '组件路径',
  `visible` char(1) DEFAULT '0' COMMENT '菜单状态（0显示 1隐藏）',
  `status` char(1) DEFAULT '0' COMMENT '菜单状态（0正常 1停用）',
  `perms` varchar(100) DEFAULT NULL COMMENT '权限标识',
  `icon` varchar(100) DEFAULT '#' COMMENT '菜单图标',
  `create_by` bigint(20) DEFAULT NULL,
  `create_time` datetime DEFAULT NULL,
  `update_by` bigint(20) DEFAULT NULL,
  `update_time` datetime DEFAULT NULL,
  `del_flag` int(11) DEFAULT '0' COMMENT '是否删除（0未删除 1已删除）',
  `remark` varchar(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8mb4 COMMENT='菜单表';

# 插入基础数据
insert into security.sys_menu (id, menu_name, path, component, visible, status, perms, icon, create_by, create_time, update_by, update_time, del_flag, remark) values (1543917775762886657, '添加用户', '/user/addUser', 'addUser', '0', '0', 'system:user:add', 'icon-add', 1, '2022-07-04 11:20:57', 1, '2022-07-04 11:20:57', 0, '添加用户按钮');
insert into security.sys_menu (id, menu_name, path, component, visible, status, perms, icon, create_by, create_time, update_by, update_time, del_flag, remark) values (1543918065589379073, '查看用户列表', '/user/userList', 'userList', '0', '0', 'system:user:list', 'icon-list', 1, '2022-07-04 11:22:06', 1, '2022-07-04 11:22:06', 0, '查看用户列表用户按钮');
```
角色表：
```
CREATE TABLE `sys_role` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `name` varchar(128) DEFAULT NULL,
  `role_key` varchar(100) DEFAULT NULL COMMENT '角色权限字符串',
  `status` char(1) DEFAULT '0' COMMENT '角色状态（0正常 1停用）',
  `del_flag` int(1) DEFAULT '0' COMMENT 'del_flag',
  `create_by` bigint(200) DEFAULT NULL,
  `create_time` datetime DEFAULT NULL,
  `update_by` bigint(200) DEFAULT NULL,
  `update_time` datetime DEFAULT NULL,
  `remark` varchar(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8mb4 COMMENT='角色表';

# 插入测试数据
insert into security.sys_role (id, name, role_key, status, del_flag, create_by, create_time, update_by, update_time, remark) values (1, '系统管理员', 'admin', '0', 0, 1, '2022-07-04 19:25:06', 1, '2022-07-04 19:25:19', '系统管理员');
insert into security.sys_role (id, name, role_key, status, del_flag, create_by, create_time, update_by, update_time, remark) values (2, '普通用户', 'user', '0', 0, 1, '2022-07-04 19:25:48', 1, '2022-07-04 19:25:52', '普通用户角色');
```
角色菜单中间表：
```
CREATE TABLE `sys_role_menu` (
  `role_id` bigint(200) NOT NULL AUTO_INCREMENT COMMENT '角色ID',
  `menu_id` bigint(200) NOT NULL DEFAULT '0' COMMENT '菜单id',
  PRIMARY KEY (`role_id`,`menu_id`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8mb4;

# 插入基础测试数据
insert into security.sys_role_menu (role_id, menu_id) values (1, 1543917775762886657);
insert into security.sys_role_menu (role_id, menu_id) values (1, 1543918065589379073);
insert into security.sys_role_menu (role_id, menu_id) values (2, 1543918065589379073);
```
用户表：
```
CREATE TABLE `sys_user` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `user_name` varchar(64) NOT NULL DEFAULT 'NULL' COMMENT '用户名',
  `nick_name` varchar(64) NOT NULL DEFAULT 'NULL' COMMENT '昵称',
  `password` varchar(64) NOT NULL DEFAULT 'NULL' COMMENT '密码',
  `status` char(1) DEFAULT '0' COMMENT '账号状态（0正常 1停用）',
  `email` varchar(64) DEFAULT NULL COMMENT '邮箱',
  `phone_number` varchar(32) DEFAULT NULL COMMENT '手机号',
  `sex` char(1) DEFAULT NULL COMMENT '用户性别（0男，1女，2未知）',
  `avatar` varchar(128) DEFAULT NULL COMMENT '头像',
  `user_type` char(1) NOT NULL DEFAULT '1' COMMENT '用户类型（0管理员，1普通用户）',
  `create_by` bigint(20) DEFAULT NULL COMMENT '创建人的用户id',
  `create_time` datetime DEFAULT NULL COMMENT '创建时间',
  `update_by` bigint(20) DEFAULT NULL COMMENT '更新人',
  `update_time` datetime DEFAULT NULL COMMENT '更新时间',
  `del_flag` int(11) DEFAULT '0' COMMENT '删除标志（0代表未删除，1代表已删除）',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8mb4 COMMENT='用户表';

# 插入测试数据
insert into security.sys_user (id, user_name, nick_name, password, status, email, phone_number, sex, avatar, user_type, create_by, create_time, update_by, update_time, del_flag) values (1501123580308578309, 'zhangsan', '张三', '$2a$10$ZqVB18PPA3P/MR9So/i8N.1UvVb.PblNl2sbj6pQJNDCgqiZqNQUm', '0', 'hly@itcast.cn', '1312103105', '0', 'http://www.itcast.cn', '1', 1, '2022-03-08 09:12:06', 1, '2022-03-08 09:12:06', 0);
insert into security.sys_user (id, user_name, nick_name, password, status, email, phone_number, sex, avatar, user_type, create_by, create_time, update_by, update_time, del_flag) values (1501123580308578310, 'admin', '系统管理员', '$2a$10$ZqVB18PPA3P/MR9So/i8N.1UvVb.PblNl2sbj6pQJNDCgqiZqNQUm', '0', 'hly@itcast.cn', '1312103105', '0', 'http://www.itcast.cn', '1', 1, '2022-03-08 09:12:06', 1, '2022-03-08 09:12:06', 0);
```
用户角色中间表：
```
CREATE TABLE `sys_user_role` (
  `user_id` bigint(200) NOT NULL AUTO_INCREMENT COMMENT '用户id',
  `role_id` bigint(200) NOT NULL DEFAULT '0' COMMENT '角色id',
  PRIMARY KEY (`user_id`,`role_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

# 插入基础数据
insert into security.sys_user_role (user_id, role_id) values (1501123580308578309, 2);
insert into security.sys_user_role (user_id, role_id) values (1501123580308578310, 1);
```
SQL测试查询某一个用户所具有的权限：
```
SELECT distinct m.perms FROM sys_user u
    left join sys_user_role ur on ur.user_id = u.id
    left join sys_role_menu rm on rm.role_id = ur.role_id
    left join sys_menu m on m.id = rm.menu_id
WHERE u.id = 1501123580308578310 ;
```
Menu实体类
```
// 菜单表(Menu)实体类
@TableName(value="sys_menu")
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Menu {

    @TableId
    private Long id;
    private String menuName;        // 菜单名
    private String path;            // 路由地址
    private String component;       // 组件路径
    private String visible;         // 菜单状态（0显示 1隐藏）
    private String status;          // 菜单状态（0正常 1停用）
    private String perms;           // 权限标识
    private String icon;            // 菜单图标
    private Long createBy;          // 创建人
    private Date createTime;        // 创建时间
    private Long updateBy;          // 更新人
    private Date updateTime;        // 更新时间
    private Integer delFlag;        // 是否删除（0未删除 1已删除）
    private String remark;          // 备注
    
}
MenuMapper接口
// 操作菜单表的Mapper接口
public interface MenuMapper extends BaseMapper<Menu> {

    // 查询某一个用户的权限信息
    public abstract List<String> findUserMenuById(Long userId) ;

}
```
application.yml修改

![](https://files.mdnice.com/user/34714/9c1becec-e0f4-4b25-9242-35b823b19281.png)

MenuMapper.xml映射文件
```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.itheima.security.mapper.MenuMapper">

    <select id="findUserMenuById" resultType="java.lang.String">
        SELECT distinct m.perms FROM sys_user u
             left join sys_user_role ur on ur.user_id = u.id
             left join sys_role_menu rm on rm.role_id = ur.role_id
             left join sys_menu m on m.id = rm.menu_id
        WHERE u.id = #{userId} ;
    </select>

</mapper>
```
##### 3.4.3 UserDetailsService修改
从数据库中查询该用户的真实权限信息：

![](https://files.mdnice.com/user/34714/8ae1cf19-82de-44ce-ae49-aa961dd4b7f4.png)


### 4 自定义失败处理
#### 4.1 实现思路
我们还希望在认证失败或者是授权失败的情况下也能和我们的接口一样返回相同结构的json，这样可以让前端能对响应进行统一的处理。要实现这个功能我们需要知道

SpringSecurity的异常处理机制。

在SpringSecurity中，如果我们在认证或者授权的过程中出现了异常会被
ExceptionTranslationFilter捕获到。在ExceptionTranslationFilter中会去判断是认证失败还是授权失败出

现的异常。

① 如果是认证过程中出现的异常会被封装成AuthenticationException然后调用AuthenticationEntryPoint对象的方法去进行异常处理。

② 如果是授权过程中出现的异常会被封装成AccessDeniedException然后调用AccessDeniedHandler对象的方法去进行异常处理。

所以如果我们需要自定义异常处理，我们只需要自定义AuthenticationEntryPoint和AccessDeniedHandler然后配置给Spring Security即可。

#### 4.5 代码实现
##### 4.5.1 认证失败处理器
```
@Component
public class AuthenticationEntryPointImpl implements AuthenticationEntryPoint {

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException) throws IOException, ServletException {
        ResponseResult result = new ResponseResult(HttpStatus.UNAUTHORIZED.value(), "认证失败请重新登录", null);
        String json = JSON.toJSONString(result) ;
        WebUtils.renderString(response,json);
    }

}
```
##### 4.5.2 授权失败处理器
```
@Component
public class AccessDeniedHandlerImpl implements AccessDeniedHandler {

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException accessDeniedException) throws IOException, ServletException {

        ResponseResult result = new ResponseResult(HttpStatus.FORBIDDEN.value(), "权限不足" , null);
        String json = JSON.toJSONString(result);
        WebUtils.renderString(response,json);

    }

}
```
##### 4.5.3 Spring Security配置处理器
实现步骤：



1、先注入对应的处理器

2、使用HttpSecurity对象的方法去配置
![](https://files.mdnice.com/user/34714/d59f6dda-f06b-40d2-885b-706733fc73aa.png)

### 5 跨域处理
#### 5.1 跨域说明
浏览器出于安全的考虑，使用 XMLHttpRequest对象发起 HTTP请求时必须遵守同源策略，否则就是跨域的HTTP请求，默认情况下是被禁止的。

同源策略要求源相同才能正常进行通信，所谓的源相同指定是：协议、域名、端口号都完全一致。

前后端分离项目，前端项目和后端项目一般都不是同源的，所以肯定会存在跨域请求的问题。

所以我们就要处理一下，让前端能进行跨域请求。

####5.2 解决方案
##### 5.2.1 Spring Boot项目添加跨域请求配置
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
      // 设置允许跨域的路径
        registry.addMapping("/**")
                // 设置允许跨域请求的域名
                .allowedOriginPatterns("*")
                // 是否允许cookie
                .allowCredentials(true)
                // 设置允许的请求方式
                .allowedMethods("GET", "POST", "DELETE", "PUT")
                // 设置允许的header属性
                .allowedHeaders("*")
                // 跨域允许时间
                .maxAge(3600);
    }
}
##### 5.2.2 Spring Security开启跨域访问支持
由于我们的资源都会收到Spring Security的保护，所以想要跨域访问还要让Spring Security运行跨域访问。

//SpringSecurityConfigurer#configure 允许跨域
http.cors();
### 6 其他问题说明
#### 6.1 其他权限校验方式
我们前面都是使用@PreAuthorize注解，然后在在其中使用的是hasAuthority方法进行校验。Spring Security还为我们提供了其它方法例如：hasAnyAuthority，hasRole，hasAnyRole等。

##### 6.1.2 hasAnyAuthority
hasAnyAuthority方法可以传入多个权限，只有用户有其中任意一个权限都可以访问对应资源。

![](https://files.mdnice.com/user/34714/504af861-12c2-4eed-92cb-9bc5eb0afea6.png)
##### 6.1.3 hasRole
hasRole要求有对应的角色才可以访问，但是它内部会把我们传入的参数拼接上 ROLE_ 后再去比较。所以这种情况下要用用户对应的权限也要有 ROLE_ 这个前缀才可以。

![](https://files.mdnice.com/user/34714/944e49d0-3101-464e-a82e-556f62801235.png)


##### 6.1.4 hasAnyRole
hasAnyRole 有任意的角色就可以访问。它内部也会把我们传入的参数拼接上 ROLE_ 后再去比较。所以这种情况下要用用户对应的权限也要有 ROLE_ 这个前缀才可以。


![](https://files.mdnice.com/user/34714/3cbef9e2-f3ad-44a2-bf17-a39cb33e3ffc.png)

#### 6.2 基于配置的权限控制
我们也可以在配置类中使用使用配置的方式对资源进行权限控制。

![](https://files.mdnice.com/user/34714/c258a918-9214-4434-8e96-d37803e64a63.png)


注意： 如果此时在方法上使用了@PreAuthorize(value = "hasAuthority('system:user:add')")指定了权限信息，那么就需要用于同时拥有两个权限才可以进行访问。

#### 6.3 CSRF
CSRF是指跨站请求伪造（Cross-site request forgery），是web常见的攻击之一。
https://blog.csdn.net/freeking101/article/details/86537087

Spring Security去防止CSRF攻击的方式就是通过csrf_token。后端会生成一个csrf_token，前端发起请求的时候需要携带这个csrf_token,后端会有过滤器进行校验，如果没有携

带或者是伪造的就不允许访问。

我们可以发现CSRF攻击依靠的是cookie中所携带的认证信息。但是在前后端分离的项目中我们的认证信息其实是token，而token并不是存储在cookie中，并且需要前端代码去把

token设置到请求头中才可以，所以CSRF攻击也就不用担心了。

7 总结
本文章给大家介绍了一下在前后端分离项目中如何使用Spring Security完成认证和授权的相关操作，并且介绍一下如何自定义认证和授权失败的处理器，以及如何解决跨域的相关

问题。大家可以参考本文章实际操作一下，相信大家很快就可以掌握Spring Security在前后端分离项目中的使用。

```
source: www.toutiao.com/article/7169500492463489551
```