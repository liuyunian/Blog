# 队列
队列也是一种特殊的线性表，特殊在于只允许在一端插入元素，在另一端删除元素，所以队列有先进先出的特性  
队列的可以分为普通队列和带有优先级的队列  

## 术语  
队头：允许删除元素的一端  
队尾：允许插入元素的一端  
入队：向队列中插入元素操作  
出队：从队列中删除元素操作

## 普通队列
``` C++
#ifndef QUEUE_H_
#define QUEUE_H_

#include <tools/base/noncopyable.h>

#include "List/Deque.h"

template <typename T, typename Container = Deque<T>>
class Queue : noncopyable {
public:
	Queue() = default;
	~Queue() = default;

	int size() const {
		return m_container.size();
	}

	bool empty() const {
		return m_container.empty();
	}

	T& front() const {
		return m_container.front();
	}

	T& back() const {
		return m_container.back();
	}

	void push(T elem){
		m_container.push_back(elem);
	}

	void pop(){
		m_container.pop_front();
	}

	// for unit test
	void print(const char* name){
		m_container.print(name);
	}

private:
	Container m_container;
};

#endif // QUEUE_H_
```
可以通过模板参数指定队列的底层的存储结构，比如：  
* Queue<T, Deque<T>>：采用动态数组Deque作为底层容器 -- 顺序存储结构
* Queue<T, List<T>>：采用双向链表List作为底层容器 -- 链式存储结构  

核心函数如下：

| 函数 | 功能 | 时间复杂度 |
| --- | :---: | :---: |
| front | 查看队头元素 | O(1) |
| back | 查看队尾元素 | O(1) |
| push | 入队 | O(1) |
| pop | 出队 | Deque: O(N), List: O(1) |  

这里就可以看出问题，采用Deque做底层容器时出队的时间复杂度为O(N)，原因在于数组下标为0的位置始终作为队头，这样出队之后，需要将数组中的其他元素向前移动  
如果数组中任何一个位置都可以作为队头，用front指针来表征队头的位置，那么出队的操作将是队头指针的改变，其他元素不用迁移，这样出队操作的时间复杂度将会是O(1)  
但是这样会出现另外的问题，随着出队，队头指针后移，那么数组前面的空间将不能存储元素，因为队列的特性是只能队尾插入元素  
可以使用***循环队列***解决这一问题

## 循环队列
循环队列是指首尾相连的队列，用head和tail指针来表明队头和队尾  
循环队列需要考虑一个问题：head == tail时是表示队列为空还是为满呢？有两种方式实现：
* 用一个bool类型flag标志进行区分，flag == true; head == tail表示队列为空，flag == false; head == tail表示队列为满
* head == tail表示队列为空，当队列中只有一个空闲空间时表示队列为满，在实现时采用的这种方式
``` C++
#ifndef CIRCLEQUEUE_H_
#define CIRCLEQUEUE_H_

#include <iostream>

#include <tools/base/noncopyable.h>

#define DEFAULT_CAPACITY 10

template <typename T>
class CircleQueue : noncopyable {
public:
	CircleQueue() : 
		m_capacity(DEFAULT_CAPACITY),
		m_head(0),
		m_tail(0),
		m_data(new T[m_capacity+1]){}

	~CircleQueue(){
		delete[] m_data;
	}

	int size() const {
		return (m_tail - m_head + m_capacity + 1) % (m_capacity + 1);
	}

	bool empty() const {
		return m_head == m_tail;
	}

	T& front() const {
		return m_data[m_head];
	}

	T& back() const {
		return m_data[(m_tail+m_capacity)%(m_capacity+1)];
	}

	void push(T elem){
		if((m_tail+1) % (m_capacity+1) == m_head){
			resize(m_capacity*2+1);
		}

		m_data[m_tail] = elem;
		++ m_tail;
		m_tail %= m_capacity+1;
	}

	void pop(){
		++ m_head;
		m_head %= m_capacity+1;

		if(m_capacity/2 >= DEFAULT_CAPACITY && size() == m_capacity/4){
			resize(m_capacity/2);
		}
	}

	// for unit test 
	void print(const char *name){
		int size = this->size();
		std::cout << name << "(CircleQueue): capacity = " << m_capacity << " size = " << size << '\n';
		if(size == 0){
			std::cout << "data = empty" << std::endl;
			return;
		}

		std::cout << "data = [";
		for(int i = 0; i < size; ++ i){
			std::cout << m_data[(m_head+i)%(m_capacity+1)] << ", ";
		}

		std::cout << "]" << std::endl;
	}

private:
	void resize(int newCapactiy){
		T *newData = new T[newCapactiy];
		int size = this->size();
		for(int i = 0; i < size; ++ i){
			newData[i] = m_data[(m_head+i)%(m_capacity+1)];
		}
		m_capacity = newCapactiy;
		m_head = 0;
		m_tail = size;
		m_data = newData;
	}

private:
	int m_capacity;
	int m_size;
	int m_head, m_tail; // tail指向队尾已有元素的下一个位置
	T *m_data;
};

#endif // CIRCLEQUEUE_H_
```

## 带有优先级的队列
带有优先级的队列是指元素在入栈时会按照排序规则（升序/降序）插入到指定的位置，这样就能按照大小顺序出队了  
带有优先级的队列更适合使用树型结构来实现（二叉堆），这样时间复杂度更低