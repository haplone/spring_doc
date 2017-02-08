# spring MVC RESTFul Web Service CRUD 例子

本文主要翻译自：http://memorynotfound.com/spring-mvc-restful-web-service-crud-example/

本文主要讲解如何使用Spring MVC4搭建RestFul Web service。我们新建一个进行CRUD操作的controller，使用http方法的POST、GET、PUT、DELETE来区分新建、查询、修改、删除。这个rest service使用json进行数据传输。我们使用CrossOrigin来解决跨域问题。

## rest Controller

使用spring MVC4实现 一个REST Web Service 有很多种方式，我们选取下面这种最简单的。使用ResponseEntity直接控制response的响应头和http状态码。

* http GET方法 /users 请求全部用户数据
* http GET方法 /users/1 请求id为1的用户
* http POST方法 /users 使用json格式新建一个用户
* http PUT方法 /users/1 修改id为1的用户
* http DELETE方法 /users/1 删除id为1的用户

```java
package com.memorynotfound.controller;

import com.memorynotfound.model.User;
import com.memorynotfound.service.UserService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.util.UriComponentsBuilder;
import java.util.List;

@RestController
@RequestMapping("/users")
public class UserController {

    private final Logger LOG = LoggerFactory.getLogger(UserController.class);

    @Autowired
    private UserService userService;

    // =========================================== Get All Users ==========================================

    @RequestMapping(method = RequestMethod.GET)
    public ResponseEntity<List<User>> getAll() {
        LOG.info("getting all users");
        List<User> users = userService.getAll();

        if (users == null || users.isEmpty()){
            LOG.info("no users found");
            return new ResponseEntity<List<User>>(HttpStatus.NO_CONTENT);
        }

        return new ResponseEntity<List<User>>(users, HttpStatus.OK);
    }

    // =========================================== Get User By ID =========================================

    @RequestMapping(value = "{id}", method = RequestMethod.GET)
    public ResponseEntity<User> get(@PathVariable("id") int id){
        LOG.info("getting user with id: {}", id);
        User user = userService.findById(id);

        if (user == null){
            LOG.info("user with id {} not found", id);
            return new ResponseEntity<User>(HttpStatus.NOT_FOUND);
        }

        return new ResponseEntity<User>(user, HttpStatus.OK);
    }

    // =========================================== Create New User ========================================

    @RequestMapping(method = RequestMethod.POST)
    public ResponseEntity<Void> create(@RequestBody User user, UriComponentsBuilder ucBuilder){
        LOG.info("creating new user: {}", user);

        if (userService.exists(user)){
            LOG.info("a user with name " + user.getUsername() + " already exists");
            return new ResponseEntity<Void>(HttpStatus.CONFLICT);
        }

        userService.create(user);

        HttpHeaders headers = new HttpHeaders();
        headers.setLocation(ucBuilder.path("/user/{id}").buildAndExpand(user.getId()).toUri());
        return new ResponseEntity<Void>(headers, HttpStatus.CREATED);
    }

    // =========================================== Update Existing User ===================================

    @RequestMapping(value = "{id}", method = RequestMethod.PUT)
    public ResponseEntity<User> update(@PathVariable int id, @RequestBody User user){
        LOG.info("updating user: {}", user);
        User currentUser = userService.findById(id);

        if (currentUser == null){
            LOG.info("User with id {} not found", id);
            return new ResponseEntity<User>(HttpStatus.NOT_FOUND);
        }

        currentUser.setId(user.getId());
        currentUser.setUsername(user.getUsername());

        userService.update(user);
        return new ResponseEntity<User>(currentUser, HttpStatus.OK);
    }

    // =========================================== Delete User ============================================

    @RequestMapping(value = "{id}", method = RequestMethod.DELETE)
    public ResponseEntity<Void> delete(@PathVariable("id") int id){
        LOG.info("deleting user with id: {}", id);
        User user = userService.findById(id);

        if (user == null){
            LOG.info("Unable to delete. User with id {} not found", id);
            return new ResponseEntity<Void>(HttpStatus.NOT_FOUND);
        }

        userService.delete(id);
        return new ResponseEntity<Void>(HttpStatus.OK);
    }
}

```

* @RestController 只是@Controller 和@ResponseBody的简化使用。就是使用了@RequestController后不再需要在每个方法上设置@ResponseBody。
* @RequestMapping 是个老朋友了，就是url到控制器的映射关系定义，这边主要是使用url+http方法进行定义。一般在类上定义根url，如/users,然后在每个方法上在此基础上定义。
* @PathVariable 方法参数从uri中提取。Spring MVC需要从URI模板中通过name查找参数值，然后注入到方法中。
* @RequestBody 暗示方法参数绑定到请求体中的数据。一个HttpMessageConverter会负责将请求体中的json数据转化并装配到参数中。
* @ResponseBody 暗示返回的数据直接写入到Response body中。就像上面说的，使用了@RestController后不再需要设置。
* ResponseEntity 作为HttpEntity的子类，可以设置HttpStatus 状态码。ResponseEntity代码整个HTTP响应，可以添加响应头，状态码，并设置Response body。
* HttpHeaders 表示 HTTP 的 请求头和响应头。这个类可以方便地设置Content-Type和Access-Content-Allow-Headers等。


## CorsFilter

跨域问题直接使用Spring MVC提供的CorsFilter可以简单解决或者在xml中设置<mvc:cors>。
```xml
<filter>
  <filter-name>cors</filter-name>
  <filter-class>org.springframework.web.filter.CorsFilter</filter-class>
</filter>
<filter-mapping>
  <filter-name>cors</filter-name>
  <url-pattern>/*</url-pattern>
</filter-mapping>
```

具体可以查看： [spring MVC cors跨域实现源码解析](http://www.cnblogs.com/leftthen/p/6378090.html)

## maven依赖
```xml

<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>
    <groupId>com.memorynotfound.spring.mvc.rest</groupId>
    <artifactId>crud</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <name>SPRING-MVC - ${project.artifactId}</name>
    <url>http://memorynotfound.com</url>
    <packaging>war</packaging>

    <properties>
        <spring.version>4.2.6.RELEASE</spring.version>
        <jackson.version>2.7.4</jackson.version>
        <logback.version>1.1.7</logback.version>
    </properties>

    <dependencies>
        <!-- spring libraries -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-webmvc</artifactId>
            <version>${spring.version}</version>
        </dependency>

        <!-- Needed for JSON View -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>${jackson.version}</version>
        </dependency>

        <!-- logging -->
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>${logback.version}</version>
        </dependency>

        <!-- servlet api -->
        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>javax.servlet-api</artifactId>
            <version>3.1.0</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.2</version>
                <configuration>
                    <source>1.7</source>
                    <target>1.7</target>
                </configuration>
            </plugin>
            <plugin>
                <artifactId>maven-war-plugin</artifactId>
                <version>2.6</version>
                <configuration>
                    <failOnMissingWebXml>false</failOnMissingWebXml>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

