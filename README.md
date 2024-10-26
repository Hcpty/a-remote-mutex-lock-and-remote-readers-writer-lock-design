# Readme
A note about Network-Based Locking System (NLS).

### Network-Based Locking System (NLS)

当位于不同机器上的应用程序并发地读写一组共享资源时，需要使用NLS。

##### 基于Database实现Mutex

当访问共享资源的应用程序都是写者的时候，使用Mutex很方便。

Database有一种特性，即创建unique记录的操作是互斥的，可以基于这种特性来实现Mutex。

基于[Apache Cassandra](https://cassandra.apache.org/_/index.html)实现Mutex的原理如下：

```python
# Prepare schema and table for Mutexes:
session.execute(
  """CREATE TABLE mutexes (
    PRIMARY KEY (resource_type, resource_id),
    resource_type VARCHAR,
    resource_id INT,
    ticket VARCHAR,
    acquired_at TIMESTAMP
  );"""
)
```

```python
# Acquire Mutex:
session.execute(
  'INSERT INTO mutexes (resource_type, resource_id, ticket, acquired_at) VALUES (%s, %s, %s, toTimestamp(now())) IF NOT EXISTS;',
  ['foobar', 123, 'fszey']
)
```

```python
# Release Mutex:
session.execute(
  'DELETE FROM mutexes WHERE resource_type=%s AND resource_id=%s AND ticket=%s;',
  ['foobar', 123, 'fszey']
)
```

基于[Redis](https://redis.io/)实现Mutex的原理如下：

```python
# Acquire Mutex:
r.set('mutexes/foobar,123', 'gluww,1729837899653', nx=True)
```

```python
# Release Mutex:
lua_script = \
  """if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
  else
    return 0
  end"""
r.eval(lua_script, 1, 'mutexes/foobar,123', 'gluww,1729837899653')
```

基于[Oracle Database](https://www.oracle.com/database/)或[Oracle In-Memory Database](https://www.oracle.com/database/)实现Mutex的原理如下：

```python
# Prepare schema and table for Mutexes:
cursor.execute(
  """CREATE TABLE mutexes (
    PRIMARY KEY (resource_type, resource_id),
    resource_type CHAR(36) NOT NULL,
    resource_id INTEGER NOT NULL,
    ticket CHAR(9) NOT NULL,
    acquired_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL
  );"""
)
```

```python
# Acquire Mutex:
cursor.execute(
  'INSERT INTO mutexes (resource_type, resource_id, ticket) VALUES (:resource_type, :resource_id, :ticket);',
  ['foobar', 123, 'auykg']
)
```

```python
# Release Mutex:
cursor.execute(
  'DELETE FROM mutexes WHERE resource_type=:resource_type AND resource_id=:resource_id AND ticket=:ticket;',
  ['foobar', 123, 'auykg']
)
```

注意上面的数据库查询都是非阻塞的，而Acquire Mutex操作通常不会一次就成功，所以通常采用轮询的方式。由于过于频繁的查询会增加数据库的开销，所以应该在每次查询之前先等待一小会儿。但是加入这种等待会让Acquire Mutex最终变成一个阻塞函数调用，而在Event-Driven Programming中不适合进行阻塞函数调用。所以合理的做法是把Mutex封装成一个独立的服务，并通过网络为应用程序提供Acquire Mutex和Release Mutex的接口。

注意附加的两个字段，使用随机生成的`ticket`以防止release了其他写者acquired的Mutex，使用`acquired_at`查找因异常情况导致的长期未释放的Mutex。

注意不要试图给Mutex设置超时，因为NLS没有有效的手段阻止已经超时的应用程序继续访问对应的共享资源，如果是因为网络故障而导致锁未被及时释放，应该先修复网络，如果是因为应用程序崩溃而导致的锁未被正常释放，应该先修复应用程序。

### Credits
- Computer Systems: A Programmer's Perspective, Third Edition
- [Readers–writers problem - Wikipedia](https://en.wikipedia.org/wiki/Readers-writers_problem)
- [Readers–writer lock - Wikipedia](https://en.wikipedia.org/wiki/Readers–writer_lock)
- [INSERT IF NOT EXISTS - Apache Cassandra](https://cassandra.apache.org/doc/latest/cassandra/developing/cql/dml.html#insert-statement) and [datastax/python-driver - GitHub](https://github.com/datastax/python-driver)
- [SET NX - Redis](https://redis.io/docs/latest/commands/set/) and [redis/redis-py - GitHub](https://github.com/redis/redis-py)
- [PRIMARY KEY - Oracle](https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/constraint.html) and [oracle/python-oracledb - GitHub](https://github.com/oracle/python-oracledb/)
