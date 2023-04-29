### pom.xml 引入
```
<dependency>
		    <groupId>com.alibaba.cloud</groupId>
		    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
		    <version>2021.1</version>
		</dependency>
```
### 找到 spring.factories 文件

`spring-cloud-starter-alibaba-nacos-discovery`jar包下找到`MATE-INF/spring.factories` 文件；可以看到
自动装配类`NacosServiceRegistryAutoConfiguration`

```
 com.alibaba.cloud.nacos.registry.NacosServiceRegistryAutoConfiguration
```

### 实例注册 NacosAutoServiceRegistration

```
@Bean
	@ConditionalOnBean(AutoServiceRegistrationProperties.class)
	public NacosAutoServiceRegistration nacosAutoServiceRegistration(
			NacosServiceRegistry registry,
			AutoServiceRegistrationProperties autoServiceRegistrationProperties,
			NacosRegistration registration) {
		return new NacosAutoServiceRegistration(registry,
				autoServiceRegistrationProperties, registration);
	}
```

NacosServiceRegistryAutoConfiguration 会自动创建关键类 NacosAutoServiceRegistration Bean；该类的父类AbstractAutoServiceRegistration 实现了 ApplicationContextAware 会在 SpringBean 创建完成后自动调用。

```
@Override
@SuppressWarnings("deprecation")
public void onApplicationEvent(WebServerInitializedEvent event) {
		bind(event);
}

@Deprecated
public void bind(WebServerInitializedEvent event) {
  ApplicationContext context = event.getApplicationContext();
  if (context instanceof ConfigurableWebServerApplicationContext) {
    if ("management".equals(((ConfigurableWebServerApplicationContext) context).getServerNamespace())) {
      return;
    }
  }
  this.port.compareAndSet(0, event.getWebServer().getPort());
  // 开始注册客户端
  this.start();
}  

public void start() {
	//.......
  //触发注册
	register();
  //......
}

protected void register() {
  //服务注册：serviceRegistry是NacosServiceRegistry的实例
  this.serviceRegistry.register(getRegistration());
}
```

### 注册具体实现类 NacosServiceRegistry

注册核心代码
```
	@Override
	public void register(Registration registration) {

		if (StringUtils.isEmpty(registration.getServiceId())) {
			log.warn("No service to register for nacos client...");
			return;
		}

		NamingService namingService = namingService();
		String serviceId = registration.getServiceId();
		String group = nacosDiscoveryProperties.getGroup();

		Instance instance = getNacosInstanceFromRegistration(registration);

		try {
      //开始注册实例
			namingService.registerInstance(serviceId, group, instance);
			log.info("nacos registry, {} {} {}:{} register finished", group, serviceId,
					instance.getIp(), instance.getPort());
		}
		catch (Exception e) {
			log.error("nacos registry, {} register failed...{},", serviceId,
					registration.toString(), e);
			// rethrow a RuntimeException if the registration is failed.
			// issue : https://github.com/alibaba/spring-cloud-alibaba/issues/1132
			rethrowRuntimeException(e);
		}
	}
```

### NamingHttpClientProxy 注册代理类

```
@Override
  public void registerService(String serviceName, String groupName, Instance instance) throws NacosException {
      //.......
      //把自己作为实例注册到nacos-server
      reqApi(UtilAndComs.nacosUrlInstance, params, HttpMethod.POST);

  }
  
  public String reqApi(String api, Map<String, String> params, Map<String, String> body, List<String> servers,
            String method) throws NacosException {
        //.......
        if (serverListManager.isDomain()) {
            //.......
            //调用服务接口
            return callServer(api, params, body, nacosDomain, method);
        } else {
            //.......
           //调用服务接口
            return callServer(api, params, body, server, method);
            }
        }
        //.......
    }
```
实例注册接口`/nacos/v1/ns/instance`

```
https://nacos.io/zh-cn/docs/open-api.html#2.1
```
