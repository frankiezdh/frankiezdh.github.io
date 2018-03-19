---
title: golang实现梯度下降算法
date: 2018-03-19 18:42:27
tags: [golang, 算法]
---

在实际工作过程中经常有一类求极值的最优化问题（比如说深度学习算法中BP算法）。对于这类问题通常可以通过梯度下降算法来计算数值极限值。以求二次函数的min(2x^2 + 3x + 1)最小值为例

1. 首先计算二次函数的“梯度”，对于二次函数来说梯度就是导数——即4x+3
2. 然后x随机赋一个值
3. 再接着x的梯度下降的方向变化，伪码表示为：x += (-1) * （4x + 3）
4. 判断是否已经到达极限值，如果达到则停止，如果没有到达跳转到第3步，不断迭代

对于上述第3步，为了防止x在范围移动导致寻优过程收敛过慢，一般会设置再乘以一个比率，一般0.001到0.5之间不等。这个比率即是深度学习中的“学习速率”参数。

<!--more-->

整个学习过程可以用下图来演示：

![梯度下降.png](https://i.loli.net/2018/03/19/5aaf9938b27dc.png)


相关代码如下
```golang
package main

import (
	"fmt"
	"math"
)

func main() {
	x, y := min(2.0)
	fmt.Printf("x = %.3f, f(x) = %.3f", x, y)
}

const RATE = 0.001
const BIAS = 0.000001

func f(x float64) float64 {
	return 2*math.Pow(x, 2) + 3*x + 1
}

func df(x float64) float64 {
	return 4*x + 3
}

func min(start float64) (x, y float64) {
	for {
		value := f(start)
		start += -1 * RATE * df(start)

		if math.Abs(value-f(start)) <= BIAS {
			return start, value
		}
	}
}
```
