# Readme
A note about Network-Based Locking System.

### 网络锁系统

很多数据库都有一个特点，那就是创建unique记录的操作是互斥的，可以利用这一个特点来实现互斥锁。

Oracle是一种高一致性数据库，基于Oracle的互斥锁的具体实现如下：
```python
cursor.execute('CREATE TABLE foo (name , token )')
cursor.execute('INSERT INTO foo (name, token) VALUES (:name, :token)', name='bar', token='token')
cursor.execute('DELETE FROM foo WHERE name=:name AND token=:token', name='bar', token='token')
```

Redis是一种高及时性数据库，基于Redis的互斥锁的具体实现如下：
```python
r.set('bar', 'token', nx=True)
r.eval('if redis.call("GET", KEYS[1]) == ARGV[1] then return redis.call("DEL", KEYS[1]) else return 0 end', 1, 'bar', 'token')
```

### Credits
- Computer Systems: A Programmer's Perspective, Third Edition
- [Readers–writers problem - Wikipedia](https://en.wikipedia.org/wiki/Readers-writers_problem)
- [Readers–writer lock - Wikipedia](https://en.wikipedia.org/wiki/Readers–writer_lock)
