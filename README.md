# Readme
A note about Network-Based Locking System.

### 网络锁系统

数据库有一个共性，那就是创建unique记录的操作是互斥的。

Oracle是一种高一致性数据库，基于Oracle的互斥锁的具体实现如下：
```python
```

Redis是一种高及时性数据库，基于Redis的互斥锁的具体实现如下：
```python
r.set('foo.lock', 'token', nx=True)

r.eval('if redis.call("GET", KEYS[1]) == ARGV[1] then return redis.call("DEL", KEYS[1]) else return 0 end', 1, 'foo.lock', 'token')
```

最佳实践：
- 给锁设置一个只有自己知道的token，解锁时验证token，以防解了别人加的锁。
- 不要设置超时，因为网络锁系统没有有效的手段阻止超时的应用程序继续访问对应的共享资源。

### Credits
- Computer Systems: A Programmer's Perspective, Third Edition
- [Readers–writers problem - Wikipedia](https://en.wikipedia.org/wiki/Readers-writers_problem)
- [Readers–writer lock - Wikipedia](https://en.wikipedia.org/wiki/Readers–writer_lock)
