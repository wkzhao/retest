---
title: springboot+mybatis数据源动态切换与加载
date: 2018-09-13 23:04:24
comments: true
toc: true
categories: "微服务"
tags:
   - springboot
   - mybatis
---
&emsp;&emsp;对于动态切换数据源，需要一个类继承AbstractRoutingDataSource,继承该抽象类的时候，必须实现一个抽象方法：protected abstract Object determineCurrentLookupKey()，该方法用于指定到底需要使用哪一个数据源。

<!--more-->
````
public abstract class AbstractRoutingDataSource extends AbstractDataSource implements InitializingBean {
    private Map<Object, Object> targetDataSources;
    private Object defaultTargetDataSource;
    private boolean lenientFallback = true;
    private DataSourceLookup dataSourceLookup = new JndiDataSourceLookup();
    private Map<Object, DataSource> resolvedDataSources;
    private DataSource resolvedDefaultDataSource;

    public AbstractRoutingDataSource() {
    }

    public void setTargetDataSources(Map<Object, Object> targetDataSources) {
        this.targetDataSources = targetDataSources;
    }

    public void setDefaultTargetDataSource(Object defaultTargetDataSource) {
        this.defaultTargetDataSource = defaultTargetDataSource;
    }

    public void setLenientFallback(boolean lenientFallback) {
        this.lenientFallback = lenientFallback;
    }

    public void setDataSourceLookup(DataSourceLookup dataSourceLookup) {
        this.dataSourceLookup = (DataSourceLookup)(dataSourceLookup != null ? dataSourceLookup : new JndiDataSourceLookup());
    }

    public void afterPropertiesSet() {
        if (this.targetDataSources == null) {
            throw new IllegalArgumentException("Property 'targetDataSources' is required");
        } else {
            this.resolvedDataSources = new HashMap(this.targetDataSources.size());
            Iterator var1 = this.targetDataSources.entrySet().iterator();

            while(var1.hasNext()) {
                Entry<Object, Object> entry = (Entry)var1.next();
                Object lookupKey = this.resolveSpecifiedLookupKey(entry.getKey());
                DataSource dataSource = this.resolveSpecifiedDataSource(entry.getValue());
                this.resolvedDataSources.put(lookupKey, dataSource);
            }

            if (this.defaultTargetDataSource != null) {
                this.resolvedDefaultDataSource = this.resolveSpecifiedDataSource(this.defaultTargetDataSource);
            }

        }
    }

    protected Object resolveSpecifiedLookupKey(Object lookupKey) {
        return lookupKey;
    }

    protected DataSource resolveSpecifiedDataSource(Object dataSource) throws IllegalArgumentException {
        if (dataSource instanceof DataSource) {
            return (DataSource)dataSource;
        } else if (dataSource instanceof String) {
            return this.dataSourceLookup.getDataSource((String)dataSource);
        } else {
            throw new IllegalArgumentException("Illegal data source value - only [javax.sql.DataSource] and String supported: " + dataSource);
        }
    }

    public Connection getConnection() throws SQLException {
        return this.determineTargetDataSource().getConnection();
    }

    public Connection getConnection(String username, String password) throws SQLException {
        return this.determineTargetDataSource().getConnection(username, password);
    }

    public <T> T unwrap(Class<T> iface) throws SQLException {
        return iface.isInstance(this) ? this : this.determineTargetDataSource().unwrap(iface);
    }

    public boolean isWrapperFor(Class<?> iface) throws SQLException {
        return iface.isInstance(this) || this.determineTargetDataSource().isWrapperFor(iface);
    }

    protected DataSource determineTargetDataSource() {
        Assert.notNull(this.resolvedDataSources, "DataSource router not initialized");
        Object lookupKey = this.determineCurrentLookupKey();
        DataSource dataSource = (DataSource)this.resolvedDataSources.get(lookupKey);
        if (dataSource == null && (this.lenientFallback || lookupKey == null)) {
            dataSource = this.resolvedDefaultDataSource;
        }

        if (dataSource == null) {
            throw new IllegalStateException("Cannot determine target DataSource for lookup key [" + lookupKey + "]");
        } else {
            return dataSource;
        }
    }

    protected abstract Object determineCurrentLookupKey();
}
````
&emsp;&emsp;自定义动态数据源类
````
import com.example.savesearchservice.util.DataSourceUtil;
import com.example.savesearchservice.util.DynamicDatasourceHolder;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.springframework.boot.jdbc.DataSourceBuilder;
import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

import javax.sql.DataSource;

public class DynamicDataSource extends AbstractRoutingDataSource {

    private static final Log logger = LogFactory.getLog(DynamicDataSource.class);

    private static Map<Object, Object> datasourceMap = new ConcurrentHashMap<>();

    @Override
    protected Object determineCurrentLookupKey() {
        logger.info("current dataSourceId:" + DynamicDatasourceHolder.get());
        return DynamicDatasourceHolder.get();
    }

    public DataSource createDatasource(String id, String url, String username, String password, String driverClassName) {
        DataSource dataSource = DataSourceBuilder.create()
                .url(url)
                .username(username)
                .password(password)
                .driverClassName(driverClassName)
                .build();
        if (dataSource != null) {
            DataSourceUtil.put(id, id);
            datasourceMap.put(id, dataSource);

            //调用父类方法赋值Map<Object, DataSource> resolvedDataSources
            super.setTargetDataSources(datasourceMap);
            super.afterPropertiesSet();
            return dataSource;
        }
        return null;
    }
}

````
&emsp;&emsp;通过ThreadLocal维护一个全局唯一的map来实现数据源的动态切换
````
public class DynamicDatasourceHolder {
    private static final ThreadLocal<String> DATASOURCE_HOLDER = new ThreadLocal<>();


    public static void add(String datasource) {
        DATASOURCE_HOLDER.set(datasource);
    }

    public static String get() {
        return DATASOURCE_HOLDER.get();
    }

    public static void clear() {
        DATASOURCE_HOLDER.remove();
    }
}
````
&emsp;&emsp;通过AOP切面实现动态切换数据源，这里假设projectId与dataSourceId有对应关系
````
import com.example.savesearchservice.annotation.DataSource;
import com.example.savesearchservice.datasource.DynamicDataSource;
import com.example.savesearchservice.util.DataSourceUtil;
import com.example.savesearchservice.util.DynamicDatasourceHolder;

import org.apache.commons.lang.StringUtils;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.After;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.beans.BeansException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

@Component
@Aspect
@Order(-1)
public class DynamicDataSourceAOP implements ApplicationContextAware {

    private static final Log logger = LogFactory.getLog(DynamicDataSourceAOP.class);

    private ApplicationContext applicationContext;

    @Pointcut("execution(* com.example.controller*(..))")
    public void pointCut() {
    }

    /**
     * 执行方法前更换数据源
     *
     * @param joinPoint 切点
     */
    @Before("@annotation(dataSource)")
    public void doBefore(JoinPoint joinPoint, DataSource dataSource) {

        logger.info("Enter DataSourceAOP");
        String projectId = DataSourceUtil.DEFAULT;
        Object[] args = joinPoint.getArgs();
        if (args.length >= 1) {
            projectId = String.valueOf(args[0]);
        }
        projectId = StringUtils.defaultIfBlank(projectId, DataSourceUtil.DEFAULT);
        if (!DataSourceUtil.contains(projectId)) {
            DynamicDataSource dynamicDataSource = applicationContext.getBean(DynamicDataSource.class);

            //这里可以根据需要从数据库或者其他地方获取数据源连接信息
            dynamicDataSource.createDatasource(projectId,
                    "jdbc:mysql://localhost:3306/dynamic_datasource?useSSL=true&useUnicode=true&characterEncoding=utf8",
                    "XXX", "XXX", "com.mysql.cj.jdbc.Driver");
            DynamicDatasourceHolder.add(DataSourceUtil.getDataSourceId(projectId));
            logger.info(String.format("change dataSource for %s,dataSourceId: %s", projectId, DataSourceUtil.getDataSourceId(projectId)));
            return;
        }
        DynamicDatasourceHolder.add(DataSourceUtil.getDataSourceId(projectId));

    }

    /**
     * 执行方法后清除数据源设置
     *
     * @param joinPoint 切点
     */
    @After("@annotation(dataSource)")
    public void doAfter(JoinPoint joinPoint, DataSource dataSource) {
        DynamicDatasourceHolder.clear();
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
}
````
&emsp;&emsp;新建DataSourceUtil类保存projectId与dataSourceId的对应关系
````
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class DataSourceUtil {

    public static final String DEFAULT = "default";

    private static final Map<String, String> dataSourceMap = new ConcurrentHashMap<>();

    private DataSourceUtil() {
    }

    public static void put(String projectId, String dataSource) {
        dataSourceMap.put(projectId, dataSource);
    }

    public static boolean contains(String projectId) {
        return dataSourceMap.get(projectId) != null;
    }

    public static String getDataSourceId(String projectId) {
        return dataSourceMap.get(projectId);
    }
}
````
&emsp;&emsp;springboot启动时的配置类配置默认datasource
````

import com.example.savesearchservice.datasource.DynamicDataSource;
import com.example.savesearchservice.util.DataSourceUtil;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.apache.ibatis.session.SqlSessionFactory;
import org.mybatis.spring.SqlSessionFactoryBean;
import org.mybatis.spring.annotation.MapperScan;
import org.mybatis.spring.mapper.MapperScannerConfigurer;
import org.springframework.context.EnvironmentAware;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.env.Environment;
import org.springframework.core.io.ClassPathResource;
import org.springframework.jdbc.datasource.DataSourceTransactionManager;

import javax.sql.DataSource;

import lombok.NonNull;

@Configuration
@MapperScan("com.example.savesearchservice.dao")
public class DataSourceConfiguration implements EnvironmentAware {

    private static Log logger = LogFactory.getLog(DataSourceConfiguration.class);

    private Environment environment;

    @Bean
    DataSource dataSource() {
        logger.info("environment:" + environment);
        DynamicDataSource dynamicDataSource = new DynamicDataSource();
        DataSource dataSource = dynamicDataSource.createDatasource(DataSourceUtil.DEFAULT,
                environment.getProperty("datasource.url"),
                environment.getProperty("datasource.username"),
                environment.getProperty("datasource.password"),
                environment.getProperty("datasource.driver-class-name"));
        dynamicDataSource.setDefaultTargetDataSource(dataSource);
        return dynamicDataSource;
    }

    @Bean
    SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(dataSource);
        sqlSessionFactoryBean.setConfigLocation(new ClassPathResource("mybatis/mybatis-conf.xml"));
        return sqlSessionFactoryBean.getObject();
    }

    @Bean
    MapperScannerConfigurer mapperScannerConfigurer() {
        MapperScannerConfigurer mapperScannerConfigurer = new MapperScannerConfigurer();
        mapperScannerConfigurer.setAnnotationClass(com.example.savesearchservice.annotation.DataSource.class);
        mapperScannerConfigurer.setBasePackage("com.example.savesearchservice.dao");
        mapperScannerConfigurer.setSqlSessionFactoryBeanName("sqlSessionFactory");
        return mapperScannerConfigurer;
    }

    @Bean
    DataSourceTransactionManager dataSourceTransactionManager(DataSource dataSource) {
        DataSourceTransactionManager dataSourceTransactionManager = new DataSourceTransactionManager();
        dataSourceTransactionManager.setDataSource(dataSource);
        return dataSourceTransactionManager;
    }

    @Override
    public void setEnvironment(@NonNull Environment environment) {
        this.environment = environment;
    }
}

````
&emsp;&emsp;可以看到，已经实现了数据源的动态切换
![](https://upload-images.jianshu.io/upload_images/13528017-397a6fa8641f93d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

