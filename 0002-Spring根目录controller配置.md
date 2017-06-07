---
title: Spring根目录controller配置
date: 2016/3/20 10:51:40
---

凌晨，外面电闪不雷鸣。最近突发奇想，想把以前的博客重构一把。

这次换上Spring吧。之前的博客托管在Google的AppEngine上，用了很多google提供的免费服务，光这一点我就很想大赞谷歌。（Google：哥不靠这个赚钱好不啦~）。

#### Java web + Spring集成
说句心里话，说这个东西估计很多人都会笑话我。N年前就用过的东西，怎么现在还要记下来，难不成你不知道怎么搞？

不好意思，一上来我还真没Hold得住，费了老大劲。

#### web.xml配置
````xml
<!-- 加载Spring Application Context -->
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:spring.xml</param-value>
</context-param>

<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>


<!-- Spring请求拦截器 -->
<servlet>
    <servlet-name>iamdigger-web</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <load-on-startup>1</load-on-startup>
</servlet>
<!-- 拦截所有的请求 -->
<servlet-mapping>
    <servlet-name>iamdigger-web</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
````

#### 静态资源配置
````xml
<!-- exclude static files -->
<mvc:annotation-driven/>
<mvc:resources mapping="/static/js/**" location="/static/js/" />
<mvc:resources mapping="/static/css/**" location="/static/css/" />
````
annotation-driven非常重要，不配的话，resources会不起作用。

#### 根目录请求Controller
````java
@Controller
public class IndexController {

    @RequestMapping(value = "/")
    public String index(Map<String, Object> context) throws Exception {
        return "index";
    }
}
````
