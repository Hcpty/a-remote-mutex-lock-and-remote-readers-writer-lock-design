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

注意附加的两个字段，使用随机生成的*ticket*以防止release了其他写者acquired的Mutex，使用*acquired_at*查找因异常情况导致的长期未释放的Mutex。

注意不要给Mutex设置超时释放，因为NLS没有有效的手段阻止已经超时的应用程序继续访问对应的共享资源。

##### 基于Database实现Readers-Writer Lock

当访问共享资源的应用程序中既有读者又有写者的时候，使用Readers-Writer Lock更高效。

在实现Readers-Writer Lock的时候用到了两种数据结构：Mutex和Doorman。每一个共享资源都对应一个Mutex，要么一群读者共同持有这个Mutex，要么一个写者独立持有这个Mutex。另外，每一个共享资源都对应一个Doorman，用于辅助Mutex的获取和释放。

在[Apache Cassandra](https://cassandra.apache.org/_/index.html)中存储Doorman数据结构：
```python
session.execute(
  """CREATE TABLE doormans (
    PRIMARY KEY (resource_type, resource_id),
    resource_type VARCHAR,
    resource_id INT,
    active_readers MAP<VARCHAR, TIMESTAMP>,
    has_pending_writer BOOLEAN,
    pending_writer TUPLE<VARCHAR, TIMESTAMP>,
    updated_at TIMESTAMP,
    created_at TIMESTAMP
  );"""
)
```

在[Redis](https://redis.io/)中存储Doorman数据结构：
```python
doorman = {
  'resource_type': 'foobar',
  'resource_id': 123,
  'active_readers': {
    'afxkv': 1729933228832,
    'ngyfs': 1729933236560,
    'pvzas': 1729933241811
  },
  'has_pending_writer': True,
  'pending_writer': ['jqpcq', 1729933260301],
  'updated_at': 1729933260301,
  'created_at': 1729932120103
}
r.json().set('foobar.doorman/123', '$', doorman)
r.json().get('foobar.doorman/123', '$')
```

在[Oracle Database](https://www.oracle.com/database/)或[Oracle In-Memory Database](https://www.oracle.com/database/)中存储Doorman数据结构：
```python
cursor.execute(
  """CREATE TABLE doormans (
    PRIMARY KEY (resource_type, resource_id),
    resource_type CHAR(36),
    resource_id INTEGER,
    active_readers JSON(OBJECT),
    has_pending_writer BOOLEAN,
    pending_writer JSON(ARRAY),
    updated_at TIMESTAMP,
    created_at TIMESTAMP
  );"""
)
```

基于Mutex和Doorman实现Readers-Writer Lock的原理如下：

```python
# Reader acquire Mutex:
acquire('foobar.doorman', 123, 'fuvub')
doorman = get_or_set_doorman('foobar', 123)
if not doorman['has_pending_writer']:
  if len(doorman['active_readers']) == 0:
    acquire('foobar', 123, 'readers')
  doorman['active_readers']['qyqen'] = int(time.time() * 1000)
  set_doorman('foobar', 123, doorman)
release('foobar.doorman', 123, 'fuvub')
```

```python
# Reader release Mutex:
acquire('foobar.doorman', 123, 'whfxo')
doorman = get_or_set_doorman('foobar', 123)
del doorman['active_readers']['qyqen']
set_doorman('foobar', 123, doorman)
if len(doorman['active_readers']) == 0:
  release('foobar', 123, 'readers')
release('foobar.doorman', 123, 'whfxo')
```

```python
# Writer acquire Mutex:
acquire('foobar.doorman', 123, 'hcmfm')
doorman = get_or_set_doorman('foobar', 123)
if not doorman['has_pending_writer']:
  doorman['has_pending_writer'] = True
  doorman['pending_writer'] = ['desjn', int(time.time() * 1000)]
  set_doorman('foobar', 123, doorman)
elif doorman['pending_writer'][0] == 'desjn' and len(doorman['active_readers']) == 0:
  acquire('foobar', 123, 'zvfwb')
release('foobar.doorman', 123, 'hcmfm')
```

```python
# Writer release Mutex:
acquire('foobar.doorman', 123, 'kxzsb')
release('foobar', 123, 'zvfwb')
doorman = get_or_set_doorman('foobar', 123)
doorman['has_pending_writer'] = False
doorman['pending_writer'] = [None, None]
set_doorman('foobar', 123, foobar)
release('foobar.doorman', 123, 'kxzsb')
```

可以看到，一次Readers-Writer Lock的使用，从获取到释放，至少要经历十数次数据库查询，这还没有算上因重试而增加的次数，但是这种多次查询的开销是可以通过Stored Procedure或Lua Script来消除的。

NLS真正的难题在于NLS无法真实地“授予”锁，也无法真实地“收回”锁，这种“授予”和“收回”都是虚拟的，因为锁对应的共享资源并不在自己的控制范围之内，因此使用NLS的场景通常对应用程序的编写质量有很高的要求。

### Credits
- Computer Systems: A Programmer's Perspective, Third Edition
- [Readers–writers problem - Wikipedia](https://en.wikipedia.org/wiki/Readers-writers_problem)
- [Readers–writer lock - Wikipedia](https://en.wikipedia.org/wiki/Readers–writer_lock)
- [INSERT IF NOT EXISTS - Apache Cassandra](https://cassandra.apache.org/doc/latest/cassandra/developing/cql/dml.html#insert-statement) and [datastax/python-driver - GitHub](https://github.com/datastax/python-driver)
- [SET NX - Redis](https://redis.io/docs/latest/commands/set/) and [redis/redis-py - GitHub](https://github.com/redis/redis-py)
- [PRIMARY KEY - Oracle](https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/constraint.html) and [oracle/python-oracledb - GitHub](https://github.com/oracle/python-oracledb/)
