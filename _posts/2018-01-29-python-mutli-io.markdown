---
layout:     post
title:      "一个python并发io的问题原因和解决方法"
subtitle:   " \"一个python并发io问题\""
date:       2018-01-29 12:00:00
author:     "Xion"
header-img: "img/post-bg-window.jpg"
catalog: true
tags:
    - python
    - 并发问题
---


# 利用Pool创建多进程任务

实现一个简单的多进程的例子如下：

```
from multiprocessing import Pool    # < 1 >
WORKER = 10

def job(arg):                      # < 2 >
  print("do something, args:%s\n" % arg)

def p_map(func,iter,worker):        # < 3 >
    pool = Pool(WORKER)             # < 4 >
    pool.map(run,iter)              # < 5 >
    pool.close()                    # < 6 >
    pool.join()                     # < 7 >

if __name__ == "__main__":
  iter_obj = range(1,20)   # < 8 >
  p_map(job, iter_obj, WORKER)
```

1. 从multiprocessing中引入Pool类，用于创建进程池
2. 创建一个函数，该函数接受一个参数，做的事情只是打印这个参数
3. 创建一个函数用于封装躲多进程的操作
4. 创建一个数量为10个进程的进程池
5. map函数的用法与buttin的map函数用法相似，只是这里会让多个进程去执行map操作
6. 不再接受新的请求，这里必须先调用pool.close再调用pool.join()否则回报错
7. 等待所有的子进程结束
8. 构造一个参数，这个参数必须是一个可以便利的类型

# 利用pool创建多线程
在python中由于有GIL的存在，一般不推荐使用多线程的方法，因为多线程无法充分的利用CPU。
但是在密集的IO操作的情况下，使用多线程同样也能提高不少的执行效率。多线程的写法和多进程的写法如出一辙，这里就顺带一提了。


```
from multiprocessing.dummy import Pool as ThreadPool   # < 1 >
WORKER = 10

def job(arg):
  print("do something, args:%s\n" % arg)

def p_map(func,iter,worker):        
    pool = ThreadPool(WORKER)      # < 2 >
    pool.map(run,iter)            
    pool.close()                 
    pool.join()                 

if __name__ == "__main__":
  iter_obj = range(1,20)   
  p_map(job, iter_obj, WORKER)
```

1. 这里导入的Pool是从dummy中导入的，也就是线程池，为了却别把别名叫做ThreadPool
2. 从代码上来看，其他部分基本都是一样的

# 比较

将这个两段代码分别保存为`test_mult_process.py` 和 `test_mult_thread.py`。然后对他们的性能进行对比：

##
