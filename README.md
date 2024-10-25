# Readme
A note about Network-Based Locking System.

### Network-Based Locking System (NLS)

可以利用database来实现NLS，这种方式not very graceful，but very efficient。

##### Mutex

Database有一种特性，即创建unique记录的操作是互斥的，可以利用这种特性来实现Mutex。

基于Apache Cassandra实现互斥锁的代码如下：
```python
```

基于Redis实现互斥锁的代码如下：
```python
r.set('foo_locks/123', '1729837899653,wsEy4', nx=True)  # acquire mutex
r.eval('if redis.call("GET", KEYS[1]) == ARGV[1] then return redis.call("DEL", KEYS[1]) else return 0 end', 1, 'foo_locks/123', '1729837899653,wsEy4')  # release mutex
```

基于Oracle实现Mutex的代码如下：
```python
cursor.execute('CREATE TABLE foo_locks (PRIMARY KEY (lock_id), lock_id INTEGER, acquired_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP, token CHAR(5) NOT NULL);')  # prepare schema and table for mutex

cursor.execute('INSERT INTO foo_locks (lock_id, token) VALUES (:lock_id, :token);', [123, 'WseAI'])  # acquire mutex
cursor.execute('DELETE FROM foo_locks WHERE lock_id=:lock_id AND token=:token', [123, 'WseAI'])  # release mutex
```

注意事项：
- `acquired_at`的用途是查找异常未释放的Mutex。
- 使用随机生成的`token`的目的是防止release了别人acquired的Mutex。
- 不要给锁设置超时，因为NLS没有有效的手段阻止已经超时的应用程序继续访问对应的共享资源。如果是因为网络故障而导致锁未被及时释放，应该解决网络故障。如果是因为应用程序崩溃而导致的锁未被正常释放，应该修复应用程序。

### Credits
- Computer Systems: A Programmer's Perspective, Third Edition
- [Readers–writers problem - Wikipedia](https://en.wikipedia.org/wiki/Readers-writers_problem)
- [Readers–writer lock - Wikipedia](https://en.wikipedia.org/wiki/Readers–writer_lock)
- [PRIMARY KEY - Apache Cassandra](https://cassandra.apache.org/doc/latest/cassandra/developing/cql/ddl.html#primary-key) and [datastax/python-driver - GitHub](https://github.com/datastax/python-driver)
- [UNIQUE Constraint - Oracle](https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/constraint.html) and [oracle/python-oracledb - GitHub](https://github.com/oracle/python-oracledb/)
- [SET NX - Redis](https://redis.io/docs/latest/commands/set/) and [redis/redis-py - GitHub](https://github.com/redis/redis-py)
