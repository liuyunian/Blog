# 栈
栈可以理解为一种受限的线性表，只能在表尾插入和删除元素。  
栈有线性表的特点：每个元素都有且之后一个前驱节点和一个后继节点（第一个节点只有后继节点，最后一个节点只有前驱节点）  
同时也有自身的特点：后入先出

## 术语
栈顶：允许插入和删除元素的一端  
栈底：与栈顶相对的一端  
入栈：插入数据操作
出栈：删除数据操作

## 抽象数据类型
```
ADT 栈(stack)
Data
	同线性表。元素具有相同的类型，相邻元素具有前驱和后继关系
Operation
	InitStack(*S)       // 初始化操作，建立一个空栈S
	DestroyStack(*S)    // 若栈存在，则销毁它
	ClearStack(*S)      // 将栈清空
	StackEmpty(S)       // 若栈为空返回true,否则返回false
	GetTop(S, *e)       // 若栈存在且非空，用e返回S的栈顶元素
	Push(*S, e)         // 若栈S存在，插入新元素e到栈S中并成为栈顶元素
	Pop(*S, *e)         // 删除S中栈顶元素，并用e返回其值
	StackLength(S)      // 返回栈S的元素个数
endADT
```

## 存储结构
同线性表一样，有两种存储结构：顺序存储结构和链式存储结构  
栈较动态数组(Deque)、链表(List)来说属于高级容器，它可以在Deque或者List基础上进行实现  
顺序存储结构的栈是在Deque基础上进一步封装实现的，链式存储结构的栈就是在List基础上实现的  
核心函数如下：  

| 函数 | 功能 | 时间复杂度 |
| --- | --- | --- |
| top | 查看栈顶元素 | O(1) |
| push | 入栈 | O(1) |
| pop | 出栈 | O(1) |

``` C++
#ifndef STACK_H_
#define STACK_H_

#include <iostream>

#include <tools/base/noncopyable.h>

#include "List/Deque.h"

template <typename T, typename Container = Deque<T>>
class Stack : noncopyable {
public:
	Stack() = default;
	~Stack() = default;

	int size() const {
		return m_container.size();
	}

	bool empty() const {
		return m_container.empty();
	}

	void push(T elem){
		m_container.push_back(elem);
	}

	void pop(){
		m_container.pop_back();
	}

	T& top() const {
		return m_container.back();
	}

	// for unit test
	void print(const char* name) const {
		m_container.print(name);
	}

private:
	Container m_container;
};

#endif // STACK_H_
```
提供了模板参数传入的方式制定栈的基础结构是Deque还是List，默认是采用Deque，采用Deque和List都是上述的时间复杂度  
如果采用ForwardList或者CircleList时间复杂度将会上升（原因在于统一采用的线性表的头部作为栈底，尾部作为栈顶）  
同时在ForwardList和CircleList中没有实现back接口（返回尾部元素），所以不允许其作为Stack的基础结构（top需要调用back接口）  
Deque、ForwardList、CircleList和List的实现代码详见博客[《数据结构 -- 线性表》](https://blog.csdn.net/sdjn_lyn/article/details/104115075)或者[GitHub源码](https://github.com/liuyunian/DataStructure-Algorithm/tree/master/List)

## 栈的思考
其实所有的情况下使用Deque或者List作为容器，完全可是编写出解决问题的程序，那为什么要封装栈、队列这样的数据结构呢？  
因为这进一步的封装将会简化程序设计的问题，划分了不同的关注层次，使得思考范围缩小，更加聚焦于我们要解决的问题核心。也就是将一些实现上的细节隐藏了，没必要再去关注  
