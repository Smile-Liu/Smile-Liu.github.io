---
title: Spring序列化和反序列化
description: Spring&SpringBoot序列化和反序列化的研究
date: 2020-03-10
category: ["Technology","Java","Spring","SpringBoot"]
tags: "Technology"
---
&emsp;&emsp;最近在开发时遇到一个现象：PO实体（从数据库里查出来的）中为`null`值的`Long`、`Integer`、`Double`等非`String`类型的字段，经过HTTP调用，得到的返回值都成了`-1`，然后页面操作再提交到后台时，这些字段的值就变成了`-1`，也就是说，本来是`null`的字段经过一次前端就变成了`-1`，然后以`-1`入库。而`-1`又可能是业务数据，去不得，改不了。所幸最后前后端协作解决了这个问题（解决方案写在最后），但我认为可以通过序列化的方式就可以解决，所以要研究透彻这个`Spring`序列化的原理，以及如何自定义序列化。
<!-- more -->

# 数据处理在HTTP中的位置
&emsp;&emsp;在`DispatchereServlet`通过`HandlerMapping`获得`HandlerExecutionChain`，拿到`HandlerAdapter`后，会针对每个请求实例化一个`ServletInvocableHandlerMethod(父类是InvocableHandlerMethod)`来进行请求（读入，`ServletInvocableHandlerMethod`的属性`argumentResolvers`来处理。该属性是`HandlerMethodArgumentResolverComposite`类，组合模式）和响应（写出，`ServletInvocableHandlerMethod`的属性`returnValueHandlerMethod`来处理。该属性是`HandlerMethodReturnValueHandlerComposite`类）的数据处理。以`Processor`结尾的类既是读取又是写入<br/>

**读入**：数据读入的核心接口是`HandlerMethodArgumentResolver`
```java
boolean supportsParameter(MethodParameter parameter); // 检测该类是否支持

Object resolverArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory); // 解析参数
```
&emsp;&emsp;`@InvocableHandlerMethod # getMethodArgumentValues()`找到方法的所有参数`MethodParameter[]`，遍历，由`HandlerMethodArgumentResolver # supportsParameter`匹配每个参数的解析器（在`HandlerMethodArgumentResolverComposite`里注册的`HandlerMethodArgumentResolver`有26个之多，在`RequestMappingHandlerAdapter`里面注册的），然后调用`HandlerMethodArgumentResolver # resolveArgument`解析出这个参数的参数值。<br/>
&emsp;&emsp;有以下几类解析器（摘自[《SpringMVC 4.3 源码分析之 HandlerMethodArgumentResolver》](https://www.jianshu.com/p/f4653fe8c935)）：
1. 基于`Name`从`URI Template Variable`, `HttpServletRequest`, `HttpSession`, `Http Header`中获取数据

取值方式|解析器类名|说明
-|-|-
直接定义的参数、`@RequestParam`修饰的简单类型|`RequestParamMethodArgumentResolver`|-
`@SessionAttribute`修饰的参数|`SessionAttributeMethodArgumentResolver`|一般是通过`HttpSevletRequest.getAttribute(name,RequestAttributes.SCOPE_SESSION)`获取
`@RequestHeader`修饰的非Map类型参数|`RequestHeaderMethodArgumentResolver`|通过`HttpServletRequest.getHeaderValues(name)`获取
`@RequestAttribute`修饰的参数|`RequestAttributeMethodArgumentResolver`|通过`HttpSevletRequest.getAttribute(name,RequestAttributes.SCOPE_REQUEST)`获取
`@PathVariable`修饰的参数|`PathVariableMethodArgumentResolver`|对应URI中的数据
`@Value`修饰的参数|`ExpressionValueMethodArgumentResolver`|返回配置数据
`@CookieValue`修饰的参数|`ServletCookieValueMethodArgumentResolver`|通过`HttpServletRequest.getCookies`获取
这类解析器的基类是`AbstractNamedValueMethodArgumentResolver`（这里有策略和模板模式）

2. 参数类型是Map的读取数据

取值方式|解析器类名|说明
-|-|-
`@RequestParam`修饰的`Map`|`RequestParamMapMethodArgumentResolver`|Map是`LinkedHashMap`
`@RequestHeader`修饰的Map|`RequestHeaderMapMethodArgumentResolver`|-
`@PathVariable`修饰的Map|`PathVariableMapMethodArgumentResolver`|把URI中定义的变量收集到参数里面
直接写的Map|`MapMethodProcessor`|从`ModelAndViewContainer`中获取`Model`里面的数据

3. 参数类型是一个自定义对象的读取数据

取值方式|解析器类名|说明
-|-|-
对象类型（Not Simple），或者加注解`@ModelAttribute`不限类型|`ModelAttributeMethodProcessor`|实际的类是`ServletModelAttributeMethodProcessor`

4. 基于`Content-Type`利用`HttpMessageConverter`将输入流转换成对应的参数

取值方式|解析器类名|说明
-|-|-
`@RequestPart`修饰的参数|`RequestPartMethodArgumentResolver`|处理`MultipartFile`或`javax.servlet.http.Part`类型的参数
`HttpEntity`或`RequestEntity`类型的参数|`HttpEntityMethodProcessor`|-
`@RequestBody`修饰的参数|`RequestResponseBodyMethodProcessor`|利用`HttpMessageConverter`进行参数转换
这类解析器的基类是`AbstractMessageConverterMethodArgumentResolver`
<p>

**写出**：数据写出的核心接口是`HandlerMethodReturnValueResolver`
```java
boolean supportsReturnType(MethodParameter returnType); // 检测该类是否支持

void handlerReturnValue(Object returnValue, MethodParameter returnType, ModelAndViewContainer mavContainer, NativeWebRequest webRequest); // 解析参数
```
&emsp;&emsp;根据返回值类型，总结以下几个常用的解析器：
1. 返回值是字符串的

解析器类名|说明
-|-
`ViewNameMethodReturnValueHandler`|把返回值做为视图名称`ViewName`放入`ModelAndView`中，并将其返回

2. 返回值是`Map`

解析器类名|说明
-|-
`MapMethodProcessor`|把返回值的每个字段和值放入`ModelAndViewContainer`

3. 返回值是自定义对象

解析器类名|说明
-|-
`ModelAttributeMethodProcessor`|实际是`ServletModelAttributeMethodProcessor`，以自定义对象的类名为Key，对象值为Value，放入`ModelAndViewContainer`，跳转到默认的`View`

4. `@ResponseBody`注解修饰

解析器类名|说明
-|-
`RequestResponseBodyMethodProcessor`|实际是`AbstractMessageConverterMethodProcessor`，通过输入输出流`InputStream&OutputStream`的方式将数据转交给`HttpMessageConverter`，由它进行后续的处理

# `HttpMessageConverter`的处理过程
``` java
boolean canRead(Class<?> clazz, MediaType mediaType>); // 校验是否可读取
boolean canWrite(Class<?> clazz, MediaType mediaType); // 校验是否可写出
List<MediaType> getSupportedMediaTypes();
T read(Class<? extends T> clazz, HttpInputMessage inputMessage); // 读取
void write(T t, MediaType mediaType, HttpOutputMessage outputMessage); // 写出
```

（摘自 [《springboot学习（三）——使用HttpMessageConverter进行http序列化和反序列化》](https://segmentfault.com/a/1190000012658289) <br/>
&emsp;&emsp;HTTP协议的处理过程，TCP字节流 <---> HttpRequest/HttpResponse <---> HttpMessageConverter <---> 内部对象，涉及到两个序列化。第一个是TCP字节流到HttpRequest和HttpResponse到TCP字节流，这一步Servlet容器已经处理好了；第二个是框架内部的HttpMessageConverter要做的工作。<br/>
![HTTP](https://s1.ax1x.com/2020/03/16/8JZ656.png)<br/>


## MediaType
类名|支持的JavaType|支持的MediaType
-|-|-
`ByteArrayHttpMessageConverter`|byte[]|application/octet-stream, */*
`StringHttpMessageConverter`|String|text/plain,*/*
`ResourceHttpMessageConverter`|Resource|*/*
`SourceHttpMessageConverter`|Source|application/xml,text/xml,application/*+xml
`AllEncompassingFormHttpMessageConverter`|Map<K,List<?>>|application/x-www-form-urlencoded,multipart/form-data
`MappingJackson2HttpMessageConverter`|Object|application/json,application/*+json
`Jaxb2BootElementHttpMessageConverter`|Object|application/xml,text/xml,application/*+xml
`FastJsonHttpMessageConverter`|Object|*/*

## `HttpMessageConverter`数据组装
&emsp;&emsp;核心接口是`HttpMessageConverter`。框架将其实现类组装成一个List，在从请求和响应中读取数据时，挨个遍历，找到能解析数据的一个实现类去执行真正的逻辑。
&emsp;&emsp;这个List的赋值渠道有三个，分别是：
1. 没有其他配置的情况下，默认使用`ha（@RequestMapping注解对应的RequestMappingHandlerAdapter）`构造器添加的这四个`HttpMessageConverter`
    - `ByteArrayHttpMessageConverter`，处理字节
    - `StringHttpMessageConverter`，处理字符串
    - `SourceHttpMessgeConverter`，
    - `AllEncompassingFormHttpMessageConverter`
2. 配置xml文件中使用`mvc:annotation-driven`的子标签`mvc:message-converters`。会在类`AnnotationDrivenBeanDefinitionParser`中解析，如果依赖了`jackson`或者`gson`，则自动将`MappingJackson2HttpMessageConverter`注册成`HttpMessageConverter`
3. 通过`RequestMappingHandlerAdapter#setMessageConverter`方法进行设置

## MediaType的匹配规则
&emsp;&emsp;HTTP请求的Header中有`Accept`和`Content-Type`这两个参数。`Accept`表示发送端（客户端）希望接受的类型；`Content-Type`代表发送端（客户端）发送的实体数据的类型。一个请求如果没有指定`Accept`，则视为默认的 `*/*`；如果没有指定`Content-Type`，视为空，而且必须是具体的类型，不能包含 `*`。
在`AbstractMessageConverterMethodProcessor#writeWithMessageConverters`中写明了匹配关系。
``` java
// 读入 @AbstractMessageConverterMethodArgumentResolver
protected <T> Object readWithMessageConverters(HttpInputMessage inputMessage, MethodParameter paramter, Type targetType) {

    MediaType contentType;
    boolean noContentType = false;
    try {
        contentType = inputMessage.getHeaders().getContentType();
    } catch (InvalidMediaTypeException ex) {
        throw new HttpMediaTypeNotSupportedException(ex.getMessage());
    }
    if (contentType == null) {
        noContentType = true;
        contentType = MediaType.APPLICATION_OCTET_STREAM;
    }

    Class<?> contextClass = parameter.getContaingClass();
    Class<T> targetClass = (targetType instanceof Class ? (Class<T>) target : null);
    if (targetClass == null) {
        ResolvableType resolvableType = ResolvableType.forMethodParameter(parameter);
        targetClass = (Class<T>) resolvableType.resolve();
    }
    HttpMethod httpMethod = (inputMessage instanceof HttpRequest ? ((HttpRequest) inputMessage).getMethod() : null);
    Object body = NO_VALUE;

    EmptyBodyCheckingHttpInputMessage message;
    try {
        message = new EmptyBodyCheckingHttpInputMessage(inputMessage);

        for (HttpMessageConverter<?> converter : this.messageConverters) {
            Class<HttpMessageConverter<?>> converterType = (Class<HttpMessageConverter<?>>)  converter.getClass();
            GenericHttpMessageConverter<?> genericConverter = (converter instanceof GrenericHttpMessageConverter ? (GenericHttpMessageConverter<?>)) converter : null;
            if (genericConverter != null ? genericConverter.canRead(targetType, contextClass, contentType) : (targetClass != null && converter.canRead(targetType, contentType))) {
                if (message.hasBody()) {
                    HttpInputMessage msgToUse = getAdvice().beforeBodyRead(message, parameter, targetType, converterTypes);
                    body = (genericConverter != null ? genericConverter.read(targetType, contextClass, msgToUse) : ((HttpMessageeConverter<T>) converter).read(targetClass, msgToUse));
                    body = getAdvice().afterBodyRead(body, msgToUse, parameter, targetType, converterType);
                } else {
                    body = getAdvice().handleEmptyBody(null, message, parameter, targetType, converterType);
                }
                break;
            }
        }
    } catch (IOException ex) {}
}


// 这是写出的一二三步 @AbstractMessageConverterMethodProcessor
// 写出第一步
// 取出客户端希望接收的MediaType
// 根据Controller的返回类型，遍历所有的Converter，找到能够返回的MediaType
HttpServletRequest request = inputMessage.getServletRequest();
List<MediaType> acceptableTypes = getAcceptableMediaTypes(request);
List<MediaType> producibleTypes = getProducibleMediaTypes(request, valueType, targetType);

if (body != null && producibleTypes.isEmpty()) {
    throw new HttpMessageNotWritableException("No converter found for return value of type: " + valueType);
}

// 写出第二步
// 遍历找到的两个List<MediaType>，判断是否兼容，然后取到更具体的MediaType
// 把具体的MediaType列表进行排序，按照 q值（QualifierFactor，品质因子） 和 具体程度 排序，一般是大的优先
List<MediaType> mediaTypesToUse = new ArrayList<>();
for (MediaType requestedType : acceptableTypes) {
    for (MediaType producibleType : producibleTypes) {
        if (requestedType.isCompatibleWith(producibleType)) {
            mediaTypesToUse.add(getMostSpecificMediaType(requestedType, producibleType));
        }
    }
}
if (mediaTypesToUse.isEmpty()) {
    if (body != null) {
        throw new HttpMediaTypeNotAcceptableException(producibleTypes);
    }
    return;
}
MediaType.sortBySpecificityAndQuality(mediaTypesToUse);

for (MediaType mediaType : mediaTypesToUse) {
    if (mediaType.isConrete()) {
        selectedMediaType = mediaType;
        break;
    } else if (mediaType.isPresentIn(ALL_APPLICATION_MEDIA_TYPES)) {
        selectedMediaType = MediaType.APPLICATION_OCTET_STREAM;
        break;
    }
}

// 写出第三步
// 选择一个能处理MediaType的Converter，根据添加顺序进行优先级排序。排在前面的优先处理
if (selectedMediaType != null) {
    selectedMediaType = selectedMediaType.removeQualityValue();
    for (HttpMessageConverter<?> converter : this.messageConverters) {
        GenericHttpMessageConverter genericConverter = (converter instanceof GenericHttpMessageConverter ? (GenericHttpMessageConverter<?>) converter : null);
        if (genericConverter != null ? ((GenericHttpMessageConverter) converter).canWrite(targetType, valueType, selectedMediaType) : converter.canWrite(valueType, selectedMediaType)) {
            body = getAdvice().beforeBodyWrite(body, returnType, selectedMediaType, (Class<? extends HttpMessageConverter<?>>) converter.getClass(), inputMessage, outputMessage);

            if (body != null) {
                Object theBody = body;
                addContentDispositionHeader(inputMessage, outputMessage);
                if (genericConverter != null) {
                    genericConverter.write(body, targetType, selectedMediaType, outputMessage);
                } else {
                    ((HttpMessageConverter) converter).write(body, selectedMediaType, outputMessage);
                }
            }
        }
    }
}
```

# SpringBoot的序列化和反序列化
&emsp;&emsp;SpringBoot也是基于SpringMVC的，所以对于参数的读取和数据的返回也是一样的原理。用的最多的还是`@RequestBody`和`@ResponseBody`（`@RestController`包含该注解）

# 如何自定义
1. 方法一：实现`HandlerMethodArgumentResolver`、`HandlerMethodReturnValueResolver`
2. 方法二：实现`HttpMessageConverter`（继承`AbstractHttpMessageConverter`），并将它放入`AbstractMessageConverterMethodArgumentResolver#messageConverters`的List中，根据前后顺序排列优先级，将它匹配到具体的处理逻辑中；编写`WebConfig`（通过继承`WebMvcOnfigurerAdapter`）

# 最后
&emsp;&emsp;学好知识，努力解决Bug

# 简单的解决方案
&emsp;&emsp;因为`SpringBoot`默认的序列化是使用`jackson`，而它的基本原理是调用`get`和`set`方法，所以在具体的实体里面重写`get`和`set`方法，对特殊的数据进行修改，然后让前端也对特殊数据进行处理就可以解决这个问题。特别注意：因为`Mybatis`的`SQL`语句中的数据也使用`get`和`set`，所以要注意查询条件和数据的正确性。
