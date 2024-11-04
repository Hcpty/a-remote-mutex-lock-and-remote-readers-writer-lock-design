# Readme
A Remote Mutex Lock and Remote Readers-Writer Lock design.

### Remote Mutex Lock

当位于不同机器上的应用程序并发地写一组位于网络中的共享资源时，可以使用Remote Mutex Lock。

Database有一种特性，即创建unique记录的操作是互斥的，可以基于这种特性来实现Remote Mutex Lock。

基于[Redis](https://redis.io/)实现Remote Mutex Lock的原理如下：

```python
# Acquire Mutex Lock:
r.set('mutex_locks/foobar&123', 'gluww&1729837899', nx=True)
```

```python
# Release Mutex Lock:
lua_script = \
  """if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
  else
    return 0
  end"""
r.eval(lua_script, 1, 'mutex_locks/foobar&123', 'gluww&1729837899')
```

注意附加的两个字段，"gluww"是**ticket**字段的值，1729837899是**required_at**字段的值，使用随机生成的**ticket**以防止release了其他写者acquired的Mutex Lock，使用**acquired_at**查找因异常情况导致的长期未释放的Mutex Locks。

可以把上面的acquire Mutex Lock和release Mutex Lock的操作封装成统一的接口供应用程序调用，调用示例：

```python
# Acquire Mutex Lock
acquire_mutex_lock('foobar', 123, 'gluww')
```

```python
# Release Mutex Lock
release_mutex_lock('foobar', 123, 'gluww')
```

### Remote Readers-Writer Lock

当位于不同机器上的应用程序并发地读写一组位于网络中的共享资源时，可以使用Remote Readers-Writer Lock。

在实现Remote Readers-Writer Lock的时候用到了两种数据结构：Mutex Lock和Doorman。每一个共享资源都对应一个Mutex Lock，要么一群读者共同持有这个Mutex Lock，要么一个写者独立持有这个Mutex Lock。每一个共享资源都对应一个Doorman，用于辅助这个共享资源上的Mutex Lock的获取和释放。

在[Redis](https://redis.io/)中存储Doorman数据结构：
```python
doorman = {
  'resource_type': 'foobar',
  'resource_id': 123,
  'active_readers': {
    'afxkv': 1729933228,
    'ngyfs': 1729933236,
    'pvzas': 1729933241
  },
  'has_pending_writer': True,
  'pending_writer': ['jqpcq', 1729933260],
  'updated_at': 1729933260,
  'created_at': 1729932120
}
r.json().set('foobar.doorman/123', '$', doorman)
r.json().get('foobar.doorman/123', '$')
```

基于Database实现Remote Readers-Writer Lock的原理如下：

```python
# Acquire Readers Lock:
acquire_mutex_lock('foobar.doorman', 123, 'fuvub')
doorman = read_or_create_doorman('foobar', 123)
if not doorman['has_pending_writer']:
  if len(doorman['active_readers']) == 0:
    acquire_mutex_lock('foobar', 123, 'readers')
  doorman['active_readers']['qyqen'] = int(time.time())
  update_doorman('foobar', 123, doorman)
release_mutex_lock('foobar.doorman', 123, 'fuvub')
```

```python
# Release Readers Lock:
acquire_mutex_lock('foobar.doorman', 123, 'whfxo')
doorman = read_or_create_doorman('foobar', 123)
if 'qyqen' in doorman['active_readers']:
  del doorman['active_readers']['qyqen']
  update_doorman('foobar', 123, doorman)
if len(doorman['active_readers']) == 0:
  release_mutex_lock('foobar', 123, 'readers')
release_mutex_lock('foobar.doorman', 123, 'whfxo')
```

```python
# Acquire Writer Lock:
acquire_mutex_lock('foobar.doorman', 123, 'hcmfm')
doorman = read_or_create_doorman('foobar', 123)
if not doorman['has_pending_writer']:
  doorman['has_pending_writer'] = True
  doorman['pending_writer'] = ['desjn', int(time.time())]
  update_doorman('foobar', 123, doorman)
elif doorman['pending_writer'][0] == 'desjn' and len(doorman['active_readers']) == 0:
  acquire_mutex_lock('foobar', 123, 'desjn')
release_mutex_lock('foobar.doorman', 123, 'hcmfm')
```

```python
# Release Writer Lock:
acquire_mutex_lock('foobar.doorman', 123, 'kxzsb')
release_mutex_lock('foobar', 123, 'desjn')
doorman = read_or_create_doorman('foobar', 123)
if doorman['has_pending_writer'] and doorman['pending_writer'][0] == 'desjn':
  doorman['has_pending_writer'] = False
  doorman['pending_writer'] = ['', 0]
  update_doorman('foobar', 123, foobar)
release_mutex_lock('foobar.doorman', 123, 'kxzsb')
```

注意**pending_writer**字段和**active_readers**字段记录了写者和读者apply for Mutex Lock或acquire Mutex Lock的时间戳，其用途和Mutex Lock中的**acquired_at**字段相似。

可以把上面的acquire Readers-Writer Lock和release Readers-Writer Lock的操作封装成统一的接口供应用程序调用，调用示例：

```python
# Acquire Readers Lock:
acquire_readers_lock('foobar', 123, 'qyqen')
```

```python
# Release Readers Lock:
release_readers_lock('foobar', 123, 'qyqen')
```

```python
# Acquire Writer Lock:
acquire_writer_lock('foobar', 123, 'desjn')
```

```python
# Release Writer Lock:
release_writer_lock('foobar', 123, 'desjn')
```

### Credits
- Computer Systems: A Programmer's Perspective, Third Edition
- [Readers–writers problem - Wikipedia](https://en.wikipedia.org/wiki/Readers-writers_problem) and [Readers–writer lock - Wikipedia](https://en.wikipedia.org/wiki/Readers–writer_lock)
- [SET NX - Redis](https://redis.io/docs/latest/commands/set/) and [redis/redis-py - GitHub](https://github.com/redis/redis-py)
