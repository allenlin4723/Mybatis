[toc]
# 基于Spring+MyBatis实现动态数据切换
## 业务背景

目前公司A和公司B运行着两套一样的系统，只是做了物理上的数据隔离。现在我们一般是业务功能在公司A开发好后，再拷贝代码到公司B。所以为了减少维护成本，在现有的新系统上，针对销售系统，仓库系统，产品系统做动态数据切换，按照一定的系统标识，在特定的切面上，进行切换。

## 实现方案

我们的系统使用的是Spring+mybatis来做的orm，在spring-jdbc包里面有一个类`AbstractRoutingDataSource`,它实现了通过lookupKey来动态切换数据源的功能。

### 1. `afterPropertiesSet()`

```
	@Override
	public void afterPropertiesSet() {
		if (this.targetDataSources == null) {
			throw new IllegalArgumentException("Property 'targetDataSources' is required");
		}
		this.resolvedDataSources = new HashMap<>(this.targetDataSources.size());
		this.targetDataSources.forEach((key, value) -> {
			Object lookupKey = resolveSpecifiedLookupKey(key);
			DataSource dataSource = resolveSpecifiedDataSource(value);
			this.resolvedDataSources.put(lookupKey, dataSource);
		});
		if (this.defaultTargetDataSource != null) {
			this.resolvedDefaultDataSource = resolveSpecifiedDataSource(this.defaultTargetDataSource);
		}
	}
```
维护了一个 `targetDataSources` 和 `defaultTargetDataSource`，初始化时将数据源分别进行复制到resolvedDataSources和resolvedDefaultDataSource中，但是在配置的时候，我们不去初始化配置一个默认数据源，因为为了避免某些数据请求没有带上系统标识，而导致查询或者更改了错误的数据源

### 2. `getConnection()`
调用此方法获取连接的时候，如下代码determineTargetDataSource().getConnection()，先调用determineTargetDataSource()方法返回当前的DataSource，然后再调用getConnection()。

```
	@Override
	public Connection getConnection() throws SQLException {
		return determineTargetDataSource().getConnection();
	}
```

### 3. `determineTargetDataSource()`
此方法的就是根据lookupkey获取map中的dataSource，而lookupkey是从determineCurrentLookupKey方法返回的，如下：

```
	protected DataSource determineTargetDataSource() {
		Assert.notNull(this.resolvedDataSources, "DataSource router not initialized");
		Object lookupKey = determineCurrentLookupKey();
		DataSource dataSource = this.resolvedDataSources.get(lookupKey);
		if (dataSource == null && (this.lenientFallback || lookupKey == null)) {
			dataSource = this.resolvedDefaultDataSource;
		}
		if (dataSource == null) {
			throw new IllegalStateException("Cannot determine target DataSource for lookup key [" + lookupKey + "]");
		}
		return dataSource;
	}
```
### 4.`determineCurrentLookupKey()`

此方法要我们自己实现，是切换数据源的方法，通过自己的实现返回lookupKey，根据lookupKey获取对应数据源达到切换动态切换的功能。


```
	@Nullable
	protected abstract Object determineCurrentLookupKey();
```

### 5.自定义`DynamicDataSource`

自定义`DynamicDataSource`继承 `AbstractRoutingDataSource`，由上得知，我们先要有个方法能设置 `targetDataSources`，然后要重写`determineCurrentLookupKey`方法，来实现动态切换，代码如下：

```
public class DynamicDataSource extends AbstractRoutingDataSource {
    /**
     * 如果不希望数据源在启动配置时就加载好，可以定制这个方法，从任何你希望的地方读取并返回数据源
     * 比如从数据库、文件、外部接口等读取数据源信息，并最终返回一个DataSource实现类对象即可
     */
    @Override
    protected DataSource determineTargetDataSource() {
        return super.determineTargetDataSource();
    }
    /**
     * 如果希望所有数据源在启动配置时就加载好，这里通过设置数据源Key值来切换数据，定制这个方法
     */
    @Override
    protected Object determineCurrentLookupKey() {
        return DynamicDataSourceContextHolder.getDataSourceKey();
    }
    /**
     * 设置数据源
     * @param dataSources
     */
    public void setDataSources(Map<Object, Object> dataSources) {
        super.setTargetDataSources(dataSources);
        // 将数据源的 key 放到数据源上下文的 key 集合中，用于切换时判断数据源是否有效
        DynamicDataSourceContextHolder.addDataSourceKeys(dataSources.keySet());
    }
}
```
### 6.`DynamicDataSourceContextHolder`

为了线程安全，我们要把`lookupKey`放入`ThreadLocal`里面，因此我们写了一个`DynamicDataSourceContextHolder`来切换数据源，就是改变当前线程保存的`lookupKey`，上面`DynamicDataSource.determineCurrentLookupKey`从当前线程取出即可：

```
public class DynamicDataSourceContextHolder {
    private static final ThreadLocal<String> contextHolder = new ThreadLocal<String>();
    /**
     * 数据源的 key集合，用于切换时判断数据源是否存在
     */
    public static List<Object> dataSourceKeys= new ArrayList<>();
    /**
     * 切换数据源
     * @param dataSoureKey
     */
    public static void setDataSourceKey(String dataSoureKey) {
        if(StringUtil.isEmpty(dataSoureKey)){
            throw new BusinessException("dataSoureKey is null");
        }
        contextHolder.set(dataSoureKey.toUpperCase());
    }
    /**
     * 获取数据源
     * @return
     */
    public static String getDataSourceKey() {
        return contextHolder.get();
    }
    /**
     * 重置数据源
     */
    public static void clearDataSourceKey() {
        contextHolder.remove();
    }
    /**
     * 判断是否包含数据源
     * @param key 数据源key
     * @return
     */
    public static boolean containDataSourceKey(String key) {
        if(StringUtil.isEmpty(key)){
            throw new BusinessException("sysUri is null");
        }
        return dataSourceKeys.contains(key.toUpperCase());
    }
    /**
     * 添加数据源
     * @return
     */
    public static boolean addDataSourceKeys(Collection<? extends Object> keys) {
        return dataSourceKeys.addAll(keys);
    }
}
```
### 7.配置动态数据源生效

在framework-common-dynamicdatasource的下面`DynamicDataMybatisPrimaryConfig`类，进行初始化dynamicDataSource

```
    @Primary
    @Bean(name = "dynamicDataSource")
    public DataSource dynamicDataSource() {
        DynamicDataSource dynamicDataSource = new DynamicDataSource();
        log.info("===>>>开始初始化主数据源:[dynamicDataSource]");
        if(jdbcAccountProperties.getAccountList().size()>0){
            Map<Object, Object> dataSourceMap = new HashMap<>();
            for(JdbcAccount jdbcAccount:jdbcAccountProperties.getAccountList()){
                DruidDataSource druidDataSource = new DruidDataSource();
                druidDataSource.setDriverClassName(jdbcAccount.getDriverClassName());
                druidDataSource.setUrl(jdbcAccount.getUrl());
                druidDataSource.setUsername(jdbcAccount.getDbUserName());
                druidDataSource.setPassword(jdbcAccount.getPassword());
                druidDataSource.setInitialSize(dynamicDataSourceJdbcPrimaryProperties.getInitialSize());
                druidDataSource.setMinIdle(dynamicDataSourceJdbcPrimaryProperties.getMinIdle());
                druidDataSource.setMaxActive(dynamicDataSourceJdbcPrimaryProperties.getMaxActive());
                druidDataSource.setMaxWait(dynamicDataSourceJdbcPrimaryProperties.getMaxWait());
                druidDataSource.setTimeBetweenEvictionRunsMillis(dynamicDataSourceJdbcPrimaryProperties.getTimeBetweenEvictionRunsMillis());
                druidDataSource.setMinEvictableIdleTimeMillis(dynamicDataSourceJdbcPrimaryProperties.getMinEvictableIdleTimeMillis());
                druidDataSource.setValidationQuery(dynamicDataSourceJdbcPrimaryProperties.getValidationQuery());
                druidDataSource.setTestWhileIdle(dynamicDataSourceJdbcPrimaryProperties.isTestWhileIdle());
                druidDataSource.setTestOnBorrow(dynamicDataSourceJdbcPrimaryProperties.isTestOnBorrow());
                druidDataSource.setTestOnReturn(dynamicDataSourceJdbcPrimaryProperties.isTestOnReturn());
                druidDataSource.setPoolPreparedStatements(dynamicDataSourceJdbcPrimaryProperties.isPoolPreparedStatements());
                druidDataSource.setMaxPoolPreparedStatementPerConnectionSize(dynamicDataSourceJdbcPrimaryProperties.getMaxPoolPreparedStatementPerConnectionSize());
                druidDataSource.setRemoveAbandoned(dynamicDataSourceJdbcPrimaryProperties.isRemoveAbandoned());
                druidDataSource.setRemoveAbandonedTimeout(dynamicDataSourceJdbcPrimaryProperties.getRemoveAbandonedTimeout());
                druidDataSource.setLogAbandoned(dynamicDataSourceJdbcPrimaryProperties.isLogAbandoned());
                try {
                    druidDataSource.setFilters(dynamicDataSourceJdbcPrimaryProperties.getFilters());
                } catch (SQLException throwables) {
                    throwables.printStackTrace();
                }
                dataSourceMap.put(jdbcAccount.getSysUri().toUpperCase(),druidDataSource);
            }
            dynamicDataSource.setDataSources(dataSourceMap);
        }
        /**
         * 必须执行此操作，才会重新初始化AbstractRoutingDataSource 中的 resolvedDataSources，也只有这样，动态切换才会起效
         */
        dynamicDataSource.afterPropertiesSet();
        return dynamicDataSource;
    }
```
### 8.`CompanyHeaderInterceptor`
请求服务的Controller时，会通过`CompanyHeaderInterceptor`拦截器拦截所有的外部请求，然后判断Header是否有sysUri的信息，如果没有则直接提示`所属公司标识sysUri不能为空`

```
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        if (handler instanceof HandlerMethod) {
            String header = request.getHeader(CompanyHolderUtils.HEADER_NAME);
            if (StringUtil.isEmpty(header)) {
                throw new BusinessException("所属公司标识sysUri不能为空!");
            }
            log.info("请求所属公司前缀：" + header);
            CompanyContext.set(header.toUpperCase());
        }
        return super.preHandle(request, response, handler);
    }
```
所以如果某个服务支持动态数据切换的话，则所有的请求都需要通过Controller作为入口，并且必须指定当前Request的sysUri。

举个例子，如果在sales-service利用spring task来做本地的定时任务调度，那么在执行数据库操作时，将会报错"Cannot determine target DataSource for lookup key[null]"

除非可以通过某种数据维度，比如sellerId，来区分当前操作要访问的是toplus还是slcx，然后手动调用` CompanyContext.set(sysUri);`

我们在写JUnit的时候，也可以按照上面的方式来解决

不过最好的实现还是，通过我们的任务调度系统xxl-job来实现，在执行任务的时候，在request header加入sysUri。针对同一个类型任务，比如session数据解析，我们可以分别为Toplus和slcx创建两个任务。目前我们的任务调度也是走的网关实现，最终会请求到Controller。

在执行完每一个request的时候，需要清空`CompanyContext`和`DynamicDataSourceContextHolder`。因为我们是把sysUri存储在ThreadLocal中的，所有如果不进行清空操作，很容易引起内存溢出。
```
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable Exception ex) throws Exception {
        //清除ThreadLocal避免内存溢出
        CompanyContext.clear();
        DynamicDataSourceContextHolder.clearDataSourceKey();
        super.afterCompletion(request, response, handler, ex);
    }
```

### 9.`DynamicDataSourceAspect`

使用面向切面编程，在service层自动切换数据源，在调用service方法之前，先通过CompayContext获取当前线程的sysUri，即对应数据源的lookupKey。

```
@Slf4j
@Aspect
@Component
@Order(1) // 请注意：这里order一定要小于tx:annotation-driven的order，即先执行DynamicDataSourceAspectAdvice切面，再执行事务切面，才能获取到最终的数据源
@EnableAspectJAutoProxy(proxyTargetClass = true)
public class DynamicDataSourceAspect {
    @Around("execution(* com.framework..*.service.impl.*.*(..)) ")
    public Object doAround(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        String tenantId = CompanyContext.get();
        log.info("当前租户Id:{}", tenantId);
        if(StringUtil.isEmpty(tenantId)){
            throw new BusinessException("当前系统标识sysUri为空!");
        }
        DynamicDataSourceContextHolder.setDataSourceKey(tenantId.toUpperCase());
        Object result = proceedingJoinPoint.proceed();
        return result;
    }
}
```
### 10.如何使用framework-common-dynamicdatasource

在服务的provider module的pom.xml引入依赖：

```
        <dependency>
            <groupId>com.framework.parent</groupId>
            <artifactId>framework-common-dynamicdatasouce</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </dependency>
```
然后早`provider`的`config`目录下，找到对应的服务的config文件，比如在`sales-service`服务的provider下面，有`SalesServiceConfig`类，在这里引入相关的动态数据源配置。对现有代码实现了零侵入


```
@Setter
@Getter
@Configuration
@EnableConfigurationProperties(DynamicDataSourceJdbcPrimaryProperties.class)
@Import({KafkaProducerConfig.class,DynamicDataSourceMybatisMainConfig.class,DynamicDataMybatisPrimaryConfig.class, DynamicDataSourceMyBatisPrimaryMapperScannerConfig.class})
public class SalesServiceConfig {
    
}
```
数据库配置如下：

```
datasource:
  primary:
    initialSize:  1
    minIdle:  1
    maxActive:  20
    maxWait:  30000
    timeBetweenEvictionRunsMillis:  60000
    minEvictableIdleTimeMillis:  300000
    validationQuery:  select 1 from dual
    testWhileIdle:  true
    testOnBorrow:  false
    testOnReturn:  false
    removeAbandoned: true
    removeAbandonedTimeout: 900
    poolPreparedStatements: true
    maxPoolPreparedStatementPerConnectionSize: 1
    logAbandoned: false
    filters: stat,log4j
jdbc:
  account:
    accountList:
      - sysUri: toplus
        driverClassName: com.mysql.jdbc.Driver
        dbUserName: root
        password: 123456
        url: jdbc:mysql://127.0.0.1:3306/amazon_sale?useUnicode=true&characterEncoding=utf8
      - sysUri: slcx
        driverClassName: com.mysql.jdbc.Driver
        dbUserName: root
        password: 123456
        url: jdbc:mysql://127.0.0.2:3306/amazon_sale?useUnicode=true&characterEncoding=utf8
```
如果后续需要在新增一个公司，那么直接在加一个对应数据库配置即可，这样可以很快的实现系统克隆功能。

### 执行流程图

[执行流程图](https://github.com/allenlin4723/pictures/blob/main/%E5%8A%A8%E6%80%81%E6%95%B0%E6%8D%AE%E6%BA%90%E5%88%87%E6%8D%A2%E6%B5%81%E7%A8%8B%E5%9B%BE.png)
