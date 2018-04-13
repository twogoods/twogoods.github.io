title: mybatis-spring的事务实现
date: 2018-04-08 18:29:21
categories: java
tags: 事务
---
打算新开一坑，搞个支持动态SQL和对象映射的异步数据访问库。说到动态SQL和对象映射这就是Mybatis的强项，可是它背后的JDBC的标准是典型的同步阻塞IO。那数据库的异步驱动有吗？当然，[postgresql-async](https://github.com/mauricio/postgresql-async)支持PostgreSQL和MySQL，但是它的API偏原始，没有ORM特性。操作数据库事务这个问题总是最烦人的，尽量不要用但你又不能不支持，今天来看看mybatis-spring是如何实现的。<!--more-->

```
@Transactional(value = "transactionManager")
    public void updateUserInfo(User u) {
        //other code ....
        userMapper.updateUserinfo(u);
        bookMapper.updateUserInfo(u);
        //other code ....
        bookApplyMapper.updateUserInfo(u); 
    }
```
上面的代码简单的展示了Spring里声明式事务的使用，可以看到Spring把事务的使用做的及其简单，一些基础的配置配好后，一个`@Transactional`解决问题。那问题来了它怎么实现的？我们知道Spring里事务的实现是AOP应用的一个典型，在没有看源码前我用大白话说一下我对实现这个功能自己的理解。事务的关键在于`connection`的`commit`和`rollback`，在一个事务中多个SQL得共用一个`connection`吧，怎么做呢？诶，动态代理。在上面的方法执行前去获取一个`connection`,这个方法里的所有SQL操作共用这一个。怎么实现共用呢？如果没有事务，方法里的每一个SQL都会经历拿连接、执行、释放等操作；如果有事务就只要获取共用的连接、执行，不需要自己释放；可以看到两种方式下逻辑不同。怎么做呢？关于这两个问题朴素的想法是：

1. 事务时我的连接存在一个地方大家都能拿到，这样就可以共用。
2. 每条SQL执行时得知道自己是不是在一个大的事务中，这样才能实现不同的逻辑，才能确定我是去拿那个共用的还是自己创建一个用完释放掉。

那么关键就是共享，static?考虑一下两个并发请求执行同一事务，static这种全局的就不适用了吧，这种情况创建了两个`connection`，但我不知道该用哪一个。所以实现事务动态代理是一个关键，**线程绑定**是另一个关键。把你的数据库连接和是否是一个大事务这个状态放`ThreadLocal`里，这样可以做贡献还能处理线程间独立。

### 原生Mybatis代码
Mybatis和Spring整合前先看看原生Mybatis的SQL执行流程。

```
String resource = "mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
SqlSession session = sqlSessionFactory.openSession(true);
UserMapper mapper = session.getMapper(UserMapper.class);
mapper.get();
session.close();
```
先得理解SqlSession，它是执行一次SQL的会话，它也封装了事务操作的commit、rollback等操作。`session.getMapper(UserMapper.class)`返回的是代理对象，代理对象由`MapperProxyFactory`创建，`MapperProxy`定义了真正的执行逻辑，最终SQL的执行又回到sqlSession里。sqlSession里有个executor成员变量，往往它是SimpleExecutor类型。

```
 public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
      stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.update(stmt);
    } finally {
      //关闭
      closeStatement(stmt);
    }
  }
  
  private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    Connection connection = getConnection(statementLog);
    stmt = handler.prepare(connection, transaction.getTimeout());
    handler.parameterize(stmt);
    return stmt;
  }
  
  //BaseExecutor 里的方法
  protected Connection getConnection(Log statementLog) throws SQLException {
    //原生代码里这个transactio是JdbcTransaction
    Connection connection = transaction.getConnection();
    if (statementLog.isDebugEnabled()) {
      return ConnectionLogger.newInstance(connection, statementLog, queryStack);
    } else {
      return connection;
    }
  }
  
```

Mybatis基本的代码很容易看懂，这里不多说。

### Mybatis-Spring
Spring整合后的事务处理代码是从Spring里AOP的TransactionInterceptor拦截器开始的

```
  public Object invoke(final MethodInvocation invocation) throws Throwable {
        Class targetClass = invocation.getThis() != null?AopUtils.getTargetClass(invocation.getThis()):null;
        return this.invokeWithinTransaction(invocation.getMethod(), targetClass, new InvocationCallback() {
            public Object proceedWithInvocation() throws Throwable {
                return invocation.proceed();
            }
        });
    }
    
    
    
     protected Object invokeWithinTransaction(Method method, Class<?> targetClass, final TransactionAspectSupport.InvocationCallback invocation) throws Throwable {
       //略去大部分，只放主要代码
       //创建事务
        TransactionAspectSupport.TransactionInfo ex = this.createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
       Object retVal = null;
       try {
           retVal = invocation.proceedWithInvocation();
       } catch (Throwable var15) {
            //回滚
           this.completeTransactionAfterThrowing(ex, var15);
           throw var15;
       } finally {
           this.cleanupTransactionInfo(ex);
       }
        //提交
       this.commitTransactionAfterReturning(ex);
       return retVal;

    }
    
```
到这里我们看到了代理的一个基本情况，我们进入到`createTransactionIfNecessary`方法里最终来到`AbstractPlatformTransactionManager`的`getTransaction`方法

```
public final TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException {
 boolean err = this.getTransactionSynchronization() != 2;
 DefaultTransactionStatus status = this.newTransactionStatus((TransactionDefinition)definition, transaction, true, err, debugEnabled, newSynchronization);
 this.doBegin(transaction, (TransactionDefinition)definition);
 this.prepareSynchronization(status, (TransactionDefinition)definition);
 return status;
            }
```
`this.doBegin`会来到`DataSourceTransactionManager`

```
    protected void doBegin(Object transaction, TransactionDefinition definition) {
        DataSourceTransactionManager.DataSourceTransactionObject txObject = (DataSourceTransactionManager.DataSourceTransactionObject)transaction;
        Connection con = null;

        try {
            if(txObject.getConnectionHolder() == null || txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
            //创建连接
                Connection ex = this.dataSource.getConnection();
                if(this.logger.isDebugEnabled()) {
                    this.logger.debug("Acquired Connection [" + ex + "] for JDBC transaction");
                }

                txObject.setConnectionHolder(new ConnectionHolder(ex), true);
            }

            txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
            con = txObject.getConnectionHolder().getConnection();
            Integer ex1 = DataSourceUtils.prepareConnectionForTransaction(con, definition);
            txObject.setPreviousIsolationLevel(ex1);
            if(con.getAutoCommit()) {
                txObject.setMustRestoreAutoCommit(true);
                if(this.logger.isDebugEnabled()) {
                    this.logger.debug("Switching JDBC Connection [" + con + "] to manual commit");
                }

                con.setAutoCommit(false);
            }

            txObject.getConnectionHolder().setTransactionActive(true);
            int timeout = this.determineTimeout(definition);
            if(timeout != -1) {
                txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
            }

            if(txObject.isNewConnectionHolder()) {
            //连接绑定到ThreadLocal   
             TransactionSynchronizationManager.bindResource(this.getDataSource(), txObject.getConnectionHolder());
            }

        } catch (Throwable var7) {
            if(txObject.isNewConnectionHolder()) {
                DataSourceUtils.releaseConnection(con, this.dataSource);
                txObject.setConnectionHolder((ConnectionHolder)null, false);
            }

            throw new CannotCreateTransactionException("Could not open JDBC Connection for transaction", var7);
        }
    }
```

`TransactionSynchronizationManager`类很关键,线程本地变量存的连接和事务状态都在这里

```
public abstract class TransactionSynchronizationManager {
    private static final Log logger = LogFactory.getLog(TransactionSynchronizationManager.class);
    private static final ThreadLocal<Map<Object, Object>> resources = new NamedThreadLocal("Transactional resources");
    private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations = new NamedThreadLocal("Transaction synchronizations");
    private static final ThreadLocal<String> currentTransactionName = new NamedThreadLocal("Current transaction name");
    private static final ThreadLocal<Boolean> currentTransactionReadOnly = new NamedThreadLocal("Current transaction read-only status");
    private static final ThreadLocal<Integer> currentTransactionIsolationLevel = new NamedThreadLocal("Current transaction isolation level");
    private static final ThreadLocal<Boolean> actualTransactionActive = new NamedThreadLocal("Actual transaction active");
    //略.....
}
```
以上还都是Spring对于事务的处理，那么来看看Mybatis都做了什么。原生mybatis的sqlsession是defaultsqlsession类型；spring整合后sqlsession是sqlsessiontemplate类型

```
public class SqlSessionTemplate implements SqlSession, DisposableBean {
  private final SqlSessionFactory sqlSessionFactory;
  private final ExecutorType executorType;
  private final SqlSession sqlSessionProxy;
  private final PersistenceExceptionTranslator exceptionTranslator;
  
    public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType,
      PersistenceExceptionTranslator exceptionTranslator) {

    notNull(sqlSessionFactory, "Property 'sqlSessionFactory' is required");
    notNull(executorType, "Property 'executorType' is required");

    this.sqlSessionFactory = sqlSessionFactory;
    this.executorType = executorType;
    this.exceptionTranslator = exceptionTranslator;
    //mybatis都是通过这个代理对象去执行SQL
    this.sqlSessionProxy = (SqlSession) newProxyInstance(
        SqlSessionFactory.class.getClassLoader(),
        new Class[] { SqlSession.class },
        new SqlSessionInterceptor());
  }
  
  private class SqlSessionInterceptor implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      //这里的SqlSession就是Defaultsqlsession类型，用于执行真正的SQL
      //sqlSession里的Executor对象内的Transaction变成了SpringManagedTransaction
      //这个sqlSession会去拿数据库连接并执行SQl,Transaction的实现整合了spring,使得数据库连接拿到那个存在线程本地变量里的那个
      SqlSession sqlSession = getSqlSession(
          SqlSessionTemplate.this.sqlSessionFactory,
          SqlSessionTemplate.this.executorType,
          SqlSessionTemplate.this.exceptionTranslator);
      try {
        Object result = method.invoke(sqlSession, args);
        if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
          // 不是在大事务中自己提交，关闭就ok了，否则交给spring管理
          sqlSession.commit(true);
     x   }
        return result;
      } catch (Throwable t) {
        Throwable unwrapped = unwrapThrowable(t);
        if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
          // release the connection to avoid a deadlock if the translator is no loaded. See issue #22
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
          sqlSession = null;
          Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException) unwrapped);
          if (translated != null) {
            unwrapped = translated;
          }
        }
        throw unwrapped;
      } finally {
        if (sqlSession != null) {
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        }
      }
    }
  }
  }
```
看看`SpringManagedTransaction`

```
  private void openConnection() throws SQLException {
    this.connection = DataSourceUtils.getConnection(this.dataSource);
    this.autoCommit = this.connection.getAutoCommit();
    //DataSourceUtils是spring 提供的类，在TransactionSynchronizationManager里得到那个数据库连接
    this.isConnectionTransactional = DataSourceUtils.isConnectionTransactional(this.connection, this.dataSource);
     }
```
到这里整合实现了共享一个connection，可以看到mybatis整合了Spring自后，sqlsession里的Transaction改变了使得支持spring事务。

---
## mapper注入
spring 扫描的Mapper 就是接口类型的Bean,mybatis 会把这个bean替换成一个factoryBean(工厂bean),getObject()方法得到的实例就是这个Bean的实例，当然这个实例一定是代理对象

```
private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    GenericBeanDefinition definition;
    for (BeanDefinitionHolder holder : beanDefinitions) {
      definition = (GenericBeanDefinition) holder.getBeanDefinition();
      String beanClassName = definition.getBeanClassName();
      LOGGER.debug(() -> "Creating MapperFactoryBean with name '" + holder.getBeanName()
          + "' and '" + beanClassName + "' mapperInterface");

     //传入原本的接口定义的那个Mapper,mapperFactoryBean这个类的构造方法就是传入一个class
     definition.getConstructorArgumentValues().addGenericArgumentValue(beanClassName); 
      //然后把bean的类型转成mapperFactoryBean，它的getObject()会产生代理对象
      definition.setBeanClass(this.mapperFactoryBean.getClass());

      definition.getPropertyValues().add("addToConfig", this.addToConfig);

     //后面代码略
    }
  }
``` 



