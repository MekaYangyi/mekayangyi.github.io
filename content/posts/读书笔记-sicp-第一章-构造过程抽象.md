---
title: '[读书笔记] sicp 第一章 构造过程抽象'
date: 2016-09-10 08:31:58
categories: 
- 读书笔记
tags:
- sicp
- 计算机程序的构造与解释
---
# 关于读书的目的
很多时候对于一本比较复杂的书，你在到达一定阶段的时候是难以读进去的。Sicp我曾经在几个月前尝试的去读了第一章。很快就读完的，但是对于其的理解实在浅薄。
我希望在这一次的阅读的过程中能够顺利的过一次不本书有所收获。
在接下来的日子里，我会记录下读书笔记以及习题的解答。
这是一个开始。

# 构造过程抽象
每一种强有力的语言为此提供了三种机制
- 基本的表达形式， 用于表示语言所关心的最简单的个体。
- 组合的方法，通过它们可以从简单的东西出发构建出复合的元素。
- 抽象的方法，通过它们可以为 复合对象命名，并将它们作为单元去操作。

在程序设计中，我们需要处理两类元素：过程和数据。非形式的说，数据是一种我们希望去操作的”东西“，而过程是有关操作这些数据的规则的描述。

```lisp
(* ( + 2 ( * 5 6 ))
	( + 3 5 7))

(define (square x) (* x x))

(square 21)
441

```

<!--more-->
## 过程应用的带换模型
对于符合过程，过程的应用的计算过程是：
- 将复合过程应用于实际参数，就是在将过程体中的每个形参用相应的实参取代之后，对这一过程求值。

```lisp
(f 5)

(sum-of-squares(+ 5 1) (* 5 2))

(+ (square 6) (square 10))

(+ (* 6 6) (* 10 10))

(+ 36 100)

136
```
这种计算过程称为过程应用的代换模型。

### 应用系和正则序
完全展开而后归约的求值模型是正则序求值。
先求值参数而后应用的方式为应用序求值。

### 条件语法
```lisp

(define (abs x)
	(cond ((> x 0) x)
			  (( = x 0) 0)
			  (( < x 0) (- x))))


(define (abs x)
	(cond (( < x 0) (- x))
			  (else x)))



(define (abs x)
	(if	( < x 0) 
        (- x)
		x))
```
## 过程与它们所产生的计算
一个过程也就是一种模式，它描述了一个计算过程的局部演化方法。

两种描述阶乘的方式。（lisp的迭代不是说形式上的迭代，实际上如果说调用本身的层面上还是递归，但是思想是迭代的思想，应用序来看的话。）
```lisp
(define (factorial n)
	(if  (= n 1)
		 1
		 (* n (factorial (- n 1)))))

;迭代的形式
(define (factorial n)
	(fact-iter 1 1 n))

;应用序的话会优先求值，那么就不会有上面的方式那么深的层次，尝试求值都能够求出来。
(define (fact-iter product counter max-count)
	(if (> counter max-count)
		product
		(fact-iter (* counter product)
					   (+ counter 1)
					   max-count)))
```
迭代计算过程就是那种其状态可以用固定数据的状态变量描述的计算过程，而与此同时，又存在着一套固定的规则，描述了计算过程从一个状态到下一个状态的转换时候，这些变量的更新方式。还有一个结束检测，它描述着一计算过程应该终止的条件。
在计算n!时候，所需要的计算不走随着n线性增长，这种过程称为线性迭代过程。
在迭代的情况下，在计算过程中的任何一点，那几个程序变量都提供了有关计算状态的一个完整描述。而递归计算过程而言，这里存在着另外的一些隐含信息，它们并未保存在程序变量里面，而是由解释器维持着，指明了在所推迟的运算所形成的链条里的漫游中，这以计算过程处在合出。链条越长，需要保存的信息越多。

###  换零钱的实现

采用递归过程：
将总数为a的现金换成n中硬币的不同方式的数目等于
 -  将现金数a换成除第一种硬币之外的所有其他硬币的不同方式数目，加上
 - 将现金数a-d换成所有种类的硬币的不同数目，其中d是第一种硬币的币值
```lisp
(define (first-denomination kinds-of-coins)
  (cond ((= kinds-of-coins 1) 1)
        ((= kinds-of-coins 2) 5)
        ((= kinds-of-coins 3) 10)
        ((= kinds-of-coins 4) 25)
        ((= kinds-of-coins 5) 50)))
  
(define (cc amount kinds-of-coins)
  (cond ((= amount 0) 1)
        ((or (< amount 0) (= kinds-of-coins 0)) 0)
        (else (+ (cc amount
                     (- kinds-of-coins 1))
                 (cc (- amount
                        (first-denomination kinds-of-coins))
                     kinds-of-coins)))))
(define (count-change amount)
  (cc amount 5))

(count-change 100)
```

### 增长的阶
空间与时间的消耗。大O记号

一个例子求幂

```lisp
(define (square n)
  (* n n))
(define (fast-expt b n)
  (cond ((= n 0) 1)
        ((even? n) (square (fast-expt b (/ n 2))))
        (else (* b (fast-expt b (- n 1))))))
(fast-expt 2 2)
```
### 最大公约数
```lisp
(define (gcd a b)
    (if ( = b 0)
        a
        (gcd b (remainder a b))))
```
###  素数检测
两种计算方法
```lisp
(define (square n)
  (* n n))

(define (smallest-divisor n)
  (find-divisor n 2))
  
;寻找最小因子
(define (find-divisor n test-divisor)
  (cond ((> (square test-divisor) n) n);根号n为检测上限
        ((divides? test-divisor n) test-divisor)
        (else (find-divisor n (+ test-divisor 1)))));查找下一个

(define (divides? a b)
  (= (remainder b a) 0))
  
;最小因子等于本身的时候为素数
(define (prime n)
  (= n (smallest-divisor n)))
```
费马检测
```lisp
(define (square n)
  (* n n))

(define (expmod base exp m)
  (cond ((= exp 0) 1)
        ((even? exp)
         (remainder (square (expmod base (/ exp 2) m))
                    m))
        (else
         (remainder (* base (expmod base (- exp 1) m))
                    m))))
(define (try-it a n)
  (= (expmod a n n) a))

;使用随机数来测试
(define (fermat-test n)
  (try-it (+ 1 (random (- n 1))) n))

;通过多次的费马测试来概率的推断是不是位素数
(define (fast-prime? n times)
  (cond ((= times 0) true)
        ((fermat-test n) (fast-prime? n (- times 1)))
        (else false)))
```
## 高阶函数抽象
以过程作为参数。以过程作为返回值。这类操作过程的过程称为高阶过程。

```lisp
#lang racket
(define (cube n)
  (* n n n))

(define (sum term a next b)
  (if (> a b)
      0
      (+ (term a)
         (sum term (next a) next b))))

(define (inc n) (+ n 1))

(define (sum-cubes a b)
  (sum cube a inc b))

(sum-cubes 1 10)

(define (integral f a b dx)
  (define (add-dx x) (+ x dx))
  (* (sum f (+ a (/ dx 2.0)) add-dx b)))

(integral cube 0 1 0.01)
```

### 用lambda构造过程
```lisp
((lambda (x) (+ 4 x)))
```
匿名的过程，对于一些简单的过程构造适合
### 用let创建局部变了
一种方法是利用辅助过程去约束局部变量
```lisp
(define (f x y)
  (let ((a (+ 1 x))
        (b (+ 2 y)))
  (+ a b)))
(f 1 2)
```
### 过程作为一般性的方法

#### 通过区间折半寻找方程的根
```lisp
(define(close-enough? x y)
  (< (abs (- x y)) 0.001))

(define (average x y)
  (/ (+ x y) 2))

(define (search f neg-point pos-point)
  (let ((midpoint (average neg-point pos-point)))
    (if (close-enough? neg-point pos-point)
        midpoint
        (let ((test-value (f midpoint)))
          (cond ((positive? test-value)
                 (search f neg-point midpoint))
                ((negative? test-value)
                 (search f midpoint pos-point))
                (else midpoint))))))

(define (half-interval-method f a b)
  (let ((a-value (f a))
        (b-value (f b)))
    (cond ((and (negative? a-value) (positive? b-value))
           (search f a ))
          ((and (negative? b-value) (positive? a-value))
           (search f b a))
          (else
           (error "Values are not of opposite sign" a b)))))

(half-interval-method sin 2.0 4.0)          
```

### 寻找函数不动点
```lisp
(define tolerance 0.00001)

(define (average x y)
  (/ (+ x y) 2))

(define (fixed-point f first-guess)
  (define (close-enought? v1 v2)
    (< (abs (- v1 v2)) tolerance))
  (define (try guess)
    (let ((next (f guess)))
      (if (close-enought? guess next)
          next
          (try next))))
  (try first-guess))

(fixed-point cos 1.0)
;不收敛
(define (sqrt x)
  (fixed-point (lambda (y) (/ x y))
               1.0))
;引入阻尼
(define (sqrt x)
  (fixed-point (lambda (y) (average y (/ x y)))
               1.0))
```
### 过程作为返回值
新的开方方法
```lisp
(define (average-damp f)
  (lambda (x) (average x (f x))))

(define (sqrt x)
  (fixed-point (average-damp (lambda(y) (/ x y)))))
```
新的牛顿法

```lisp
(define (deriv g)
  (lambda (x)
    (/ (- (+ x dx) (g x))
       dx)))

(define (newton-transform g)
  (lambda (x)
    (- x (/ (g x) ((deriv g) x)))))
```
# 抽象和第一级过程
复合过程是一种至关重要的抽象机制，因为它使得我们能将一般性的计算方法，用这一程序设计语言里的元素明确描述。现在我们又看到，高阶函数能如何去操作这一些一般性的方法，以便建立起进一步的抽象。
作为编程者，我们应该对这类可能性保持高度敏感，设法从中识别出程序里的基本抽象，基于它们去进一步构造，程序设计专家指导的如何根据工作中的情况，去选择合适的抽象层次。但是，能够基于这种抽象去思考确实是最重要的，只有这样才可能在新的上下文中去应用它们。高阶的过程的重要性，就在于使我们能够显式的用程序设计语言的要素去描述这些抽象，使我们能够像操作其他计算元素一样去操作它们。
一般而言，程序设计语言总会对计算元素的可能使用方式强加上某些显式。带有最少限制的元素被称为具有第一级状态。第一级元素的某些权利与特权包括:
 - 可以用变量命名
 - 可以提供给过程作为参数
 - 可以由过程作为结果返回
 - 可以包含在数据结构中
 lisp给了过程完全的第一级的状态。
