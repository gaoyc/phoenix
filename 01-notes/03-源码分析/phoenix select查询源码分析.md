
phoenix select 查询源码分析
=

如下执行一个Sql的查询语句
```java
ResultSet rst = conn.createStatement().executeQuery("select * from  test.person");
```
创建的statement对象为 PhoenixStatement，然后进入到这个对象的方法中进行查询

```java
    @Override
    public ResultSet executeQuery(String sql) throws SQLException {
        if (logger.isDebugEnabled()) {
            logger.debug(LogUtil.addCustomAnnotations("Execute query: " + sql, connection));
        }
        CompilableStatement stmt = parseStatement(sql); // 将输入sql语句格式为select或者upsert statement.
        if (stmt.getOperation().isMutation()) {
            throw new ExecuteQueryNotApplicableException(sql);
        }
        return executeQuery(stmt);
    }
```

然后创建一个stmt对象，该对象为 ExecutableSelectStatement   
然后执行executeQuery 方法，内容如下  
```java
       // 执行compile过程。(识别limit、having、where、order、projector等操作，生成ScanPlan）
        QueryPlan plan = stmt.compilePlan(PhoenixStatement.this, Sequence.ValueOp.VALIDATE_SEQUENCE); // 创建QueryCompiler
        // Send mutations to hbase, so they are visible to subsequent reads.
        // Use original plan for data table so that data and immutable indexes will be sent
        // TODO: for joins, we need to iterate through all tables, but we need the original table,
        // not the projected table, so plan.getContext().getResolver().getTables() won't work.
        Iterator<TableRef> tableRefs = plan.getSourceRefs().iterator();
        connection.getMutationState().sendUncommitted(tableRefs);
        plan = connection.getQueryServices().getOptimizer().optimize(PhoenixStatement.this, plan);
         // this will create its own trace internally, so we don't wrap this
         // whole thing in tracing
        ResultIterator resultIterator = plan.iterator();
        if (logger.isDebugEnabled()) {
            String explainPlan = QueryUtil.getExplainPlan(resultIterator);
            logger.debug(LogUtil.addCustomAnnotations("Explain plan: " + explainPlan, connection));
        }
        StatementContext context = plan.getContext();
        context.getOverallQueryMetrics().startQuery();
        PhoenixResultSet rs = newResultSet(resultIterator, plan.getProjector(), context);
        resultSets.add(rs);
        setLastQueryPlan(plan);
        setLastResultSet(rs);
        setLastUpdateCount(NO_UPDATE);
        setLastUpdateOperation(stmt.getOperation());
        // If transactional, this will move the read pointer forward
        if (connection.getAutoCommit()) {
            connection.commit();
        }
        connection.incrementStatementExecutionCounter();
        return rs;
```

在stmt 中执行 compilePlan方法，生成查询计划，如下方法  
在 ExecutableSelectStatement 中  

```java
        public QueryPlan compilePlan(PhoenixStatement stmt, Sequence.ValueOp seqAction) throws SQLException {
            if(!getUdfParseNodes().isEmpty()) {
                stmt.throwIfUnallowedUserDefinedFunctions(getUdfParseNodes());
            }
            SelectStatement select = SubselectRewriter.flatten(this, stmt.getConnection());
            ColumnResolver resolver = FromCompiler.getResolverForQuery(select, stmt.getConnection());
            select = StatementNormalizer.normalize(select, resolver);
            SelectStatement transformedSelect = SubqueryRewriter.transform(select, resolver, stmt.getConnection());
            if (transformedSelect != select) {
                resolver = FromCompiler.getResolverForQuery(transformedSelect, stmt.getConnection());
                select = StatementNormalizer.normalize(transformedSelect, resolver);
            }
            QueryPlan plan = new QueryCompiler(stmt, select, resolver, Collections.<PDatum>emptyList(), stmt.getConnection().getIteratorFactory(), new SequenceManager(stmt), true).compile();
            plan.getContext().getSequenceManager().validateSequences(seqAction);
            return plan;
        }
```

我们看一下 getResolverForQuery 中对sql的解释器，里面会创建 SingleTableColumnResolver 对象
```java
    public static ColumnResolver getResolverForQuery(SelectStatement statement, PhoenixConnection connection)
    		throws SQLException {
    	TableNode fromNode = statement.getFrom();
    	if (fromNode == null)
    	    return EMPTY_TABLE_RESOLVER;
        if (fromNode instanceof NamedTableNode)
            return new SingleTableColumnResolver(connection, (NamedTableNode) fromNode, true, 1, statement.getUdfParseNodes());

        MultiTableColumnResolver visitor = new MultiTableColumnResolver(connection, 1, statement.getUdfParseNodes());
        fromNode.accept(visitor);
        return visitor;
    }
```

SingleTableColumnResolver 对象构造器  
```java
        public SingleTableColumnResolver(PhoenixConnection connection, NamedTableNode tableNode,
                boolean updateCacheImmediately, int tsAddition,
                Map<String, UDFParseNode> udfParseNodes) throws SQLException {
            super(connection, tsAddition, updateCacheImmediately, udfParseNodes);
            alias = tableNode.getAlias();
            TableRef tableRef = createTableRef(tableNode, updateCacheImmediately);
            tableRefs = ImmutableList.of(tableRef);
        }
```

调用到父类当中去
```java
        private BaseColumnResolver(PhoenixConnection connection, int tsAddition, boolean updateCacheImmediately, Map<String, UDFParseNode> udfParseNodes) throws SQLException {
        	this.connection = connection;
            this.client = connection == null ? null : new MetaDataClient(connection); //创建MetaDataClient
            this.tsAddition = tsAddition;
            functionMap = new HashMap<String, PFunction>(1);
            if (udfParseNodes.isEmpty()) {
                functions = Collections.<PFunction> emptyList();
            } else {
                // 这个就是Sql中查询用到的udf函数了
                functions = createFunctionRef(new ArrayList<String>(udfParseNodes.keySet()), updateCacheImmediately);
                // 然后判断一下缓存当中是否有函数对象，有就在本地缓存拿，否则就在client.updateCache(functionNames)进行远程查询
                for (PFunction function : functions) {
                    functionMap.put(function.getFunctionName(), function);
                }
            }
        }
```

可以看到上面有一个 createFunctionRef 方法，这个就是Sql中查询用到的udf函数了  
然后判断一下缓存当中是否有函数对象，有就在本地缓存拿，否则就在client.updateCache(functionNames)进行远程查询  
```java
        private List<PFunction> createFunctionRef(List<String> functionNames, boolean updateCacheImmediately) throws SQLException {
            long timeStamp = QueryConstants.UNSET_TIMESTAMP;
            int numFunctions = functionNames.size();
            List<PFunction> functionsFound = new ArrayList<PFunction>(functionNames.size());
            if (updateCacheImmediately || connection.getAutoCommit()) {
                getFunctionFromCache(functionNames, functionsFound, true);
                if(functionNames.isEmpty()) {
                    return functionsFound;
                }
                MetaDataMutationResult result = client.updateCache(functionNames);
 ```

当本地缓存找不到时，调用 client.updateCache(functionNames)  

```java
    private MetaDataMutationResult updateCache(PName tenantId, List<String> functionNames,
            boolean alwaysHitServer) throws SQLException { // TODO: pass byte[] herez
        long clientTimeStamp = getClientTimeStamp();
        List<PFunction> functions = new ArrayList<PFunction>(functionNames.size());
        List<Long> functionTimeStamps = new ArrayList<Long>(functionNames.size());
        Iterator<String> iterator = functionNames.iterator();
        while (iterator.hasNext()) {
            PFunction function = null;
            try {
                String functionName = iterator.next();
                function =
                        connection.getMetaDataCache().getFunction(
                            new PTableKey(tenantId, functionName));
```

当上面都找不到的情况，就调用如下方法
```java
        result = connection.getQueryServices().getFunctions(tenantId, functionsToFecth, clientTimeStamp);
```

调用到 ConnectionQueryServicesImpl 当中  
```java

```

