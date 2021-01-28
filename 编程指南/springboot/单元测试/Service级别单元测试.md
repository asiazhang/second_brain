# Springboot中如何进行Service级别的单元测试

Springboot中主要就是测试时如何让`@Autowired`注解生效。

## 编写测试类

```java
@SpringBootTest  
@ActiveProfiles("test")  
public class AnnouncementServiceTest {  
  
    @Autowired  
 	private AnnouncementData data;  
  
    @Test  
 	public void testQueryCurrentAnnouncement() {  
 }  
  
}
```

测试代码中几个注意的地方：

1. 使用的测试库是`spring-boot-starter-test`，默认已经引用了junit5，因此写注解时无需使用`@RunWith(...)`，新版本SpringBoot已经支持junit5。
2. 需要增加`@SpringBootTest`注解
3. 建议增加`@ActiveProfiles`注解，然后设定加载profile为`test`。这样测试的时候就会自动加载`application-test.yml`文件，不用跟dev环境冲突。


## 设定依赖注入扫描范围

之前的版本会把默认的Application中所有依赖注入全部启用，但是我们测试的时候可能需要关闭某些注入，因此建议单独编写一个类，用于指定扫描范围。

```java
@SpringBootApplication  
@ComponentScan(basePackages = {  
        "com.oa.coding.platform.notify.controller",  
 		"com.oa.coding.platform.notify.service",  
 		"com.oa.coding.platform.notify.util",  
 		"com.oa.coding.platform.notify.data.announcement"  
})  
public class CoreApp {  
    public static void main(String[] args) {  
        SpringApplication.run(CoreApp.class, args);  
 }  
}
```

然后在程序中指定使用此`Application`。

```java
@SpringBootTest(classes = CoreApp.class)
@ActiveProfiles("test")
public class AnnouncementServiceTest {
}
```

这样就避免某些功能的开启，加快了测试速度。