---
layout:     post   						
title:      PG中的 xmax与 multixact机制(1) # 标题 
subtitle:    #副标题
date:       2021-06-26 				# 时间
author:     CCH 					# 作者
header-img: img/post-bg-map.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - greenplum
    - PostgreSQL
---

# 0
之前一直认为Postgres中tuple头部信息的xmax只记录了删除/更新这条记录的事务号，为PG的MVCC机制服务。调试了一下PG行锁的实现机制，发现xmax所包含的信息没有那么简单，它还负责记录该tuple上的行锁以及行锁的模式信息(EXCLUDE/SHARE)，以及持有该锁的事务号; 另外，因为多个事务可以持有某个行的SHARE锁，PG不可能将所有持有这个行锁的事务的事务号信息都放在tuple的头部，所以又引入了一个multixact机制，将这多个事务号"合并"起来放在共享内存里，tuple头部信息保存一个映射指向共享内存里这个"合并"的多个事务号即可。

本文作为阅读PG行锁代码机制和multixact代码的笔记和总结 (代码基于 PostgreSQL 9.6.10 版本)

# 1 PG的行锁
PG 支持4种行锁模式: FOR KEY SHARE, FOR SHARE, FOR NO KEY UPDATE, FOR UPDATE, 其中FOR KEY SHARE和 FOR SHARE为 SHARE模式的行锁, FOR NO KEY UPDATE和 FOR UPDATE是EXCLUSIVE模式的行锁。

锁冲突表如下

||  FOR KEY SHARE   | FOR SHARE  |  FOR NO KEY UPDATE   | FOR UPDATE  |
|---|  ----  | ----  | ----   | ----  |
|FOR KEY SHARE |  |  |   | X |
|FOR SHARE|   |  | X  | X |
|FOR NO KEY UPDATE|   | X | X  | X |
|FOR UPDATE| X  | X | X  | X |


# 2 行锁上锁流程

PG 使用LockRows算子对行进行上锁. LockRows 对下层查询执行之后得到的tuple依次进行上锁

```SQL
postgres=# explain select * from t1 where a = 1 for key share;
                        QUERY PLAN
-----------------------------------------------------------
 LockRows  (cost=0.00..38.36 rows=11 width=14)
   ->  Seq Scan on t1  (cost=0.00..38.25 rows=11 width=14)
         Filter: (a = 1)
(3 rows)
```

ExecLockRows执行过程，会先调用ExecProcNode 来执行下层的执行计划，获得一个结果tuple, 然后调用heap_lock_tuple来对该tuple上锁。

```C
HTSU_Result
heap_lock_tuple(Relation relation, HeapTuple tuple,
				CommandId cid, LockTupleMode mode, LockWaitPolicy wait_policy,
				bool follow_updates,
				Buffer *buffer, HeapUpdateFailureData *hufd)
```

heap_lock_tuple会根据传入的LockTupleMode，对tuple上对应模式的锁。

它首先会调用HeapTupleSatisfiesUpdate去使用tuple头部的infomask信息，去看这个tuple目前的状态(可见性以及是否正在被改写，是否已存在行锁等等)。HeapTupleSatisfiesUpdate判断tuple状态的逻辑细节比较复杂，总体来讲就是使用infomask的位信息以及xmax,xmin信息得到tuple的可见性以及是否已经上行锁，这里先省略细节。HeapTupleSatisfiesUpdate回返回HeapTupleInvisible，HeapTupleMayBeUpdated，HeapTupleSelfUpdated，HeapTupleUpdated，HeapTupleBeingUpdated

```C
typedef enum
{
	HeapTupleMayBeUpdated,
	HeapTupleInvisible,
	HeapTupleSelfUpdated,
	HeapTupleUpdated,
	HeapTupleBeingUpdated,
	HeapTupleWouldBlock			/* can be returned by heap_tuple_lock */
} HTSU_Result;
```
HeapTupleInvisible: 表示该tuple不可见
HeapTupleMayBeUpdated: 表示该tuple可见且有效，可以用来被当前函数改写
HeapTupleSelfUpdated: tuple被当前事务修改过
HeapTupleUpdated: 该tuple被其他已经提交的事务修改过
HeapTupleBeingUpdated: tuple正在被其他事务修改，且该事务还没提交

根据HeapTupleSatisfiesUpdate的不同的返回值，需要采取不同的动作。
1. 当HeapTupleSatisfiesUpdate返回为HeapTupleBeingUpdated或者HeapTupleUpdated时，说明有并发的事务正在或者已经修改过该tuple并对该tuple上了行锁，那么根据行锁的语义，我们就需要等待这个其他事务结束。根据传入的行锁模式,与infomask中记录的该tuple已有的行锁，判断要上的锁是否与已存在的锁冲突，如果冲突，则将require_sleep这个flag设置为true，否则设置为false。
   
   
判断完毕之后，会根据require_sleep这个flag决定要不要进行等待。如果require_sleep为true, 表示需要等待其他已持有行锁的事务结束, 于是会调用XactLockTableWait,传入正在持有行锁的事务xid, 等待这个事务的结束. XactLockTableWait会使用xid去计算一个哈希值, 这个哈希值在共享内存中会映射到这个xid的事务锁. 于是当前进程会等待这个事务锁. 当持有行锁的事务结束时,它会释放它自己的事务锁, 于是当前等待这个事务锁的进程会从XactLockTableWait中被唤醒. 当我们被唤醒之后,说明我们等待的事务已经结束了,它也就释放了持有的行锁, 那么我们就可以进行上行锁的逻辑(见2).

2. 当HeapTupleSatisfiesUpdate返回为HeapTupleMayBeUpdated时,说明我们可以对该tuple进行上锁. PG的行锁信息实质上是记录在tuple头部的infomask中. 首先调用compute_new_xmax_infomask根据锁模式和tuple当前的xmax以及当前事务的xid计算新的infomask和infomask2,然后通过位运算将这些信息写到tuple头部.

```C
compute_new_xmax_infomask(xmax, old_infomask, tuple->t_data->t_infomask2,
							  GetCurrentTransactionId(), mode, false,
							  &xid, &new_infomask, &new_infomask2);

....

tuple->t_data->t_infomask &= ~HEAP_XMAX_BITS;
tuple->t_data->t_infomask2 &= ~HEAP_KEYS_UPDATED;
tuple->t_data->t_infomask |= new_infomask;
tuple->t_data->t_infomask2 |= new_infomask2;
if (HEAP_XMAX_IS_LOCKED_ONLY(new_infomask))
	HeapTupleHeaderClearHotUpdated(tuple->t_data);
HeapTupleHeaderSetXmax(tuple->t_data, xid);
```

infomask中,有位信息用来表示tuple的xmax字段记录的xid是持有该tuple行锁的事务,还是修改(update/delete)该tuple的事务. 在compute_new_xmax_infomask中,实质上就是根据mode
来设置 HEAP_XMAX_KEYSHR_LOCK, HEAP_XMAX_EXCL_LOCK , HEAP_XMAX_SHR_LOCK 到infomask中, 将xmax设置为当前事务xid. 另外注意的是, 对于最强的FOR UPDATE 锁,还需要将infomask2中设置HEAP_KEYS_UPDATED位.

```C
#define HEAP_XMAX_KEYSHR_LOCK	0x0010	/* xmax is a key-shared locker */
#define HEAP_XMAX_EXCL_LOCK		0x0040	/* xmax is exclusive locker */
#define HEAP_XMAX_LOCK_ONLY		0x0080	/* xmax, if valid, is only a locker */
 /* xmax is a shared locker */
#define HEAP_XMAX_SHR_LOCK	(HEAP_XMAX_EXCL_LOCK | HEAP_XMAX_KEYSHR_LOCK)

#define HEAP_LOCK_MASK	(HEAP_XMAX_SHR_LOCK | HEAP_XMAX_EXCL_LOCK | \
						 HEAP_XMAX_KEYSHR_LOCK)

/*
 * information stored in t_infomask2:
 */
 ...
#define HEAP_KEYS_UPDATED		0x2000	/* tuple was updated and key cols
										 * modified, or tuple deleted */
```

修改完tuple的以上这些信息之后, 还需要写一条关于这个行锁的XLOG记录, 这个XLOG里会记录tuple tid, 持有该行锁的事务xid, 以及tuple的infomask,infomask2信息.


