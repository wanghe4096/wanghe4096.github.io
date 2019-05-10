---
title: 基础数据结构--链表
date: 2019-1-14
---
# 介绍
这篇文章重点在于描述一个链表的设计和实现
## 链表的数据结构
- <font color="green"> **单链表** </font>

    最简单的链表。 元素之间由一个单独的指针链接。 这种结构的链表允许从第一个元素开始遍历到最后一个元素。


- <font color="green"> **双向链表** </font>

    这种形式的链表元素之间通过两个指针链接。 双向链表可以正向遍历， 也可以反向遍历。 

- <font color="green"> **循环链表** </font>

    这种形式的链表最后个元素的指针指向链表的首元素， 而不是设置为NULL。 这种结构的链表允许循环遍历。 


## 链表的应用场景：
- <font color="green"> **滚动列表** </font>

    图形用户界面中常能看到这种组件。 通常在滚动列表中与每个条目相关的数据并不显示出来。 一种管理这种“隐藏”数据的方式是维护一个链表， 链表中每个元素保存一个在滚动列表中出现的条目数据。

- <font color="green"> **多项式计算**</font>

    数学中的一个重要组成部分。 但对于大多数编程语言来说，并没有将其作为原生的数据类型加以支持。如果让链表中的每个元素存储多项式中的一项， 那么链表对于表示一个多项式是非常有用的,比如 $3x^2+2x+1$

- <font color="green">**内存管理**</font>

    内存管理是操作系统中一个重要组成部分。 操作系统必须决定如何为系统中运行的进程分配和回收存储空间。 链表能够用来跟踪可供分配的内存片段信息。 

- <font color="green">**文件的链式分配**</font>
     
    为了消除磁盘上的外部碎片而采用的一种文件分配方式， 但只适合于顺序访问。 文件按照块的方式进行组织， 每一块都包含一个指向文件下一块数据指针。 

- <font color="green">**其他数据结构**</font>

    有一些数据结构的实现需要依赖于链表， 它们是栈、队列、集合、哈希表和图。 

<!-- more -->


## 链表描述


# C语言实现

## 链表的抽象数据类型头文件

```c

#ifndef LIST_H
#define LIST_H
#include<stdlib.h>

/* Define a structure for linked list elements. */ 

typedef struct ListElmt_ 
{
	void *data;
	struct ListElmt_ *next; 
}ListElmt;


/* Defin a structure for linked list. */

typedef struct List_ 
{
	int size; 
	int (*match)(const void *key1, const void*key2); 
	void (*destroy)(void *data);

	ListElmt *head; 
	ListElmt *tail;

}List;

/* Public Interface */
void list_init(List *list, void (*destroy)(void *data)); 
void list_destroy(List *list);
int list_ins_next(List *list, ListElmt *element, const void *data);
int list_rem_next(List *list, ListElmt *element, void **data);
#define list_size(list) ((list)->size)

#define list_head(list) ((list)->head)
#define list_tail(list) ((list)->tail)
#define list_is_head(list, element) ((element) == (list)->head ? 1 : 0)
#define list_is_tail(list, element) ((element) == (list)->tail ? 1: 0 ) 
#define list_data(element) ((element)->data)
#define list_next(element) ((element)->next)


#endif

```

## 链表的抽象数据类型的实现

```c
#include<stdio.h>
#include<stdlib.h>
#include<string.h>
#include "list.h"

/* list_init */ 

void list_init(List *list, void (*destroy)(void *data))
{
	/* Initilize the list. */ 
	list->size = 0; 
	list->destroy = destroy; 
	list->head = NULL;
	list->tail = NULL;
	return ;
}


/* list_destroy */
void list_destroy(List *list) 
{
	void *data; 

	/* Remove each element. */
	while(list_size(list)>0) 
	{
		if (list_rem_next(list, NULL, (void **)&data) == 0 && list->destroy != NULL)
		{
			/* Call a user-defined function to free dynamically allocated data */
			list->destroy(data);
		}
	}

	/* No operations are allowed now, but clear the structure as precaution. */
	memset(list, 0, sizeof(List));
	return ; 
}

/*
 * list_int_next
 **/ 
int list_int_next(List *list, ListElmt *element, const void *data)
{
	ListElmt *new_element = NULL; 

	/* Allocate storage for the element. */ 
	if ((new_element = (ListElmt *)malloc(sizeof(ListElmt))) == NULL) 
	{
		return -1;
	}

	/* Insert the element into the list. */
	new_element->data = (void *)data; 

	if (element == NULL) {
		/* Handle insertion at the head of the list. */ 
		if (list_size(list) == 0) {
			// 只有一个节点的情况下， 头节点=尾节点，当前节点的下一个节点也是头节点
			list->tail = new_element;
			new_element->next = list->head; 
			list->head = new_element; 
		}

	}else {
		/* Handle insertion somewhere other than at the head */
		if (element->next == NULL) {
			list->tail = new_element;
		}

		new_element->next = element->next;
		element->next = new_element;
	}

	/* Adjust the size of the list to acount for the inserted element. */
	list->size ++ ;
	return 0;
}

/*
 * list_rem_next
 */
int list_rem_next(List *list, ListElmt *element, void **data) 
{
	ListElmt *old_element = NULL; 
	/* Do not allow removal from an empty list. */ 
	if (list_size(list) == 0) {
		return -1;
	}

	/* Remove the element from the list. */
	if (element == NULL) {
		/* Handle removal from the head of the list. */
		*data = list->head->data; 
		old_element = list->head; 
		list->head = list->head->next;

		if (list_size(list) == 1){
			list->tail = NULL;
		}

	}else {
		/* Handle removal from somewhere other than the head. */ 
		if (element->next == NULL) {
			return  -1;
		}

		*data = element->next->data;
		old_element = element->next; 
		element->next = element->next->next;

		if (element->next == NULL){
			list->tail = element;
		}
	}

	/* Free the storage allocated by the abstract datatype. */ 
	free(old_element);

	/* Adjust the size of the list to account for the removed element.*/ 
	list->size--; 
	return 0;
}

```

# Go语言实现

```golang
package list

// Node  链表节点
type Node struct {
	Data interface{}
	Next *Node

}

// LinkList 链表定义
type LinkList struct {
	Size int
	Head *Node
	Tail *Node
}

// New 初始链表
func New() *LinkList {
	return &LinkList{Size: 0, Head: nil, Tail:nil}
}

// InsertNode 插入节点
func (linklist *LinkList) InsertNode(prevNode *Node, data interface{}) (err error) {


	newNode := &Node{Data:data}

	if prevNode == Nil {
		if linklist.Size == 0 {
			linklist.Tail = newNode
			newNode.Next = linklist.Head
			linklist.Head = newNode
		}
	}else {
		if prevNode.Next == Nil {
			linklist.Tail = newNode
		}
		newNode.Next = prevNode.Next
		prevNode.Next = newNode
	}
	linklist.Size ++

	return
}

// RemoveNode 移除节点
func (linklist *LinkList) RemoveNode(prevNode *Node, data interface{})  {
	if linklist.Size == 0 {
		return
	}


	// 如果前一个节点是空的， 删除头节点
	if prevNode == Nil {
		data = linklist.Head.Data
		linklist.Head = linklist.Head.Next

		if linklist.Size == 1 {
			linklist.Tail = nil
		}

	}else {
		// 如果前一个节点是空的
		if prevNode == nil {
			return
		}

		// 前一个节点不是空的
		data = prevNode.Next.Data
		prevNode.Next = prevNode.Next.Next

		// 当前节点是空的
		if prevNode.Next == nil {
			linklist.Tail = prevNode
		}
		linklist.Size--

	}

	return



}

```