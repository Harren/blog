---
title: SpringBoot自定义方法参数解析器
date: 2017-05-07 14:31:43
categories:
- 技术杂谈
tags:
- SpringBoot
---
最近整个部门在向Java技术转型，新的业务全部都是Java实现，但是由于之前老的系统主要是php接口，其定义的参数都是下划线的个是，因此，在写Rest服务接口时特别别扭，但是为了兼容老系统的接口，需要在请求和接受参数和原来的保持一致，所以实现一个参数解析器将参数下划线形式转换为Java常用的驼峰形式，之后查阅了Springboot的相关资料，大致思路如下：自定义参数解析器实现HandlerMethidArgumentResolver接口，将带有下划线的参数转换成驼峰形式，这样就可以在请求上就直接映射到实体类上到属性

首先，我们需要注明请求中到哪些参数需要进行处理，类似于注解 @RequestParam 等，自定义一个参数注解如下

```Java
@Target(value = ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface ParamModel {
}
```

其次，是定义下划线参数转换为驼峰参数到解析器UnderlineToCamelArgumentResolver，其代码如下：

```Java
public class UnderlineToCamelArgumentResolver implements HandlerMethodArgumentResolver {
    /**
     * 匹配下划线的格式
     */
    private static Pattern pattern = Pattern.compile("_(\\w)");

    private static String underLineToCamel(String source) {
        Matcher matcher = pattern.matcher(source);
        StringBuffer sb = new StringBuffer();
        while (matcher.find()) {
            matcher.appendReplacement(sb, matcher.group(1).toUpperCase());
        }
        matcher.appendTail(sb);
        return sb.toString();
    }

    @Override
    public boolean supportsParameter(MethodParameter methodParameter) {
        return methodParameter.hasParameterAnnotation(ParamModel.class);
    }


    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer container,
            NativeWebRequest webRequest, WebDataBinderFactory binderFactory) {
        return handleParameterNames(parameter, webRequest);
    }

    private Object handleParameterNames(MethodParameter parameter, NativeWebRequest webRequest) {
        Object obj = BeanUtils.instantiate(parameter.getParameterType());
        BeanWrapper wrapper = PropertyAccessorFactory.forBeanPropertyAccess(obj);
        Iterator<String> paramNames = webRequest.getParameterNames();
        while (paramNames.hasNext()) {
            String paramName = paramNames.next();
            Object o = webRequest.getParameter(paramName);
            try {
                wrapper.setPropertyValue(underLineToCamel(paramName), o);
            } catch (BeansException e) {

            }
        }
        return obj;
    }


}
```

其中，在 handleParameterNames 方法中去实现参数名的转换，underLineToCamel 就是将下划线转成驼峰的形式，然后将修改完的参数名和值重新放入请求。

在定义了解析器后，需要将自定义的解析器添加到配置中去，首先，需要定义一个WebConfig配置类需要继承WebMvcConfigurerAdapter,使用@Configuration注解，并重写addArgumentResolvers方法，将自定义解析器加入到argumentResolvers的List中去

```Java
@Configuration
public class WebConfig extends WebMvcConfigurationSupport {

//    如果请求方式为Put，则需要注册该过滤器
//    @Bean
//    public HttpPutFormContentFilter httpPutFormContentFilter() {
//        return new HttpPutFormContentFilter();
//    }

    /**
     * 添加参数解析，将参数的形式从下划线转化为驼峰
     * @param argumentResolvers
     */
    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
        argumentResolvers.add(new UnderlineToCamelArgumentResolver());
        super.addArgumentResolvers(argumentResolvers);
    }

}
```

该解析器只会对Post和Get的参数进行解析，对于Put方法的参数无法获取到，也无法直接映射到实体对象，jsp或者说html中的form的method值只能为post或get

**如果是使用的是PUT方式，SpringMVC默认将不会辨认到请求体中的参数，或者也有人说是Spirng MVC默认不支持 PUT请求带参数**

我们可以通过HiddenHttpMethodFilter获取put表单中的参数-值，而在Spring3.0中获取put表单的参数-值还有另一种方法，即使用HttpPutFormContentFilter过滤器。HttpPutFormContentFilter过滤器的作为就是获取put表单的值，并将之传递到Controller中标注了method为RequestMethod.put的方法中（PS:  需要注意的是，该过滤器只能接受enctype值为application/x-www-form-urlencoded的表单）