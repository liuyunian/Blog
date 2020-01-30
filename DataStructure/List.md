# 线性表
## 定义
0个或多个数据元素的有限序列  
理解：相同类型的元素之间是有顺序的，第一个元素只有一个后继节点，没有前驱节点，最后一个元素只有一个前驱节点，没有后继节点，其他元素各有一个前驱节点和后继节点

## 抽象数据类型
```
ADT 线性表（List）
Data
	数据元素之间满足线性表的定义
Operation
	InitList(*L)            // 初始化操作，建立一个空的线性表
	ListEmpty(L)            // 若线性表为空，返回true，否则返回false
  ListLength(L)           // 返回线性表L的元素个数
	ClearList(*L)           // 将线性表清空
	GetElem(L, i, *e)       // 将线性表L中的第i个元素值返回给e
	LocateElem(L, e)        // 在线性表L中查找与给定值e相等的元素，如果成功，返回该元素在表中的序号，否则返回0
	ListInsert(*L, i ,e)    // 在线性表L中第i个位置插入新元素e
	ListDelete(*L, i, e)    // 删除线性表L中第i个位置元素，并用e返回其值
endADT
```

## 存储结构
从存储结构上分，线性表的实现可以分成两种：  
* 顺序存储结构：数组
* 链式存储结构：单链表、双向链表、循环链表

## 动态数组(Deque)
C/C++中只提供的静态数组（不考虑STL），静态数组的优点是随机访问，存取元素的时间复杂度是O(1)，缺点是容量固定，往往会造成内存浪费  
动态数组可以根据数组存储元素的多少，动态调整容量  
核心函数如下：  

| 函数 | 功能 | 时间复杂度 |
| --- | :---: | :---: |
| push_front | 向动态数组头部添加元素 | O(N) |
| pop_front | 移除动态数组头部的元素 | O(N) |
| push_back | 向动态数组尾部添加元素 | O(1) |
| pop_back | 移除动态数组尾部的元素 | O(1) |
| insert | 向指定位置插入元素 | O(N) |
``` C++
#ifndef DEQUE_H_
#define DEQUE_H_

#include <iostream>

#include <assert.h>

#include "tools/base/copyable.h"

#define DEFAULT_CAPACITY 10

template <typename T>
class Deque : copyable {
public:
  explicit Deque(int capacity) : 
    m_capacity(capacity),
    m_size(0),
    m_data(new T[m_capacity]){}

  Deque() : Deque(DEFAULT_CAPACITY){}

  Deque(const Deque& other) :                     // 拷贝构造函数
    m_capacity(other.capacity()),
    m_size(other.size()),
    m_data(new T[m_capacity])
    {
      for(int i = 0; i < m_size; ++ i){
        m_data[i] = other.at(i);                  // m_data[i] = other[i]报错
      }
    }

  ~Deque(){
    delete[] m_data;
  }

  T& operator[](int index){                       // 重载[]
    assert(index >= 0 && index < m_size);
    return m_data[index];
  }

  void operator=(const Deque& other){             // 重载=
    m_capacity = other.capacity();
    m_size = other.size();
    delete[] m_data;
    m_data = new T[m_capacity];
    for(int i = 0; i < m_size; ++ i){
        m_data[i] = other.at(i);
    }
  }

  int size() const {
    return m_size;
  }

  int capacity() const {
      return m_capacity;
  }

  bool empty() const {
    return m_size == 0;
  }

  void insert(int index, T elem){
    if(m_size == m_capacity){
      resize(m_capacity * 2);                 // 扩容成原来的两倍
    }

    assert(index >= 0 && index <= m_size);
    for(int i = m_size; i > index; -- i){
      m_data[i] = m_data[i-1];
    }

    m_data[index] = elem;
    ++ m_size;
  }

  void remove(int index){
    assert(index >= 0 && index < m_size);
    for(int i = index + 1; i < m_size; ++ i){
      m_data[i - 1] = m_data[i]; 
    }

    -- m_size;

    if(m_capacity / 2 >= DEFAULT_CAPACITY && m_size == m_capacity / 4){
      resize(m_capacity / 2);
    }
  }

  void push_front(T elem){
    insert(0, elem);
  }

  void pop_front(){
    remove(0);
  }

  void push_back(T elem){
    insert(m_size, elem);
  }

  void pop_back(){
    remove(m_size - 1);
  }

  T& at(int index) const {
    assert(index >= 0 && index < m_size);
    return m_data[index];
  }

  T& front() const {
    return at(0);
  }

  T& back() const {
    return at(m_size - 1);
  }

  void resize(int newCapacity){
    T* newData = new T[newCapacity];
    for(int i = 0; i < m_size; ++ i){
      newData[i] = m_data[i];
    }

    m_data = newData;
    m_capacity = newCapacity;
  }

  // for test
  void print(const char* name) const {
    std::cout << name << "(Deque): capacity = " << m_capacity << ", size = " << m_size << '\n';
    if(m_size == 0){
      std::cout << "data = empty" << std::endl;
      return;
    }
    
    std::cout << "data = [";
    for(int i = 0; i < m_size - 1; ++ i){
      std::cout << m_data[i] << ", ";
    }
    std::cout << m_data[m_size - 1] << "]" << std::endl;
  }

private:
  int m_capacity;
  int m_size;
  T *m_data;
}; 

#endif // DEQUE_H_
```

## 单链表(ForwardList)
每个节点有一个数据域（存储数据）和一个指针域（指向后继节点），最后一个节点的指针域指向nullptr  
使用头指针操作链表，遍历链表只能从头指针开始，所以访问链表中第一个节点的时间复杂度为O(1)，访问最后一个节点的时间复杂度为O(N)  

核心函数如下：  

| 函数 | 功能 | 时间复杂度 |
| --- | :---: | :---: |
| push_front | 向链表头部添加元素 | O(1) |
| pop_front | 移除链表头部的元素 | O(1) |
| push_back | 向链表尾部添加元素 | O(N) |
| pop_back | 移除链表尾部的元素 | O(N) |

单链表操作尾部的元素需要从头部遍历到尾部之后才能操作，所以时间复杂度为O(N)  
当然可以维护一个尾指针指向最后一个结点，但是这相当于空间换时间
``` C++
#ifndef FORWARDLIST_H_
#define FORWARDLIST_H_

#include <iostream>

#include <tools/base/copyable.h>
#include <tools/base/noncopyable.h>

template <typename T>
class ForwardList : copyable {
private:
	struct Node : noncopyable {
		T data;
		Node *next;

		Node(T elem) : data(elem), next(nullptr) {}
	};

public:
	ForwardList() : 
		m_head(nullptr), 
		m_size(0){}                                 // 默认构造函数

	ForwardList(const ForwardList& other) :         // 拷贝构造函数
		m_head(nullptr), 
		m_size(other.size())
	{         
		if(other.empty()){
			return;
		}

		m_head = new Node(other.head()->data);
		if(other.size() == 1){
			return;
		}

		Node *preNode = m_head;
		Node *selfNode;
		Node *otherNode = other.head()->next;
		while(otherNode != nullptr){
			selfNode = new Node(otherNode->data);
			preNode->next = selfNode;
			preNode = preNode->next;
			otherNode = otherNode->next;
		}
	}

	~ForwardList(){
		Node *curNode = m_head;
		Node *nextNode;
		while(curNode != nullptr){
			nextNode = curNode->next;
			delete curNode;
			curNode = nextNode;
		}
	}

	int size() const {
		return m_size;
	}

	bool empty() const {
		return m_size == 0;
	}

	Node *head() const {
		return m_head;
	}

	void push_front(T elem){                        // 向链表头部添加元素
		Node *node = new Node(elem);
		node->next = m_head;
		m_head = node;

		++ m_size; 
	}

	void pop_front(){                               // 移除链表头部的元素
		if(m_head == nullptr){
			return;
		}

		Node *node = m_head->next;
		delete m_head;
		m_head = node;

		-- m_size;
	}

	void push_back(T elem){                         // 向链表尾部添加元素
		Node *node = new Node(elem);
		if(m_head == nullptr){
			m_head = node;
		}
		else{
			Node *tailNode = m_head;
			while(tailNode->next != nullptr){
				tailNode = tailNode->next;
			}
			tailNode->next = node;
		}

		++ m_size;
	}

	void pop_back(){                                // 移除链表尾部的元素
		if(m_head == nullptr){
			return;
		}

		if(m_size == 1){
			delete m_head;
			m_head = nullptr;
			-- m_size;
			return;
		}

		Node *preNode = m_head;
		Node *tailNode = preNode->next;
		while(tailNode->next != nullptr){
			preNode = tailNode;
			tailNode = tailNode->next;
		}
		preNode->next = nullptr;
		delete tailNode;

		-- m_size;
	}

	void operator=(const ForwardList &other){       // 重载赋值运算符
		this->~ForwardList();

		m_head = nullptr;
		m_size = other.size();

		if(other.empty()){
			return;
		}

		m_head = new Node(other.head()->data);
		if(other.size() == 1){
			return;
		}
		
		Node *preNode = m_head;
		Node *selfNode;
		Node *otherNode = other.head()->next;
		while(otherNode != nullptr){
			selfNode = new Node(otherNode->data);
			preNode->next = selfNode;
			preNode = preNode->next;
			otherNode = otherNode->next;
		}
	}

	// for pratice and it doesn't
	void insert(int index, T elem){}                // 向指定位置插入元素

	void remove(int index){}                        // 删除指定位置的元素

	void set(int index, T elem){}                   // 修改指定位置的元素

	T& get(int index) const {}                      // 获取指定位置的元素

	// for unit test
	void print(const char *name) const {
		std::cout << name << "(ForwardList): size = " << size() << '\n';
		std::cout << "data = ";
		Node *curNode = m_head;
		while(curNode != nullptr){
				std::cout << curNode->data << " -> ";
				curNode = curNode->next;
		}
		std::cout << "nullptr" << std::endl;
	}

private:
	Node *m_head;
	int m_size;
};

#endif // FORWARDLIST_H_
```

## 循环链表(CircleList)
循环链表的每个节点也是有一个数据域和一个指针域构成，与单链表不同的是链表最后一个节点的指针域指向第一个节点，这样链表就围成了一个环  
循环链表的好处是：从链表中的任意一个节点开始都能遍历完整个链表  
循环链表中使用尾指针来操作循环链表，这样访问最后一个节点和第一个节点的时间复杂度就都是O(1)

核心函数如下：

| 函数 | 功能 | 时间复杂度 |
| --- | :---: | :---: |
| push_front | 向链表头部添加元素 | O(1) |
| pop_front | 移除链表头部的元素 | O(1) |
| push_back | 向链表尾部添加元素 | O(1) |
| pop_back | 移除链表尾部的元素 | O(N) |

由于pop_back()需要找到最后一个节点的前驱节点，所以需要遍历整个链表，时间复杂度为O(N)
``` C++
#ifndef CIRCLELIST_H_
#define CIRCLELIST_H_

#include <iostream>

#include <tools/base/copyable.h>
#include <tools/base/noncopyable.h>

template <typename T>
class CircleList : copyable {
public:
	struct Node : noncopyable {
		T data;
		Node *next;

		Node(T elem) : data(elem), next(nullptr){}
	};

	CircleList() : m_tail(nullptr), m_size(0){}     // 默认构造函数

	CircleList(const CircleList &other) : 
		m_tail(nullptr),
		m_size(other.size())
	{
		if(m_size == 0){
			return;
		}

		m_tail = new Node(other.tail()->data);

		Node *preNode = m_tail;
		Node *selfNode;
		Node *otherNode = other.head();
		while(otherNode != other.tail()){
			selfNode = new Node(otherNode->data);
			preNode->next = selfNode;
			preNode = preNode->next;
			otherNode = otherNode->next;
		}
		preNode->next = m_tail;
	}

	~CircleList(){
		Node *curNode = m_tail;
		Node *nextNode;
		for(int i = 0; i < m_size; ++ i){
			nextNode = curNode->next;
			delete curNode;
			curNode = nextNode;
		}
	}

	Node* tail() const {
		return m_tail;
	}

	Node* head() const {
		return m_tail->next;
	}

	int size() const {
		return m_size;
	}

	bool empty() const {
		return m_size == 0;
	}

	void push_front(T elem){                        // 向链表头部添加元素
		Node *node = new Node(elem);
		if(m_tail == nullptr){
			m_tail = node;
			m_tail->next = m_tail;
		}
		else{
			node->next = m_tail->next;
			m_tail->next = node;
		}
		
		++ m_size;
	}

	void pop_front(){                               // 移除链表头部的元素
		if(m_tail == nullptr){
			return;
		}

		if(m_size == 1){
			delete m_tail;
			m_tail = nullptr;
			-- m_size;
			return;
		}

		Node *node = m_tail->next;
		m_tail->next = node->next;
		delete node;

		-- m_size;
	}

	void push_back(T elem){                         // 向链表尾部添加元素
		Node *node = new Node(elem);
		if(m_tail == nullptr){
			m_tail = node;
			m_tail->next = m_tail;
		}
		else{
			node->next = m_tail->next;
			m_tail->next = node;
			m_tail = node;
		}

		++ m_size;
	}

	void pop_back(){                                // 移除链表尾部的元素
		if(m_tail == nullptr){
			return;
		}

		if(m_size == 1){
			delete m_tail;
			m_tail = nullptr;
			-- m_size;
			return;
		}

		Node *preNode = m_tail->next;
		while(preNode->next != m_tail){
			preNode = preNode->next;
		}
		preNode->next = m_tail->next;
		delete m_tail;
		m_tail = preNode;

		-- m_size;
	}

	void operator=(const CircleList& other){
		this->~CircleList();

		m_tail = nullptr;
		m_size = other.size();

		if(m_size == 0){
			return;
		}

		m_tail = new Node(other.tail()->data);
		Node *preNode = m_tail;
		Node *selfNode;
		Node *otherNode = other.head();
		while(otherNode != other.tail()){
			selfNode = new Node(otherNode->data);
			preNode->next = selfNode;
			preNode = selfNode;
			otherNode = otherNode->next;
		}
		preNode->next = m_tail;
	}

	// for pratice and it doesn't
	void insert(int index, T elem){}                // 向指定位置插入元素

	void remove(int index){}                        // 删除指定位置的元素

	void set(int index, T elem){}                   // 修改指定位置的元素

	T& get(int index) const {}                      // 获取指定位置的元素

	// for unit test
	void print(const char *name) const {
		std::cout << name << "(CircleList): size = " << m_size << '\n';
		std::cout << "data: ";
		if(m_size == 0){
			std::cout << "nullptr" << std::endl;
			return;
		}

		Node *node = head();
		for(int i = 0; i < m_size; ++ i){
			std::cout << node->data << " -> ";
			node = node->next;
		}
		std::cout << head()->data << std::endl;
	}

private:
	Node *m_tail;
	int m_size;
};

#endif // CIRCLELIST_H_
```

### [双向链表(List)](List.h)
双向链表每个节点是由一个数据域和两个指针域（分别指向前驱节点和后继节点）构成  
优点在于可以正向或者反向遍历链表，灵活性提高  
缺点是每个节点多了一个指针域，多占用了存储空间（32位4字节，64位8字节）
一般情况下，双向链表都会以双向循环链表方式实现，这样可以优势最大化

核心函数如下：

| 函数 | 功能 | 时间复杂度 |
| --- | :---: | :---: |
| push_front | 向链表头部添加元素 | O(1) |
| pop_front | 移除链表头部的元素 | O(1) |
| push_back | 向链表尾部添加元素 | O(1) |
| pop_back | 移除链表尾部的元素 | O(1) |

``` C++
#ifndef LIST_H_
#define LIST_H_

#include <iostream>

#include <tools/base/copyable.h>
#include <tools/base/noncopyable.h>

template <typename T>
class List : copyable {
public:
	struct Node {
		T data;
		Node *pre;
		Node *next;

		Node(T elem) : data(elem), pre(nullptr), next(nullptr){}
	};

	List() : m_head(nullptr), m_size(0){}                           // 默认构造函数

	List(const List& other) :                                       // 拷贝构造函数
		m_head(nullptr),
		m_size(other.size())
	{
		copy(other);
	}

	~List(){
		Node *curNode = m_head;
		Node *nextNode;
		for(int i = 0; i < m_size; ++ i){
			nextNode = curNode->next;
			delete curNode;
			curNode = nextNode;
		}
	}

	Node* head() const {
		return m_head;
	}

	int size() const {
		return m_size;
	}

	bool empty() const {
		return m_size == 0;
	}

	void push_front(T elem){                                        // 向链表头部添加元素
		Node *node = new Node(elem);
		if(m_head == nullptr){
				m_head = node;
				m_head->next = m_head;
				m_head->pre = m_head;
		}
		else{
				Node *tailNode = m_head->pre;
				node->next = m_head;
				node->pre = tailNode;
				m_head->pre = node;
				tailNode->next = node;
				m_head = node;
		}
		
		++ m_size;
	}

	void pop_front(){                                               // 移除链表头部的元素
		if(m_head == nullptr){
			return;
		}

		if(m_size == 1){
			delete m_head;
			m_head = nullptr;
		}
		else{
			Node *nextNode = m_head->next;
			Node *tailNode = m_head->pre;

			tailNode->next = nextNode;
			nextNode->pre = tailNode;
			delete m_head;
			m_head = nextNode;
		}

		-- m_size;
	}

	void push_back(T elem){                                         // 向链表尾部添加元素
		Node *node = new Node(elem);
		if(m_head == nullptr){
			m_head = node;
			m_head->next = m_head;
			m_head->pre = m_head;
		}
		else{
			Node *tailNode = m_head->pre;
			node->next = m_head;
			node->pre = tailNode;
			tailNode->next = node;
			m_head->pre = node;
		}

		++ m_size;
	}

	void pop_back(){                                                // 移除链表头部的元素
		if(m_head == nullptr){
			return;
		}

		if(m_size == 1){
			delete m_head;
			m_head = nullptr;
		}
		else{
			Node *tailNode = m_head->pre;
			tailNode->pre->next = m_head;
			m_head->pre = tailNode->pre;
			delete tailNode;
		}

		-- m_size;
	}

	void operator=(const List& other){
		this->~List();
		m_head = nullptr;
		m_size = other.size();

		copy(other);
	}

	T& front() const {
		return m_head->data;
	}

	T& back() const {
		return m_head->pre->data;
	}

	// for pratice and it doesn't
	void insert(int index, T elem){}                // 向指定位置插入元素

	void remove(int index){}                        // 删除指定位置的元素

	void set(int index, T elem){}                   // 修改指定位置的元素

	T& get(int index) const {}                      // 获取指定位置的元素

	// for unit test
	void print(const char *name) const {
		std::cout << name << "(List): size = " << m_size << '\n';
		std::cout << "data: ";
		if(m_size == 0){
			std::cout << "nullptr" << std::endl;
			return;
		}

		Node *node = m_head;
		for(int i = 0; i < m_size; ++ i){
			std::cout << node->data << " <=> ";
			node = node->next;
		}
		std::cout << m_head->data << std::endl;
	}

private:
	void copy(const List& other){
		if(m_size == 0){
				return;
		}

		m_head = new Node(other.head()->data);
		Node *preNode = m_head;
		Node *selfNode;
		Node *otherNode = other.head()->next;
		for(int i = 1; i < m_size; ++ i){
			selfNode = new Node(otherNode->data);
			selfNode->pre = preNode;
			preNode->next = selfNode;
			preNode = selfNode;

			otherNode = otherNode->next;
		}

		preNode->next = m_head;
		m_head->pre = preNode;
	}

private:
	Node *m_head;
	int m_size;
};

#endif // LIST_H_
```