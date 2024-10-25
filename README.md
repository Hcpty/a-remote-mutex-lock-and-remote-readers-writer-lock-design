# Readme
A note about Network-Based Locking System.

### Network-Based Locking System (NLS)

可以利用database来实现NLS，这种方法不graceful，但efficient。

##### 互斥锁

Database有一种特性，即创建unique记录的操作是互斥的，可以利用这种特性来实现互斥锁。

Oracle是一种高一致性数据库，基于Oracle实现互斥锁的代码如下：
```python
cursor.execute('CREATE TABLE foo_locks (lock_id INTEGER NOT NULL UNIQUE, token CHAR(5) NOT NULL);')  # prepare schema and table
cursor.execute('INSERT INTO foo_locks (lock_id, token) VALUES (:lock_id, :token);', [123, 'WseAI'])  # acquire mutex
cursor.execute('DELETE FROM foo_locks WHERE lock_id=:lock_id AND token=:token', [123, 'WseAI'])  # release mutex
```

Redis是一种高及时性数据库，基于Redis实现互斥锁的代码如下：
```python
r.set('foo_locks:123', 'wsEy4', nx=True)  # acquire mutex
r.eval('if redis.call("GET", KEYS[1]) == ARGV[1] then return redis.call("DEL", KEYS[1]) else return 0 end', 1, 'foo_locks:123', 'wsEy4')  # release mutex
```

使用随机生成的token的目的是防止release了别人acquired的互斥锁。

不要给锁设置某种超时，因为NLS没有有效的手段阻止已经超时的应用程序继续访问对应的共享资源。应该用合理的方式解决因应用程序崩溃或网络故障导致的锁未被正常释放的问题。

### Credits
- Computer Systems: A Programmer's Perspective, Third Edition
- [Readers–writers problem - Wikipedia](https://en.wikipedia.org/wiki/Readers-writers_problem)
- [Readers–writer lock - Wikipedia](https://en.wikipedia.org/wiki/Readers–writer_lock)
- [UNIQUE Constraint - Oracle](https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/constraint.html) and [oracle/python-oracledb - GitHub](https://github.com/oracle/python-oracledb/)
- [SET NX - Redis](https://redis.io/docs/latest/commands/set/) and [redis/redis-py - GitHub](https://github.com/redis/redis-py)
