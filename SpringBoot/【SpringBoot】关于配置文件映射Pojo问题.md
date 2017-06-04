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



