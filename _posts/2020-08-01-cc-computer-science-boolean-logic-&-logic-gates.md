---
layout: post
title: 第三节：布尔逻辑 和 逻辑门
subtitle: Crash Course 计算机科学系列课程笔记
categories: Notes
tags: [crash course, EECS]
---

![](https://www.youtube.com/watch?v=gI-qXk7XojA)

## Recall

什么是二进制，为什么用二进制，布尔逻辑？  
NOT、AND、OR 的逻辑是什么样的？电路是什么样的？  


## Notes

### 二进制

> In Computers, an "on" state, when electricity is flowing, represents true. the "off" state, on electricity flowing represents false.  

- 为什么采用二进制？  
  状态越多，越难区分信号（早期某些电子计算机是三进制、五进制）  
  ![01-01.png](/assets/images/2020/08/01-01.png)

- George Boole
  > Boole's approach allowed truth to be systematically and formally proven, through logic equations which he introduced in his first book, "The Mathematical analysis of logic" in 1847.

  在布尔代数，变量值为 true 和 false，并对这些变量进行逻辑处理。布尔代数中有三个操作：“非”，“与”，“或”。


### 三个基本操作：NOT, AND, OR

- NOT
  ![01-02.png](/assets/images/2020/08/01-02.png)
    
- AND
  ![01-03.png](/assets/images/2020/08/01-03.png)
    
- OR
  ![01-04.png](/assets/images/2020/08/01-04.png)
    
- XOR
  ![01-05.png](/assets/images/2020/08/01-05.png)


当计算机工程师在设计处理器时，很少会在晶体管在这个层面工作。作为代替，他们通常使用更大的区块，例如逻辑门。或者由逻辑门组成的更大的组件。

通过掌握逻辑门，我们可以使用它构建一个鉴定复杂逻辑命题的机器。
