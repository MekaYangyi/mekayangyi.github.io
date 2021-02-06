# [读书笔记] sicp 第二章 抽象数据的多重表示

对于一个数据对象可以能存在多种有用的表示形式，而且我们也希望所涉及的系统能够处理多种表示形式。
例子：复数的极坐标形式和直角坐标的形式
构造通用型过程：可以在不止一种数据表示上操作的过程。采用的技术：让它们在带有类型标志的数据对象上工作。也就是说，让数据对象包含着它们应该如何处理的明确信息。
# 复数的表示
为一个数据提供了多种操作，存在多种形式
```lisp
(define (real-part z) (car z))

(define (imag-part z) (cdr z))

(define (magnitude z)
  (sqrt (+ (square (real-part z)) (square (imag-part z)))))

(define (angle z)
  (atan (imag-part z) (real-part z)))

(define (make-from-real-imag x y) (cons x y))

(define (make-from-mag-ang r a)
  (cons (* r (cos a)) (* r (sin a))))

(define (add-complex z1 z2)
  (make-from-real-imag (+ (real-part z1) (real-part z2))
                       (+ (imag-part z1) (imag-part z2))))

(define (sub-complex z1 z2)
  (make-from-real-imag (- (real-part z1) (real-part z2))
                       (- (imag-part z1) (imag-part z2))))

(define (mul-complex z1 z2)
  (make-from-mag-ang (* (magnitude z1) (magnitude z2))
                     (+ (angle z1) (angle z2))))

(define (div-complex z1 z2)
  (make-from-mag-ang (/ (magnitude z1) (magnitude z2))
                     (- (angle z1) (angle z2))))
```
<!--more-->
# 带标志数据
认识数据抽象的一种方式是将其看做”最小允诺原则“的一个应用。在实现上面的复数系统的时候，采用两种形式，由选择函数和构造函数形成的抽象屏障，使我们可以把为自己所用的数据对象选择具体表现形式的事情尽量往后推，而且还能够保持系统设计的最大灵活性。
方式，利用类型标志，来确定什么类型，选择什么函数。
增加类型标示
```lisp
(define (attach-tag type-tag contents)
  (cons type-tag contents))

(define (type-tag datum)
  (if (pair? datum)
      (car datum)
      (error "Bad tagged datum -- TYPE-TAG" datum)))

(define (contents datum)
  (if (pair? datum)
      (cdr datum)
      (error "Bad tagged datum -- CONTENTS" datum)))

(define (rectangular? z)
  (eq? (type-tag z) 'rectangular))

(define (polar? z)
  (eq? (type-tag z) 'polar))
```
修改新的直角坐标表示
```lisp
(define (real-part-rectangular z) (car z))

(define (imag-part-rectangular z) (cdr z))

(define (magnitude-rectangular z)
  (sqrt (+ (square (real-part-rectangular z)) (square (imag-part-rectangular z)))))

(define (angle-rectangular z)
  (atan (imag-part-rectangular z) (real-part-rectangular z)))

(define (make-from-real-imag-rectangular x y) (attach-tag 'rectangular (cons x y)))

(define (make-from-mag-ang-rectangular r a)
  (attach-tag 'rectangular (cons (* r (cos a)) (* r (sin a)))))
```
修改极坐标的表现形式
```lisp
(define (real-part-polat z)
  (* (magnitude-polat z) (cos (angle-polat z))))

(define (imag-part-polat z)
  (* (magnitude-polat z) (sin (angle-polat z))))

(define (magnitude-polat z)
  (car z))

(define (angle-polat z)
  (cdr z))

(define (make-from-real-imag-polatr x y)
  (attach-tag 'polar
              (cons (sqrt (+ (square x) (square y)))
                    (atan y x))))

(define (make-from-mag-ang-polat r a)
  (attach-tag 'polar (cons r a)))
```
在通用选择函数都添加检查类型的标志，调用合适的函数。
```lisp
(define (real-patr z)
  (cond ((rectangular? z)
         (real-part-rectangular (contents z)))
        ((polar? z)
         (real-part-polat (contents z)))
        (else (error "Unknown type -- REAL-PART" z))))

(define (imag-patr z)
  (cond ((rectangular? z)
         (imag-part-rectangular (contents z)))
        ((polar? z)
         (imag-part-polat (contents z)))
        (else (error "Unknown type -- IMAG-PART" z))))

(define (magnitude z)
  (cond ((rectangular? z)
         (magnitude-rectangular (contents z)))
        ((polar? z)
         (magnitude-polat (contents z)))
        (else (error "Unknown type -- MAGNITUDE" z))))

(define (angle z)
  (cond ((rectangular? z)
         (angle-rectangular (contents z)))
        ((polar? z)
         (angle-polat (contents z)))
        (else (error "Unknown type -- ANGLE" z))))
```
实现算数操作的时候不需要改变。还是原来的形式就可以。
```lisp
(define (add-complex z1 z2)
  (make-from-real-imag (+ (real-part z1) (real-part z2))
                       (+ (imag-part z1) (imag-part z2))))
```
需要修改下构造函数
```lisp
(define (make-from-real-imag x y)
  (make-from-real-imag-rectangular x y))

(define (make-from-mag-ang r a)
  (make-from-mag-ang-polat r a))
```
# 数据导向的程序设计和可加性
检查一个数据项的类型，并据此去调用某个适当的过程称为基于类型的分派。
在系统设计中，这是一种获得模块性的强有力策略（可能oo是更好的方式，检测类型还是比较麻烦的）。
存在两个弱点：
1. 其中的通用型接口过程，必须知道素有的不同表示。需要检测类型，选择适当函数
2. 独立的表现形式分别设计，需要拥有不同的名字。
那么这就导致，这种实现不具有可加性。在每一次增加一种新形式的时候，需要去修改原过程，修改类型判断，增加代码，修改过程名字。

现在我们需要的是一种能够将系统设计进一步模块化的方法。一种称为数据导向的程序设计都编程技术提供了这种能力（其实在数据中保存能够处理数据的过程，就能够不用选择函数直接处理了嘛）。
（实际上这里讲的是一种注册机制）
假定存在put get来制造表格
```lisp
(define (install-rectangular-package)
  ;internal procedures
  (define (real-part z) (car z))
  (define (imag-part z) (cdr z))
  (define (make-from-real-imag x y) (cons x y))
  (define (magnitude z)
  (sqrt (+ (square (real-part z)) (square (imag-part z)))))
  (define (angle z)
  (atan (imag-part z) (real-part z)))
  (define (make-from-mag-ang r a)
  (cons (* r (cos a)) (* r (sin a))))
  ;interface the rest of the system
  (define (tag x) (attach-tag 'rectangular x))
  (put 'real-part '('rectangular) real-part)
  (put 'imag-part '('rectangular) imag-part)
  (put 'magnitude '('rectangular) magnitude)
  (put 'angle '('rectangular) angle)
  (put 'make-from-real-imag '('rectangular)
       (lambda (x y) (tag (make-from-real-imag x y))))
  (put 'make-from-mag-ang '('rectangular)
       (lambda (r a) (tag (make-from-mag-ang r a))))
  'done)
```
```lisp
(define (install-polar-package)
  ;internal procedures
  (define (magnitude z) (car z))
  (define (angle z) (cdr z))
  (define (make-from-mag-ang r a) (cons r a))
  (define (real-part z)
  (* (magnitude z) (cos (angle z))))
  (define (imag-part z)
   (* (magnitude z) (sin (angle z))))
  (define (make-from-real-imag x y)
    (cons (sqrt (+ (square x) (square y)))
          (atan y x)))
  ;interface the rest of the system
  (define (tag x) (attach-tag 'polar x))
  (put 'real-part '('polar) real-part)
  (put 'imag-part '('polar) imag-part)
  (put 'magnitude '('polar) magnitude)
  (put 'angle '('polar) angle)
  (put 'make-from-real-imag '('polar)
       (lambda (x y) (tag (make-from-real-imag x y))))
  (put 'make-from-mag-ang '('polar)
       (lambda (r a) (tag (make-from-mag-ang r a))))
  'done)
```
下面操作用于访问表格
```lisp
(define (apply-generic op . args)
  (let ((type-tags (map type-tag args)))
    (let ((proc (get op type-tags)))
      (if proc
          (apply proc (map contents args))
          (error
           "No method for these types -- APPLY-GENERIC"
           (list op type-tags))))))
```
那么通用操作。
```lisp
(define (real-part z) (apply-generic 'real-part z))
(define (imag-part z) (apply-generic 'imag-part z))
(define (magnitude z) (apply-generic 'magnitude z))
(define (angle z) (apply-generic 'angle z))
```

```lisp
(define (make-from-real-imag x y)
  ((get 'make-from-real-imag 'rectangular) x y))

(define (make-from-mag-ang r a)
  ((get 'make-from-mag-ang 'polat) r a))
```
## 消息传递
数据导向的程序设计，最关键的思想是通过显式的操作-类型表格的方式，管理程序中的各种通用性操作。上面使用的程序设计风格是一种基于类型进行分派的组织方式，其中让每个操作管理自己的分派。从效果上看，这种方式就是将操作-类型表哥格分解位一行一行，每个通用型过程表示表格中的一行。
另一种实现策略是将这一表格按列进行分解，不是采用一批“只能”操作区基于数据类型进行分派，而是采用“只能数据对象”，让它们基于操作名完成所需要的分派工作。
需要做的，将每一个数据对象表示为一个过程。（实际上类似于数据封装，每个数据对象保存专有的函数，利用虚函数就行了。思想是一致的。stl）

```lisp
(define (make-from-real-imag x y)
  (define (dispatch op)
    (cond ((eq? op 'real-part) x)
          ((eq? op 'iamg-part) y)
          ((eq? op 'magnitude)
           (sqrt (+ (square x) (square y))))
          ((eq? op 'angle) (atan y x))
          (else
           (error "Unkonown op -- MAKE-FROM-REAL-IMAG" op))))
dispatch)
;查找函数
(define (apply-generic op arg)
  (arg op))
```
这种风格的程序设计称为消息传递，将数据对象设想位一个实体，它以消息的方式接受所需要操作的名字。（设计模式里面有一种这种模式）。

