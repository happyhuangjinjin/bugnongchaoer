### Webflux 概述
简单来说，Webflux 是响应式编程的框架，与其对等的概念是 SpringMVC。两者的不同之处在于 Webflux 框架是异步非阻塞的，其可以通过较少的线程处理高并发请求。

Webflux 的框架底层采用了 Reactor 响应式编程框架以及 Netty，关于这两部分内容可以参看我之前的学习笔记：

【基础】Netty 的基础概念及使用
```
https://blog.csdn.net/zqf787351070/article/details/128724222
```
【基础】Reactor 响应式编程
```
https://blog.csdn.net/zqf787351070/article/details/128724411
```

作为一个异步框架来说，必须保证整个程序链中的每一步都是异步操作，如果在某一步出现了同步阻塞（如等待数据库 IO），则整个程序还是回出现阻塞的问题。因此本文主要介绍 Webflux 框架的基本使用，并通过异步数据库驱动 R2DBC 实现了对 MySQL 数据库的异步操作。

注意，单纯使用 Webflux 框架并不一定会提高接口的响应速度，其作用是提高系统的吞吐量。具体接口的响应速度还要看我们本身的业务逻辑。

### Webflux 基本使用
首先创建 maven 项目，在项目的 pom 文件中引入相应的依赖
```
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.3</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-r2dbc</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-webflux</artifactId>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>dev.miku</groupId>
            <artifactId>r2dbc-mysql</artifactId>
            <version>0.8.2.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.24</version>
        </dependency>
    </dependencies>
```

创建项目的启动类

```
@SpringBootApplication
public class WebfluxDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(WebfluxDemoApplication.class);
    }

}
```
此时，就可以编写一个简单的 Controller 来感受一下 Webflux 框架异步相应的概念
```
@RestController
@RequestMapping("/test")
public class TestController {

    @GetMapping("/hello")
    public String hello() {
        long start = System.currentTimeMillis();
        String helloStr = getHelloStr();
        System.out.println("普通接口耗时：" + (System.currentTimeMillis() - start));
        return helloStr;
    }

    @GetMapping("/helloWebFlux")
    public Mono<String> hello0() {
        long start = System.currentTimeMillis();
        Mono<String> hello0 = Mono.fromSupplier(this::getHelloStr);
        System.out.println("WebFlux 接口耗时：" + (System.currentTimeMillis() - start));
        return hello0;
    }

    private String getHelloStr() {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "hello";
    }

}
```

上述代码中，我们定义了一个普通的接口和一个异步响应的接口，启动程序调用相应接口，观察两个接口的耗时可以发现，异步相应接口在处理任务时不会阻塞，而是直接向下运行，当业务产生结果后，再将结果通过“预留的通道”反向推送到请求者；而普通接口的整个过过程都是同步的。


![](https://files.mdnice.com/user/34714/68c07358-7cfb-48a5-9b76-144cecaeac6e.png)

同时，观察 Postman 调用接口的接口响应时间，我们可以发现，无论是普通接口还是异步接口，其接口相应时间均为 2s 多。这也印证了 Webflux 框架并不一定会提高接口的响应时间，起主要作用是提高系统的吞吐量。

### Webflux + R2DBC 操作 MySQL
R2DBC 是一个异步操作数据库的驱动，区别于传统的同步数据库驱动 JDBC，R2DBC 与数据库的各种操作也是异步的，这将大量节省高并发系统的线程数量。

创建配置文件`application.yml`

```
spring:
  r2dbc:
    driver-class-name: com.mysql.jdbc.Driver
    username: root
    password: root
    url: r2dbc:pool:mysql://localhost:3306/spiderflow?useSSL=false&useUnicode=true&characterEncoding=UTF8&autoReconnect=true
```

创建一个 User 实体类用于测试，同时在 MySQL 中创建相应的数据库以及表结构
```
@Data
@AllArgsConstructor
@NoArgsConstructor
@Table("webflux_user")
public class User {

    @Id
    private int id;

    private String username;

    private String password;

}
```
编写数据仓库层，使用 Spring-data 封装好的简单 CRUD 接口（用法类似 JPA）

```
public interface UserRepository extends ReactiveCrudRepository<User, Integer> {

}
```
此时就可以调用封装好的 CRUD 方法进行简单的增删改查操作了。在 Webflux 框架中，我们可以使用 SpringMVC 中 Controller + Service 的模式进行开发，也可以使用 Webflux 中 route + handler 的模式进行开发。

### Controller + Service

编写 Service 调用 UserRepository

```
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    public Mono<User> addUser(User user) {
        return userRepository.save(user);
    }

    public Mono<ResponseEntity<Void>> delUser(int id) {
        return userRepository.findById(id)
                .flatMap(user -> userRepository.delete(user).then(Mono.just(new ResponseEntity<Void>(HttpStatus.OK))))
                .defaultIfEmpty(new ResponseEntity<Void>(HttpStatus.NOT_FOUND));
    }

    public Mono<ResponseEntity<User>> updateUser(User user) {
        return userRepository.findById(user.getId())
                .flatMap(user0 -> userRepository.save(user))
                .map(user0 -> new ResponseEntity<User>(user0, HttpStatus.OK))
                .defaultIfEmpty(new ResponseEntity<User>(HttpStatus.NOT_FOUND));
    }

    public Flux<User> getAllUser() {
        return userRepository.findAll();
    }

}

```

编写 Controller 进行测试

```
@RestController
@RequestMapping("/user")
public class UserController {

    @Autowired
    private UserService userService;

    @PostMapping
    public Mono<User> addUser(@RequestBody User user) {
        return userService.addUser(user);
    }

    @DeleteMapping("/{id}")
    public Mono<ResponseEntity<Void>> delUser(@PathVariable int id) {
        return userService.delUser(id);
    }

    @PutMapping
    public Mono<ResponseEntity<User>> updateUser(@RequestBody User user) {
        return userService.updateUser(user);
    }

    @GetMapping
    public Flux<User> getAllUser() {
        return userService.getAllUser();
    }

}
```

### Route + Handler
handler 就相当于定义很多处理器，其中不同的方法负责处理不同路由的请求，其对应的是传统的 Service 层

```
@Component
public class UserHandler {

    @Autowired
    private UserRepository userRepository;

    public Mono<ServerResponse> addUser(ServerRequest request) {
        return ServerResponse.ok()
                .contentType(MediaType.APPLICATION_JSON)
                .body(userRepository.saveAll(request.bodyToMono(User.class)), User.class);
    }

    public Mono<ServerResponse> delUser(ServerRequest request) {
        return userRepository.findById(Integer.parseInt(request.pathVariable("id")))
                .flatMap(user -> userRepository.delete(user).then(ServerResponse.ok().build()))
                .switchIfEmpty(ServerResponse.notFound().build());
    }

    public Mono<ServerResponse> updateUser(ServerRequest request) {
        return ServerResponse.ok()
                .contentType(MediaType.APPLICATION_JSON)
                .body(userRepository.saveAll(request.bodyToMono(User.class)), User.class);
    }

    public Mono<ServerResponse> getAllUser(ServerRequest request) {
        return ServerResponse.ok()
                .contentType(MediaType.APPLICATION_JSON)
                .body(userRepository.findAll(), User.class);
    }

    public Mono<ServerResponse> getAllUserStream(ServerRequest request) {
        return ServerResponse.ok()
                .contentType(MediaType.TEXT_EVENT_STREAM)
                .body(userRepository.findAll(), User.class);
    }

}
```

route 就是路由配置，其规定路由的分发规则，将不同的请求路由分发给相应的 handler 进行业务逻辑的处理，其对应的就是传统的 Controller 层

```
@Configuration
public class RouteConfig {

    @Bean
    RouterFunction<ServerResponse> userRoute(UserHandler userHandler) {
        return RouterFunctions.nest(
                RequestPredicates.path("/userRoute"),
                RouterFunctions.route(RequestPredicates.POST(""), userHandler::addUser)
                        .andRoute(RequestPredicates.DELETE("/{id}"), userHandler::delUser)
                        .andRoute(RequestPredicates.PUT(""), userHandler::updateUser)
                        .andRoute(RequestPredicates.GET(""), userHandler::getAllUser)
                        .andRoute(RequestPredicates.GET("/stream"), userHandler::getAllUserStream)
        );
    }

}
```
### 参考文章
```
【应用】SpringBoot -- Webflux + R2DBC 操作 MySQL
https://blog.csdn.net/zqf787351070/article/details/128725327
探究WebFlux之WebFlux 的秘密
https://zhuanlan.zhihu.com/p/143614001
用 WebFlux 写个 CURD 是什么体验？
http://www.javaboy.org/2021/0617/webflux-crud.html
WebFlux 中的请求地址路由怎么玩？
http://www.javaboy.org/2021/0618/webflux-router-function.html
WebFlux 详解
https://blog.csdn.net/zhang33565417/article/details/122012459
```