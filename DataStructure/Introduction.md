# 数据结构绪论
## 数据结构是做什么的？
数据结构是研究数据如何在计算机内存中进行组织和存储，使得我们可以高效的获取数据或者修改数据

## 学习数据结构有用吗？
目前在学习、做项目中确实没有感受到哪里需要用到数据结构，这是为什么呢？  
原因很简单，你使用的编程语言，IDE、操作系统已经把数据结构都实现好了，而你需要的是怎么使用它，怎么在它的基础上增加一些业务逻辑实现一个产品  
另外一个档次就是开发IDE、编成语言、操作系统，在这个层次上就需要了解更多的数据结构和算法方面的知识了

## 数据结构有哪些内容？
主要有三种数据结构：
* 线性结构
* 树结构
* 图结构

## 数据结构中的基本概念
**逻辑结构**：表征了数据元素之间的关系  

**存储结构**：表征了具有逻辑结构的元素是如何在计算机内存中存储的

其实每种数据结构都有两种存储结构：顺序存储结构和链式存储结构
两者的区别就是：顺序存储是将数据存放在连续的内存区域，链式存储可以将元素存放在任意的内存区域，可以连续，也可以不连续

**抽象数据类型（Abstract Data Type，ADT）**：一个数学模型及定义在该模型上的一组操作  
理解：对编程中所用到的数据容器的一种抽象，比如int整型就是一种抽象数据类型，整型的具体实现在不同的操作系统，不同的编程语言中是不一样的，但是在不考虑具体实现，仅考虑如何使用的情况下，可以抽象出int整型  
再比如说线性表的抽象数据类型，线性表是一种有自己特点的数据结构，每种语言在实现时肯定是不一样的，但是它们都可以忽略实现的细节，将本质进行抽象概括，就构成了线性表的抽象数据类型

## 源码
在GitHub上建了一个仓库记录学习数据结构和算法的过程，大家可以参考学习：[https://github.com/liuyunian/DataStructure-Algorithm](https://github.com/liuyunian/DataStructure-Algorithm)