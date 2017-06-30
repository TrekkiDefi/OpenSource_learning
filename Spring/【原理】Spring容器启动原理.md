Servlet3.0+规范后，允许Servlet，Filter，Listener不必声明在web.xml中，而是以硬编码的方式存在，实现容器的零配置。

`ServletContainerInitializer`：启动容器时负责加载相关配置

```java
package javax.servlet;

import java.util.Set;

public interface ServletContainerInitializer {
    void onStartup(Set<Class<?>> var1, ServletContext var2) throws ServletException;
}
```

容器启动时会自动扫描当前服务中`ServletContainerInitializer`的实现类。

并调用其onStartup方法，其参数Set<Class<?>> c，可通过在实现类上声明注解javax.servlet.annotation.HandlesTypes(xxx.class)注解自动注入。

@HandlesTypes会自动扫描项目中所有的xxx.class的实现类，并将其全部注入Set。

Spring为其提供了一个实现类：`SpringServletContainerInitializer`类:
```java
package org.springframework.web;

import java.lang.reflect.Modifier;
import java.util.Iterator;
import java.util.LinkedList;
import java.util.List;
import java.util.Set;
import javax.servlet.ServletContainerInitializer;
import javax.servlet.ServletContext;
import javax.servlet.ServletException;
import javax.servlet.annotation.HandlesTypes;
import org.springframework.core.annotation.AnnotationAwareOrderComparator;

@HandlesTypes({WebApplicationInitializer.class})// 自动扫描项目中所有的WebApplicationInitializer.class的实现类，并将其Class对象全部注入Set
public class SpringServletContainerInitializer implements ServletContainerInitializer {
    @Override
    public void onStartup(Set<Class<?>> webAppInitializerClasses, ServletContext servletContext)
            throws ServletException {

        // WebApplicationInitializer.class的实现类的对象集合
        List<WebApplicationInitializer> initializers = new LinkedList<WebApplicationInitializer>();

        if (webAppInitializerClasses != null) {
            for (Class<?> waiClass : webAppInitializerClasses) {// 遍历WebApplicationInitializer.class实现类的Class对象集合
                // 一些servlet容器为我们提供了无效的类，所以这里做校验
                // 1. 不能为接口；2. 不能为抽象类；3. 必须是WebApplicationInitializer的实现类
                if (!waiClass.isInterface() && !Modifier.isAbstract(waiClass.getModifiers()) &&
                        WebApplicationInitializer.class.isAssignableFrom(waiClass)) {
                    try {
                        initializers.add((WebApplicationInitializer) waiClass.newInstance());
                    }
                    catch (Throwable ex) {
                        throw new ServletException("Failed to instantiate WebApplicationInitializer class", ex);
                    }
                }
            }
        }

        if (initializers.isEmpty()) {
            servletContext.log("No Spring WebApplicationInitializer types detected on classpath");
            return;
        }

        servletContext.log(initializers.size() + " Spring WebApplicationInitializers detected on classpath");
        AnnotationAwareOrderComparator.sort(initializers);// 根据@Order排序
        for (WebApplicationInitializer initializer : initializers) {
            initializer.onStartup(servletContext);
        }
    }
}
```
从中可以看出，WebApplicationInitializer才是我们需要关心的接口，

我们只需要将相应的Servlet，Filter，Listener等硬编码到该接口的实现类中即可。

Spring为我们提供了一些WebApplicationInitializer的抽象类，我们只需要继承并按需修改即可。

这里我们拿Spring Session提供的AbstractHttpSessionApplicationInitializer类做说明。

```java
package org.springframework.session.web.context;

import java.util.Arrays;
import java.util.EnumSet;
import javax.servlet.DispatcherType;
import javax.servlet.Filter;
import javax.servlet.ServletContext;
import javax.servlet.ServletException;
import javax.servlet.FilterRegistration.Dynamic;
import org.springframework.core.Conventions;
import org.springframework.core.annotation.Order;
import org.springframework.util.Assert;
import org.springframework.web.WebApplicationInitializer;
import org.springframework.web.context.ContextLoaderListener;
import org.springframework.web.context.support.AnnotationConfigWebApplicationContext;
import org.springframework.web.filter.DelegatingFilterProxy;

@Order(100)
public abstract class AbstractHttpSessionApplicationInitializer implements WebApplicationInitializer {
    private static final String SERVLET_CONTEXT_PREFIX = "org.springframework.web.servlet.FrameworkServlet.CONTEXT.";
    public static final String DEFAULT_FILTER_NAME = "springSessionRepositoryFilter";
    private final Class<?>[] configurationClasses;

    protected AbstractHttpSessionApplicationInitializer() {
        this.configurationClasses = null;
    }

    protected AbstractHttpSessionApplicationInitializer(Class... configurationClasses) {
        this.configurationClasses = configurationClasses;
    }

    public void onStartup(ServletContext servletContext) throws ServletException {
        this.beforeSessionRepositoryFilter(servletContext);
        if(this.configurationClasses != null) {
            AnnotationConfigWebApplicationContext rootAppContext = new AnnotationConfigWebApplicationContext();
            rootAppContext.register(this.configurationClasses);
            servletContext.addListener(new ContextLoaderListener(rootAppContext));
        }

        this.insertSessionRepositoryFilter(servletContext);
        this.afterSessionRepositoryFilter(servletContext);
    }

    private void insertSessionRepositoryFilter(ServletContext servletContext) {
        String filterName = "springSessionRepositoryFilter";
        DelegatingFilterProxy springSessionRepositoryFilter = new DelegatingFilterProxy(filterName);
        String contextAttribute = this.getWebApplicationContextAttribute();
        if(contextAttribute != null) {
            springSessionRepositoryFilter.setContextAttribute(contextAttribute);
        }

        this.registerFilter(servletContext, true, filterName, springSessionRepositoryFilter);
    }

    protected final void insertFilters(ServletContext servletContext, Filter... filters) {
        this.registerFilters(servletContext, true, filters);
    }

    protected final void appendFilters(ServletContext servletContext, Filter... filters) {
        this.registerFilters(servletContext, false, filters);
    }

    private void registerFilters(ServletContext servletContext, boolean insertBeforeOtherFilters, Filter... filters) {
        Assert.notEmpty(filters, "filters cannot be null or empty");
        Filter[] var4 = filters;
        int var5 = filters.length;

        for(int var6 = 0; var6 < var5; ++var6) {
            Filter filter = var4[var6];
            if(filter == null) {
                throw new IllegalArgumentException("filters cannot contain null values. Got " + Arrays.asList(filters));
            }

            String filterName = Conventions.getVariableName(filter);
            this.registerFilter(servletContext, insertBeforeOtherFilters, filterName, filter);
        }

    }

    private void registerFilter(ServletContext servletContext, boolean insertBeforeOtherFilters, String filterName, Filter filter) {
        Dynamic registration = servletContext.addFilter(filterName, filter);
        if(registration == null) {
            throw new IllegalStateException("Duplicate Filter registration for '" + filterName + "'. Check to ensure the Filter is only configured once.");
        } else {
            registration.setAsyncSupported(this.isAsyncSessionSupported());
            EnumSet<DispatcherType> dispatcherTypes = this.getSessionDispatcherTypes();
            registration.addMappingForUrlPatterns(dispatcherTypes, !insertBeforeOtherFilters, new String[]{"/*"});
        }
    }

    private String getWebApplicationContextAttribute() {
        String dispatcherServletName = this.getDispatcherWebApplicationContextSuffix();
        return dispatcherServletName == null?null:"org.springframework.web.servlet.FrameworkServlet.CONTEXT." + dispatcherServletName;
    }

    protected String getDispatcherWebApplicationContextSuffix() {
        return null;
    }

    protected void beforeSessionRepositoryFilter(ServletContext servletContext) {
    }

    protected void afterSessionRepositoryFilter(ServletContext servletContext) {
    }

    protected EnumSet<DispatcherType> getSessionDispatcherTypes() {
        return EnumSet.of(DispatcherType.REQUEST, DispatcherType.ERROR, DispatcherType.ASYNC);
    }

    protected boolean isAsyncSessionSupported() {
        return true;
    }
}
```

```java
package com.github.ittalks.fn.config;

import org.springframework.session.web.context.AbstractHttpSessionApplicationInitializer;

/**
 * Created by 刘春龙 on 2017/6/30.
 */
public class HttpSessionApplicationInitializer extends AbstractHttpSessionApplicationInitializer {

    public HttpSessionApplicationInitializer() {
        super(HttpSessionConfig.class);
    }
}
```

---
对于一个web应用，其部署在web容器中，web容器提供其一个全局的上下文环境，这个上下文就是ServletContext，其为后面的Spring IoC容器提供宿主环境；

其次，在web.xml中会提供有`ContextLoaderListener`。这也是上面`AbstractHttpSessionApplicationInitializer`类注册的监听。

在web容器启动时，会触发容器初始化事件，此时ContextLoaderListener会监听到这个事件，其`contextInitialized`方法会被调用


```java

package org.springframework.web.context;

import javax.servlet.ServletContextEvent;
import javax.servlet.ServletContextListener;

public class ContextLoaderListener extends ContextLoader implements ServletContextListener {
    public ContextLoaderListener() {
    }

    public ContextLoaderListener(WebApplicationContext context) {
        super(context);
    }

    public void contextInitialized(ServletContextEvent event) {
        this.initWebApplicationContext(event.getServletContext());
    }

    public void contextDestroyed(ServletContextEvent event) {
        this.closeWebApplicationContext(event.getServletContext());
        ContextCleanupListener.cleanupAttributes(event.getServletContext());
    }
}
```

在`contextInitialized`这个方法中，spring会初始化一个启动上下文，这个上下文被称为根上下文，即WebApplicationContext，这是一个接口类。

确切的说，其实现类是`XmlWebApplicationContext`(基于web.xml配置)或者上面提及的`AnnotationConfigWebApplicationContext`(基于JavaConfig配置)。

这个就是Spring IoC容器，其对应的Bean定义的配置由web.xml中的context-param标签指定。

在这个IoC容器初始化完毕后，Spring以`WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE`为属性Key，将其存储到ServletContext中，便于获取。

```java
public interface WebApplicationContext extends ApplicationContext {
    String ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE = WebApplicationContext.class.getName() + ".ROOT";
}
```

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

再次，`ContextLoaderListener`监听器初始化完毕后，开始初始化web.xml中配置的Servlet，这个servlet可以配置多个，以最常见的`DispatcherServlet`为例， 

这个servlet实际上是一个标准的前端控制器，用以转发、匹配、处理每个servlet请求。 

`DispatcherServlet`上下文在初始化的时候会建立自己的IoC上下文，用以持有spring mvc相关的bean。 

在建立`DispatcherServlet`自己的IoC上下文时，会利用`WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE`先从`ServletContext`中获取之前的根上下文(即WebApplicationContext)作为自己上下文的parent上下文。 
有了这个parent上下文之后，再初始化自己持有的上下文。

这个`DispatcherServlet`初始化自己上下文的工作在其`initStrategies`方法中可以看到，大概的工作就是初始化处理器映射、视图解析等。

```java
//TODO
public class DispatcherServlet extends FrameworkServlet {
    
    protected void initStrategies(ApplicationContext context) {
            this.initMultipartResolver(context);
            this.initLocaleResolver(context);
            this.initThemeResolver(context);
            this.initHandlerMappings(context);
            this.initHandlerAdapters(context);
            this.initHandlerExceptionResolvers(context);
            this.initRequestToViewNameTranslator(context);
            this.initViewResolvers(context);
            this.initFlashMapManager(context);
        }
    
}
```

这个servlet自己持有的上下文默认实现类也是XmlWebApplicationContext。 

初始化完毕后，spring以与servlet的名字相关(此处不是简单的以servlet名为Key，而是通过一些转换，具体可自行查看源码)的属性为属性Key，也将其存到ServletContext中，以便后续使用。

```java
public abstract class FrameworkServlet extends HttpServletBean implements ApplicationContextAware {

    public static final String SERVLET_CONTEXT_PREFIX = FrameworkServlet.class.getName() + ".CONTEXT.";

    protected WebApplicationContext initWebApplicationContext() {
        ...
        if (this.publishContext) {
            // Publish the context as a servlet context attribute.
            String attrName = getServletContextAttributeName();
            getServletContext().setAttribute(attrName, wac);
        }
        ...
    }

    /**
     * Return the ServletContext attribute name for this servlet's WebApplicationContext.
     * <p>The default implementation returns
     * {@code SERVLET_CONTEXT_PREFIX + servlet name}.
     * @see #SERVLET_CONTEXT_PREFIX
     * @see #getServletName
     */
    public String getServletContextAttributeName() {
        return SERVLET_CONTEXT_PREFIX + getServletName();
    }
}
```

这样每个servlet就持有自己的上下文，即拥有自己独立的bean空间，同时各个servlet共享相同的bean，即根上下文(第2步中初始化的上下文)定义的那些bean。


