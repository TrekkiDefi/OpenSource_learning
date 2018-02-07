```java
public class ContextLoader {
    
    private static final String DEFAULT_STRATEGIES_PATH = "ContextLoader.properties";
    
    private static final Properties defaultStrategies;

    static {
        // Load default strategy implementations from properties file.
        // This is currently strictly internal and not meant to be customized
        // by application developers.
        try {
            ClassPathResource resource = new ClassPathResource(DEFAULT_STRATEGIES_PATH, ContextLoader.class);
            defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
        }
        catch (IOException ex) {
            throw new IllegalStateException("Could not load 'ContextLoader.properties': " + ex.getMessage());
        }
    }
    
    
    public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
        // 如果已经存在根上下文，则抛出异常。故web.xml配置了ContextLoaderListener，就不要在JavaConfig中重复配置。
        if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
            throw new IllegalStateException(
                    "Cannot initialize context because there is already a root application context present - " +
                    "check whether you have multiple ContextLoader* definitions in your web.xml!");
        }
    
        Log logger = LogFactory.getLog(ContextLoader.class);
        servletContext.log("Initializing Spring root WebApplicationContext");
        if (logger.isInfoEnabled()) {
            logger.info("Root WebApplicationContext: initialization started");
        }
        long startTime = System.currentTimeMillis();
    
        try {
            // 在本地实例变量中存储上下文，以确保它在ServletContext关闭时可用。
            if (this.context == null) {
                this.context = createWebApplicationContext(servletContext);
            }
            if (this.context instanceof ConfigurableWebApplicationContext) {
                ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
                if (!cwac.isActive()) {
                    // 上下文尚未刷新 -> 提供设置父上下文，设置应用程序上下文ID等服务
                    if (cwac.getParent() == null) {
                        // The context instance was injected without an explicit parent ->
                        // determine parent for root web application context, if any.
                        // 上下文实例被注入，没有明确的父上下文 -> 为根Web应用程序上下文确定父上下文（如果有的话）。
                        ApplicationContext parent = loadParentContext(servletContext);
                        cwac.setParent(parent);
                    }
                    // TODO
                    configureAndRefreshWebApplicationContext(cwac, servletContext);
                }
            }
            // 以`WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE`为属性Key，将根上下文存储到ServletContext中
            servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
    
            // TODO
            ClassLoader ccl = Thread.currentThread().getContextClassLoader();
            if (ccl == ContextLoader.class.getClassLoader()) {
                currentContext = this.context;
            }
            else if (ccl != null) {
                currentContextPerThread.put(ccl, this.context);
            }
    
            if (logger.isDebugEnabled()) {
                logger.debug("Published root WebApplicationContext as ServletContext attribute with name [" +
                        WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE + "]");
            }
            if (logger.isInfoEnabled()) {
                long elapsedTime = System.currentTimeMillis() - startTime;
                logger.info("Root WebApplicationContext: initialization completed in " + elapsedTime + " ms");
            }
    
            return this.context;
        }
        catch (RuntimeException ex) {
            logger.error("Context initialization failed", ex);
            servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
            throw ex;
        }
        catch (Error err) {
            logger.error("Context initialization failed", err);
            servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, err);
            throw err;
        }
    }
    
    /**
    * 
    * 返回要使用的`WebApplicationContext`实现类，默认的`XmlWebApplicationContext`或指定的自定义上下文类。
    * 
    * @param servletContext 当前的servlet上下文
    * @return 要使用的WebApplicationContext实现类
    * @see #CONTEXT_CLASS_PARAM
    * @see org.springframework.web.context.support.XmlWebApplicationContext
    */
    protected Class<?> determineContextClass(ServletContext servletContext) {
        // 如果在web.xml中指定了<context-param> contextClass，则使用该指定上下文类。
        String contextClassName = servletContext.getInitParameter(CONTEXT_CLASS_PARAM);
        if (contextClassName != null) {
            try {
                return ClassUtils.forName(contextClassName, ClassUtils.getDefaultClassLoader());
            }
            catch (ClassNotFoundException ex) {
                throw new ApplicationContextException(
                        "Failed to load custom context class [" + contextClassName + "]", ex);
            }
        }
        else {
            // 使用默认策略，去ContextLoader.properties配置文件中查找类名为org.springframework.web.context.WebApplicationContext的上下文类，
            // 即：org.springframework.web.context.support.XmlWebApplicationContext
            contextClassName = defaultStrategies.getProperty(WebApplicationContext.class.getName());
            try {
                return ClassUtils.forName(contextClassName, ContextLoader.class.getClassLoader());
            }
            catch (ClassNotFoundException ex) {
                throw new ApplicationContextException(
                        "Failed to load default context class [" + contextClassName + "]", ex);
            }
        }
    }
    
    /**
    * 
    * 实例化此加载器的根WebApplicationContext，默认上下文类或自定义上下文类。
    * 
    * <p>此实现期望自定义上下文实现{@link ConfigurableWebApplicationContext}接口。可以在子类中被覆盖。
    * <p>此外，{@link #customizeContext}在刷新上下文之前被调用，允许子类对上下文执行自定义修改。
    * 
    * @param sc 当前的servlet上下文
    * @return 根WebApplicationContext
    * @see ConfigurableWebApplicationContext
    */
    protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
        // 获取上下文类Class对象
        Class<?> contextClass = determineContextClass(sc);
        if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {// 如果上下文类不是ConfigurableWebApplicationContext类型，则抛出异常
            throw new ApplicationContextException("Custom context class [" + contextClass.getName() +
                    "] is not of type [" + ConfigurableWebApplicationContext.class.getName() + "]");
        }
        return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);// 创建上下文类对象
    }
    
    /**
    * 具有默认实现的模板方法（可能被子类覆盖），以加载或获取将用作根WebApplicationContext(root WebApplicationContext)的父上下文的ApplicationContext实例。如果方法的返回值为空，则不设置父上下文。
    *  
    * <p>这里加载父上下文的主要原因是允许多个根Web应用程序上下文都是共享EAR上下文的子上下文，或者也可以共享EJB可见的相同的父上下文。对于纯Web应用程序，通常不需要担心根Web应用程序上下文(root web application context)有一个父上下文。
    * 
    * <p>默认实现使用{@link org.springframework.context.access.ContextSingletonBeanFactoryLocator}，通过{@link #LOCATOR_FACTORY_SELECTOR_PARAM}和{@link #LOCATOR_FACTORY_KEY_PARAM}进行配置。
    *      
    * @param servletContext 当前的servlet上下文
    * @return 父应用程序上下文，否则返回{@code null}
    * @see org.springframework.context.access.ContextSingletonBeanFactoryLocator
    */
    protected ApplicationContext loadParentContext(ServletContext servletContext) {
        ApplicationContext parentContext = null;
        String locatorFactorySelector = servletContext.getInitParameter(LOCATOR_FACTORY_SELECTOR_PARAM);
        String parentContextKey = servletContext.getInitParameter(LOCATOR_FACTORY_KEY_PARAM);

        if (parentContextKey != null) {
            // locatorFactorySelector may be null, indicating the default "classpath*:beanRefContext.xml"
            BeanFactoryLocator locator = ContextSingletonBeanFactoryLocator.getInstance(locatorFactorySelector);
            Log logger = LogFactory.getLog(ContextLoader.class);
            if (logger.isDebugEnabled()) {
                logger.debug("Getting parent context definition: using parent context key of '" +
                        parentContextKey + "' with BeanFactoryLocator");
            }
            this.parentContextRef = locator.useBeanFactory(parentContextKey);
            parentContext = (ApplicationContext) this.parentContextRef.getFactory();
        }

        return parentContext;
    }
}
```