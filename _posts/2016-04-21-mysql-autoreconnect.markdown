---
layout: post
title:  "不要使用MySQL autoReconnect"
date:   2016-04-21 15:11:45
published: true
---

很多使用MySQL的示范代码中都会出现autoReconnect=true这个参数, 但是实际上这个参数已经被deprecated. 以下是mysql的官方文档

>>autoReconnect: Should the driver try to re-establish stale and/or dead connections? If enabled the driver will throw an exception for a queries issued on a stale or dead connection, which belong to the current transaction, but will attempt reconnect before the next query issued on the connection in a new transaction. The use of this feature is not recommended, because it has side effects related to session state and data consistency when applications don't handle SQLExceptions properly, and is only designed to be used when you are unable to configure your application to handle SQLExceptions resulting from dead and stale connections properly. Alternatively, as a last option, investigate setting the MySQL server variable "wait_timeout" to a high value, rather than the default of 8 hours.

因为这是一个1.1版本的JDBC connector就支持的参数, 很多人使用这个参数来解决MySQL默认8小时断开idle连接的问题, 但是实际上这个参数会导致很多问题. 比如在使用ReplicationDriver的时候, 如果一个服务器crash之后, 会发生很微妙的问题.

如果我们设定autoReconnect=true, maxReconnects=3, initialTimeout=5, 有三个slave服务器, 当其中一个服务器crash的时候, 尽管我们还有两个正常的slave, 但是响应时间仍然会非常抖动. 我们来看看原因.


ReplicationDriver新增数据库连接到连接池的时候, 会调用com.mysql.jdbc.ConnectionImpl的构造函数来创建数据库连接, 具体代码在connectWithRetries中:

{% highlight java linenos %}
private void connectWithRetries(boolean isForReconnect, 
    Properties mergedProps) throws SQLException {
    double timeout = getInitialTimeout();
    boolean connectionGood = false;

    Exception connectionException = null;

    for (int attemptCount = 0; (attemptCount < getMaxReconnects()) &&
        !connectionGood; attemptCount++) {
        try {
            if (this.io != null) {
                this.io.forceClose();
            }

            coreConnect(mergedProps);
            pingInternal(false, 0);

            boolean oldAutoCommit;
            int oldIsolationLevel;
            boolean oldReadOnly;
            String oldCatalog;

            synchronized (getConnectionMutex()) {
                this.connectionId = this.io.getThreadId();
                this.isClosed = false;

                // save state from old connection
                oldAutoCommit = getAutoCommit();
                oldIsolationLevel = this.isolationLevel;
                oldReadOnly = isReadOnly(false);
                oldCatalog = getCatalog();

                this.io.setStatementInterceptors(this.statementInterceptors);
            }

            initializePropsFromServer();

            if (isForReconnect) {
                // Restore state from old connection
                setAutoCommit(oldAutoCommit);

                if (this.hasIsolationLevels) {
                    setTransactionIsolation(oldIsolationLevel);
                }

                setCatalog(oldCatalog);
                setReadOnly(oldReadOnly);
            }

            connectionGood = true;

            break;
        } catch (Exception EEE) {
            connectionException = EEE;
            connectionGood = false;
        }

        if (connectionGood) {
            break;
        }

        if (attemptCount > 0) {
            try {
                Thread.sleep((long) timeout * 1000);
            } catch (InterruptedException IE) {
                // ignore
            }
        }
    } // end attempts for a single host
    // .... 省略 ....
}

{% endhighlight %}

因为coreConnect的调用会失败, 所以在第54行会抓住异常,因为maxReconnets=3, 接下来会循环一次重试, 失败后, 由于attemptCount &gt; 0接下来会进入第82行, 当前的worker线程会休眠5秒(initiaTimeout=5). 这对于一个高并发的应用来说是无法接受的, 尤其是你命名还有两个正常的slave. 更严重的是, 因为maxReconnects=3, 当前线程还会继续休眠5秒. 我个人认为backoff, 休眠, 重连这些逻辑放在连接池或者应用层比放到MySQL的驱动内去处理要更合适, 这是我觉得autoReconnect这个参数就不应该提供出来的主要原因.

你可以尝试启动三个数据去打印客户端调用的响应时间, 你会看到尽管你有两个正常的slave, 但是响应时间总是在20ms到10秒之间波动. 一旦你禁止autoReconnect, 你会看到响应时间会稳定到20ms, 因为一旦连接到crash的服务器, 驱动会立即失败, 返回到com.mysql.jdbc.RandomBalanceStrategy的时候, 驱动会立即触发连接切换, 去尝试其他slave服务器. (假设我们默认使用Random的BalanceStrategy)

代码如下:

{% highlight java linenos %}
public ConnectionImpl pickConnection(LoadBalancedConnectionProxy proxy, 
    List<String> configuredHosts, Map<String, ConnectionImpl> liveConnections,
    long[] responseTimes, int numRetries) throws SQLException {
    int numHosts = configuredHosts.size();

    SQLException ex = null;

    List<String> whiteList = new ArrayList<String>(numHosts);
    whiteList.addAll(configuredHosts);

    Map<String, Long> blackList = proxy.getGlobalBlacklist();

    whiteList.removeAll(blackList.keySet());

    Map<String, Integer> whiteListMap = this.getArrayIndexMap(whiteList);

    for (int attempts = 0; attempts < numRetries;) {
        int random = (int) Math.floor((Math.random() * whiteList.size()));
        if (whiteList.size() == 0) {
            throw SQLError.createSQLException("No hosts configured", null);
        }

        String hostPortSpec = whiteList.get(random);

        ConnectionImpl conn = liveConnections.get(hostPortSpec);

        if (conn == null) {
            try {
                conn = proxy.createConnectionForHost(hostPortSpec);
            } catch (SQLException sqlEx) {
                ex = sqlEx;

                if (proxy.shouldExceptionTriggerConnectionSwitch(sqlEx)) {

                    Integer whiteListIndex = whiteListMap.get(hostPortSpec);

                    // exclude this host from being picked again
                    if (whiteListIndex != null) {
                        whiteList.remove(whiteListIndex.intValue());
                        whiteListMap = this.getArrayIndexMap(whiteList);
                    }
                    proxy.addToGlobalBlacklist(hostPortSpec);

                    if (whiteList.size() == 0) {
                        attempts++;
                        try {
                            Thread.sleep(250);
                        } catch (InterruptedException e) {
                        }

                        // start fresh
                        whiteListMap = new HashMap<String, Integer>(numHosts);
                        whiteList.addAll(configuredHosts);
                        blackList = proxy.getGlobalBlacklist();

                        whiteList.removeAll(blackList.keySet());
                        whiteListMap = this.getArrayIndexMap(whiteList);
                    }
                    continue;
                }
                throw sqlEx;
            }
        }
        return conn;
    }
    if (ex != null) {
        throw ex;
    }
    return null; // we won't get here, compiler can't tell
}

{% endhighlight %}

第33行调用shouldExceptionTriggerConnectionSwitch的时候, 驱动会检查这个异常是否是SQLException或者ExceptionChecker认为应该应该换连接的异常, 如果是的, 那么接下来只要第44行发现还有其他的slave可用, 那么就会立即执行到59行的continue然后发生连接切换.

这里你可以定制自己的ExceptionChecker, 针对除了SQLException之外的其他类型的异常也触发连接切换.

autoReconnect还有其他的一些问题, 比如处理断开连接, 此处不再详述. 由于他引入的各种问题, 官方已经废除了这个参数, 如果你看到有些文档中还保留着这个参数, 请自己删掉它.

本文的MySQL Connector是5.1.38版本, 并且为了方便阅读对代码格式稍微做了一点调整, 你所看到的源代码可能会不一样, 也不排除将来MySQL驱动行为发生变化.





