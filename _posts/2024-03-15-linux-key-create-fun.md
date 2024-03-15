---
layout: post
title: Linux多线程之pthread_key_create
tags: Linux多线程
---

#### 一、说明

**线程特定数据（TSD）**：pthread_key_create()
			每个线程都有自己独立的数据副本，线程可以访问和修改这些数据，而不影响其他线程的数据。

**应用场景：**
线程上下文管理：有些情况下，我们需要在线程执行过程中保存一些临时的上下文信息，以便在后续的函数调用中使用。通过pthread_key_create()函数创建线程特定数据的键，并在每个线程中保存这些临时上下文信息。这种方式比通过函数参数传递数据更简洁方便，同时避免了函数参数的传递和维护成本。

*需要注意的是，pthread_key_create()函数用于线程内的数据访问，对于多个线程之间需要共享的数据，仍然需要使用其他线程同步机制，如互斥锁或信号量等。

#### 二、参考链接

https://blog.csdn.net/Lid_23/article/details/132036106

#### 三、示例代码

##### 

```
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
 
// 定义线程特定数据键
pthread_key_t key;
 
// 析构函数
void destructor_function(void* data) {
    printf("Free thread-specific data\n");
    free(data);
}
 
void print_threadID()
{
    int* data = pthread_getspecific(key);
    printf("Print Thread ID: %u\n", *data);
}
 
// 线程函数
void* thread_func(void* arg) {
    // 获取线程特定数据
    void* data = pthread_getspecific(key);
    if (data == NULL) {
        // 初始化线程特定数据
        data = malloc(sizeof(int));
        *(int*)data = pthread_self();
        printf("Thread ID: %u\n", pthread_self());
        pthread_setspecific(key, data);
    }
 
    // 打印线程特定数据
    print_threadID();
 
    return NULL;
}
 
int main() {
    // 创建线程特定数据键
    pthread_key_create(&key, destructor_function);
 
    // 创建两个线程
    pthread_t thread1, thread2;
    pthread_create(&thread1, NULL, thread_func, NULL);
    pthread_create(&thread2, NULL, thread_func, NULL);
 
    // 等待线程结束
    pthread_join(thread1, NULL);
    pthread_join(thread2, NULL);
 
    return 0;
}

```

