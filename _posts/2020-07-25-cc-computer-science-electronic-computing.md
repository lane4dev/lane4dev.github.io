---
layout: post
title: 第二节：电子计算机
subtitle: Crash Course 计算机科学系列课程笔记
categories: Notes
tags: [crash course, EECS]
related: [
  "_posts/2020-07-18-cc-computer-science-early-computing.md"
]
---

![](https://www.youtube.com/watch?v=LN0ucKNX0hc)

📌 **SUMMARY:**  

## Recall

## Notes

早期计算设备都只是针对特定用途（制表机）。二战后人类社会大规模增长，全球贸易和运输变得愈发紧密，工程和科学的复杂度达到新高（登月），数据量暴增。对计算力要求迅速提升，柜子大小的计算机发展到房间大小（维护费用高，容易出错）。

### 哈佛 Mark 1 号

IBM 1994 年制造（二战同盟国，“曼哈顿计划”计算模拟数据）  
76 万 5 千个组件，300 万个连接点 和 500 英里长的导线。有一个 50 英尺的传动轴，由一个 5 马力的电机驱动

- 机械继电器  
  用电控制的机械开关。继电器里，有根“控制线路”，控制电路是开还是关。  
  “控制线路”连着一个线圈，当电流流过线圈，线圈产生电磁场，吸引金属臂，从而闭合电路。（水龙头）  

  ![25-01.png](/assets/images/2020/07/25-01.png)

  > Unfortunately, the mechanical arm inside of a relay **has mass**, and therefore **can't move instantly** between opened and closed states.

  1940 年代一个好的继电器 1 秒能翻转 50 次。 

- 缺陷

  - 速度慢  
  Mark 1 号，1 秒能做 3 次加法或减法运算。一次乘法要花 6 秒，除法要花 15 秒。更复杂的操作 比如三角函数，可能要花一分钟以上。

  - 齿轮磨损   
  Mark 1 号，大约 3500 个继电器（假设继电器的使用寿命是 10 年，平均每天都需要换一个继电器）
  <aside>
  📌 任何会动的机械都会随时间磨损，有些部件完全损坏，有些则是变粘，变慢，变得不可靠。随着继电器数量增加，故障率也会增加。
  </aside>

  - 吸引 Bugs  
  1947 年 9 月，Mark 2 型的操作员从故障继电器里，拔出一只死虫
  ![25-02.png](/assets/images/2020/07/25-02.png)

  > From then on ,when anything went wrong with a computer, we said it had bugs in it.  —— Grace Hopper

### 真空管

<aside>
📌 电流只能单向流动的电子部件叫“二极管(Diode)”。
</aside>

1904 年，英国物理学家 “约翰·安布罗斯·弗莱明” 发明了一种新的电子组件，叫 “热电子管”（Thermionic valve）。  
把两个电极装在一个气密的玻璃灯泡里，这是世界上第一个真空管。其中一个电极可以加热，从而发射电子，即：“热电子发射 ( thermionic emission )”，另一个电极会吸引电子，形成 “水龙头” 的电流。只有带正电才能通过，如果带负电荷或中性电荷，电子就没办法被吸引，越过真空区域，因此没有电流。  

### 继电器

<aside>
📌 标志着计算机，从机电转向电子。
</aside>

1906 年，美国发明家 “李·德富雷斯特”，在 弗莱明 设计的两个电极之间，加入了第三个 “控制” 电极。向控制电极施加正电荷，它会允许电子流过，如果施加负电荷，它会阻止电子流动。通过控制线路，可以断开或闭合电路。（真空管里没有会动的组件，意味着没有损耗，每秒可以开闭数千次）  

![25-03.png](/assets/images/2020/07/25-03.png)

这些 “三级真空管” 成为无线电，长途电话，以及其他电子设备的基础，持续了接近半个实际。（像灯泡一样会烧坏，成本高，体积大）  


### 巨人 1 号

<aside>
📌 被认为是第一个可编程的电子计算机
</aside>

由工程师 Tommy Flowers 设计，完工于 1943 年 12 月。  
在英国 布莱切利园（用于破解纳粹通信）首次大规模使用真空管（1600 个）。  

> Programming was done by plugging hundreds of wires into plugboards, sort of like old school telephone switchboards, in order to set up the  computer to perform the right operations. so while "programmable", it still had to be configured to perform any specific computation.


### 电子数值积分计算机（ENIAC）

<aside>
📌 世界上第一个真正的通用，可编程，电子计算机。
</aside>

![25-04.png](/assets/images/2020/07/25-04.png)

1946 年，宾西法尼亚大学的 ENIAC。设计者是 Jhon Mauchly 和 J. Presper Eckert 。  

每秒可执行 5000 次十位数加减法。  
它运行了十年。据估计，它完成的计算，比全人类加起来还多。  
真空管很多，所以故障很常见。ENIAC 运行半天左右就会出一次故障。  
1950 年，真空管计算机达到极限。  
1955 年，美国空军的 AN/FSQ-7 计算机。  

### 晶体管 [ Transistors ]

降低成本和大小，提高可靠性和速度。  
1947 年，贝尔实验室作出了晶体管，一个全新的计算机时代诞生。  
晶体管有两个电极，电极之间有一种材料隔开它们，这种材料有时候导电，有时候不导电（半导体）。控制线连到一个“门”电极，通过改变“门”的电荷，来控制半导体的导电性。  

![25-05.png](/assets/images/2020/07/25-05.png)

对比真空管  
  - 晶体管每秒可以开关 10000 次。
  - 晶体管是固态的，比玻璃的真空管更可靠
  - 晶体管可以远远小于继电器或晶体管

使用晶体管可以制造更小更便宜的计算机。

1957 年发布的 IBM 608（第一个完全用晶体管，面向一般消费者的计算机）。   
  - 包含 3000 个晶体管
  - 每秒执行 4500 次加法
  - 每秒能执行 80 次左右的乘除法。

![25-06.png](/assets/images/2020/07/25-06.png)

晶体管有诸多好处，IBM 很快全面转向晶体管  

> Today, computers use transistors that are smaller than 50 nanometers in size - for reference, a sheet of paper is roughly 10000 nanometers thick. They're not only incredibly small, they're super fast - they can switch states millions of times per second, and can run for decades.  

- 硅谷的典故  
  很多晶体管和半导体的开发都是 “圣克拉拉谷” 做的（加州，位于 旧金山 和 圣荷西 之间）。而生产半导体最常见的材料是硅，这个地区被成为 “硅谷”。

- 英特尔  
  肖克利半导体（William Shockley） → 仙童半导体 → 英特尔


### Reference:

- [Transistor](https://en.wikipedia.org/wiki/Transistor)
- [How do transistors work?](https://www.explainthatstuff.com/howtransistorswork.html)
