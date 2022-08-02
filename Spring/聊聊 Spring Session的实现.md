# 聊聊 Spring Session的实现

## 引言

在Web项目由于Http的无状态性，如何存储用户的会话信息是一个普遍的问题，在传统的单体项目中用户会话信息Session存储在系统内存中，也就是堆内存的Map中。随着前后端分离和微服务的架构遍地开花，传统的Session-Cookie方案已不能满足项目的要求。因为项目集群化以后，不能在每台服务器上单独管理会话，需要一个集中管理的方式。通俗的说就是如何在集群环境中实现Session共享

Spring总是为开发者着想，所以为了解决上面的问题，开发了一个名为Spring Session的框架结合其对Redis的实现可以完美的解决问题。

[!Spring Session官方文档]https://docs.spring.io/spring-session/reference/

## 使用

Spring Boot版本 2.5.6

pom文件

```XML
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <version>2.5.6</version>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
    <version>2.5.6</version>
</dependency>

<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
    <version>2.4.6</version>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
    <version>2.5.6</version>
</dependency>
```

application配置文件

```yaml
spring:
  redis:
    port: 16379
    database: 1 #Redis相关配置
```

启动类

```JAVA
@SpringBootApplication
@EnableRedisHttpSession //开启以Redis作为基础存储设施的HttpSession功能
public class SessionApplication {
    public static void main(String[] args) {
        SpringApplication.run(SessionApplication.class,args);
    }
}
```

测试

```JAVA
@RestController
@RequestMapping("/session")
public class SessionController {

    @GetMapping("/set")
    public void session(HttpServletRequest request,String name) {
        request.getSession().setAttribute("name",name);
    }

    @GetMapping("/current")
    public String getName(HttpServletRequest request){
        return (String) request.getSession().getAttribute("name");
    }
}
```

然后在IDEA中启动两个JVM进程端口号分别是8080和8090，模拟集群部署的方案。

先在8089的服务上设置，设置当前用户的姓名为张三，调用如下

`http://localhost:8089/session/set?name=张三`

然后在8080和8090两个服务上分别调用`/session/current`接口获取当前用户信息

![image-20220401134008922](https://s2.loli.net/2022/04/01/VoXagQ4jZMUuAr3.png)

![image-20220401134048621](https://s2.loli.net/2022/04/01/72mi63ylpVJKeB1.png)

可以看到都是能获取到用户信息的？那么不免就有一个疑问了，为什么在8089上设置用户信息，能在8080上也获取到呢？原因是因为设置的用户信息，被统一存储到了Redis中。

![image-20220401134327315](https://s2.loli.net/2022/04/01/9zCQBSL2wv1YcJf.png)

那么其中的执行过程又是如何的呢？接下来分析Spring Session的流程

## 源码分析

### @EnableRedisHttpSession

```JAVA
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Documented
@Import({RedisHttpSessionConfiguration.class}) //开启基于Redis的Http Session的配置
@Configuration(
    proxyBeanMethods = false
)
public @interface EnableRedisHttpSession {
    int maxInactiveIntervalInSeconds() default 1800; //Session的最大有效时间 30分钟

    String redisNamespace() default "spring:session"; 

    /** @deprecated */
    @Deprecated
    RedisFlushMode redisFlushMode() default RedisFlushMode.ON_SAVE;

    FlushMode flushMode() default FlushMode.ON_SAVE;

    String cleanupCron() default "0 * * * * *"; //默认执行清除

    SaveMode saveMode() default SaveMode.ON_SET_ATTRIBUTE;
}
```

这个注解的目的，就是告诉Spring容器，让它帮我们加载`RedisHttpSessionConfiguration`类，所以让我们看看这个配置类

### RedisHttpSessionConfiguration

```JAVA
public class RedisHttpSessionConfiguration extends SpringHttpSessionConfiguration ...{
    //注册一个RedisIndexedSessionRepository的Bean，目的是实现对Redis的读取和写入操作，底层用的还是RedisTeamplate而已
    @Bean
    public RedisIndexedSessionRepository sessionRepository() {
        RedisTemplate<Object, Object> redisTemplate = this.createRedisTemplate();
        RedisIndexedSessionRepository sessionRepository = new RedisIndexedSessionRepository(redisTemplate);
        sessionRepository.setApplicationEventPublisher(this.applicationEventPublisher);
        if (this.indexResolver != null) {
            sessionRepository.setIndexResolver(this.indexResolver);
        }

        if (this.defaultRedisSerializer != null) {
            sessionRepository.setDefaultSerializer(this.defaultRedisSerializer);
        }

        sessionRepository.setDefaultMaxInactiveInterval(this.maxInactiveIntervalInSeconds);
        if (StringUtils.hasText(this.redisNamespace)) {
            sessionRepository.setRedisKeyNamespace(this.redisNamespace);
        }

        sessionRepository.setFlushMode(this.flushMode);
        sessionRepository.setSaveMode(this.saveMode);
        int database = this.resolveDatabase();
        sessionRepository.setDatabase(database);
        this.sessionRepositoryCustomizers.forEach((sessionRepositoryCustomizer) -> {
            sessionRepositoryCustomizer.customize(sessionRepository);
        });
        return sessionRepository;
    }
    ...
}
```

这时候还有一个疑问：当我们在使用`request.getSession()`的时候，是如何调用了`redisTemplate`的查询呢?

注意到一个类`SpringHttpSessionConfiguration`玄机就这它里面

### SpringHttpSessionConfiguration

```JAVA
public class SpringHttpSessionConfiguration implements ApplicationContextAware {
    //注册了一个过滤器，这个过滤器中使用到了SessionRepository
    @Bean
    public <S extends Session> SessionRepositoryFilter<? extends Session> springSessionRepositoryFilter(SessionRepository<S> sessionRepository) {
        SessionRepositoryFilter<S> sessionRepositoryFilter = new SessionRepositoryFilter(sessionRepository);
        sessionRepositoryFilter.setHttpSessionIdResolver(this.httpSessionIdResolver);
        return sessionRepositoryFilter;
    }
}
```

`SessionRepository`是`RedisIndexedSessionRepository`的抽象而已，所以可以断定一切对Session的管理都在这个过滤器中实现的

### SessionRepositoryFilter

```JAVA
public class SessionRepositoryFilter<S extends Session> extends OncePerRequestFilter {
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        //在请求中设置SessionReporitory方便后续使用
        request.setAttribute(SESSION_REPOSITORY_ATTR, this.sessionRepository);
        //将原始的request和response包装为Session相关
        SessionRepositoryFilter<S>.SessionRepositoryRequestWrapper wrappedRequest = new SessionRepositoryFilter.SessionRepositoryRequestWrapper(request, response);
        SessionRepositoryFilter.SessionRepositoryResponseWrapper wrappedResponse = new SessionRepositoryFilter.SessionRepositoryResponseWrapper(wrappedRequest, response);
        try {
            filterChain.doFilter(wrappedRequest, wrappedResponse);
        } finally {
            //方法执行完后对Session操作的回调
            wrappedRequest.commitSession();
        }
    } 
}
```

也就是说当我们执行`request.getSession`的时候，其实并不是调用了`HttpServletRequest的getSession`方法，而是调用了

`SessionRepositoryRequestWrapper`中的`getSession`方法

![image-20220401141706357](https://s2.loli.net/2022/04/01/Xxy97ZYoWpaKQdA.png)

```JAVA
private final class SessionRepositoryRequestWrapper extends HttpServletRequestWrapper {
    
    //这里就是真正被执行getSession的方法
    public SessionRepositoryFilter<S>.SessionRepositoryRequestWrapper.HttpSessionWrapper getSession(boolean create) {
        //获取当前的会话信息
        SessionRepositoryFilter<S>.SessionRepositoryRequestWrapper.HttpSessionWrapper currentSession = this.getCurrentSession();
        if (currentSession != null) {
            return currentSession;
        } else {
            S requestedSession = this.getRequestedSession();
            if (requestedSession != null) {
                if (this.getAttribute(SessionRepositoryFilter.INVALID_SESSION_ID_ATTR) == null) {
                    requestedSession.setLastAccessedTime(Instant.now());
                    this.requestedSessionIdValid = true;
                    currentSession = new SessionRepositoryFilter.SessionRepositoryRequestWrapper.HttpSessionWrapper(requestedSession, this.getServletContext());
                    currentSession.markNotNew();
                    this.setCurrentSession(currentSession);
                    return currentSession;
                }
            } else {
                this.setAttribute(SessionRepositoryFilter.INVALID_SESSION_ID_ATTR, "true");
            }

            if (!create) {
                return null;
            } else if (SessionRepositoryFilter.this.httpSessionIdResolver instanceof CookieHttpSessionIdResolver && this.response.isCommitted()) {
                throw new IllegalStateException("Cannot create a session after the response has been committed");
            } else {
                //创建新的Session会话
                S session = SessionRepositoryFilter.this.sessionRepository.createSession();
                session.setLastAccessedTime(Instant.now());
                currentSession = new SessionRepositoryFilter.SessionRepositoryRequestWrapper.HttpSessionWrapper(session, this.getServletContext());
                //设置当前会话
                this.setCurrentSession(currentSession);
                return currentSession;
            }
        }
    }


    private void commitSession() {
        SessionRepositoryFilter<S>.SessionRepositoryRequestWrapper.HttpSessionWrapper wrappedSession = this.getCurrentSession();
        if (wrappedSession == null) {
            //不合法的Session
            if (this.isInvalidateClientSession()) {
                //清除Session
                SessionRepositoryFilter.this.httpSessionIdResolver.expireSession(this, this.response);
            }
        } else {
            S session = wrappedSession.getSession();
            this.clearRequestedSessionCache();
            //保存Sesssion到Redis中
            SessionRepositoryFilter.this.sessionRepository.save(session);
            String sessionId = session.getId();
           
            //Session有效，并且sessionId变化了，将SessionId重新写入Cookie中，后回写到客户端
            if (!this.isRequestedSessionIdValid() || !sessionId.equals(this.getRequestedSessionId())) {
                SessionRepositoryFilter.this.httpSessionIdResolver.setSessionId(this, this.response, sessionId);
            }
        }
    }
}
```

## 总结

一句话描述一Spring Session的流程：当我们开启了`@EnableRedisHttpSession`，Spring Boot会注册一个名为`SessionRepositoryFilter`的过滤器，这个过滤器的作用将原始的`HttpServletRequest`和`HttpServletReponse`包装为`SessionRepositoryRequestWrapper`和`SessionRepositoryResponseWrapper`,其中`SessionRepositoryRequestWrapper`重写了`getSession`的方法,实现了从Redis从中读取缓存、设置过期时间等操作。



我们应该学到的东西：

* 可以在Filter中包装HttpServletRequest，实现对request的增强，其实这种做法在SpringCloud GateWay是很常见的。
* 回顾一下装饰者设计模式
* Spring Session的基本使用
* 回顾代理和委派设计模式
