这篇文章**主要介绍**下**`HttpMessageConverter`整个注册过程**，
包含**自定义的`HttpMessageConverter`**，然后对一些`HttpMessageConverter`进行具体介绍。

### HttpMessageConverter:
```java
public interface HttpMessageConverter<T> {  
  
    /** 
     * Indicates whether the given class can be read by this converter. 
     * @param clazz the class to test for readability 
     * @param mediaType the media type to read, can be {@code null} if not specified. 
     * Typically the value of a {@code Content-Type} header. 
     * @return {@code true} if readable; {@code false} otherwise 
     */  
    boolean canRead(Class<?> clazz, MediaType mediaType);  
  
    /** 
     * Indicates whether the given class can be written by this converter. 
     * @param clazz the class to test for writability 
     * @param mediaType the media type to write, can be {@code null} if not specified. 
     * Typically the value of an {@code Accept} header. 
     * @return {@code true} if writable; {@code false} otherwise 
     */  
    boolean canWrite(Class<?> clazz, MediaType mediaType);  
  
    /** 
     * Return the list of {@link MediaType} objects supported by this converter. 
     * @return the list of supported media types 
     */  
    List<MediaType> getSupportedMediaTypes();  
  
    /** 
     * Read an object of the given type form the given input message, and returns it. 
     * @param clazz the type of object to return. This type must have previously been passed to the 
     * {@link #canRead canRead} method of this interface, which must have returned {@code true}. 
     * @param inputMessage the HTTP input message to read from 
     * @return the converted object 
     * @throws IOException in case of I/O errors 
     * @throws HttpMessageNotReadableException in case of conversion errors 
     */  
    T read(Class<? extends T> clazz, HttpInputMessage inputMessage)  
            throws IOException, HttpMessageNotReadableException;  
  
    /** 
     * Write an given object to the given output message. 
     * @param t the object to write to the output message. The type of this object must have previously been 
     * passed to the {@link #canWrite canWrite} method of this interface, which must have returned {@code true}. 
     * @param contentType the content type to use when writing. May be {@code null} to indicate that the 
     * default content type of the converter must be used. If not {@code null}, this media type must have 
     * previously been passed to the {@link #canWrite canWrite} method of this interface, which must have 
     * returned {@code true}. 
     * @param outputMessage the message to write to 
     * @throws IOException in case of I/O errors 
     * @throws HttpMessageNotWritableException in case of conversion errors 
     */  
    void write(T t, MediaType contentType, HttpOutputMessage outputMessage)  
            throws IOException, HttpMessageNotWritableException;  
  
}  
```

- 从`HttpInputMessage`中读取数据： 
    `T read(Class<? extends T> clazz, HttpInputMessage inputMessage)`，前提`clazz`能够通过`canRead(Class<?> clazz, MediaType mediaType)`测试。
    
- 向`HttpOutputMessage`中写入数据：
    `void write(T t, MediaType contentType, HttpOutputMessage outputMessage)`，前提`clazz`能够通过`canWrite(Class<?> clazz, MediaType mediaType)`测试。
    
简单举例： 如`StringHttpMessageConverter`，
`read`方法就是根据**编码类型**将`HttpInputMessage`中的数据变为**字符串**。
`write`方法就是根据**编码类型**将**字符串**数据写入`HttpOutputMessage`中。

`HttpMessageConverter`的使用场景：
它主要是用来**转换request的内容到一定的格式**，**转换输出的内容的到response**。

看下自定义的使用方式：
```xml
<mvc:annotation-driven>  
    <mvc:message-converters register-defaults="true">  
        <bean class="org.springframework.http.converter.StringHttpMessageConverter">  
            <constructor-arg value="UTF-8"/>  
        </bean>  
    </mvc:message-converters>  
</mvc:annotation-driven>  
```
首先，在**对`mvc:annotation-driven`解析的`AnnotationDrivenBeanDefinitionParser`**中，有这么一个方法：
```java
ManagedList<?> messageConverters = getMessageConverters(element, source, parserContext); 
```
**获取所有的`HttpMessageConverter`**，最终设置到`RequestMappingHandlerAdapter`的`private List<HttpMessageConverter<?>> messageConverters`属性上。

看下具体的获取过程：
```java
private ManagedList<?> getMessageConverters(Element element, Object source, ParserContext parserContext) {  
    Element convertersElement = DomUtils.getChildElementByTagName(element, "message-converters");  
    ManagedList<? super Object> messageConverters = new ManagedList<Object>();  
    if (convertersElement != null) {  
        messageConverters.setSource(source);  
        for (Element beanElement : DomUtils.getChildElementsByTagName(convertersElement, "bean", "ref")) {  
            Object object = parserContext.getDelegate().parsePropertySubElement(beanElement, null);  
            messageConverters.add(object);  
        }  
    }  

    if (convertersElement == null || Boolean.valueOf(convertersElement.getAttribute("register-defaults"))) {
        messageConverters.setSource(source);  
        messageConverters.add(createConverterDefinition(ByteArrayHttpMessageConverter.class, source));  

        RootBeanDefinition stringConverterDef = createConverterDefinition(StringHttpMessageConverter.class, source);  
        stringConverterDef.getPropertyValues().add("writeAcceptCharset", false);  
        messageConverters.add(stringConverterDef);  

        messageConverters.add(createConverterDefinition(ResourceHttpMessageConverter.class, source));  
        messageConverters.add(createConverterDefinition(SourceHttpMessageConverter.class, source));  
        messageConverters.add(createConverterDefinition(AllEncompassingFormHttpMessageConverter.class, source));  

        if (romePresent) {  
            messageConverters.add(createConverterDefinition(AtomFeedHttpMessageConverter.class, source));  
            messageConverters.add(createConverterDefinition(RssChannelHttpMessageConverter.class, source));  
        }  
        if (jaxb2Present) {  
            messageConverters.add(createConverterDefinition(Jaxb2RootElementHttpMessageConverter.class, source));  
        }  
        if (jackson2Present) {  
            messageConverters.add(createConverterDefinition(MappingJackson2HttpMessageConverter.class, source));  
        }  
        else if (jacksonPresent) {  
            messageConverters.add(createConverterDefinition(  
                    org.springframework.http.converter.json.MappingJacksonHttpMessageConverter.class, source));  
        }  
    }
    return messageConverters;  
}
```
- 该过程**第一步**： **解析并获取我们自定义的HttpMessageConverter**，

- 该过程**第二步**： 
    由`if (convertersElement == null || Boolean.valueOf(convertersElement.getAttribute("register-defaults")))`知：
    `<mvc:message-converters register-defaults="true">`有一个**`register-defaults`属性，当为true时**，**仍然注册默认的HttpMessageConverter**，**当为false则不注册**，仅仅**使用用户自定义的HttpMessageConverter**。
    或者`message-converters`用户自定义的HttpMessageConverter为空时，也会注册默认的HttpMessageConverter。

获取完毕，便会将这些`HttpMessageConverter`设置进`RequestMappingHandlerAdapter`的`messageConverters`属性中。

然后，就是它的使用过程，`HttpMessageConverter`主要针对**那些不会返回view视图的response**： 即方法含有`@ResponseBody`或者返回值为`HttpEntity`等类型的，它们都会用到`HttpMessageConverter`。

以`@ResponseBody`举例： 首先**先决定由哪个`HandlerMethodReturnValueHandler`来处理返回值**，由于是`@ResponseBody`所以将会由`RequestResponseBodyMethodProcessor`来处理，然后就是如下的写入：
```java
protected <T> void writeWithMessageConverters(T returnValue, MethodParameter returnType,  
        ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage)  
        throws IOException, HttpMediaTypeNotAcceptableException {  

    Class<?> returnValueClass = returnValue.getClass();  
    HttpServletRequest servletRequest = inputMessage.getServletRequest();  
    List<MediaType> requestedMediaTypes = getAcceptableMediaTypes(servletRequest);  
    List<MediaType> producibleMediaTypes = getProducibleMediaTypes(servletRequest, returnValueClass);  

    Set<MediaType> compatibleMediaTypes = new LinkedHashSet<MediaType>();  
    for (MediaType requestedType : requestedMediaTypes) {  
        for (MediaType producibleType : producibleMediaTypes) {  
            if (requestedType.isCompatibleWith(producibleType)) {  
                compatibleMediaTypes.add(getMostSpecificMediaType(requestedType, producibleType));  
            }  
        }  
    }  
    if (compatibleMediaTypes.isEmpty()) {  
        throw new HttpMediaTypeNotAcceptableException(producibleMediaTypes);  
    }  

    List<MediaType> mediaTypes = new ArrayList<MediaType>(compatibleMediaTypes);  
    MediaType.sortBySpecificityAndQuality(mediaTypes);  

    MediaType selectedMediaType = null;  
    for (MediaType mediaType : mediaTypes) {  
        if (mediaType.isConcrete()) {  
            selectedMediaType = mediaType;  
            break;  
        }  
        else if (mediaType.equals(MediaType.ALL) || mediaType.equals(MEDIA_TYPE_APPLICATION)) {  
            selectedMediaType = MediaType.APPLICATION_OCTET_STREAM;  
            break;  
        }  
    }  

    if (selectedMediaType != null) {  
        selectedMediaType = selectedMediaType.removeQualityValue();  
        for (HttpMessageConverter<?> messageConverter : this.messageConverters) {  
            if (messageConverter.canWrite(returnValueClass, selectedMediaType)) {  
                ((HttpMessageConverter<T>) messageConverter).write(returnValue, selectedMediaType, outputMessage);  
                if (logger.isDebugEnabled()) {  
                    logger.debug("Written [" + returnValue + "] as \"" + selectedMediaType + "\" using [" +  
                            messageConverter + "]");  
                }  
                return;  
            }  
        }  
    }  
    throw new HttpMediaTypeNotAcceptableException(this.allSupportedMediaTypes);  
}
```
选取一个合适的`content-type`，再由这个**`content-type`**和**返回类型**来选取合适的`HttpMessageConverter`，找到合适的`HttpMessageConverter`后，便调用它的`write`方法。

接下来就说一说一些具体的`HttpMessageConverter`。 

- `AbstractHttpMessageConverter`：提供了进一步的抽象，将是否支持相应的MediaType这一共有的功能实现，它的子类只需关心是否支持返回类型。
- `AbstractHttpMessageConverter`子类 - `StringHttpMessageConverter`：如用于处理字符串到response中，这就要涉及编码问题，这一过程在本系列的第四篇文章中做过详细说明，这里跳过。
- `AbstractHttpMessageConverter`子类 - `ByteArrayHttpMessageConverter`：

    ```java
    public class ByteArrayHttpMessageConverter extends AbstractHttpMessageConverter<byte[]> {  
      
        /** Creates a new instance of the {@code ByteArrayHttpMessageConverter}. */  
        public ByteArrayHttpMessageConverter() {  
            super(new MediaType("application", "octet-stream"), MediaType.ALL);  
        }  
      
        @Override  
        public boolean supports(Class<?> clazz) {  
            return byte[].class.equals(clazz);  
        }  
      
        @Override  
        public byte[] readInternal(Class<? extends byte[]> clazz, HttpInputMessage inputMessage) throws IOException {  
            long contentLength = inputMessage.getHeaders().getContentLength();  
            ByteArrayOutputStream bos =  
                    new ByteArrayOutputStream(contentLength >= 0 ? (int) contentLength : StreamUtils.BUFFER_SIZE);  
            StreamUtils.copy(inputMessage.getBody(), bos);  
            return bos.toByteArray();  
        }  
      
        @Override  
        protected Long getContentLength(byte[] bytes, MediaType contentType) {  
            return (long) bytes.length;  
        }  
      
        @Override  
        protected void writeInternal(byte[] bytes, HttpOutputMessage outputMessage) throws IOException {  
            StreamUtils.copy(bytes, outputMessage.getBody());  
        }  
      
    }  
    ```
    源码就很清晰明了。它专门负责byte[]类型的转换。
    
- `AbstractHttpMessageConverter`子类 -`MappingJacksonHttpMessageConverter`：用于转换Object到json字符串类型。已过时,使用的是http://jackson.codehaus.org中Jackson 1.x的ObjectMapper，取代者为`MappingJackson2HttpMessageConverter`。依赖为：

    ```xml
    <dependency>   
        <groupId>org.codehaus.jackson</groupId>   
        <artifactId>jackson-core-asl</artifactId>   
        <version>1.9.11</version>   
    </dependency>   
      
    <dependency>   
        <groupId>org.codehaus.jackson</groupId>   
        <artifactId>jackson-mapper-asl</artifactId>   
        <version>1.9.11</version>   
    </dependency>   
    ```
    
- `AbstractHttpMessageConverter`子类 - `MappingJackson2HttpMessageConverter`： 它所使用的json转换器是http://jackson.codehaus.org中Jackson 2.x的ObjectMapper。 依赖的jar包为有3个，`jackson-databind`和它的两个依赖`jackson-annotations`、`jackson-core`，但是有了jackson-databind的pom文件会去自动下载它的依赖，所以只需增添jackson-databind的pom即可获取上述3个jar包：
    
    ```xml
    <dependency>  
        <groupId>com.fasterxml.jackson.core</groupId>  
        <artifactId>jackson-databind</artifactId>  
        <version>2.4.2</version>   
    </dependency>  
    ```
    
接下来便说道：在注册HttpMessageConverter过程中的一些问题：
```java
private ManagedList<?> getMessageConverters(Element element, Object source, ParserContext parserContext) {  
    Element convertersElement = DomUtils.getChildElementByTagName(element, "message-converters");  
    ManagedList<? super Object> messageConverters = new ManagedList<Object>();  
    if (convertersElement != null) {  
        messageConverters.setSource(source);  
        for (Element beanElement : DomUtils.getChildElementsByTagName(convertersElement, "bean", "ref")) {  
            Object object = parserContext.getDelegate().parsePropertySubElement(beanElement, null);  
            messageConverters.add(object);  
        }  
    }  

    if (convertersElement == null || Boolean.valueOf(convertersElement.getAttribute("register-defaults"))) {  
        messageConverters.setSource(source);  
        messageConverters.add(createConverterDefinition(ByteArrayHttpMessageConverter.class, source));  

        RootBeanDefinition stringConverterDef = createConverterDefinition(StringHttpMessageConverter.class, source);  
        stringConverterDef.getPropertyValues().add("writeAcceptCharset", false);  
        messageConverters.add(stringConverterDef);  

        messageConverters.add(createConverterDefinition(ResourceHttpMessageConverter.class, source));  
        messageConverters.add(createConverterDefinition(SourceHttpMessageConverter.class, source));  
        messageConverters.add(createConverterDefinition(AllEncompassingFormHttpMessageConverter.class, source));  

        if (romePresent) {  
            messageConverters.add(createConverterDefinition(AtomFeedHttpMessageConverter.class, source));  
            messageConverters.add(createConverterDefinition(RssChannelHttpMessageConverter.class, source));  
        }  
        if (jaxb2Present) {  
            messageConverters.add(createConverterDefinition(Jaxb2RootElementHttpMessageConverter.class, source));  
        }  
        if (jackson2Present) {  
            messageConverters.add(createConverterDefinition(MappingJackson2HttpMessageConverter.class, source));  
        }  
        else if (jacksonPresent) {  
            messageConverters.add(createConverterDefinition(  
                    org.springframework.http.converter.json.MappingJacksonHttpMessageConverter.class, source));  
        }  
    }  
    return messageConverters;  
}
```
这段代码是在注册默认的`HttpMessageConverter`，但是个别`HttpMessageConverter`也是有条件的。即相应的jar包存在，才会去注册它。
如`MappingJackson2HttpMessageConverter`，
```java
if (jackson2Present) {
messageConverters.add(createConverterDefinition(MappingJackson2HttpMessageConverter.class, source));
```
当`jackson2Present`为true时才会注册。而jackson2Present的值如下：
```java
private static final boolean jackson2Present =  
    ClassUtils.isPresent("com.fasterxml.jackson.databind.ObjectMapper", AnnotationDrivenBeanDefinitionParser.class.getClassLoader()) &&  
            ClassUtils.isPresent("com.fasterxml.jackson.core.JsonGenerator", AnnotationDrivenBeanDefinitionParser.class.getClassLoader());  
```
也就是当`com.fasterxml.jackson.databind.ObjectMapper`和`com.fasterxml.jackson.core.JsonGenerator`存在在classpath中才会去加载`MappingJackson2HttpMessageConverter`。 

同理，`MappingJacksonHttpMessageConverter`的判断如下：
```java
private static final boolean jacksonPresent =  
    ClassUtils.isPresent("org.codehaus.jackson.map.ObjectMapper", AnnotationDrivenBeanDefinitionParser.class.getClassLoader()) &&  
        ClassUtils.isPresent("org.codehaus.jackson.JsonGenerator", AnnotationDrivenBeanDefinitionParser.class.getClassLoader());  
```

所以当我们程序没法转换json时，你就需要考虑是否已经把`MappingJacksonHttpMessageConverter`或者`MappingJackson2HttpMessageConverter`的依赖加进来了，
**官方推荐使用`MappingJackson2HttpMessageConverter`**。