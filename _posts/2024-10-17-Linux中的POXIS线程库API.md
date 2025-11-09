---
layout:     post
title: Linux中的POXIS线程库API
date:       2024-10-17
author:     MZ
header-img: img/post-bg-keybord.jpg
catalog: true
categories:
    - Linux应用篇
tags:
    - Linux
    - POXIS
    - 线程	
    - pthread
---

# 线程基础用法

## 1.线程概述

早期Linux只有进程的概念，一个进程对应一个地址空间，进程与进程之间不能互相访问，保证隔离性和安全性。直到Windows的线程可以很好的处理多任务时，Linux才引入了多线程的概念，它只是在进程的基础上再次分为线程，与Windows有一定的区别。

多个线程在同一进程下，共享进程的资源，可以当作一个应用下的多个子任务，CPU的最小调度单位就是这些任务，也就是线程。

## 2.创建线程

每个线程都有一个id，类型为pthread_t，其实是无符号长整型类型——unsigned long。

```cmake
pthread_t pthread_self(void);	// 返回当前线程的线程ID
```

创建线程API：

```cmake
#include <pthread.h>
int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                   void *(*start_routine) (void *), void *arg);
// Compile and link with -pthread, 线程库的名字叫pthread, 全名: libpthread.so libptread.a
```

参数：

- thread：创建完成后传出线程id
- attr：属性，一般为NULL
- start_routine：函数指针，就是线程要执行的任务
- arg：函数中的参数

## 3.线程退出

在编写多线程程序的时候，如果想要让线程退出，但是不会导致虚拟地址空间的释放（针对于主线程），我们就可以调用线程库中的线程退出函数，**只要调用该函数当前线程就马上退出了，并且不会影响到其他线程的正常运行，不管是在子线程或者主线程中都可以使用。**

```cmake
#include <pthread.h>
void pthread_exit(void *retval);
```

## 4.线程回收

线程和进程一样，子线程退出的时候其内核资源主要由主线程回收，线程库中提供的线程回收函叫做`**pthread_join()**`，这个函数是一个**阻塞函数**，如果还有子线程在运行，调用该函数就会阻塞，子线程退出函数解除阻塞进行资源的回收，函数被调用一次，只能回收一个子线程，如果有多个子线程则需要循环进行回收。

```cmake
#include <pthread.h>
// 这是一个阻塞函数, 子线程在运行这个函数就阻塞
// 子线程退出, 函数解除阻塞, 回收对应的子线程资源, 类似于回收进程使用的函数 wait()
int pthread_join(pthread_t thread, void **retval);
```

retval其实是可以从子线程退出程序回收的数据，因为退出时可以带一个参数`**pthread_exit(void \*retval)**`然后主线程就可以得到这个数据。

## 5.线程分离

在一般情况下，主线程需要回收子线程相关资源，需要调用`**pthread_join**`等待子线程运行完再回收，而如果不想回收子线程，或者不想等待，就可以线程分离，让子线程结束后，系统自动回收。

```cmake
#include <pthread.h>
// 参数就子线程的线程ID, 主线程就可以和这个子线程分离了
int pthread_detach(pthread_t thread);
```

## 6.线程取消

用于在一个线程中结束另一个线程，但是不是调用就直接结束另一个线程，需要有一个**条件：另一个线程调用了系统调用函数。**

```cmake
#include <pthread.h>
// 参数是子线程的线程ID
int pthread_cancel(pthread_t thread);
```

# 线程同步

## 1.线程同步概念

**为什么要实现线程同步？不实现有什么问题？**

在学习多线程时，我们了解到Linux的线程是共享进行下的资源的，也就是说同一进程下的线程可以彼此相互访问。那么多个线程同时访问一个资源就会出现问题，读操作可能还好，但是写操作就很容易出现问题。

例如：i++操作，i是保存在内存堆中或全局数据中的变量，两个线程都要i++的话，都需要从内存读取i的值，然后操作，但是如果两个线程几乎同时发生，那么一个线程的操作还没写回内存，另一个线程就已经开始读内存中未被修改的i值了，所以最后i只加了一次，明明两个线程都i++了，却只加了一次，所以会出现问题。

**怎么实现线程同步呢？**

有四种方式：**互斥锁**、**读写锁**、**条件变量**、**信号量**

## 2.互斥锁

在Linux中互斥锁的类型为`**pthread_mutex_t**`，创建一个这种类型的变量就得到了一把互斥锁：

```cmake
pthread_mutex_t  mutex;
```

创建和销毁操作：

```cmake
// 初始化互斥锁
// restrict: 是一个关键字, 用来修饰指针, 只有这个关键字修饰的指针可以访问指向的内存地址, 其他指针是不行的
int pthread_mutex_init(pthread_mutex_t *restrict mutex,
           const pthread_mutexattr_t *restrict attr);
// 释放互斥锁资源            
int pthread_mutex_destroy(pthread_mutex_t *mutex);
```

加锁操作：

```cmake
// 修改互斥锁的状态, 将其设定为锁定状态, 这个状态被写入到参数 mutex 中
int pthread_mutex_lock(pthread_mutex_t *mutex);
```

解锁操作：

```cmake
// 对互斥锁解锁
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```

## 3.读写锁

读写锁是互斥锁的**升级版**, 在做**读操作的时候可以提高程序的执行效率**，如果**所有的线程都是做读操作, 那么读是并行的**，但是使用互斥锁，读操作也是串行的。

读写锁是一把锁，锁的类型为`**pthread_rwlock_t**`，有了类型之后就可以创建一把互斥锁了：

```cmake
pthread_rwlock_t rwlock;
```

之所以称其为读写锁，是因为这把锁既可以锁定读操作，也可以锁定写操作。

通过一把读写锁可以锁定读或者写操作，下面介绍一下关于读写锁的特点：

1. 读锁锁住临界区时，线程是并行访问临界区的。`**读锁是共享的**`**。**
2. 写锁锁住临界区时，线程只能串行访问临界区。`**写锁是独占的**`**。**
3. 如果两个线程同时用写锁和读锁同时锁住临界区，并且两个线程同时访问两个临界区，那么写锁先执行，读锁会阻塞线程。

总之，读写锁与互斥锁的区别就在于**读是并行的**，这样极大提高了效率。（在读操作较多，写操作较少的情况下）

创建和销毁操作：

```cmake
#include <pthread.h>
pthread_rwlock_t rwlock;
// 初始化读写锁
int pthread_rwlock_init(pthread_rwlock_t *restrict rwlock,
           const pthread_rwlockattr_t *restrict attr);
// 释放读写锁占用的系统资源
int pthread_rwlock_destroy(pthread_rwlock_t *rwlock);
```

加与解 **读**锁：

```cmake
// 在程序中对读写锁加读锁, 锁定的是读操作
int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock);

// 在程序中对读写锁加写锁, 锁定的是写操作
int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock);
```

加与解 **写**锁：



## 4.条件变量

严格意义上来说，条件变量的主要作用不是处理线程同步, 而是进行**线程的阻塞**。如果在多线程程序中只使用条件变量无法实现线程的同步, 必须要配合互斥锁来使用。



一般情况下条件变量用于处理生产者和消费者模型，并且和互斥锁配合使用。条件变量类型对应的类型为`**pthread_cond_t**`，这样就可以定义一个条件变量类型的变量了：

```cmake
pthread_cond_t cond;
```

条件变量操作函数函数原型如下：

```cmake
#include <pthread.h>
pthread_cond_t cond;
// 初始化
int pthread_cond_init(pthread_cond_t *restrict cond,
      const pthread_condattr_t *restrict attr);
// 销毁释放资源        
int pthread_cond_destroy(pthread_cond_t *cond);
```

线程阻塞操作：

```cmake
// 线程阻塞函数, 哪个线程调用这个函数, 哪个线程就会被阻塞
int pthread_cond_wait(pthread_cond_t *restrict cond, pthread_mutex_t *restrict mutex);
```

通过函数原型可以看出，该函数在阻塞线程的时候，需要一个**互斥锁参数**，这个互斥锁主要功能是进行线程同步，让线程顺序进入临界区，避免出现数共享资源的数据混乱。该函数会对这个互斥锁做以下几件事情：

1. 在阻塞线程时候，如果线程已经对互斥锁`**mutex**`**上锁**，那么会将这把**锁打开**，这样做是为了避免死锁
2. 当线程解除阻塞的时候，函数内部会帮助这个线程**再次将这个mutex互斥锁锁上**，继续向下访问临界区

线程唤醒操作：

```cmake
// 唤醒阻塞在条件变量上的线程, 至少有一个被解除阻塞
int pthread_cond_signal(pthread_cond_t *cond);
// 唤醒阻塞在条件变量上的线程, 被阻塞的线程全部解除阻塞
int pthread_cond_broadcast(pthread_cond_t *cond);
```

调用上面两个函数中的任意一个，都可以换线被**pthread_cond_wait**或者**pthread_cond_timedwait**阻塞的线程，区别就在于pthread_cond_signal是**唤醒至少一个被阻塞的线程**（总个数不定），pthread_cond_broadcast是**唤醒所有被阻塞的线程**。

## 5.信号量

信号量是用在多线程多任务的情况下，我的理解是信号量（除了二元信号量也就是互斥变量）就是对资源的计数，当资源为0时，锁住所有消耗资源的线程，当资源计数满时，锁住所有生产资源的线程。信号量同样可以用于生产者和消费者模式之中。

信号的类型为`**sem_t**`对应的头文件为`**<semaphore.h>**`：

```cmake
#include <semaphore.h>
sem_t sem;
```

信号量操作原型：

```cmake
#include <semaphore.h>
// 初始化信号量/信号灯
int sem_init(sem_t *sem, int pshared, unsigned int value);
// 资源释放, 线程销毁之后调用这个函数即可
// 参数 sem 就是 sem_init() 的第一个参数            
int sem_destroy(sem_t *sem);
```

阻塞：

```cmake
// 参数 sem 就是 sem_init() 的第一个参数  
// 函数被调用sem中的资源就会被消耗1个, 资源数-1
int sem_wait(sem_t *sem);
```

唤醒：

```cmake
// 调用该函数给sem中的资源数+1
int sem_post(sem_t *sem);
```

获取资源计算：

```cmake
// 查看信号量 sem 中的整形数的当前值, 这个值会被写入到sval指针对应的内存中
// sval是一个传出参数
int sem_getvalue(sem_t *sem, int *sval);
```