# redlock

## 官方
[官方原理](https://redis.io/topics/distlock/#the-redlock-algorithm)

Redlock算法：
在算法的分布式版本中，我们假设我们有N个Redis母版。这些节点是完全独立的，因此我们不使用复制或任何其他隐式协调系统。我们已经描述了如何在单个实例中安全地获取和释放锁。我们认为该算法将使用此方法在单个实例中获取和释放锁，这是理所当然的。在我们的示例中，我们将N = 5设置为一个合理的值，因此我们需要在不同的计算机或虚拟机上运行5个Redis主服务器，以确保它们将以大多数独立的方式发生故障。

为了获取锁，客户端执行以下操作：

1. 它以毫秒为单位获取当前时间。
2. 它尝试在所有N个实例中顺序使用所有实例中相同的键名和随机值来获取锁定。在第2步中，当在每个实例中设置锁时，客户端使用的超时时间小于总锁自动释放时间，以便获取该超时时间。例如，如果自动释放时间为10秒，则超时时间可能在5到50毫秒之间。这样可以防止客户端长时间与处于故障状态的Redis节点通信时保持阻塞：如果一个实例不可用，我们应该尝试与下一个实例尽快通信。
3. 客户端通过从当前时间中减去在步骤1中获得的时间戳，来计算获取锁所需的时间。当且仅当客户端能够在大多数实例（至少3个）中获取锁时， ，并且获取锁所花费的总时间小于锁有效时间，则认为已获取锁。
4. 如果获取了锁，则将其有效时间视为初始有效时间减去经过的时间，如步骤3中所计算。
5. 如果客户端由于某种原因（无法锁定N / 2 + 1实例或有效时间为负数）而未能获得该锁，它将尝试解锁所有实例（即使它认为不是该实例）能够锁定）

**注意**：
1. 这个锁是不可重入的。可以改为hash数据结构实现可重入。
2. 这个算法并没有实现锁续期，需另开线程去实现，java版的redisson利用watchdog实现。


## Python实现

```python
import logging
import string
import random
import time
from collections import namedtuple

import redis
from redis.exceptions import RedisError

# Python 3 compatibility
string_type = getattr(__builtins__, 'basestring', str)

try:
    basestring
except NameError:
    basestring = str


Lock = namedtuple("Lock", ("validity", "resource", "key"))


class CannotObtainLock(Exception):
    pass


class MultipleRedlockException(Exception):
    def __init__(self, errors, *args, **kwargs):
        super(MultipleRedlockException, self).__init__(*args, **kwargs)
        self.errors = errors

    def __str__(self):
        return ' :: '.join([str(e) for e in self.errors])

    def __repr__(self):
        return self.__str__()


class Redlock(object):

    default_retry_count = 3
    default_retry_delay = 0.2
    clock_drift_factor = 0.01
    unlock_script = """
    if redis.call("get",KEYS[1]) == ARGV[1] then
        return redis.call("del",KEYS[1])
    else
        return 0
    end"""

    def __init__(self, connection_list, retry_count=None, retry_delay=None):
        self.servers = []
        for connection_info in connection_list:
            try:
                if isinstance(connection_info, string_type):
                    server = redis.StrictRedis.from_url(connection_info)
                elif type(connection_info) == dict:
                    server = redis.StrictRedis(**connection_info)
                else:
                    server = connection_info
                self.servers.append(server)
            except Exception as e:
                raise Warning(str(e))
        self.quorum = (len(connection_list) // 2) + 1

        if len(self.servers) < self.quorum:
            raise CannotObtainLock(
                "Failed to connect to the majority of redis servers")
        self.retry_count = retry_count or self.default_retry_count
        self.retry_delay = retry_delay or self.default_retry_delay

    def lock_instance(self, server, resource, val, ttl):
        try:
            assert isinstance(ttl, int), 'ttl {} is not an integer'.format(ttl)
        except AssertionError as e:
            raise ValueError(str(e))
        return server.set(resource, val, nx=True, px=ttl)

    def unlock_instance(self, server, resource, val):
        try:
            server.eval(self.unlock_script, 1, resource, val)
        except Exception as e:
            logging.exception("Error unlocking resource %s in server %s", resource, str(server))

    def get_unique_id(self):
        CHARACTERS = string.ascii_letters + string.digits
        return ''.join(random.choice(CHARACTERS) for _ in range(22)).encode()

    def lock(self, resource, ttl):
        retry = 0
        val = self.get_unique_id()

        # Add 2 milliseconds to the drift to account for Redis expires
        # precision, which is 1 millisecond, plus 1 millisecond min
        # drift for small TTLs.
        drift = int(ttl * self.clock_drift_factor) + 2  # 时钟漂移时间

        redis_errors = list()
        while retry < self.retry_count:
            n = 0
            start_time = int(time.time() * 1000)
            del redis_errors[:]
            for server in self.servers:
                try:
                    if self.lock_instance(server, resource, val, ttl):  # 尝试在多个服务器获取锁
                        n += 1
                except RedisError as e:
                    redis_errors.append(e)
            elapsed_time = int(time.time() * 1000) - start_time
            validity = int(ttl - elapsed_time - drift)
            if validity > 0 and n >= self.quorum:  # 如果在有效时间内获得锁且服务器数量大于等于[所有服务器数量/2] + 1，则认为获取锁
                if redis_errors:
                    raise MultipleRedlockException(redis_errors)
                return Lock(validity, resource, val)
            else:  # 否则释放已经获取的锁
                for server in self.servers:
                    try:
                        self.unlock_instance(server, resource, val)
                    except:
                        pass
                retry += 1
                time.sleep(self.retry_delay)
        return False

    def unlock(self, lock):
        redis_errors = []
        for server in self.servers:
            try:
                self.unlock_instance(server, lock.resource, lock.key)
            except RedisError as e:
                redis_errors.append(e)
        if redis_errors:
            raise MultipleRedlockException(redis_errors)

```


## 参考
1. [redis分布式锁-知乎](https://www.zhihu.com/question/300767410?sort=created)
2. [官方原理](https://redis.io/topics/distlock/#the-redlock-algorithm)