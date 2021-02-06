# [读书笔记] sicp 第二章 带有通用型操作的系统

# 通用型算术运算

```lisp
(define (add x y) (apply-generic 'add x y))
(define (sub x y) (apply-generic 'sub x y))
(define (mul x y) (apply-generic 'mul x y))
(define (div x y) (apply-generic 'div x y))

(define (install-scheme-number-package)
  (define (tag x)
    (attach-tag 'scheme-number x))
  (put 'add '(scheme-number scheme-number)
       (lambda (x y) (tag (+ x y))))
  (put 'sub '(scheme-number scheme-number)
       (lambda (x y) (tag (- x y))))
  (put 'mul '(scheme-number scheme-number)
       (lambda (x y) (tag (* x y))))
  (put 'div '(scheme-number scheme-number)
       (lambda (x y) (tag (/ x y))))
  (put 'make '(scheme-number scheme-number)
       (lambda (x) (tag x)))
  'done)
  
  (define (make-scheme-number n)
  ((get 'make 'scheme-number) n))
```
利用同样的方法可以加入有理数/复数等操作

<!--more-->
# 不同类型数据的组合
处理跨类型的操作。
为每一种跨类型操作提供专门的过程处理，是可以，但是太麻烦。每添加一种类型，要增加太多过程。

## 强制
类型转换处理能够解决一部分问题。
```lisp
;实数转虚数
(define (scheme-number->complex n)
  (make-complex-from-real-imag (contents n) 0))
```
将这些强制过程安装到一个特护的表格里，用两个类型的名字作为索引。
```lisp
(put-coercion 'scheme-number 'complex scheme-number->complex)
```
```lisp
(define (apply-generic op . args)
  (let ((type-tags (map type-tag args)))
    (let ((proc (get op type-tags)))
      (if proc
          (apply proc (map contents args))
          (if (= (length args) 2)
              (let ((type1 (car type-tags))
                    (type2 (cadr type-tags))
                    (a1 (car args))
                    (a2 (cadr args)))
                (let ((t1->t2 (get-coercion type1 type2))
                      (t2->t1 (get-coercion type2 type1)))
                  (cond (t1->t2
                         (apply-generic op (t1->t2 a1) a2))
                        (t2->t1
                         (apply-generic op a1 (t2->t1 a2)))
                        (else
                         (error "No method for these types"
                                (list op type-tags))))))
              (error "No method for these types"
                     (list op type-tags)))))))
```
# 类型的层次结构
就是继承嘛。子类型有父类型的所有操作。
#层次结构的不足
可能产生菱形的层次结构。

在设计大型系统时，处理好一大批相互有关的类型而同时又能保持模块性，这是一个困难的问题，也是当前正在继续研究的领域。
编者注：这句话出现在书的第一版本。它的现在就像20年前写出时候正确。开发出一种有用的，具有一般意义的框架，以描述不同类型对象之间的关系(哲学中本体论)，看来是一件极其困难的工作。在10年前存在的混乱和今天存在的混乱之间的主要差异在于，今天已经有了一批各式各样的并不合适的本体理论，它们已经嵌入数量过多而又先天不足的各种程序设计语言里。举例来说，面向对象语言的大部分复杂性-以及当前各种面向对象语言之间细微的而且诗人迷惑的差异-的核心，就是类型之间通用型操作的处理。我们在第三章有关计算性对象的讨论中完全避免了这些问题。熟悉面向对象程序涉及到读者将会注意到，在第三章里关于局部状态说了许多东西，但是却根本没有提到“类”或者“继承”。事实上，我们的猜想是，如果没有知识表示和自动推理工作的帮助，这些问题是无法仅仅通过计算机语言设计的方式合理处理的。

