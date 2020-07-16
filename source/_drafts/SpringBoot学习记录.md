# SpringBoot学习记录

## 第一天

1. 回顾ssm的整合

2. **SpringBoot的创建和starter的理解**

3. 自定义banner

## 第二天，Web开发相关

1. json解析
   1. 使用Jackson，配置MappingJackson2HttpMessageConverter或者配置ObjectMapper
   2. 使用Fastjson，必须配置一个FastJsonHttpMessageConverter
2. 文件上传
3. @ControllerAdvice的三种用法：处理全局异常、预设全局数据、请求参数预处理
4. 组件：拦截器、类型转换器、整合AOP

## 第三天，数据相关

1. 整合持久层：JDBCTemplate、**MyBatis**、SpringData
2. 整合NoSQL：**Redis**、Mongodb
3. Session共享：Nginx

## 第四天，开发测试

1. Devtools介绍，配合游览器的插件LiveReload进行测试
2. MockMvcBuilders和TestRestTemplate两种测试方法测试Controller

## 第五天，Spring Security

1. Spring Security配置：WebSecurityConfigurerAdapter接口
   1. UserDetail、UserDetailService接口，访问（角色）认证管理
   2. FilterInvocationSecurityMetadataSource、AccessDecisionManager接口，访问（资源）授权管理
   3. PasswordEncoder：BCryptPasswordEncoder

## 第六天，OAuth2授权码模式

​	写到博客了

## 第七天，RabbitMQ，邮件发送，定时任务

   	1. RabbitMQ的四种交换机

