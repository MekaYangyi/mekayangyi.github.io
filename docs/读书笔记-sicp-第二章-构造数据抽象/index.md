# [读书笔记] sicp 第二章 构造数据抽象


第一章关注的是计算过程，以及过程在程序中所扮演的角色。
本章，讲将数据对象组合起来，形成复合数据的方式。
复合数据：能够提升我们在设计程序时所位于的概念层次，提高设计的模块性，增强语言的表达能力。
将程序中处理数据对象的表示的部分与处理数据对象的使用部分相互 隔离的技术，称为数据抽象。
复合数据中的一个关键性思想是闭包的概念，也就是说，用于组合数据对象的粘合剂不但能用于组合基本的数据对象，也能组合复合数据对象。
复合数据对象能够称为以混合与匹配的方式组合程序模块的方便接口。

# 数据抽象引导
数据抽象的基本思想，就是设法构造出一些使用复合数据对象的程序，使它们就像是在抽象数据上操作一样。

## 有理数的算数运算
假定存在构造函数与选择函数
```lisp
(make-rat n d);返回一个有理数，分子是整数n，分母是整数d
(numer x);返回有理数x的分子
(denom x));返回有理数x的分母
```
那么可以定义以下的规则
```lisp
;加法
(define (add-rat x y)
  (make-rat (+ (* (numer x) (denom y))
               (* (numer y) (denom x)))
            (* (denom x) (denom y))))
;减法
(define (sub-rat x y)
  (make-rat (- (* (numer x) (denom y))
               (* (numer y) (denom x)))
            (* (denom x) (denom y))))
;乘法
(define (mul-rat x y)
  (make-rat (* (numer x) (numer y))
            (* (denom x) (denom y))))
;除法
(define (div-rat x y)
  (make-rat (* (numer x) (denom y))
            (* (denom x) (numer y))))
;等于？
(define (equal-rat? x y)
  (= (* (numer x) (denom y))
     (* (numer y) (denom x))))
```
<!--more-->
### 序对
lisp存在基本过程cons，car,cdr。存在下列关系
```lisp
(define x (cons 1 2))

(car x)
1

(cdr x)
2
```

### 有理数的表示
利用序对完成有理数的实现
```lisp
(define (make-rat n d)
  (cons n d))

(define (numer x)
  (car x))

(define (denom x)
  (cdr x))
```
打印有理数
```lisp
(define (print-rat x)
  (newline)
  (display (numer x))
  (display "/")
  (display (denom x)))
```
化简有理数
```lisp
(define (make-rat n d)
  (let ((g (gcd n d)))
  (cons (/ n g) (/ d g))))
```

## 抽象屏障
每一个层次中国策构成了所定义的抽象 屏障的接口，联系起系统中的不同层次。使得系统简单，修改容易。

## 数据意味着什么
一般而言，我们总可以将数据定义为一组适当的选择函数和构造函数，以及使这些过程成为一套合法的表示，它们必须满足一组特定的条件。
使用过程实现cons car cdr
```lisp
(define (cons x y)
  (define (dispatch m)
    (cond ((= m 0) x)
          ((= n 1) y)
          (else (error "Argument not 0 or 1 -- CONS" m))))
  dispatch)

(define (car z) (z 0))

(define (cdr z) (z 1))
```
尽管实际语言的实现不是上面的形式，但是我们定义的函数已经能够正常完成工作了。过程和数据的界限被模糊，满足关于序对的描述。

# 层次性数据和闭包性质
某种组合数据对象的操作满足闭包性质，那就是说，通过它组合起数据对象得到的结果本身可以通过同样的操作再进行组合。闭包性质是任何一种组合功能的威力的关键要素，因为它使我们能够建立起层次性的结构，这种结构由一些部分构成，而其中的各个部分又是由它们的部分构成，并且可以继续下去。

## 序列的表示
```lisp
(list <a1> <a2>...<an>)
;等于
(cons <a1> (cons <a2> (cons ... (cons <an> nil)...)))
```
nil是拉丁词汇nihil的缩写，拉丁语表示什么也没有，表示序对的链结束，代表一个不包含任何元素的序对，空表。

### 表操作
```lisp
;第n个元素
(define (list-ref items n)
  (if (= n 0)
      (car items)
      (list-ref (cdr items) (- n 1))))

(define squares (list 1 4 9 16 25))

(list-ref squares 3)
16

;表长 递归
(define (length items)
  (if (null? items)
      0
      (+ 1 (length (cdr items)))))
  
;表长 迭代
(define (length items)
  (define (length-iter a count)
    (if (null? a)
        count
        (length-iter (cdr a) (+ 1 count))))
  (length-iter items 0))
  
(length squares)
5

(define odds (list 1 3 5 7))

;组合两个表
(define (append list1 list2)
  (if (null? list1)
      list2
      (cons (car list1) (append (cdr list1) list2))))

(append odds squares)
'(1 3 5 7 1 4 9 16 25)
```

### 对表的映射
```lisp
;DrRacket中未定义nil，若要使用nil
(define nil '())

;缩放
(define (scale-list items factor)
  (if (null? items)
      nil
      (cons (* (car items) factor)
            (scale-list (cdr items) factor))))
            
(scale-list (list 1 2 3 4 5) 10)
'(10 20 30 40 50)

;更一般的过程
(define (map proc items)
  (if (null? items)
      nil
      (cons (proc (car items))
            (map proc (cdr items)))))
            
(define (scale-list items factor)
  (map (lambda (x) (* x factor))
       items))
```
map构建一层抽象屏障，将实现表变换的过程的实现，与如何提取表中元素以及组合结果的细节隔离开。

## 层次性结构
树的分支，而那些本身也是序列的元素就形成了树中的子树。
实现count-leaver
```lisp
(define (count-leaves x)
  (cond ((null? x) 0)
        ((not (pair? x)) 1)
        (else (+ (count-leaves (car x))
                 (count-leaves (cdr x))))))
```

##  对树的映射
```lisp
(define (sacle-tree tree factor)
  (cond ((null? tree) nil)
        ((not (pair? tree)) (* tree factor))
        (else (cons (sacle-tree (car tree) factor)
                    (sacle-tree (cdr tree) factor)))))

(define (scale-tree tree factor)
  (map  (lambda (sub-tree)
          (if (pair? sub-tree)
              (scale-tree sub-tree factor)
              (* sub-tree factor)))
        tree))
```

## 序列作为一种约定的接口
强有力的设计原理-使用约定的接口。
考虑下面的过程，它以一棵树为参数，计算出那些值为奇数的叶子的平方和。
```lisp
(define (sum-odd-squares tree)
  (cond ((null? tree) 0)
        ((not (pair? tree))
         (if (odd? tree) (square tree) 0))
        (else (+ (sum-odd-squares (car tree))
                 (sum-odd-squares (cdr tree))))))
```
偶数的斐波那契数列的表。
```lisp
(define (even-fibs n)
  (define (next k)
    (if (> k n)
        nil
        (let ((f (fib k)))
          (if (even? f)
              (cons f (next (+ k l)))
              (next (+ k 1))))))
  (next 0))
```
虽然表面上结构差异大，但是计算的抽象描述存在极大的相似性。
都是从枚举器开始，产生给定的树的树叶组成的信号。信号流过过滤器，过滤掉不符合规则的信号。通过一个映射，转换每一个元素。积累器把所有的元素组合起来。
但是上面的程序是将以上的操作混合在一起。

### 序列操作
修改map过程

```lisp
(map square (list 1 2 3 4 5))
(1 4 9 16 25)
```
过滤器
```lisp
(define (filter predicate sequence)
  (cond ((null? sequence) nil)
        ((predicate (car sequence))
         (cond (car sequence);真就保留
               (filter predicate (cdr sequence))))
        (else (filter predicate (cdr sequence)))))
```
积累器
```lisp
(define (accumulate op initial sequence)
  (if (null? sequence)
      initial
      (op (car sequence)
          (accumulate op initial (cdr sequence)))))
```
fib枚举器
```lisp
(define (enumerate-interval low high)
  (if (> low high)
      nil
      (cons low (enumerate-interval (+ low 1) high))))
```
树叶枚举器
```lisp
(define (enumerate-tree tree)
  (cond ((null? tree) nil)
        ((not (pair? tree)) (list tree))
        (else (append (enumerate-tree (car tree))
                      (enumerate-tree (cdr tree))))))
```
重构
```lisp
(define (sum-odd-squares tree)
  (accumulate +
              0
              (map square
                   (filter odd?
                           (enumerate-tree tree)))))
```

```lisp
(define (even-fibs n)
  (accumulate cons
              nil
              (filter even?
                      (map fib
                           (enumerate-interval 0 n)))))
```
通过提供一个标准不见的库，并使这些不见都有着一些能以灵活方式互相连接的约定接口，将能够进一步推动模块化设计。
在工程设计中，模块化结构是控制复杂性的一种威力巨大的策略。（类似西门子的自动化软件 plc等）

