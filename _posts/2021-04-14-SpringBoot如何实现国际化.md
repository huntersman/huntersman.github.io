---
title: SpringBoot如何实现国际化
categories:
  - SpringBoot
tags:
  - SpringBoot
  - 国际化
---

项目基于SpringBoot 2.4.4版本

# 方法一

在resources文件夹下面建立messages.properties，以及相关的国际化内容。zh_CN表示中文，en_US表示美国英语，其他等等。

![image.png](https://www.hualigs.cn/image/6076f1fd790b0.jpg)

调用非常简单，直接注入MessageSource，第一个参数是properties里面自己写好的，中间参数暂时不管，写成null，第三个参数为LocaleContextHolder.getLocale()

```java
 	@Autowired
    MessageSource messageSource;

    @GetMapping("/hello")
    public String hello() {
        return messageSource.getMessage("user.name",null, LocaleContextHolder.getLocale());
    }
```

调用的时候需要在Headers里面注明Accept-Language是什么，这样就会相应的返回对应的语言。


![image.png](https://www.hualigs.cn/image/6076f3a9b456a.jpg)

# 方法二

自己实现一个拦截器

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Bean
    LocaleResolver localeResolver() {
        SessionLocaleResolver resolver = new SessionLocaleResolver();
        resolver.setDefaultLocale(Locale.SIMPLIFIED_CHINESE);
        return resolver;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        LocaleChangeInterceptor interceptor = new LocaleChangeInterceptor();
        interceptor.setParamName("lang");
        registry.addInterceptor(interceptor);
    }
}
```

只需要在访问后面加lang=en-us即可

![image.png](https://www.hualigs.cn/image/6076f60aede85.jpg)

并且之后访问不需要加lang=en-us，也是默认的en-us，非常省力

![image.png](https://www.hualigs.cn/image/6076f6e267f34.jpg)
