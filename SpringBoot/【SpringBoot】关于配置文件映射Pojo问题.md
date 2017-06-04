**场景描述**
自定义一个配置文件`project.properties`
```properties
project.info.groupId=com.github.ittalks
project.info.artifactId=spring_boot_learning
project.info.version=0.0.1-SNAPSHOT
```
对应的创建一个类`ProjectInfo`
```java
package com.github.ittalks.spring_boot_learning.config.properties;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties(prefix = "project.info")
public class ProjectInfo {

    private String groupId;
    private String artifactId;
    private String version;

    //getter、setter方法
}
```
然后是接口：
```java
package com.github.ittalks.spring_boot_learning.api;

import com.github.ittalks.spring_boot_learning.config.properties.ProjectInfo;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class PropertiesTController {

    @Autowired
    private ProjectInfo projectInfo;

    @RequestMapping("properties/print")
    public Object printProps() {
        return projectInfo;
    }
}
```
运行Spring Boot，调用接口：`http://127.0.0.1:8080/properties/print`
输出结果如下：
```json
{
    "groupId": null,
    "artifactId": null,
    "version": null
}
```
说明配置文件并没有成功映射。

---

将`project.properties`配置文件的内容放入默认的`application.properties`文件，重新运行，输出结果：
```json
{

   "groupId": "com.github.ittalks",
   "artifactId": "spring_boot_learning",
   "version": "0.0.1-SNAPSHOT"

}
```

---

上面的`ProjectInfo`类添加了`@Component`注解，这样才能通过`@Autowired`注入该属性类。

另外一种方式是删除`@Component`注解，并通过JavaConfig配置添加`@EnableConfigurationProperties(value = ProjectInfo.class)`注解，
这样会自动注册标有该注解的bean。

```java
package com.github.ittalks.spring_boot_learning.config.properties;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "project.info")
public class ProjectInfo {

    private String groupId;
    private String artifactId;
    private String version;

    //getter、setter方法
}

```

```java
package com.github.ittalks.spring_boot_learning.config;

import com.github.ittalks.spring_boot_learning.config.properties.ProjectInfo;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableConfigurationProperties(value = ProjectInfo.class)
public class PropertiesConfig {
}
```

---


还原上述配置，将配置属性重新放入`project.properties`文件，如何使自定义配置文件生效呢？

参见：[【SpringBoot】SpringBoot1.5以上版本@ConfigurationProperties取消location后的替代方案](https://github.com/itTalks/Anti_pit_guide/blob/master/SpringBoot/%E3%80%90SpringBoot%E3%80%91SpringBoot1.5%E4%BB%A5%E4%B8%8A%E7%89%88%E6%9C%AC%40ConfigurationProperties%E5%8F%96%E6%B6%88location%E5%90%8E%E7%9A%84%E6%9B%BF%E4%BB%A3%E6%96%B9%E6%A1%88.md)

1. 取消`@EnableConfigurationProperties`激活自定义的配置类
```java
@Configuration//Spring Boot自动导入该注解注释的配置类
//@EnableConfigurationProperties(value = ProjectInfo.class)
public class PropertiesConfig {
}
```
2. 在配置类中采用`@Component`的方式注册为组件，然后使用`@PropertySource`来指定自定义的资源目录
```java
@Component
@ConfigurationProperties(prefix = "project.info")
@PropertySource("classpath:/project.properties")
public class ProjectInfo {
    ...
}
```

