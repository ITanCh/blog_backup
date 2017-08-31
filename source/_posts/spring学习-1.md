title: spring学习_bean的装配
date: 2017-06-25 19:48:39
tags: [java,spring,技术]
---

## 装配bean

DI可以降低模块之间的耦合性，一个类通过定义接口来被注入依赖。配置文件描述了对象之间的依赖关系，Spring容器会根据配置文件装配bean。

有三种装配bean的方式：   
1. 在java中现式配置；   
2. 在XML文件中现式配置；   
3. 隐式bean发现和装配。  

<!--more--> 

## 项目实践

针对上述三种装配bean的方式，我创建了一个简单的项目进行学习。

首先创建了maven项目，并且pom.xml加入所需的依赖：  

```xml   
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.pache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.tc</groupId>
    <artifactId>spring</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>4.3.9.RELEASE</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.springframework/spring-test -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-test</artifactId>
            <version>4.3.9.RELEASE</version>
            <scope>test</scope>
        </dependency>

        <!-- https://mvnrepository.com/artifact/junit/junit -->
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
            <scope>test</scope>
        </dependency>

    </dependencies>

</project>
```



### 自动化装配bean

示例中有两个bean，其中CDPlayer需要装配CompactDisk。

```java    
package com.tc;

import org.springframework.stereotype.Component;

//注解指明了需要被自动发现的bean
@Component
public class CompactDisc {
    private String title = "Sgt";
    private String artist = "Beatles";
    static int count = 0;

    public CompactDisc() {
        count++;
    }

    public String play() {
        return "Playing " + title + " by " + artist + " " + count;
    }
}

```

```java    
package com.tc;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class CDPlayer {
    private CompactDisc cd;

    //自动装配依赖的bean
    @Autowired
    public CDPlayer(CompactDisc cd) {
        this.cd = cd;
    }

    public String play() {
        return cd.play();
    }
}
```


通过配置文件指明自动扫描

```java    
package com.tc;

import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

//指明该class是一个配置，并且采用自动扫描
@Configuration
@ComponentScan
public class CDPlayerConfig {

}
```   

进行测试：   

```java   
package com.tc;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertNotNull;

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = CDPlayerConfig.class)
public class CDPlayerTest {
    @Autowired
    private CompactDisc cd;

    @Autowired
    private CDPlayer player;

    //cd会被自动装配，不会为null
    @Test
    public void cdShouldNotBeNull() {
        assertNotNull(cd);
    }

    
    @Test
    public void playIsRight() {
        assertEquals("Playing Sgt by Beatles 1", player.play());
    }


    //CompactDisc实例只被创建了一次，spring默认bean是单例模式
    @Test
    public void cdIsSingle() {
        assertEquals(cd.play(), player.play());
    }
}
```    

### 通过java装配bean

通过java装配bean和自动装配CompactDisc与CDPlayer的区别在于，去掉了`@Component`和`@Autowired`标注，将这些类变成了普通的类。

而在配置文件中，需要如下修改：   

```java   
package com.tc;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class CDPlayerConfig {
    @Bean
    public CompactDisc compactDisc(){
        return new CompactDisc();
    }

    @Bean
    public CDPlayer cdPlayer(CompactDisc cd){
        return new CDPlayer(cd);
    }
}
```   

使用原来的测试代码，可以发现测试可以成功的通过。


### 通过XML装配bean

创建xml配置文件`src\main\resources\play.xml`，并填入如下配置内容：   

```xml    
<?xml version = "1.0" encoding = "UTF-8"?>

<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
   http://www.springframework.org/schema/beans/spring-beans-3.0.xsd">

    <bean id="compactDisc" class="com.tc.CompactDisc"/>
    <bean id="cdPlayer" class="com.tc.CDPlayer">
        <constructor-arg ref="compactDisc"/>
    </bean>
</beans>
```   

把原来的java配置文件CDPlayerConfig.java删除，可以运行测试，结果依然正确。


