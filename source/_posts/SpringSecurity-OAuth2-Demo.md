---
title: OAuth2授权码模式Demo
urlname: SpringSecurity-OAuth2-Demo
tags:
  - Spring Security
  - Oauth2
categories: 安全
author: Atomsk
date: 2020-5-26
---

## 案例架构

| 项目            | 端口 | 备注       |
| :-------------- | :--- | :--------- |
| auth-server     | 8080 | 授权服务器 |
| resource-server | 8081 | 资源服务器 |
| client-app      | 8082 | 第三方应用 |

<!-- more -->

***

## 授权服务器

创建SpringBoot应用并加入三个依赖：

- web
- spring cloud security
- spirng cloud OAuth2

1.对Spring Security做一些基本配置：

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.inMemoryAuthentication()
                .withUser("atomsk")
                .password(new BCryptPasswordEncoder().encode("123"))
                .roles("admin")
                .and()
                .withUser("guest")
                .password(new BCryptPasswordEncoder().encode("123"))
                .roles("user");
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable().formLogin();
    }
}
```

2.用Redis存储token

```java
@Configuration
public class AccessTokenConfig {
    @Autowired
    RedisConnectionFactory redisConnectionFactory;

    @Bean
    TokenStore tokenStore(){
        return new RedisTokenStore(redisConnectionFactory);
    }
}
```

3.**配置授权服务器**

```java
@Configuration
@EnableAuthorizationServer//开启授权服务器的自动化配置
public class AuthorizationServer extends AuthorizationServerConfigurerAdapter {
    
    @Autowired
    TokenStore tokenStore;
    @Autowired
    ClientDetailsService clientDetailsService;

    @Bean
    AuthorizationCodeServices authorizationCodeServices(){
        /*
        授权码和令牌有什么区别？授权码是用来获取令牌的，使用一次就失效，令牌则是用来获取资源的
        */
        return new InMemoryAuthorizationCodeServices();//将授权码存储到内存中
    }

    @Bean
    AuthorizationServerTokenServices tokenServices() {
        DefaultTokenServices services = new DefaultTokenServices();
        services.setClientDetailsService(clientDetailsService);
        services.setSupportRefreshToken(true);//是否支持刷新
        services.setTokenStore(tokenStore);//指定token的存储方式
        services.setAccessTokenValiditySeconds(60 * 60 * 2);//access_token的有效期
        services.setRefreshTokenValiditySeconds(60 * 60 * 24 * 3);//refresh_token的有效期
        return services;
    }

    @Override//配置token的安全约束
    public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
        security.checkTokenAccess("permitAll()")//设置token的校验端点为可以直接访问
                .allowFormAuthenticationForClients();//支持client_id以及client_secret作登录认证
    }

    @Override//配置客户端的信息用于校验发来请求的客户端
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.inMemory()//为了简单示范直接将客户端信息存储在内存里
                .withClient("atomsk")//客户端id
                .secret(new BCryptPasswordEncoder().encode("321"))
                .resourceIds("res1")
                .authorizedGrantTypes("authorization_code","refresh_token")
                .scopes("all")
                .redirectUris("http://localhost:8082/index.html");
    }

    @Override//配置token的访问端点和token服务
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        endpoints.authorizationCodeServices(authorizationCodeServices())
                .tokenServices(tokenServices());
    }
}
```

***

## 资源服务器

tip：小项目授权服务器和资源服务器可以放在一起。

和创建授权服务器一样，创建SpringBoot应用并加入三个依赖：

- web
- spring cloud security
- spirng cloud OAuth2

1.加入如下配置：

```java
@Configuration
@EnableResourceServer
public class ResourceServerConfig extends ResourceServerConfigurerAdapter {

    @Bean//如果资源服务器和授权服务器是放在一起的，就不需要配置 RemoteTokenServices 
    RemoteTokenServices tokenServices(){
        RemoteTokenServices services=new RemoteTokenServices();
        services.setCheckTokenEndpointUrl("http://localhost:8080/oauth/check_token");//校验地址
        services.setClientId("atomsk");
        services.setClientSecret("321");
        return services;
    }

    @Override
    public void configure(ResourceServerSecurityConfigurer resources) throws Exception {
        resources.resourceId("res1").tokenServices(tokenServices());
    }

    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .antMatchers("/admin/**").hasAnyRole("admin")
                .anyRequest().authenticated();
    }
}
```

2.添加两个配置接口

```java
@RestController
public class HelloController {
    @GetMapping("/hello")
    public String hello() {
        return "hello";
    }
    @GetMapping("/admin/hello")
    public String admin() {
        return "admin";
    }
}
```

***

## 第三方应用

第三方应用就是一个普通的 Spring Boot 工程，创建时加入 Thymeleaf 依赖和 Web 依赖即可

1.在templates创建一个index.html

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Tittle</title>
</head>
<body>
Hello,Oauth2
    
<a href="http://localhost:8080/oauth/authorize?client_id=atomsk&response_type=code">第三方登录</a>
<!-- &scope=all&redirect_uri=http://localhost:8082/index.html 在这不加也没影响-->
<h1 th:text="${msg}"></h1>
</body>
</html>
```

超链接里的参数：

- client_id 客户端 ID，根据我们在授权服务器中的实际配置填写。
- response_type 表示响应类型，这里是 code 表示响应一个授权码。

2.定一个Controller

```java
@Controller
public class HelloController {
    @Autowired
    RestTemplate restTemplate;

    @GetMapping("/index.html")
    public String hello(String code, Model model) {
        if (code != null) {
            MultiValueMap<String, String> map = new LinkedMultiValueMap<>();
            map.add("code", code);
            map.add("client_id", "atomsk");
            map.add("client_secret", "321");
            map.add("redirect_uri", "http://localhost:8082/index.html");
            map.add("grant_type", "authorization_code");
            Map<String,String> resp = restTemplate.postForObject("http://localhost:8080/oauth/token", map, Map.class);
            String access_token = resp.get("access_token");
            System.out.println(access_token);
            HttpHeaders headers = new HttpHeaders();
            headers.add("Authorization", "Bearer " + access_token);
            HttpEntity<Object> httpEntity = new HttpEntity<>(headers);
            ResponseEntity<String> entity = restTemplate.exchange("http://localhost:8081/admin/hello", HttpMethod.GET, httpEntity, String.class);
            model.addAttribute("msg", entity.getBody());
        }
        return "index";
    }
}
```

如果 code 不为 null，也就是如果是通过授权服务器重定向到这个地址来的，那么我们做如下两个操作：

1. 根据拿到的 code，去请求 `http://localhost:8080/oauth/token` 地址去获取 Token，返回的数据结构如下：

   ```json
   {
       "access_token": "e7f223c4-7543-43c0-b5a6-5011743b5af4",
       "token_type": "bearer",
       "refresh_token": "aafc167b-a112-456e-bbd8-58cb56d915dd",
       "expires_in": 7199,
       "scope": "all"
   }
   ```

2. 接下来，根据我们拿到的 access_token，去请求资源服务器，注意 access_token 通过请求头传递，最后将资源服务器返回的数据放到 model 中

***

本博文完全参照江南一点雨的Oauth2的授权码模式案例写成，感兴趣的点击下面链接访问原文：

http://www.javaboy.org/2020/0414/oauth2_authorization_code.html