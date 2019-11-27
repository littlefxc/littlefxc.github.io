---
title: SpringEl+aop+jedis实现缓存管理
date: 2019-09-02 19:40:16
categories: springboot
tags:
- spring expression
- aop
- jedis
- redis
- aspect
---

## SpringEl + aop + jedis实现缓存管理

目的：使用 aop 配合 jedis 和 spring expression 实现缓存管理

### 切点

```java
import java.lang.annotation.*;

/**
 * 配合{@link CacheAspect}，该注解是切入点
 * @author fengxuechao
 * @version 0.1
 * @date 2019/8/26
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
public @interface Cacheable {

    /**
     * redis key 的前缀
     * @return
     */
    String prefix() default "";

    /**
     * redis key
     * @return
     */
    String key();
}
```

### 切面

```java
/**
 * @author fengxuechao
 * @version 0.1
 * @date 2019/9/2
 */
@Slf4j
@Aspect
@Component
public class CacheAspect {

    @Autowired
    private JedisCluster jedisCluster;

    @Autowired
    private AthenaIotProperties properties;

    /**
     * 模仿spring cache
     *
     * @param pjp
     * @return
     * @throws Throwable
     */
    @Around("@annotation(Cacheable)")
    public Object cacheable(ProceedingJoinPoint pjp) throws Throwable {
        boolean info = log.isInfoEnabled();
        Method method = AopUtils.getMethod(pjp);
        Cacheable cacheable = method.getAnnotation(Cacheable.class);
        String key = getELString(cacheable, method, pjp.getArgs());
        int expireIn = properties.getTokenExpireIn();
        Boolean existKey = jedisCluster.exists(key);
        if (!existKey) {
            if (info) {
                log.info("缓存中没有 key:{}", key);
            }
            Object obj = pjp.proceed();
            String value = JSON.toJSONString(obj);
            jedisCluster.setex(key, expireIn, value);
            if (info) {
                log.info("缓存token:key={}, expireIn={}, value={}", key, expireIn, value);
            }
            return obj;
        }
        String athenaIotTokenString = jedisCluster.get(key);
        if (info) {
            log.info("缓存token:key={}, expireIn={}, value={}", key, expireIn, athenaIotTokenString);
        }
        return JSON.parseObject(athenaIotTokenString, method.getReturnType());
    }

    private String getELString(Cacheable cacheable, Method method, Object[] args) {
        ExpressionParser parser = new SpelExpressionParser();
        EvaluationContext context = new StandardEvaluationContext();
        //获取被拦截方法参数名列表(使用Spring支持类库)
        LocalVariableTableParameterNameDiscoverer nameDiscoverer = new LocalVariableTableParameterNameDiscoverer();
        String[] paraNameArr = nameDiscoverer.getParameterNames(method);
        //把方法参数放入SPEL上下文中
        for (int i = 0; i < paraNameArr.length; i++) {
            context.setVariable(paraNameArr[i], args[i]);
        }
        String keyPrefix = parser.parseExpression(cacheable.prefix()).getValue(context, String.class);
        String key = parser.parseExpression(cacheable.key()).getValue(context, String.class);
        return MessageFormat.format("{0}{1}", keyPrefix, key);
    }

}
```

### 实现

```java
@Cacheable(prefix = "'token:'", key = "#authRequest.appId+':'+#authRequest.secret")
//@Cacheable(value = "token", key = "#p0.appId+':'+#p0.secret")
@Override
public AccessToken getAccessToken(AuthenticationRequest authRequest) {
    String tokenUrl = authRequest.getTokenUrl();
    ParameterizedTypeReference typeReference = new ParameterizedTypeReference<ResultBean<AccessToken>>(){};
    ResponseEntity<ResultBean<AccessToken>> entity = restTemplate.exchange(tokenUrl, HttpMethod.GET, null, typeReference);
    return entity.getBody().getData();
}
```
