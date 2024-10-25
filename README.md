# Readme
A note about Network-Based Locking System.

### 网络锁系统

几乎Database都有一种特性，即创建unique记录的操作是互斥的，可以利用这种特性来实现互斥锁。

Oracle是一种高一致性数据库，基于Oracle实现互斥锁的代码如下：
```python
```

Redis是一种高及时性数据库，基于Redis实现互斥锁的代码如下：
```python
r.set('bar', 'token', nx=True)  # acquire mutext
r.eval('if redis.call("GET", KEYS[1]) == ARGV[1] then return redis.call("DEL", KEYS[1]) else return 0 end', 1, 'bar', 'token')  # release mutext
```

### Credits
- Computer Systems: A Programmer's Perspective, Third Edition
- [Readers–writers problem - Wikipedia](https://en.wikipedia.org/wiki/Readers-writers_problem)
- [Readers–writer lock - Wikipedia](https://en.wikipedia.org/wiki/Readers–writer_lock)
- [UNIQUE - Oracle](https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/constraint.html)
- [oracle/python-oracledb - GitHub](https://github.com/oracle/python-oracledb/)
- [SET NX - Redis](https://redis.io/docs/latest/commands/set/)
- [redis/redis-py - GitHub](https://github.com/redis/redis-py)
