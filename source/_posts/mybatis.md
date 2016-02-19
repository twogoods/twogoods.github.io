title: mybatis中的几个问题
date: 2015-12-07 22:25:01
categories: mybatis
tags: 问题小计
---
公司里用mybatis，写着写着会有几个问题冒出来，终于有空去看了一下源代码。
- mybatis 方法签名返回list，LinkedList，如何实现不同的list?
解：单纯使用mybatis时  `List<Object> user = session.selectList(statement);`  只能返回list且实际为arraylist
<!--more-->
`org.apache.ibatis.executor.resultset.DefaultResultSetHandler`里
```
@Override
  public List<Object> handleResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance().activity("handling results").object(mappedStatement.getId());
    final List<Object> multipleResults = new ArrayList<Object>();
    int resultSetCount = 0;
    ResultSetWrapper rsw = getFirstResultSet(stmt);
    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    int resultMapCount = resultMaps.size();
    validateResultMapsCount(rsw, resultMapCount);
    while (rsw != null && resultMapCount > resultSetCount) {
      ResultMap resultMap = resultMaps.get(resultSetCount);
      handleResultSet(rsw, resultMap, multipleResults, null);
      rsw = getNextResultSet(stmt);
      cleanUpAfterHandlingResultSet();
      resultSetCount++;
    }
    String[] resultSets = mappedStatement.getResulSets();
    if (resultSets != null) {
      while (rsw != null && resultSetCount < resultSets.length) {
        ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
        if (parentMapping != null) {
          String nestedResultMapId = parentMapping.getNestedResultMapId();
          ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
          handleResultSet(rsw, resultMap, null, parentMapping);
        }
        rsw = getNextResultSet(stmt);
        cleanUpAfterHandlingResultSet();
        resultSetCount++;
      }
    }
    return collapseSingleResultList(multipleResults);
  }
```

配合spring时，实际的sqlSession是一个代理，在SqlSessionTemplate能看到sqlSessionProxy这个代理类，并且代理类里有一个实现InvocationHandler的SqlSessionInterceptor。spring读取配置初始化`MapperFactoryBean`,方法执行完后对返回类型转换。`org.apache.ibatis.binding.MapperMethod` 里关于方法返回不同类型集合的转换MapperProxy的invoke方法里调mapperMethod.execute()。
```
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if (Object.class.equals(method.getDeclaringClass())) {
      try {
        return method.invoke(this, args);
      } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
      }
    }
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }
```

```

  private <E> Object executeForMany(SqlSession sqlSession, Object[] args) {
    List<E> result;
    Object param = method.convertArgsToSqlCommandParam(args);
    if (method.hasRowBounds()) {
      RowBounds rowBounds = method.extractRowBounds(args);
      result = sqlSession.<E>selectList(command.getName(), param, rowBounds);
    } else {
      result = sqlSession.<E>selectList(command.getName(), param);
    }
    // issue #510 Collections & arrays support
    if (!method.getReturnType().isAssignableFrom(result.getClass())) {
      if (method.getReturnType().isArray()) {
        return convertToArray(result);
      } else {
        return convertToDeclaredCollection(sqlSession.getConfiguration(), result);
      }
    }
    return result;
  }
```
- resulttype为map时，map里的泛型<String,Object>,<String,String>如何转化？
解：配置有 resultType="java.util.Map" xml配置会组装在类 `MappedStatement` mybatis 把结果包装成`Map<Object,Object>`

