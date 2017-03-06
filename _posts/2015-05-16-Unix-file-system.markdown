---
layout:     post
title:      "Unix 类文件系统模拟"
date:       2015-05-16 21:56:00
author:     "DongYeo"
header-img: "img/post-bg-02.jpg"
tags: ["C++","Linux"]
---
**注：本篇为老博客平台系统迁移过来的**
---
# 项目背景
完成一个 UNIX文件系统的子集的模拟实现。
1. 完成文件卷结构设计
2. I节点结构设计
3. 目录结构
4. 用户及组结构
5. 文件树结构
6. 实现功能如下命令:
>Ls				显示文件目录
Chmod			改变文件权限
Chown			改变文件拥有者
Chgrp			改变文件所属组
Pwd				显示当前目录
Cd				改变当前目录
Mkdir			创建子目录
Rmdir			删除子目录
Umask			文件创建屏蔽码
Mv				改变文件名
Cp				文件拷贝
Rm				文件删除
Ln           	建立文件联接
Cat				连接显示文件内容
Passwd			修改用户口令

---
# 开发环境
操作系统	：windows 7 64位
开发工具	：visual studio 2010
程序类型	：win32 控制台应用程序

---
# 分析
## 文件系统特特性
文件数据除了文件的实际内容外，通常还包含很多额外的属性，例如Linux操作系统的文件权限（RWX）与晚间属性（所有者、族群、时间参数等）。文件系统通常会将这两部分数据分别存放在不同的块，权限与属性放置到Inode中，至于时间数据则放置到Data Block中。另外，还有一个超级块 Superblock 会记录整个文件系统的整体信息，包括inode和block的总量、使用量、剩余量。

- super block：记录此文件系统的整体信息，包括inode/block 的总量，使用情况、剩余量。以及文件系统的格式相关信息等；
- inode：记录文件的属性，一个文件占用一个inode,同时记录此文件的数据所在的block的号码；
- block：实际记录文件的内容，若文件太大的时候，会占用多个block

---
## 盘块的管理
### 位示图法
位示图是利用二级制的以为来表示磁盘的中有个盘块的使用情况。当其值为“0”时，表示对应的盘块空闲；为“1”的时，表示已经分配。
### 成组链接法
在UNIX系统中，将空闲块分成若干组，每100个空闲块为一组，每组的第一空闲块登记了下一组空闲块的物理盘块号和空闲块总数。如果一个组的第二个空闲块号等于0，则有特殊的含义，意味着该组是最后一组，即无下一个空闲块。

分配空闲块的时候，从前往后分配，先从第一组开始分配，第一组空闲的100块分完了，才进入第二组。

释放空闲块的时候正好相反，从后往前分配，先将释放的空闲块放到第一组，第一组满了，在第一组前再开辟一组，之前的第一组变成第二组。

---
# 设计
## 系统流程
![系统流程图](http://img.blog.csdn.net/20150516224626333)

程序开始执行后，程序会先将资源文件读取，并以一文件指针保存在全局变量virtualDisk中。
读取文件成功以后，通过文件指针访问超级块所在盘块，读取内容保存在全局变量super中。
加载超级块以后，程序通过iget操作获取到root节点，保存在全局变量root中。
接下来，进入登录界面，用户输入密码和口令，程序根据用户的输入，读取用户文件，判断是否登录成功，成功以后进入程序的主界面。

---
## 文件卷设计
![这里写图片描述](http://img.blog.csdn.net/20150516225008322)

本程序的模拟磁盘的大小是8MB，每个盘块的大小设定为1KB，共8192个盘块。其中0号盘块在本程序中没有使用，但保留了下来。1号盘块保存超级块的信息。2-911盘块保存的是finode节点的信息。从912盘块开始，都是存储文件内容的盘块，使用成组链接法来管理，每组的盘块数是20。关于结点等数据结构会在下面部分详细介绍。

---
## 数据结构设计
### 超级快数据结构
1#块为超级块（superblock）。磁盘的回收和索引结点的分配与回收，将涉及到超级块。超级块是专门用于记录文件系统中盘块和磁盘索引节点使用情况的一个盘块，其中含有以下各个字段：

1. size	：文件系统的盘块数。
2. freeBlock：空闲盘块号栈，即用于记录当前可用的空闲盘块编号的栈。
3. nextFreeBlock：当前空闲盘快号数，即在空闲盘块号栈中保存的空闲盘块号的数目。他也可以被视为空闲盘块号栈的指针。
4. freeInode：空闲磁盘i结点编号栈，即记录了当前可用的素偶皮空闲结点编号的栈。
5. nextFreeInode：空闲磁盘i结点数目，指在磁盘i结点栈中保存的空闲i结点编号的数目，也可以视为当前空闲i结点栈顶的指针。
6. freeBlockNum：空闲盘块数，用于记录整个文件系统中未被分配的盘块个数。
7. freeInodeNum：空闲i结点个数，用于记录整个文件系统中未被分配的节点个数。
8. lastLogin：上次登录时间

```c++
struct supblock
{
	unsigned int size;					//the size of the disk
	unsigned int freeBlock[BLOCKNUM];	//the stack of the free block
	unsigned int	nextFreeBlock;		//the pointer of the next free block in the stack
	unsigned int freeBlockNum;			//the totally number of the free block in the disk
	unsigned int freeInode[INODENUM];	//the stack of the free node
	unsigned int freeInodeNum;			//the totally number of the free inode in the disk
	unsigned int nextFreeInode;			//the next free inode in the stack
	unsigned int lastLogin;
};
```

---
### 结点数据结构

关于索引结点，本程序的主要由两种结构的定义，分别是内存索引结点和磁盘索引结点。
磁盘索引结点：
1. mode		：文件类型及属性
2. fileSize		：文件大小
3. fileLink		：文件连接数
4. owner		：所属用户名
5. group		：所属用户组
6. modifyTime	：修改时间
7. createTime	：创建时间
8. addr		：盘块地址数组

```c++
truct finode
{
	int				mode;
	long		int	fileSize;
	int				fileLink;
	char			owner[MAXNAME];
	char			group[GROUPNAME];
	long		int	modifyTime;
	long		int	createTime;
	int				addr[6];
	char			black[45];				//留空，以备内容扩充时不会影响结构大小
};
```
内存索引结点:
内存索引结点是保存在内存中索引结点的数据结构，当文件第一次被打开时，文件的索引结点从模拟磁盘上读出，并保存在内存中，方便下一次文件的打开。
1. finode：磁盘索引结点结构，保存从磁盘读出的索引结点信息
2. parent：父级内存索引结点指针
3. inodeID：索引结点号

```c++
struct inode
{
	struct						finode finode;
	struct						inode *parent;
	unsigned short int		    inodeID;				//the node id
	int							userCount;					//the number of process using the inode
};
```

---

### 文件目录项：
文件目录项由文件名和文件索引结点号组成。
1. directName：文件名或目录名
2. inodeID：文件索引结点号

```c++
struct direct
{
	char					directName[DIRECTNAME];
	unsigned short int	    inodeID;
};
```
---
### 目录数据结构
目录结构：
1. dirNum：目录数目
2. direct：目录项数组

```c++
struct dir
{
	int		dirNum;
	struct	direct direct[DIRNUM];
};
```
对于目录类，它的内容都是以dir结构保存在磁盘中的，并以dir结构读取。

---

# 代码实现

## 主函数

主函数中，最最核心的函数就是dispatch函数，它解析用户输入的指令，并解析出参数，调用用户需要的函数。其流程图如下：
![这里写图片描述](http://img.blog.csdn.net/20150516234314015)

---
## 核心函数
一下函数为整一个文件系统最最核心的功能，所有的操作都是建立在一下函数的基础上进行的：
1. 盘块读函数
			int bread(void * _Buf,unsigned short int bno,long int offset,int size,int count=1)
		该函数将指定的盘块号内容的读取到对应的数据结构中。

2. 盘块写函数
			int bwrite(void * _Buf,unsigned short int bno,long int offset,int size,int count=1)

3. 结点分配函数
			struct inode* ialloc()
		该函数的作用是为从超级块的空闲结点栈中取出一个新的结点，并	出示化该结点。其流程图如图所示。
![这里写图片描述](http://img.blog.csdn.net/20150516235211674)

4. 盘块分配函数
			int balloc()
该函数的主要功能是从超级快的空闲盘块栈中取出一个空闲盘块号，若栈只剩一个空闲盘块，那么采用成组链接法，读取下一组的空闲盘块栈。函数流程图如图所示。
![这里写图片描述](http://img.blog.csdn.net/20150516235233697)

5. 盘块回收函数
			int bfree(int bno)
该函数的主要功能是回收空闲盘块，若超级块中的空闲盘块栈未满，则回收盘块号入栈，若空闲盘块栈满，则将栈内容写到新回收的盘块上，清空栈，并将新的盘块号入栈。流程图如图。
![这里写图片描述](http://img.blog.csdn.net/20150516235258517)

# 代码实现
[点我点我](https://github.com/linhuahua/FileSystem.git)
