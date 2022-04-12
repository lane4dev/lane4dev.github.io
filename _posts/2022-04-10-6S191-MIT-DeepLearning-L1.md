---
layout: post
title: 深度学习基础
subtitle: 6S191 MIT DeepLearning 课程笔记
categories: Notes
use_math: true
banner:
  image: /assets/images/2022/04/10/banner.png
  opacity: 0.618
  height: "100vh"
tags: [Deep Learning, MIT OpenCourseWare]
---

这篇文章是 6S191 MIT DeepLearning 系列课程第一课的笔记总结，我以原有课程内容为脉络，参考像是李宏毅老师的课程和其他一些资料，在 Activation Function 的数学意义、 Backpropagation 过程的推导等这些自己感兴趣的话题作出了横向扩展，希望更进一步加深对深度学习的理解。深度学习在近些年发展迅速，而 MIT 6S191 DeepLearning 作为一门每年都会更新的介绍深度学习的入门类课程，在保持着极高的时效性同时，还拥有极高的课程质量，是不可多得的学习资料。

## 什么是 “深度学习” ？
在文章的一开始需要理清楚几个基本的概念，分别是：“人工智能”、“机器学习” 和 “深度学习”。这几个概念在大众的认知中常常被有意或无意的混为一谈。  
首先，[人工智能](https://zh.wikipedia.org/wiki/%E4%BA%BA%E5%B7%A5%E6%99%BA%E8%83%BD) 这一概念是 1956 年由约翰·麦卡锡等计算机科学家在 [达特矛斯会议](https://zh.wikipedia.org/wiki/%E8%BE%BE%E7%89%B9%E7%9F%9B%E6%96%AF%E4%BC%9A%E8%AE%AE) 上提出。由于意识到计算机科学巨大的发展潜力，当时的科学家提出了一种构想，即：是否有可以找到一种方法通过计算机来模拟人类的智力活动？多年后的今天，[机器学习](https://zh.wikipedia.org/wiki/%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0) 对这个问题给出了其中一种答案。机器学习使用了统计学方法，针对特定问题，通过海量样本数据进行数学建模，发现这些数据的内在规律，并且利用发现的规律解决相似问题。而 [深度学习](https://zh.wikipedia.org/wiki/%E6%B7%B1%E5%BA%A6%E5%AD%A6%E4%B9%A0) 又是机器学习的一个分支，是以 [人工神经网络](https://zh.wikipedia.org/wiki/%E4%BA%BA%E5%B7%A5%E7%A5%9E%E7%BB%8F%E7%BD%91%E7%BB%9C) 的架构对样本数据建模的一种方法。  
从概念上来看，三者是相互包含的关系。“人工智能” 包含了 “机器学习”，“机器学习” 又包含了 “深度学习”。  
![人工智能_vs_机器学习_vs_深度学习](/assets/images/2022/04/10/2RwctP_VTHKLPG_KVDb_dm8zmoJTuKdParnrzEa3Td3d-TVkkfXkklnIArZglrrFNZHXpablbFFjRw15FADn58z8nsUxZY5Roecnn2JLWfY3zqfhkS-BAucWWzeBkb5W.png)


## 为什么需要机器学习 
传统计算机程序只能通过 `if ... else` 这样的条件判断，或是 `for / while` 这样的循环以一种线性的方式来处理问题。但是像是人脸识别、语音识别这一类的问题，没办法用传统的线性编程手段一步一步实现，我们迫切的需要一种新的算法来解决这些更加 “抽象” 的问题。


## 为什么是现在 
![为什么现在是进行机器学习的好时候](/assets/images/2022/04/10/gCb3x47M72emg0WR5bk9Gm146z5ih_lcS5P82WKGdbtXQlQKcbTFzHmqGtKXEfvcupPrQt-_eY0KCb7r7dnZ0Alhm08WsvK8_Sxox9mykAO0V2jFsoZ-tOkuTo9TwCwe.png)
1. 互联网的爆炸式发展已经积累起海量数据，为机器学习的算法研究提供数据支撑  
2. 硬件的快速迭代，尤其是显卡算力的高速增长为机器学习的程序运算提供硬件支撑  
3. 配套软件诸如 scikit-learn 、pytorch、tenserflow 这样的支撑机器学习的软件已经比较成熟，为机器学习的程序运算提供了软件支撑  


## 神经元（The Perceptron）
### 深度学习是基于对人类神经系统的模拟
关于人类大脑的神经网络是如何运作的，参考以下内容：
  - [TED Speech Sebastian Seung: I am my connectome](https://www.youtube.com/watch?v=HA7GwKXfJB0&ab_channel=TED)
  - [Wikipedia: Computational neuroscience](https://en.wikipedia.org/wiki/Computational_neuroscience)

### 神经元的数学表达
- 单个神经元由这样几部分组成： **输入**、**权重**、**求和函数**、**Activation Function** 和 **输出**。  
![单个神经元的数学表达](/assets/images/2022/04/10/YjtXa6x_-rzo-TRbBVJ7IAWQ0eQUz0i3eFKas7DUFtmYD7u4aVlt8LuJZH0tcc_NanI-IGAvUZmXmNL8sLe4BJFl0v8Tj0HZ-kvUKVc74Q_rSCFxlHdNn6ifTnkLWXUi.png)
- 具体计算的过程是这样： 
  - 对 $m + 1$ 个输入 $x_0(x_0 = 1), x_1 ... x_m$ 
  - 分别乘以各自的权重 $w_0, w_1 ... w_m$ 
  - 把得到的结果通过求和函数相加
  - 之后再传递给 Activation Function 最终得到一个输出 $\hat{y}$
- 数学公式可以表达为：  
  ![神经元的数学公式可以表达](/assets/images/2022/04/10/swMGz3n8zFhuAIDierQ64I7aKcq3iDzCIstdy5Xv5W5wUhHJUgsVKwEPEo4zE_t3DI8nkua4cLadChvnxQq30FwCG6JmEUfEN1Bwla3-M46W-JOA7sURlbyfJuAMXI_X.png)
- 用矩阵的方式来表示数学公式：   
  ![用矩阵的方式来表示神经元的数学公式](/assets/images/2022/04/10/A1TEieRfE7jie6S96rJimW4K-wfzMD7RE95OdK0O2dY3wyOXh021SNCQnivx2A9UIcxcr_Zbfs1A8dMu5nbp7thZnoYXrgE-J75qe4_IBDZKm5uTlDbv2FO-79cPDdGi.png)
- 考虑激活函数后的完整式子  
  ![考虑激活函数后完整神经元的式子](/assets/images/2022/04/10/yzznkUc9JiYAB9qAh2AneafxaYlZ6_DViSkgwWTghDVZBMMCuNtiZpMgvwqHeQL64yOG4UKZPH2IqT4w6ovK5aAvgCjVYNxsh6NtOMy6hvsatEJjCvZLQ7158zfnJ6tv.png)

### 关于激活函数
#### 常见的激活函数有哪些？  
![常见的激活函数有哪些](/assets/images/2022/04/10/A6tLY1Vyx1mV1MAEWnxYp99Q9lLjCOcYrzaMqktrUJ8ytuAaPdSRkvXLWsLODHaJblL2oYx62ekAxdB881PLnoEMVxVxM2_0SJq77Gb74qQZmeUN8rGMZDHXDrA0Qclo.png)

#### 为什么要有激活函数？
参考资料： <i class="fa fa-youtube-play" aria-hidden="true" style="color: red"></i> [Why Non-linear Activation Functions (C1W3L07)](https://www.youtube.com/watch?v=NkOv_k7r6no&ab_channel=DeepLearningAI)  

要回答这个问题，不妨先换个思路。思考另一个问题：  
> **如果没有激活函数会怎么样？**   

- 对一个两层的神经网络，我们能得到以下公式：  

$$
\begin{align}
& \text Z^{[1]} = W^{[1]}x + b^{[1]}  \\
& \text a^{[1]} = g^{[1]}(Z^{[1]})  \\
& \text Z^{[2]} = W^{[2]}x + b^{[2]}  \\
& \text a^{[2]} = g^{[2]}(Z^{[2]})  \\
\end{align}
$$

- 现在假设激活函数不存在，也即是在上面的式子中 $a^{[1]} = Z^{[1]}$ 和 $a^{[2]} = Z^{[2]}$  

- 又因为第一层是第二层的输入（ $a^{[1]}$ 等于 $a^{[2]}$ 式子中的输入 $x$ ），我们可以推导以下公式：

$$
\begin{align}
& \text a^{[1]} = Z^{[1]} = W^{[1]}x + b^{[1]}  \\
& \text a^{[2]} = Z^{[2]} = W^{[2]}x + b^{[2]}  \\
& \text a^{[2]} = W^{[2]}(W^{[1]}x + b^{[1]}) + b^{[2]}  \\
& \text a^{[2]} = W^{[2]}W^{[1]}x + W^{[2]}b^{[1]} + b^{[2]}  \\
\end{align}
$$

- 在上面的式子里，$W^{[2]}W^{[1]}$ 的结果可以用一个矩阵 $W^{[i]}$ 替代， $W^{[2]}b^{[1]} + b^{[2]}$ 的结果可以用另一个矩阵 $b^{[i]}$ 替代，于是就有了下面的式子：  

$$
a^{[2]} = W^{[i]}x = b^{[i]}
$$


通过以上推理过程能发现一个怎样的结论呢？ **如果没有激活函数，神经网络叠再多层都没有用，因为它始终都是线性的。**

所以激活函数的作用就显而易见了：   
**激活函数的存在为神经网络的结构引入了非线性。让它能够通过一层一层的叠加来处理复杂问题。**
![激活函数为神经网络的结构引入了非线性](/assets/images/2022/04/10/buY7B7yj6oDjQDwc-_LSx-QXpdOD-fEW6vdX7fNPx6EyrycIcVl6JNdkn5-7o2ov26YbR9cwdpGXkUi9XsCIBhGqDRKa8phcrBnLdx4grkxBs1dR-xyCex6Ne5QGtg_Y.png)


### 为什么需要有一个 $w_0$ ？

参考资料：[Glossary of Deep Learning: Bias](https://medium.com/deeper-learning/glossary-of-deep-learning-bias-cf49d9c895e2)

#### 从数学的角度理解
$w_{0}$ 被称为：bias ( 偏移 )。从数学上的解释来说，bias 的作用是激活函数能左右移动，以更好的拟合数据。
![bias 的作用](/assets/images/2022/04/10/3LZNU4PjVhWLH3LKsFb5RmX7_4I9IaEle6F44CWqMf71HZFdrj9Hq81Q3osjFun7VFmpQxBNVH0jAo_tikL8728kXy3ST6wYf8cvCfgLiBDEy35x8fykAAzlyE9NeNA9.png)

#### **从更简单（更符合直觉）的角度理解**
因为 bias 的存在，事实上决定了神经元在没有没有任何输入的情况下默认保持打开还是关闭的状态。 （或者这个神经元在多大程度上容易被打开，或容易被关闭）

### 所以 …… 这些 W 的权重是怎么算出来的？
简单的答案是 [Gradient Descent](#h-gradient-descent) + [Backpropagation](#h-backpropagation)，会在后面详细展开。


## 神经网络
### 神经元组成的网络
深度学习中的神经网络就是由这样一个一个单独神经元为基础不断叠加起来的
  ![神经网络](/assets/images/2022/04/10/pHwVvEygwG-68h5sOWvR-KCgtukdu2OZNp06gOvvSmz3lhxapA_ZZ3XC-IjW6ezAnapVPWqAbenSb1seZfGwJVG7mChK9P3bW2itKleOr-p1MKh-jh3cdcD0kbzl2DO6.png)

单层神经网络  
  ![单层神经网络](/assets/images/2022/04/10/h5E60NGn5X5K9JGywnXGvlE1uIgKts9NqQ5lx13lPAXgt0QyHVroDEbW2bzW9TQf6pZL_oVzoICEPzF93sy1n-RDNJoQpN6wD0YLNxKXXfrYdDHsqX4lIZgUbU9keHFC.png)

深度（多层）神经网络  
  ![多层神经网络](/assets/images/2022/04/10/vKo98iX56TE3uX7S1YO-LDlkKRBUhTMXfsoEeFUpi1HSH9JtEVYd1baITJ5VorDRfHJ2rn0QyxY-SI5yPfPw0_KxwcDb7b7JDa9gnbSsjL-joJWvJ4u2xq4UyWPaqQ7O.png)

### 评估预测结果的准确性
我们使用 **Loss Function** $J(W)$ 在给定权重值 $W$ 的情况下评估模型预测的准确性。有以下几种不同的评估方式：

#### Quantifying Loss  
旨在预测出错时衡量预测值与实际值之间的偏差。  
![Quantifying Loss](/assets/images/2022/04/10/pqgVQJf2DhXsGixp9eeDTYzI3f1pCTJqxLL2hPw9GZ6MwKJjPB7IziRIQeKIYlJn7Kz_Cp0tCWYtI6f56kZRw8JwX6dUGs75GuAOW44QXTtX6uXGuPxOngF2Qdg2OZm3.png)
#### Empirical Loss
衡量所有数据中预测值与实际值之间的偏差。  
![Empirical Los](/assets/images/2022/04/10/2qHUVpMC2hVzLQTOGm9SFmBY7aGq3aAj7z_jk4Ky01neU1BFFVUUqs3UDDr9uA2ntVWMHpvygHoGpSo2fVz_cZ9docl34wYubxNV5GImt3_KefxuEEeCYoOs_mE2nNbV.png)
#### Binary Cross Entropy Loss
当模型预测的结果是 0 到 1 之间的概率时，用 **Binary Cross Entropy Loss** 衡量预测准确性 。
![Binary Cross Entropy Loss](/assets/images/2022/04/10/5gqcd-FZByXTI_VILd-LSFiHPDzs4pHAbsafEePnxfPliOMoolBiPQiCDVEm0a7CSKLppaCG_0IVf9XNIcjCF5A_BbNLvAvb8ipBvn-VDcHEiJXG7gX-179C1R4_jy5r.png)
#### Mean Squared Error Loss  
一般用 **Mean squared error loss** 评价输出是连续值的回归类模型中实际值与预测值之间的差异。
![Mean Squared Error Loss](/assets/images/2022/04/10/ygWy0zgZ1WAdOXYeNojUX7PEiFf-1f4F2b4VzbTNUFqoQFIEXFw9yn2gNaT4lTMBUKR7YxBJ8MENuKfCvNgm6M0uCc8abvS6esdHLKVxCaFwH4SH6ywWTi7fNCvuuItG.png)


## 训练神经网络

在前一部分的内容里，在假定权重 $W$ 的值已经确定的条件下介绍了神经网络的结构以及如何评估预测结果的准确性。  
在这一部分会重点关注神经网络中的权重 $W$ 是如何产生的。  

理论上，我们需要找到这样一个 $W$ ，使得它在所有的数据集的预测结果上损失最小
![achieve the lowest loss](/assets/images/2022/04/10/39j-OVLsiPsIeQu5gIfUXWCG00ug6INMorCZFk2OhPyL2TWVGWffkqEGmbX_vy-LvhfHvZ3vH3uen6U2MmVjGaXpV7dXM1_UPFiVErO4RQbKDQSsHp9U3mTz-v1q2Uol.png)

如果可以把所有 $W$ 组合下对应的损失函数的值分布绘制出来，很直观的就能发现最小损失函数对应的 $w$ 值是多少（下图示范了只有两个 $w$ 的情况下情况下损失函数值域分布图 ）
![损失函数值分布图](/assets/images/2022/04/10/iwy-IcfibL_58w_ZJkBCeZIvkdT5TCLCi3XsI9LL2uHKxfWKAHokq1emjXv9DIEPH8MjDNsGNQJm32bqp2DAlvGkXQ4BO9X5K8FyNyr-hdWVkK6BRaotx4y2Nh0eMWBm.png)

### Gradient Descent
在实际应用场景里，一个普通的神经网路结构，轻轻松松就能包含上万个 $w$ 参数。想要穷举所有 $w$ 的组合即使对于现代计算机而言也是绝无可能的。更实际的做法是使用名为 **[Gradient Descent](https://en.wikipedia.org/wiki/Gradient_descent)** 的方法，思路是这样的： 
![Gradient Descent](/assets/images/2022/04/10/hUxc3mQ1NSywT6giIFvSCmHhMenT9E42eDS-rHbuLUHKupjtkHV6ue0L6oE2tyAjriK-dIWJ0mYzdzrAsFQr_UeTwMa7WuPQgntDc_NEGMOp1v-jWV3chz4wxCUugQoh.png)

- 在一开始随机给定 $W$ 初始值
- 对每一个 $w_i$ 分别计算它对损失函数 $J(W)$ 的斜率，也就是计算 $\frac{\partial J(W)}{\partial w_i}$
- 在使 $J(W)$ 总体变小的方向上选定一个步长 $\eta$ 来更新 $w_i$ 的值，然后前进到下一个点
- 重复以上步骤，直到 $\frac{\partial J(W)}{\partial W}$ 收敛，此时斜率为零，达到值局部最小的位置。

Gradient Descent 的算法描述为  
![Gradient Descent 的算法描述](/assets/images/2022/04/10/hyyDRw_fvLAyJTiPiYgdIkMSfH1hPqgCHE8PNCDSX8Cl2ob2cF2JifpmSISoI6njA5ERHhg6bGb1DQFBoSg5vEywO9g6SPRasJqQQBlvcx9CfLzs5_GodRW-goXvcJcb.png)

### Backpropagation

参考资料：
- [Backpropagation](https://en.wikipedia.org/wiki/Backpropagation)
- [Understanding Backpropagation Algorithm](https://towardsdatascience.com/understanding-backpropagation-algorithm-7bb3aa2f95fd)
- [Neural Networks from Scratch](https://www.youtube.com/playlist?list=PLQVvvaa0QuDcjD5BAw2DxE6OF2tius3V3)

接下来关注 $\frac{\partial J(W)}{\partial W}$ 具体的计算过程，在这个过程中使用了一种叫 **[Backpropagation](https://en.wikipedia.org/wiki/Backpropagation)** 的方法【 Backpropagation 是机器之所以能够 “学习” 的核心，因此需要重点掌握 】。

#### 公式推导过程
考虑下面这样一个经过简化后的神经网络模型：
![简化神经网络](/assets/images/2022/04/10/bDs8vBU-uU2FtIJuY1l3fdSmQ03FNA6PUtOTuhcVzdqdSvqk0ZqSLaREBT9Q6ys66ZKOz4mfqIVz3JXpee3K8Kb7_PWFKjZkbYywR4GclcgaobUafxeE8WfkonOcBCL2.jpeg)

在这个模型中，输入为 $x_1$ 和 $x_2$，预测结果为 $y_1$ 和 $y_2$。实际的真实值为 $T_1$ 和 $T_2$ 。Activation Function 使用 Sigmoid 函数 $g(x) = \frac{1}{1 + e^{-x}}$。  Loss Function 使用 Mean squared error loss： $J(W) = \frac{1}{n} \sum_{i=1}^{n} (y^{(i)} - f(x^{(i)}, W))^2$。

在推导公式之前，先复习一下 [Chain Rule](https://en.wikipedia.org/wiki/Chain_rule) ：
![Chain Rule](/assets/images/2022/04/10/chapter14-2.png)

首先，从正向来看：

$$
\begin{align}
& \text y_1 = g( z_{2,1} ) = g( g( z_{1,1} ) \times w_5 + g( z_{1,2} ) \times w_6 ) \\
& \text y_1 = g( g( x_1 \times w_1 + x_2 \times w_2 ) \times w_5 + g( x_1 \times w_3 + x_2 \times w_4 ) \times w_6 )  \\
& \text {} \\
& \text y_2 = g( z_{2,2} ) = g( g( z_{1,1} ) \times w_7 + g( z_{1,2} ) \times w_8 ) \\
& \text y_2 = g( g( x_1 \times w_1 + x_2 \times w_2 ) \times w_7 + g( x_1 \times w_3 + x_2 \times w_4 ) \times w_8 )  \\
\end{align}
$$

因此可以很容易得到 Loss Function $J(W)$：

$$
\begin{align}
& \text J(W) = \frac{1}{2} (T_1 - y_1)^2 + \frac{1}{2} (T_2 - y_2)^2
\end{align}
$$

接下来，先求 $\frac{\partial J(W)}{w_5}$，由 chain Rule 可得：

$$
\begin{align}
& \text {} \frac{\partial J(W)}{\partial w_5} = \frac{\partial J(W)}{\partial y_1} \times \frac{\partial y_1}{\partial Z_(2,1)} \times \frac{\partial Z_(2,1)}{\partial w_5}  \\
& \text {} \frac{\partial J(W)}{\partial w_5} = ( y_1 - T_1 ) \times g(Z_{2,1}) \times (1 - g(Z_{2,1})) \times g(Z_{1,1}) \\
\end{align}
$$

同理，可求得 $\frac{\partial J(W)}{w_6}$、 $\frac{\partial J(W)}{w_7}$、 $\frac{\partial J(W)}{w_8}$ 。

接下来，再求 $\frac{\partial (W)}{w_1}$，由 chain Rule 可得：

$$
\begin{align}
& \text {} \frac{\partial J(W)}{\partial w_1} = \frac{\partial J(W)}{\partial Z_{1,1}} \times \frac{\partial Z_{1,1}}{\partial w_1}  \\
& \text {} \frac{\partial J(W)}{\partial w_1} = \frac{\partial J(W)}{\partial g(Z_{1,1})} \times \frac{\partial g(Z_{1,1})}{\partial Z_{1,1}} \times \frac{\partial Z_{1,1}}{\partial w_1} \\
\end{align}
$$

其中：  

$$
\begin{align}
\text {} \frac{\partial J(W)}{\partial g(Z_{1,1})} &= \frac{\partial J(W)}{\partial y_1} \times \frac{\partial y_1}{\partial g(Z_{1,1})} \\
&+ \frac{\partial J(W)}{\partial y_2} \times \frac{\partial y_2}{\partial g(Z_{1,1})} \\
{} \\
\text {} \frac{\partial J(W)}{\partial g(Z_{1,1})} &= \frac{\partial J(W)}{\partial y_1} \times \frac{\partial y_1}{\partial Z_{2,1}} \times \frac{\partial Z_{2,1}}{\partial g(Z_{1,1})} \\
&+ \frac{\partial J(W)}{\partial y_2} \times \frac{\partial y_2}{\partial Z_{2,2}} \times \frac{\partial Z_{2,2}}{\partial g(Z_{1,1})} \\
{} \\
\text {} \frac{\partial J(W)}{\partial g(Z_{1,1})} &= (y_1 - T_1) \times g(Z_{2,1}) \times (1 - g(Z_{2,1})) \times w_5 \\
&+ (y_2 - T_2) \times g(Z_{2,2}) \times (1 - g(Z_{2,2})) \times w_7 \\
\end{align}
$$

最终合在一起就是：  

$$
\begin{align}
\text {} \frac{\partial J(W)}{\partial w_1} &= ((y_1 - T_1) \times g(Z_{2,1}) \times (1 - g(Z_{2,1})) \times w_5 \\
& + (y_2 - T_2) \times g(Z_{2,2}) \times (1 - g(Z_{2,2})) \times w_7) \times g(Z_{1,1}) \times (1 - g(Z_{1,1})) \times x_1)
\end{align}
$$

同理，可求得 $\frac{\partial J(W)}{w_2}$、 $\frac{\partial J(W)}{w_3}$、 $\frac{\partial J(W)}{w_4}$ 。

事实上，对于任意一个神经元节点来说：
![任意一个神经元](/assets/images/2022/04/10/IMG_0060.jpg)
都有：

$$
\begin{align}
& \text {} \frac{\partial J(W)}{\partial Z_{t,i}} = g(Z_{t,i}) \times \sum_{k=i}^{j} w_{t+1, k} \frac{\partial J(W)}{\partial Z_{t+1,k}} \\
\end{align}
$$

这也就是为什么这个操作叫 **Backpropagation** 的原因。

#### Python 代码实现

以下是使用 Python 代码重现了 Backpropagation 的过程。  

```python
import numpy as np

def sigmoid(a):
    return 1 / (1 + np.exp(-a))

def diff_sigmoid(a):
    return ( 1 - sigmoid(a)) * sigmoid(a)

def loss_function(actual, predict):
    return np.sum( ( ( actual - predict ) ** 2 ) / 2 )

def p_JW_of_H(layer, row_num):
    if layer == max_layer:
        return (y[row_num] - t[row_num]) * diff_sigmoid(A[layer - 1][row_num])
    else:
        out = 0
        m = A[layer - 1].shape[0]

        for i in range(m):
            out += p_JW_of_H(layer+1, row_num) * W[layer][row_num, i] * diff_sigmoid(A[layer - 1][row_num])

        return out

def p_JW_of_W(row_num, col_num, layer):
    return p_JW_of_H(layer, row_num) * X[layer - 1][col_num]


x1 = np.array([[0.05], [0.10]])

w1 = np.array([[0.15, 0.20], [0.25, 0.30]])
a1 = w1.dot(x1)

x2 = sigmoid(a1)

w2 = np.array([[0.40, 0.45], [0.50, 0.55]])
a2 = w2.dot(x2)

y = sigmoid(a2)
t = np.array([0.01, 0.99])

X = [x1, x2]
A = [a1, a2]
W = [w1, w2]

eta = 0.01
for _ in range(10000):
    for l in range(len(X)):
        for r in range(W[l].shape[0]):
            for c in range(W[l].shape[1]):
                W[l][r, c] = W[l][r, c] - eta * p_JW_of_W(r, c, l + 1)

    A[0] = W[0].dot(X[0])
    X[1] = sigmoid(A[0])
    A[1] = W[1].dot(X[1])

    y = sigmoid(A[1])
```


## 神经网络优化实践

现实中的神经网络训练往往是一个很让人头疼的问题，抛开数据量很大，计算复杂这一点。另一方面的问题是在 Gradient Descent 算法中步长 $\eta$ 选择上。选择的步长过大，会导致结果轻易越过 W 的局部最优，离预期差十万八千里。选择的步长过小，会导致每一次的迭代中 W 的值几乎没有变化，需要浪费大量的时间和计算资源才能得到预期结果。  
针对这一问题，最先想到的解决办法自然就是尝试不同的步长值，试出一种时间和计算资源成本不高且最终结果良好的步长值。采用这种方式的一个最主要的问题是缺乏通用性，每一次训练新的模型都需要经历同样的试验步长的步骤。  
另一种可能的解决办法是不使用固定的步长，而是根据 Gradient 的大小、学习时间的长短、权重 W 的数量等因素设计一套算法，能够在每次迭代过程中自动选择最佳步长。实际上，类似 tensorflow 这样的框架里已经集成了一些实现上述目标的算法，像是：`SGD`、 `Adam`、 `Adadelta` 等等。  
![tensorflow-eta-algo](/assets/images/2022/04/10/L3gMHLgTBZnj_EmsK0IKBOeUvGoL64PDg0XjJgz2ArhWxdaalydFRA7yG1sqe8cdHjugugbJlBoDCOVWyG65557Bnty6sT7f2xXqaNib-UaL2mpfQ08_n30ATLBNE_EH.png)


## Mini-batches

正如同前面提到的，现实中使用的神经网络结构一般都十分复杂。要在百万次级别的迭代中计算全部数据的 Loss Function J(W) 对权重 W 的偏微分是一件极其困难的事。换个角度讲，也可以说是一件极其低效率的事，毕竟在每一次的迭代中我们期望的实际上是权重 W 向理想中的值靠近一点点。那么，更 "经济" 的做法是使用 **[Stochastic Gradient Descent](https://en.wikipedia.org/wiki/Stochastic_gradient_descent)**，也就是说，每次从训练的样本中随机挑选 n 个，计算这些样本 Gradient 的平均值，将得到的平均值带入后续的计算中，指导权重 W 的更新。 
![Stochastic Gradient Descent](/assets/images/2022/04/10/Kl6rWhbj-v__EVhdM2ELeld1HDxE-oSXEz-e88fqX46Ks40WBGHxub5XRsaHaqyEnNbdBk4VKYBsAhBE7Dl4bGkdbG8x8rNcbQnCVFDUbDNkh5elbxvnXtaRzdv-ZmgO.png)


## 过拟合的问题

所谓的过拟合，是指模型结构过于复杂，在对训练样本的预测结果变现异常完美，但面对新数据时却表现的一塌糊涂。  
![过拟合](/assets/images/2022/04/10/UTAmbHMK5_h_YINrMDTATZp2dp9L15QJGW7ai9xjEvTwmFeOvmDTMbzTWof7gYSUoZ6_HOs1Zg90HyVDpHk5kF6wodOb6XeCXB42pKqQT_DgQHuGpq8XinaPf8jm4T0C.png)
解决这个问题有两种思路，一种方法是 **Dropout**，在每次迭代的过程中随机将部分（一般是 50%） 中间隐藏层神经元的输出设置为零，以此来避免其中单独某一个节点对输出结果产生决定性影响。
![Dropout](/assets/images/2022/04/10/PbjXv6iy17qJozb2RS2v8jQjEdHuQkEhXr9JSm1yX9mADql_TML43lm1yY3NTI6DARwWDYj8TQWL4cyhjnH7XpAkoD41xqreSCpZQ10ZQc7XzCH7X_L8wMQaoLBYsoc2.png)
另一种方法是 **Early Stopping**，也就是在过拟合问题出现之前提前停止训练。在每次训练过程中，把数据按比例（通常是 4:1）拆分成两部分：训练数据和测试数据，测试数据不参与训练，只用于验证模型训练结果。在每一次的迭代中观察模型对训练数据和测试数据的表现，如果在某一时刻模型对训练数据 Loss 保持降低的同时对测试数据 Loss 有上升的趋势，说明模型可能开始出现过拟合，应该在此时停止训练。
![Early Stopping](/assets/images/2022/04/10/qmljpsfEPyMIj1vonVKaE4EXgECDjFdD17fnD-qcigTl2mOmmrWLLjTBanNvgDYAX2QWdHdfEpR8wT8o68MD5s2ukXead1blWhOetjlMxUO_m3qT-qis8sLOizGNejLb.png)


## 总结


