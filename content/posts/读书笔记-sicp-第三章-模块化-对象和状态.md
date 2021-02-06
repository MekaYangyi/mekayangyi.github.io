---
title: '[读书笔记] sicp 第三章 模块化 对象和状态'
date: 2016-09-10 08:33:32
categories: 
- 读书笔记
tags:
- sicp
- 计算机程序的构造与解释
---
有效的程序综合还需要一些组织原则，它们能够指导我们系统化地完成系统的整体设计。特别的需要一些能够帮助我们构造起模块化的大型系统策略，也就是说，使这些系统能够“自然地”划分为一些具有内聚力的部分，使这些部分可以分别进行开发和维护。
有一种非常强有力的设计策略，特别适合用于构造那类模拟真实物理系统的程序，那就是基于被模拟程序的结构去设计程序的结构。那么有关的物理系统里的每一个对象，我们构造起一个与之对象的计算对象；对该系统里的每种活动，我们在自己的计算系统里顶一种符号操作。采用这一策略时的希望是，在需要针对系统中的新对象或者新活动扩充对应的计算模型时候，我们能够不必对程序的组织方面做得很成功，那么在需要添加新特城或者排除旧东西里的错误时候，就只需要在系统里的一些小局部中工作。
本章研究两种特点鲜明的策略。第一种策略将注意力集中在对象上，将大型系统看成一大批对象，它们的行为可能随着时间的进展而不断的变化。另一种组织策略将注意力集中在流过的系统的信息流上，非常像电子工程师观察一个信号处理系统。
对于对象途径而言，我们必须关注计算对象可以怎样变化而又同时保持其标识。这将迫使我们抛弃老的计算的代换模型，转向更机械式的，理论上也更不同意把握的计算的环境模型。在处理对象、变化和标识时，各种困难的基本根源在于我们需要在这一计算模型中与时间搏斗。如果允许程序并发指向的可能性，事情会变得更困难许多。流方式特别能用于松解在我们模型中对时间的模拟与计算机求值过程中的各种时间发生的顺序。我们将通过一种称为延时求值的技术做到这一点。
# 赋值和局部状态
一个由许多对象组成的系统里，其中的这些对象极少会是完全独立的。每个对象都可能通过交互作用，影响其他对象的状态，所谓交互就是建立起一个对象的状态变量与其他对象的状态变量之间的联系。确实如果一个系统中的状态变量可以分组，形成一些内部紧密结合的子系统，每个子系统与其他子系统之间只存在松散的练习，此时将这个系统看作是由一些独立对象组成的观点就会特别有用。
对于一个系统的这种观点，有可能成为组织这一系统的计算模型的有力框架。要使这样的一个模型成为模块化的，就要求它能够分解为一批计算对象，使它们能够模拟系统里的实际对象。每个计算对象必须有它自己的一些局部状态变量，用于描述实际对象的状态。对于被模拟系统里的对象的状态是随着时间的变化的，与它们对象的计算对象的状态也必须变化。
<!--more-->
## 局部状态变量
```lisp
(set! <name> <new-value>);设置值
(begin <exp1> <exp2>);顺序求值

(define balance 100)

(define (withdraw amount)
  (if (>= balance amount)
      (begin (set! balance (- balance amount))
             balance)
      "Insufficient funds"))
```
上面使用了全局变量
下面使用局部变量

```lisp
(define new-withdraw
  (let ((balance 100))
    (lambda (amount)
      (if (>= balance amount)
          (begin (set! balance (- balance amount))
                 balance)
          "Insufficient funds"))))
```
构建一个提款机
```lisp
(define (make-withdraw balance)
  (lambda (amount)
    (if (>= balance amount)
        (begin (set! balance (- balance amount))
               balance)
        "Insufficient funds")))
```
创建一个账户
```lisp
(define (make-withdraw balance)
  (define (withdraw amount)
    (if (>= balance amount)
        (begin (set! balance (- balance amount))
               balance)
        "Insufficient funds"))
  (define (deposit amount)
    (set! balance (+ balance amount))
    balance)
  (define (dispatch m)
    (cond ((eq? m 'withdraw) withdraw)
          ((eq? m 'deposit) deposit)
          (else (error "Unknoew request --MAKE-ACCOUNT"
                       m))))
  dispatch)
```
## 引进赋值带来的收益
能够简化一部分需要变量状态的过程。

## 引进赋值的代价
相比函数是编程，输入什么结果就是什么。显然引进赋值让程序变得更复杂。
需要存在一个位置存储变量。
### 命令式程序设计的缺陷
与函数式程序设计相对，广泛采用的赋值程序设计被称为命令是程序设计。
求值顺序需要保证。

# 求值的环境模型
类似于C++的区域。
过程也是对象。
调用过程就会产生新的上下文环境，过程内的是过程内的环境。过程外全局环境等。

过程应用的环境模型两条规则：
1. 将一个过程对象应用于一集实际参数，将构造出一个新框架，其中将过程的形式参数约束到调用时的实际参数，而后在构造起的这一新环境的上下文中求
2. 相对于一个给定的环境求值一个lambda表达式，将创建其一个过程对象，这个过程对象是一个序对，由该lambda表达式的征文和一个指向环境的指针组成，这一指针指向的就是创建这个过程对象时候的环境。

## 简单过程的应用
## 将框架看做局部状态的展台
## 内部定义
1. 局部过程的名字不会与包容它们的过程之外的名字互相干扰，这是因为这些局部过程名 都是该过程运行时创建的框架里约束的，而不是在全局环境里约束的。
2. 局部过程只需将包含着它们的过程的形参作为自由变量，就可以访问该过程的实际参数，这是因为对于局部过程体的求值所在的环境是外围过程求值所在的环境的下属。

# 用变动的数据做模拟
```lisp
(define (append! x y)
  (set-cdr! (last-pair x) y)
  x)

(define (last-pair x)
  (if (null? (cdr x))
      x
      (last-pair (cdr x))))
```
### 共享与相等
共享会导致多个对象都拥有同一个对象，修改一个会导致另外的也跟着被修改。
```lisp
(eq? x y);检查是不是一个对象
```
###  改变也就是赋值
主要是构建了一前一后两个指针。
```lisp
(define (cons x y)
  (define (set-x! v) (set! x v))
  (define (set-y! v) (set! y v))
  (define (dispatch m)
    (cond ((eq? m 'car) x)
          ((eq? m 'cdr) y)
          ((eq? m 'set-car!) set-x!)
          ((eq? m 'set-cdr!) set-y!)
          (else (error "Undefined operation -- CONS" m))))
  dispatch)

(define (car z) (z 'car))
(define (cdr z) (z 'cdr))
(define (set-car! z new-value)
  ((z 'set-car!) new-value)
  z)
(define (set-cdr! z new-value)
  ((z 'set-cdr!) new-value)
  z)
```
##  队列的表示
```lisp
;构造函数
(define (make-queue) (cons '() '()))
;选择函数
(define (front-queue queue)
  (if (empty-queue? queue)
      (error "FRONT called with an empty queue" queue)
      (car (front-ptr queue))))
;检测队列是否为空
(define (empty-queue? queue) (null? (front-ptr queue)))
;改变函数
(define (insert-queue! queue item)
  (let ((new-pair (cons item '())))
    (cond ((empty-queue? queue)
           (set-front-ptr! queue new-pair)
           (set-rear-ptr! queue new-pair)
           queue)
          (else
           (set-cdr! (rear-ptr queue) new-pair)
           (set-rear-ptr! queue new-pair)
           queue))))

(define (delete-queue! queue)
  (cond ((empty-queue? queue)
         (error "DELETE! called with an empty queue" queue))
        (else
         (set-front-ptr! queue (cdr (front-ptr queue)))
         queue)))

(define (front-ptr queue) (car queue))
(define (rear-ptr queue) (cdr queue))
(define (set-front-ptr! queue item) (set-car! queue item))
(define (set-rear-ptr! queue item) (set-car! queue item))
```
## 表格的表示
```lisp
(define (lookup key table)
  (let ((record (assoc key (cdr table))))
    (if record
        (cdr record)
        false)))

(define (assoc key records)
  (cond ((null? records) false)
        ((equal? key (caar records)) (car records))
        (else (assoc key (cdr records)))))

(define (insert! key value table)
  (let ((record (assoc key (cdr table))))
    (if record
        (set-cdr! record value)
        (set-cdr! table
                  (cons (cons key value) (cdr table))))))
(define (make-table)
  (list '*table*))
```
## 数字电路的模拟器

# 并发：时间本质是个问题
和普通的并发问题是一致的。
串行化共享部分。
```lisp
(define s make-serializer)
(define (make-account balance)
  (define (withdraw amount)
    (if (>= balance amount)
        (begin (set! balance (= balance amount))
               balance)
        "Insufficient funds"))
  (define (deposit amount)
    (set! balance (+ balance amount))
    balance)
  (let ((protected (make-serializer)))
    (define (dispatch m)
      (cond ((eq? m 'withdraw) (protected withdraw))
            ((eq? m 'deposit) (protected deposit))
            ((eq? m 'balance) balance)
            (else (error "Unknown request --MAKE-ACCOUNT"
                         m))))
    dispatch))
```

## 串行化的实现
互斥元同步机制。
```lisp
(define (make-serializer)
  (let ((mutex (make-mutex)))
    (lambda (p)
      (define (serialized-p . args)
        (mutex 'acquire)
        (let ((val (apply p args)))
          (mutex 'release)
          val))
      serialized-p)))


(define (make-mutex)
  (let ((cell (list false)))
    (define (the-mutex m)
      (cond ((eq? m 'acquire)
             (if (test-and-set! cell)
                 (the-mutex 'acquire)))
            ((eq? m 'release) (clea! cell))))
    the-mutex))

(define (test-and-set! cell)
  (if (car cell)
      true
      (begin (set-car! cell true)
             false)))
```
