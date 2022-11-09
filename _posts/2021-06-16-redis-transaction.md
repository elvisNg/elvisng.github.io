---
layout: post
title: "Redis-Transaction"
subtitle: 'redis事务'
author: "Elvis"
header-style: text
mermaid: true
tags:
  - Redis
---

### Redis事务命令

Redis事务（transaction）提供了以下五个命令，用于用户操作事务功能，其分别是：

| 命令    | 描述                                                         |
| ------- | ------------------------------------------------------------ |
| MULTI   | 标记一个事务块的开始                                         |
| EXEC    | 执行所有事务块内的命令                                       |
| DISCARD | 取消事务，放弃执行事务块内的所有命令                         |
| WATCH   | 监视一个（或多个）key，如果在事务执行之前这个（或多个）key被其他命令所改动，那么事务将被打断 |
| UNWATCH | 取消 WATCH 命令对所有 keys 的监视                            |



#### Redis事务命令用法

`MULTI `命令用于开启一个事务，它总是返回 `OK` 。

MULTI 执行之后， 客户端可以继续向服务器发送任意多条命令， 这些命令不会立即被执行， 而是被放到一个队列中， 当 EXEC 命令被调用时， 所有队列中的命令才会被执行。

另一方面， 通过调用 `DISCARD` ， 客户端可以清空事务队列， 并放弃执行事务。

以下是一个事务例子， 它原子地增加了 `foo` 和 `bar` 两个键的值：

```javascript
> MULTI
OK

> INCR foo
QUEUED

> INCR bar
QUEUED

> EXEC
1) (integer) 1
2) (integer) 1
```

`EXEC `命令的回复是一个数组， 数组中的每个元素都是执行事务中的命令所产生的回复。 其中， 回复元素的先后顺序和命令发送的先后顺序一致。

当客户端处于事务状态时， 所有传入的命令都会返回一个内容为 `QUEUED` 的状态回复（status reply）， 这些被入队的命令将在 EXEC 命令被调用时执行。



#### 事务的错误与取消

- 事务在执行 EXEC 之前，入队的命令可能会出错。比如说，命令可能会产生语法错误（参数数量错误，参数名错误等等），或者其他更严重的错误，比如内存不足（如果服务器使用 maxmemory 设置了最大内存限制的话）。

如：

```javascript
>MULTI
+OK
>SET a 3
+QUEUED
>INCR a b c
-ERR wrong number of arguments for 'incr' command
```

> 这是由于`INCR`命令的语法错误，将在调用`EXEC`之前被检测出来，并终止事务（version2.6.5+）。

此时，MULTI开启的事务会被关闭，第一条的SET a 3 队列会被清空，所有这个事务内的命令提交将会不被执行



- 命令可能在 EXEC 调用之后失败。举个例子，事务中的命令可能处理了错误类型的键，比如将列表命令用在了字符串键上面，诸如此类。

如：

```javascript
>MULTI
+OK
>SET a 3
+QUEUED
>LPOP a
+QUEUED
>EXEC
*2
+OK
-ERR Operation against a key holding the wrong kind of value //a 为int 不能做LPOP操作

```

> - `EXEC`返回一个包含两个元素的字符串数组，一个元素是`OK`，另一个是`-ERR……`。

此时，事务内的两条命令，报错的`LPOP a`将不会被执行，但`SET a 3`将会被执行，即Redis事务的原子性只能保持在单命令之中。



#### Watch命令实现乐观锁

WATCH 命令可以为 Redis 事务提供 check-and-set （CAS）行为。

被 WATCH 的键会被监视，并会发觉这些键是否被改动过了。 如果有至少一个被监视的键在 EXEC 执行之前被修改了， 那么整个事务都会被取消， EXEC 返回空多条批量回复（null multi-bulk reply）来表示事务已经失败。

举个例子：

clientA:

```javascript
redis> WATCH name
OK

redis> MULTI
OK

redis> SET name peter
QUEUED

redis> EXEC
(nil)
```

此时，clientB:

```javascript
redis> SET name elvis
OK
```

此时clientA对于此次事务的set name peter执行失败。

> 对于UNWATCH&DISCARD命令就不进行演示了，相对的取消监听key值和关闭事务。



### Redis事务的ACID

#### 原子性（Atomicity）

单个 Redis 命令的执行是原子性的，但 Redis 没有在事务上增加任何维持原子性的机制，所以 Redis 事务的执行并不是原子性的。

如果一个事务队列中的所有命令都被成功地执行，那么称这个事务执行成功。

另一方面，如果 Redis 服务器进程在执行事务的过程中被停止 —— 比如接到 KILL 信号、宿主机器停机，等等，那么事务执行失败。

当事务失败时，Redis 也不会进行任何的重试或者回滚动作。（Redis是不支持回滚操作的）

#### 一致性（Consistency）

Redis 的一致性问题可以分为三部分来讨论：入队错误、执行错误、Redis 进程被终结。

##### 入队错误

在命令入队的过程中，如果客户端向服务器发送了错误的命令，比如命令的参数数量不对，等等， 那么服务器将向客户端返回一个出错信息， 并且将客户端的事务状态设为 `REDIS_DIRTY_EXEC` 。

当客户端执行 [EXEC](http://redis.readthedocs.org/en/latest/transaction/exec.html#exec) 命令时， Redis 会拒绝执行状态为 `REDIS_DIRTY_EXEC` 的事务， 并返回失败信息。

```
redis 127.0.0.1:6379> MULTI
OK

redis 127.0.0.1:6379> set key
(error) ERR wrong number of arguments for 'set' command

redis 127.0.0.1:6379> EXISTS key
QUEUED

redis 127.0.0.1:6379> EXEC
(error) EXECABORT Transaction discarded because of previous errors.
```

因此，带有不正确入队命令的事务不会被执行，也不会影响数据库的一致性。

##### 执行错误

如果命令在事务执行的过程中发生错误，比如说，对一个不同类型的 key 执行了错误的操作， 那么 Redis 只会将错误包含在事务的结果中， 这不会引起事务中断或整个失败，不会影响已执行事务命令的结果，也不会影响后面要执行的事务命令， 所以它对事务的一致性也没有影响。

##### Redis 进程被终结

如果 Redis 服务器进程在执行事务的过程中被其他进程终结，或者被管理员强制杀死，那么根据 Redis 所使用的持久化模式，可能有以下情况出现：

- 内存模式：如果 Redis 没有采取任何持久化机制，那么重启之后的数据库总是空白的，所以数据总是一致的。

- RDB 模式：在执行事务时，Redis 不会中断事务去执行保存 RDB 的工作，只有在事务执行之后，保存 RDB 的工作才有可能开始。所以当 RDB 模式下的 Redis 服务器进程在事务中途被杀死时，事务内执行的命令，不管成功了多少，都不会被保存到 RDB 文件里。恢复数据库需要使用现有的 RDB 文件，而这个 RDB 文件的数据保存的是最近一次的数据库快照（snapshot），所以它的数据可能不是最新的，但只要 RDB 文件本身没有因为其他问题而出错，那么还原后的数据库就是一致的。

- AOF 模式：因为保存 AOF 文件的工作在后台线程进行，所以即使是在事务执行的中途，保存 AOF 文件的工作也可以继续进行，因此，根据事务语句是否被写入并保存到 AOF 文件，有以下两种情况发生：

  1）如果事务语句未写入到 AOF 文件，或 AOF 未被 SYNC 调用保存到磁盘，那么当进程被杀死之后，Redis 可以根据最近一次成功保存到磁盘的 AOF 文件来还原数据库，只要 AOF 文件本身没有因为其他问题而出错，那么还原后的数据库总是一致的，但其中的数据不一定是最新的。

  2）如果事务的部分语句被写入到 AOF 文件，并且 AOF 文件被成功保存，那么不完整的事务执行信息就会遗留在 AOF 文件里，当重启 Redis 时，程序会检测到 AOF 文件并不完整，Redis 会退出，并报告错误。需要使用 redis-check-aof 工具将部分成功的事务命令移除之后，才能再次启动服务器。还原之后的数据总是一致的，而且数据也是最新的（直到事务执行之前为止）。

#### 隔离性（Isolation）

Redis 是单进程程序，并且它保证在执行事务时，不会对事务进行中断，事务可以运行直到执行完所有事务队列中的命令为止。因此，Redis 的事务是总是带有隔离性的。

#### 持久性（Durability）

因为事务不过是用队列包裹起了一组 Redis 命令，并没有提供任何额外的持久性功能，所以事务的持久性由 Redis 所使用的持久化模式决定：

- 在单纯的内存模式下，事务肯定是不持久的。

- 在 RDB 模式下，服务器可能在事务执行之后、RDB 文件更新之前的这段时间失败，所以 RDB 模式下的 Redis 事务也是不持久的。

- 在 AOF 的“总是 SYNC ”模式下，事务的每条命令在执行成功之后，都会立即调用 `fsync` 或 `fdatasync` 将事务数据写入到 AOF 文件。但是，这种保存是由后台线程进行的，主线程不会阻塞直到保存成功，所以从命令执行成功到数据保存到硬盘之间，还是有一段非常小的间隔，所以这种模式下的事务也是不持久的。

  其他 AOF 模式也和“总是 SYNC ”模式类似，所以它们都是不持久的。



小结：

Redis 事务保证了其中的一致性（C）和隔离性（I），但并不保证原子性（A）和持久性（D）。