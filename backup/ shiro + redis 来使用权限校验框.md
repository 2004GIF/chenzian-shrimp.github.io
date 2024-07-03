

1. 使用 `shiro-redis-spring-boot-starter` 

shiro 只提供了对 ehcache 和 parallelHashMap 的支持，下面介绍一个 shiro 可以使用的 redis cache 实现，希望对大家有帮助！ 文档官方：https://alexxiyang.github.io/shiro-redis/

坐标：

```xml
<!-- 安全框架 -->
<!-- springboot + shiro  + redis  -->
<dependency>
    <groupId>org.crazycake</groupId>
    <artifactId>shiro-redis-spring-boot-starter</artifactId>
    <version>3.3.1</version>
</dependency>
```

2. 创建类实现 `AuthorizingRealm`  来实现认证方法和权限校验方法

   ```java
   package com.lostpropery.shiro;
   
   
   import cn.hutool.jwt.JWTException;
   import com.lostpropery.common.UserStatus;
   import com.lostpropery.entity.po.SysRole;
   import com.lostpropery.entity.po.SysUser;
   import com.lostpropery.service.ISysUserService;
   import com.lostpropery.utils.JwtUtil;
   import lombok.extern.slf4j.Slf4j;
   import org.apache.shiro.authc.*;
   import org.apache.shiro.authz.AuthorizationInfo;
   import org.apache.shiro.authz.SimpleAuthorizationInfo;
   import org.apache.shiro.realm.AuthorizingRealm;
   import org.apache.shiro.subject.PrincipalCollection;
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.stereotype.Component;
   
   import java.util.List;
   import java.util.Map;
   import java.util.stream.Collectors;
   
   @Component
   @Slf4j
   public class UserRealm extends AuthorizingRealm {
       @Autowired
       private ISysUserService userService;
   
       @Override
       // 支持自定义的 Token
       public boolean supports(AuthenticationToken token) {
           return token instanceof JwtToken;
       }
   
       @Override
       // 授权方法
       protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {
           System.out.println("授权方法");
           SysUser user = (SysUser) principalCollection.getPrimaryPrincipal();
           if (user == null) {
               throw new JWTException("令牌错误");
           }
           List<SysRole> roleList = userService.findByRolesUserId(user.getId());
           List<String> roleNames = roleList.stream().map(SysRole::getRoleName).collect(Collectors.toList());
           System.out.println("当前用户拥有的角色信息：" + roleNames);
           if (!roleNames.isEmpty()) {
               SimpleAuthorizationInfo info = new SimpleAuthorizationInfo();
               info.addRoles(roleNames);
               return info;
           }
           return null;
       }
   
       @Override
       // 认证方法
       protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken authenticationToken) throws AuthenticationException {
           System.out.println("认证方法执行了");
           JwtToken jwtToken = (JwtToken) authenticationToken;
           Map<String, Object> map = JwtUtil.parseJWT(jwtToken.getCredentials().toString());
           Integer id = (Integer) map.get("userId");
           SysUser one = userService.findById(id);
           if (one == null) {
               throw new UnknownAccountException("用户不存在");
           }
           one.setPassword(null);
           //用户信息  密钥token 用户名字
           // 第一个参考要传递一个 对象，并且提供一个id属性和对应的get set方法 给redis进行缓存作为key
           return new SimpleAuthenticationInfo(one, jwtToken.getCredentials(), getName());
       }
   
   }
   ```

3. 创建 JwtToken 类 , 使用生成的jwt 来作为主体身份的凭证 ，默认是用 `UsernamePasswordToken` 实现的，当时我们现在使用的 jwt 作为主体身份的凭证

   ```java
   package com.lostpropery.shiro;
   
   import org.apache.shiro.authc.AuthenticationToken;
   
   public class JwtToken implements AuthenticationToken {
       private String token;
       public JwtToken(String token) {
           this.token = token;
       }
       @Override
       public Object getPrincipal() {
           return token;
       }
   
       @Override
       public Object getCredentials() {
           return token;
       }
   }
   ```

4. 创建 UserInfo 类 ,  决定 token 中封装的什么信息，这信息必须要生成token的返回的一致

   ```java
   package com.lostpropery.shiro;
   
   import lombok.Data;
   
   /**
    * 将这些信息封装token
    */
   @Data
   public class UserInfo {
       private String id;
       private String account;
   }
   
   ```

5. 定义认证拦截器

   ````java
   package com.lostpropery.filter;
   
   import cn.hutool.core.util.StrUtil;
   import cn.hutool.json.JSONUtil;
   import com.lostpropery.common.R;
   import com.lostpropery.shiro.JwtToken;
   import com.lostpropery.utils.JwtUtil;
   import org.apache.shiro.authc.AuthenticationException;
   import org.apache.shiro.authc.AuthenticationToken;
   import org.apache.shiro.web.filter.authc.BasicHttpAuthenticationFilter;
   import org.springframework.stereotype.Component;
   
   import javax.servlet.ServletOutputStream;
   import javax.servlet.ServletRequest;
   import javax.servlet.ServletResponse;
   import javax.servlet.http.HttpServletRequest;
   import javax.servlet.http.HttpServletResponse;
   import java.io.IOException;
   import java.nio.charset.StandardCharsets;
   import java.util.Map;
   
   @Component
   public class JwtFilter extends BasicHttpAuthenticationFilter {
   
   
       @Override
       // 创建token
       protected AuthenticationToken createToken(ServletRequest servletRequest, ServletResponse servletResponse)  {
           HttpServletRequest request = (HttpServletRequest) servletRequest;
           String token = request.getHeader(JwtUtil.JWT_TOKEN_NAME);
           if (StrUtil.isEmptyIfStr(token)){
               return null;
           }
           return new JwtToken(token);
       }
   
       @Override
       // 拦截请求
       protected boolean onAccessDenied(ServletRequest servletRequest, ServletResponse servletResponse) throws Exception {
           HttpServletRequest request = (HttpServletRequest) servletRequest;
           String token = request.getHeader(JwtUtil.JWT_TOKEN_NAME);
           if (StrUtil.isEmptyIfStr(token)){
               return true;
           }
           Map<String, Object> map = null;
           try {
               map = JwtUtil.parseJWT(token);
           } catch (Exception e) {
               return true;
           }
   
           if (map == null || JwtUtil.isExpired(token)){
               return false;
           }else {
               // 执行登录（有 sub 对象调用login 方法）
               return executeLogin(servletRequest,servletResponse) ;
           }
       }
   
       @Override
       // 当登录 sub 对象调用 login 方法登录失败的时候调用该方法 （ realm 的认证方法获取到 token 就会调用该方法）
       protected boolean onLoginFailure(AuthenticationToken token, AuthenticationException e, ServletRequest request, ServletResponse response) {
           HttpServletResponse servletResponse = (HttpServletResponse) response;
           servletResponse.setCharacterEncoding("UTF-8");
           servletResponse.setContentType("application/json;charset=utf-8");
           Throwable throwable =  e.getCause() == null ? e : e.getCause();
           R r = R.error(throwable.getMessage());
           String res = JSONUtil.toJsonStr(r);
           try {
               ServletOutputStream outputStream = servletResponse.getOutputStream();
               outputStream.write(res.getBytes(StandardCharsets.UTF_8) );
           } catch (IOException ex) {
               throw new RuntimeException(ex);
           }
           return false;
       }
   
   }
   
   ````

6. 定义配置

   ````java
   package com.lostpropery.config;
   
   import com.lostpropery.filter.JwtFilter;
   import com.lostpropery.shiro.UserRealm;
   import org.apache.shiro.cache.CacheManager;
   import org.apache.shiro.mgt.DefaultSessionStorageEvaluator;
   import org.apache.shiro.mgt.DefaultSubjectDAO;
   import org.apache.shiro.mgt.SecurityManager;
   import org.apache.shiro.session.mgt.SessionManager;
   import org.apache.shiro.spring.security.interceptor.AuthorizationAttributeSourceAdvisor;
   import org.apache.shiro.spring.web.ShiroFilterFactoryBean;
   import org.apache.shiro.spring.web.config.DefaultShiroFilterChainDefinition;
   import org.apache.shiro.spring.web.config.ShiroFilterChainDefinition;
   import org.apache.shiro.web.mgt.DefaultWebSecurityManager;
   import org.apache.shiro.web.session.mgt.DefaultWebSessionManager;
   import org.crazycake.shiro.RedisCacheManager;
   import org.crazycake.shiro.RedisSessionDAO;
   import org.crazycake.shiro.serializer.StringSerializer;
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.context.annotation.Bean;
   import org.springframework.context.annotation.Configuration;
   
   import javax.servlet.Filter;
   import java.util.Arrays;
   import java.util.HashMap;
   import java.util.LinkedHashMap;
   import java.util.Map;
   
   @Configuration
   public class ShiroConfig {
       @Autowired
       private JwtFilter jwtFilter;
   
       // 将 knife4j 的资源屏蔽掉
       private final String[] knife4j = new String[]{
               "/doc.html",
               "/webjars/**",
               "/v3/**",
               "/swagger-resources/**"
       };
   	/**
   	如果项目使用 SpringAOP 就需要加上的这个 AuthorizationAttributeSourceAdvisor 
   	为应用程序的方法调用自动应用Shiro的权限（授权）检查 ， 如果不加上导致访问加上权限校验注解的资源出现 404 的问题
   	**/
       @Bean
       public AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor() {
           return new AuthorizationAttributeSourceAdvisor();
       }
   
   
       /**
        * 配置Session管理器，使用Redis存储Session数据
        *
        * @param redisSessionDAO RedisSessionDAO对象，用于操作Redis中的Session数据
        * @return 返回配置好的SessionManager对象
        */
       @Bean
       public SessionManager sessionManager(RedisSessionDAO redisSessionDAO) {
           DefaultWebSessionManager sessionManager = new DefaultWebSessionManager();
           // 设置存储到redis中用户信息的前缀
           redisSessionDAO.setKeyPrefix("shiro:");
           sessionManager.setSessionDAO(redisSessionDAO);
           return sessionManager;
       }
   
   
       /**
        * 配置安全管理员，整合用户认证和授权信息
        *
        * @param accountRealm      用户认证Realm
        * @param sessionManager    Session管理器
        * @param redisCacheManager Redis缓存管理器
        * @return 返回配置好的DefaultWebSecurityManager对象
        */
       @Bean
       public DefaultWebSecurityManager securityManager(
           UserRealm accountRealm,
           SessionManager sessionManager,
           RedisCacheManager redisCacheManager
       ) {
           DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager(accountRealm);
           // 生成的用户的信息缓存前缀
           redisCacheManager.setKeyPrefix("shiro:");
           securityManager.setSessionManager(sessionManager);
           securityManager.setCacheManager(redisCacheManager);
           /*
            * 关闭shiro自带的session，详情见文档
            */
           DefaultSubjectDAO subjectDAO = new DefaultSubjectDAO();
           DefaultSessionStorageEvaluator defaultSessionStorageEvaluator = new DefaultSessionStorageEvaluator();
           defaultSessionStorageEvaluator.setSessionStorageEnabled(false);
           subjectDAO.setSessionStorageEvaluator(defaultSessionStorageEvaluator);
           securityManager.setSubjectDAO(subjectDAO);
           return securityManager;
       }
   
   
       /**
        * 定义过滤器链，指定不同路径的过滤器规则
        *
        * @return 返回配置好的ShiroFilterChainDefinition对象
        */
       @Bean
       public ShiroFilterChainDefinition shiroFilterChainDefinition() {
           DefaultShiroFilterChainDefinition chainDefinition = new DefaultShiroFilterChainDefinition();
           Map<String, String> filterMap = new LinkedHashMap<>();
           // 无需校验的接口
           filterMap.put(Arrays.toString(knife4j), "anon");
           filterMap.put("/common/**", "anon");
   
           // 需要校验的接口 
           filterMap.put("/**", "jwt"); // 主要通过注解方式校验权限
           chainDefinition.addPathDefinitions(filterMap);
           return chainDefinition;
       }
   
       /**
        * 配置ShiroFilterFactoryBean，整合过滤器和安全管理器
        *
        * @param securityManager            安全管理器
        * @param shiroFilterChainDefinition 过滤器链定义
        * @return 返回配置好的ShiroFilterFactoryBean对象
        */
       @Bean("shiroFilterFactoryBean")
       public ShiroFilterFactoryBean shiroFilterFactoryBean(SecurityManager securityManager,
                                                            ShiroFilterChainDefinition shiroFilterChainDefinition) {
           ShiroFilterFactoryBean shiroFilter = new ShiroFilterFactoryBean();
           shiroFilter.setSecurityManager(securityManager);
           Map<String, Filter> filters = new HashMap<>();
           filters.put("jwt", jwtFilter);
           shiroFilter.setFilters(filters);
   
           Map<String, String> filterMap = shiroFilterChainDefinition.getFilterChainMap();
   
           shiroFilter.setFilterChainDefinitionMap(filterMap);
   
           return shiroFilter;
       }
   }
   
   ````

7. 取消使用 ShiroAnnotationProcessorAutoConfiguration  ( 使用 Shiro 框架的同时使用了 SpringAOP 才要做 )

   ShiroAnnotationProcessorAutoConfiguration：  是Apache Shiro的Spring Boot自动配置类，它负责处理Shiro的注解处理器。这个类在Spring Boot应用启动时，如果被包含在配置中，会自动配置Spring AOP以便处理Shiro的注解，如@RequiresAuthentication, @RequiresRoles, @RequiresPermissions等。这些注解用于在方法级别控制访问权限。

   在使用 Shiro 框架的同时使用了 SpringAOP 框架，这就有可能导致 Controller初始化时被增强了两次，并且在第二次增强的使用丢掉了逐渐，导致了该Controller无法被正常映射。

   ```yaml
    spring:
      autoconfigure:
         exclude: org.apache.shiro.spring.boot.autoconfigure.ShiroAnnotationProcessorAutoConfiguration
   ```

   

   

   

   

