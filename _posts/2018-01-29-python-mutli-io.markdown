---
layout:     post
title:      "python多进程和多线程性能比较&并发print错乱问题原因"
subtitle:   " \"顺便介绍了一下调试问题的工具\""
date:       2018-01-29 12:00:00
author:     "Xion"
header-img: "img/post-bg-window.jpg"
catalog: true
tags:
    - python
    - 并发问题
---


# 一、性能比较

## 利用pool创建多进程
实现一个简单的多进程的例子如下：

```
from multiprocessing import Pool    # < 1 >

WORKER = 10


def job(arg):                      # < 2 >
    print("do something, args:%s\n" % arg)


def p_map(func, iter, worker):        # < 3 >
    pool = Pool(WORKER)             # < 4 >
    pool.map(func, iter)              # < 5 >
    pool.close()                    # < 6 >
    pool.join()                     # < 7 >


def main():
    iter_obj = range(1, 20)   # < 8 >
    p_map(job, iter_obj, WORKER)


if __name__ == "__main__":
    main()  
```

1. 从multiprocessing中引入Pool类，用于创建进程池
2. 创建一个函数，该函数接受一个参数，做的事情只是打印这个参数
3. 创建一个函数用于封装躲多进程的操作
4. 创建一个数量为10个进程的进程池
5. map函数的用法与buttin的map函数用法相似，只是这里会让多个进程去执行map操作
6. 不再接受新的请求，这里必须先调用pool.close再调用pool.join()否则回报错
7. 等待所有的子进程结束
8. 构造一个参数，这个参数必须是一个可以便利的类型

## 利用pool创建多线程
在python中由于有GIL的存在，一般不推荐使用多线程的方法，因为多线程无法充分的利用CPU。
但是在密集的IO操作的情况下，使用多线程同样也能提高不少的执行效率。多线程的写法和多进程的写法如出一辙，这里就顺带一提了。


```
from multiprocessing.dummy import Pool as ThreadPool   # < 1 >
WORKER = 10


def job(arg):
    print("do something, args:%s\n" % arg)


def p_map(func, iter, worker):
    pool = ThreadPool(WORKER)      # < 2 >
    pool.map(func, iter)
    pool.close()
    pool.join()


def main():
    iter_obj = range(1, 20)
    p_map(job, iter_obj, WORKER)


if __name__ == "__main__":
    main()
```

1. 这里导入的Pool是从dummy中导入的，也就是线程池，为了区别把别名叫做ThreadPool
2. 从代码上来看，其他部分基本都是一样的

# 比较

将这个两段代码分别保存为`test_mult_process.py` 和 `test_mult_thread.py`。然后对他们的性能进行对比：

## 通过时间比较

比较代码如下：
```
from timeit import Timer


t1 = Timer("main()", "from test_mult_process import main")
t2 = Timer("main()", "from test_mult_thread import main")
time1 = t1.timeit(100)
time2 = t2.timeit(100)
print(time1, time2)
```
结果：
```
12.191851092000434 10.70985237900095
```

可以看出使用进程池的时候，开销相对大一些

## 看内部调用

为了效果的明显，把print删除，并且把循环的次数增加一下使用下面两条命令来查看调用情况

```
 strace -c python test_mult_process.py
 strace -c python test_mult_thread.py
```
### 线程
```
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 52.00    0.000013           0       101           close
 48.00    0.000012           0       532       431 open
  0.00    0.000000           0       159           read
  0.00    0.000000           0       159       119 stat
  0.00    0.000000           0       151           fstat
  0.00    0.000000           0         5           lstat
  0.00    0.000000           0         4           lseek
  0.00    0.000000           0        52           mmap
  0.00    0.000000           0        29           mprotect
  0.00    0.000000           0         8           munmap
  0.00    0.000000           0        11           brk
  0.00    0.000000           0        68           rt_sigaction
  0.00    0.000000           0         1           rt_sigprocmask
  0.00    0.000000           0         5         1 ioctl
  0.00    0.000000           0         8         8 access
  0.00    0.000000           0        13           clone
  0.00    0.000000           0         1           execve
  0.00    0.000000           0         2           fcntl
  0.00    0.000000           0         4           getdents
  0.00    0.000000           0         2           getcwd
  0.00    0.000000           0         4         2 readlink
  0.00    0.000000           0         2           getrlimit
  0.00    0.000000           0         1           sysinfo
  0.00    0.000000           0         1           getuid
  0.00    0.000000           0         1           getgid
  0.00    0.000000           0         1           geteuid
  0.00    0.000000           0         1           getegid
  0.00    0.000000           0         1           arch_prctl
  0.00    0.000000           0        77        14 futex
  0.00    0.000000           0         1           set_tid_address
  0.00    0.000000           0         1           set_robust_list
------ ----------- ----------- --------- --------- ----------------
100.00    0.000025                  1406       575 total

```
### 多进程
```
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
100.00    0.000111           9        13           clone
  0.00    0.000000           0       207           read
  0.00    0.000000           0         5           write
  0.00    0.000000           0       698       564 open
  0.00    0.000000           0       138           close
  0.00    0.000000           0       192       152 stat
  0.00    0.000000           0       200           fstat
  0.00    0.000000           0         9         4 lstat
  0.00    0.000000           0         4           lseek
  0.00    0.000000           0        62           mmap
  0.00    0.000000           0        27           mprotect
  0.00    0.000000           0        17           munmap
  0.00    0.000000           0        14           brk
  0.00    0.000000           0        68           rt_sigaction
  0.00    0.000000           0         1           rt_sigprocmask
  0.00    0.000000           0         5         1 ioctl
  0.00    0.000000           0        10        10 access
  0.00    0.000000           0         2           pipe
  0.00    0.000000           0         1           execve
  0.00    0.000000           0        56         1 wait4
  0.00    0.000000           0         2           fcntl
  0.00    0.000000           0         4           getdents
  0.00    0.000000           0         2           getcwd
  0.00    0.000000           0         4           link
  0.00    0.000000           0         8           unlink
  0.00    0.000000           0         4         2 readlink
  0.00    0.000000           0         2           getrlimit
  0.00    0.000000           0         1           sysinfo
  0.00    0.000000           0         1           getuid
  0.00    0.000000           0         1           getgid
  0.00    0.000000           0         1           geteuid
  0.00    0.000000           0         1           getegid
  0.00    0.000000           0         1           statfs
  0.00    0.000000           0         1           arch_prctl
  0.00    0.000000           0        47         6 futex
  0.00    0.000000           0         1           set_tid_address
  0.00    0.000000           0         1           set_robust_list
------ ----------- ----------- --------- --------- ----------------
100.00    0.000111                  1811       740 total

```
可以看出多线程的开销主要是在open和close两个系统调用，而多进程的开销主要是在clone这个系统调用上。

虽然多进程的创建比多线程的开销大，但是多进程的执行效率更高。上面的结果虽然显示多线程的时间和系统调用都更少，但是当同时有很多并发的时候，多线程的执行效率并不高。

# 二、print错乱问题

## 复现

使用第一节中的代码就能够复现在print错乱的问题，例子如下：
```
test mult thread 1
test mult thread 2
 test mult thread 3
test mult thread 4
 test mult thread 5
test mult thread 6
test mult thread 7
 test mult thread 9test mult thread 11

test mult thread 12
test mult thread 10
test mult thread 13
 test mult thread 15test mult thread 14

test mult thread 16

```

打印到console的日志，有的发生了错行，有强迫症的同学一定受不了。

然后我用多进程执行了相同的代码，执行结果如下：
```
test mult process 1
test mult process 2
test mult process 3
test mult process 4
test mult process 5
test mult process 6
test mult process 7
test mult process 8
test mult process 9
test mult process 11
test mult process 12
test mult process 13
test mult process 15
test mult process 16

```
打印的内容没有问题

## 原因

### 原子操作

在多线程中，所有线程共用一个buffer，只要在print的时候发生了线程切换，就会导致出现打印混乱的。也就是说print xxx这个表达并不是一个python的原子操作！
为了证实这一想法，可以使用dis模块来将一个只有print的函数，转化为python的字节码。

代码如下

```
import dis

def bar():
    print 'helloword'

dis.dis(bar)
```

输出如下：
```
2           0 LOAD_CONST               1 ('helloworld')
            3 PRINT_ITEM          
            4 PRINT_NEWLINE       
            5 LOAD_CONST               0 (None)
            8 RETURN_VALUE
```

通过上马的结果可以看出，print被转化成来两句，PRINT_ITEM和PRINT_NEWLINE，所以print xxx这个表达不是原子(并不是说只有一句就是原子的，而是有两句一定不是原子的)的。
### 缓存

可是为什么多进程没有问题呢？

其实只是刚好没发现问题，由于console是行缓冲的(缓冲区遇到\n或者缓冲区满才会打印出来)。而在函数中正好打印的是一行，一行满了刚好打出来，
所以没有相互交错的错乱问题，可以通过下面的代码证实这个猜想：


```
from multiprocessing import Pool
WORKER = 10


def job(arg):
    print("do something, args:%s\n\n\n" % arg)


def p_map(func, iter, worker):
    pool = Pool(WORKER)      
    pool.map(func, iter)
    pool.close()
    pool.join()


def main():
    iter_obj = range(1, 20)
    p_map(job, iter_obj, WORKER)


if __name__ == "__main__":
    main()
```

上面的代码输出如下：

```
do something, args:1


do something, args:2


do something, args:3


do something, args:4


do something, args:7

do something, args:5

do something, args:9



do something, args:10


do something, args:6


do something, args:11


do something, args:12

```

在打印来多行之后，一样会出现错乱的情况

# 解决方法

## 不用python的换行，而是自己写换行

将`print "xxx"`写为`print "xxx\n",`

差异如下
```
>>> def bar():
...     print "abc"
...
>>> def foo():
...     print "abc\n",
...
>>> dis.dis(foo)
  2           0 LOAD_CONST               1 ('abc\n')
              3 PRINT_ITEM          
              4 LOAD_CONST               0 (None)
              7 RETURN_VALUE        
>>> dis.dis(bar)
  2           0 LOAD_CONST               1 ('abc')
              3 PRINT_ITEM          
              4 PRINT_NEWLINE       
              5 LOAD_CONST               0 (None)
              8 RETURN_VALUE        
>>>
```

但是实际上，虽然此处只有一条PRINT_ITEM，着一条是否是原子的还是在于python本身的实现。

此处有print方法的具体实现：

https://github.com/python/cpython.git

但是在这里就不不展开讲解了。

有一个简单的测试方法：将缓冲关闭，并且查看这一句调用的时候有多少个write的调用。

因为write这个系统调用是原子的，可以粗略的估计这个操作是否是原子的。

结果如下：
```
➜  test strace -c python -u -c "print 123"     
123
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
  0.00    0.000000           0        95           read
  0.00    0.000000           0         2           write
  0.00    0.000000           0       231       168 open
  0.00    0.000000           0        63           close
  0.00    0.000000           0        86        59 stat
  0.00    0.000000           0        94           fstat
  0.00    0.000000           0         4           lstat
  0.00    0.000000           0        34           mmap
  0.00    0.000000           0        14           mprotect
  0.00    0.000000           0         8           munmap
  0.00    0.000000           0        10           brk
  0.00    0.000000           0        68           rt_sigaction
  0.00    0.000000           0         1           rt_sigprocmask
  0.00    0.000000           0         4           ioctl
  0.00    0.000000           0         8         8 access
  0.00    0.000000           0         1           execve
  0.00    0.000000           0         4           getdents
  0.00    0.000000           0         3         1 readlink
  0.00    0.000000           0         1           getrlimit
  0.00    0.000000           0         1           sysinfo
  0.00    0.000000           0         1           getuid
  0.00    0.000000           0         1           getgid
  0.00    0.000000           0         1           geteuid
  0.00    0.000000           0         1           getegid
  0.00    0.000000           0         1           arch_prctl
  0.00    0.000000           0         1           set_tid_address
  0.00    0.000000           0         1           set_robust_list
------ ----------- ----------- --------- --------- ----------------
100.00    0.000000                   739       236 total

➜  test strace -c python -u -c "print '123\n',"
123
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
  0.00    0.000000           0        95           read
  0.00    0.000000           0         1           write
  0.00    0.000000           0       231       168 open
  0.00    0.000000           0        63           close
  0.00    0.000000           0        86        59 stat
  0.00    0.000000           0        94           fstat
  0.00    0.000000           0         4           lstat
  0.00    0.000000           0        34           mmap
  0.00    0.000000           0        14           mprotect
  0.00    0.000000           0         8           munmap
  0.00    0.000000           0        10           brk
  0.00    0.000000           0        68           rt_sigaction
  0.00    0.000000           0         1           rt_sigprocmask
  0.00    0.000000           0         4           ioctl
  0.00    0.000000           0         8         8 access
  0.00    0.000000           0         1           execve
  0.00    0.000000           0         4           getdents
  0.00    0.000000           0         3         1 readlink
  0.00    0.000000           0         1           getrlimit
  0.00    0.000000           0         1           sysinfo
  0.00    0.000000           0         1           getuid
  0.00    0.000000           0         1           getgid
  0.00    0.000000           0         1           geteuid
  0.00    0.000000           0         1           getegid
  0.00    0.000000           0         1           arch_prctl
  0.00    0.000000           0         1           set_tid_address
  0.00    0.000000           0         1           set_robust_list
------ ----------- ----------- --------- --------- ----------------
100.00    0.000000                   738       236 total

```
基本能证实上面的猜想

## 使用sys.stdout.write()

`sys.stdout`是一个文件描述符[3],它指向当期进程的标准输出，一般而言就是指向console。但是
在unix的世界观下，一切都是文件，console也是个文件。这里调用write就是调用的文件的write，查看一些资料这里
的写是原子的(由于GIL的存在)。

## 使用python3

python3 在很多地方已经有了不小的改动，并且在2020年python2会停止维护。很多在python2中出现的
问题，在python3中已经没有来。

使用python3运行一样的代码，将print写成函数的形式，就已经不会有打印错乱的现象了。

# 总结

1. 介绍了使用进程池和线程持来进行并发的方法，对比了两者的性能。python中多线程开销小，但是执行的效率没有多进程高。
2. 示范了timeit，strace的例子。
3. 分析来print错乱的原因。
4. 给出了三种解决方法。



# 注释
[1] 缓存和缓冲这两个概念虽然是截然不同的，但是由于中文长得有点像。有点容易搞混，但是只要记住writer-buffer和read-cache就能清晰的区别他们
