---
layout:     post
title:      "C++ 用critical_section 代码临界区模拟信号量，解决生产者消费者的问题"
date:       2014-10-26 21:36:00 +0800
author:     "DongYeo"
header-img: "img/post-bg-02.jpg"
tags: ["C++"]
---
**注：本篇为老博客平台系统迁移过来的**

PS.不要问我为什么不直接用CRITICAL_SECTION，我只是突然脑洞大开，想自己写个信号量的操作函数。
## 问题描述：
生产者消费者的问题描述了两个共享固定大小的线程——即所谓的“生产者”和“消费者”——在实际运行时会发生的问题。生产者的主要作用是生成一定量的数据放到缓冲区中，然后重复此过程。与此同时，消费者也在缓冲区消耗这些数据。该问题的关键就是要保证生产者不会在缓冲区满时加入数据，消费者也不会在缓冲区中空时消耗数据。

## 函数描述：
```c++
producer:
while(count==n) do no-op;//内存满，不操作
wait(S);
produce an item into buffer...
in=(in+1)%n;
counter++;
signal(S);
consumer:
while(count==0) do no-op;
wait(S);
consume an item from buffer;
out=(out+1)%n;
count--;
signal(S);
```

实现目标，通过CRITICAL_SECTION 代码临界区模拟实现信号量的wait，和signal功能

wait(S)以及signal(S)函数：
在操作系统中，wait和signal是一对信号量的操作函数，他们都具有原子性。
操作描述：

```
	wait(S):
	while(S<=0) do no-op;
	S:=S-1;

	signal(S):
	S:=S+1;
```

这两个函数都是源自操作，因此，他们在执行是是不可中断的。亦即，当一个函数在进程中修改某个信号量时，是没有其他进程可同时对该信号量进行操作。此外，在wait函数中的对S的测试和等待是不可中断的。

关于CRITICAL_SECTION这里就不在多说，主要介绍几个函数
CRITICAL_SECTION cs;
initializeCriticalSection(&cs);//初始化一个临界资源
enterCriticalSection(&cs);//进入临界资源
leaveCriticalSection(&cs);//退出临界资源

直接上代码

```c++
#include <iostream>
#include <windows.h>
using namespace std;


DWORD WINAPI producer(LPVOID lpParameter);//生产者的函数
DWORD WINAPI consumer(LPVOID lpParameter);//消费者函数
void waitSemaphore(int &semephore);
void releaseSemaphore(int &semaphore);

int n=10;
int in=0;
int out=0;
int *buffer=new int[n];//用数组来模拟资源
int counter=0;
CRITICAL_SECTION cs;
CRITICAL_SECTION cs1;
int mutex=1;
int empty=1;
int full=0;
void main()
{
	for(int i=0;i<n;i++)
	{
		buffer[i]=0;
	}
	InitializeCriticalSection(&cs);
	InitializeCriticalSection(&cs1);
	HANDLE Thread1;
	HANDLE Thread2;
	HANDLE Thread3;
	Thread1=CreateThread(NULL,0,producer,NULL,0,NULL);
	Thread2=CreateThread(NULL,0,consumer,NULL,0,NULL);
	Thread3=CreateThread(NULL,0,producer,NULL,0,NULL);

	Sleep(40000);

	CloseHandle(Thread1);
	CloseHandle(Thread2);
	CloseHandle(Thread3);
	DeleteCriticalSection(&cs);
	DeleteCriticalSection(&cs1);
}
void waitSemaphore(int &semaphore)
{
	EnterCriticalSection(&cs);
	while(semaphore==0)
	{
		NULL;
	}
	semaphore=0;
	LeaveCriticalSection(&cs);
}
void releaseSemaphore(int &semaphore)
{
	EnterCriticalSection(&cs1);
	semaphore=1;
	LeaveCriticalSection(&cs1);
}
DWORD WINAPI producer(LPVOID lpParameter)
{
	while(true)
	{

		while(counter==n)
		{
			NULL;
		}
		waitSemaphore(mutex);
		Sleep(3000);
		int nextp=rand()%10+1;
		buffer[in]=nextp;
		in=(in+1)%n;
		counter=counter+1;
		cout<<"生产:"<<endl;
		cout<<"counter:"<<counter<<endl;
		for(int i=0;i<n;i++)
		{
			cout<<buffer[i]<<" ";
		}
		cout<<endl;
		releaseSemaphore(mutex);
	}
	return 0;
}
DWORD WINAPI consumer(LPVOID lpParameter)
{
	while(true)
	{
		//waitSemaphore(full);
		while(counter==0)
		{
			NULL;
		}
		waitSemaphore(mutex);
		Sleep(3000);
		int nextc=buffer[out];
		buffer[out]=0;
		out=(out+1)%n;
		counter=counter-1;
		cout<<"消费:"<<endl;
		cout<<"counter:"<<counter<<endl;
		cout<<"netxc:"<<nextc<<endl;
		for(int i=0;i<n;i++)
		{
			cout<<buffer[i]<<" ";
		}
		cout<<endl;
		//releaseSemaphore(empty);
	}
	return 0;
}
```
