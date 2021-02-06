# [读书笔记] sicp 第二章 符号数据


```lisp
(list 'a 'b)
(eq? a b)
```
# 符号求导
```lisp

;e是变量吗
(define (variable? x)
  (symbol? x));symbol?判断变量是不是符号

;v1和v2是同一个变量吗
(define (same-variable? v1 v2)
  (and (variable? v1) (variable? v2) (eq? v1 v2)))

;e是和式吗
(define (sum? x)
  (and (pair? x) (eq? (car x) '+)))

;e的被加数
(define (addend s) (cadr s))

;e的加数
(define (augend s) (caddr s))

;构造起a1和a2的和式
(define (make-sum a1 a2) (list '+ a1 a2))

;e是乘式吗
(define (product? x)
  (and (pair? x) (eq? (car x) '*)))

;e的被乘数
(define (multiplier p) (cadr p))

;e的乘数
(define (multiplicand p) (caddr p))

;构造起来m1与m2的乘式
(define (make-product m1 m2)
       (list '* m1 m2))

;求导
(define (deriv exp var)
  (cond ((number? exp) 0)
        ((variable? exp)
         (if (same-variable? exp var) 1 0))
        ((sum? exp)
         (make-sum (deriv (addend exp) var)
                   (deriv (augend exp) var)))
        ((product? exp)
         (make-sum
          (make-product (multiplier exp)
                        (deriv (multiplicand exp) var))
          (make-product (deriv (multiplier exp) var)
                        (multiplicand exp))))
        (else
         (error "unknown expression type -- DERIV" exp))))

(deriv '(+ x 3) 'x)
(deriv '(* x y) 'x)
(deriv '(* (* x y) (+ x 3)) 'x)

'(+ 1 0)
'(+ (* x 0) (* 1 y))
'(+ (* (* x y) (+ 1 0)) (* (+ (* x 0) (* 1 y)) (+ x 3)))
```

<!--more-->

# 集合的表示
## 集合作为未排序的表
```lisp
;判断是不是表成员
(define (element-of-set? x set)
  (cond ((null? set) false)
        ((equal? x (car set)) true)
        (else (element-of-set? x (cdr set)))))
;向表增加一项
(define (adjoin-set x set)
  (if (element-of-set? x set)
      set
      (cons x set)))
;合并
(define (intersection-set set1 set2)
  (cond ((or (null? set1) (null? set2)) '())
        ((element-of-set? (car set1) set2)
         (cons (car set1)
               (intersection-set (cdr set1) set2)))
        (else (intersection-set (cdr set1) set2))))
```
## 集合作为排序的表
排序的存在的好处就是减少复杂度
```lisp
(define (element-of-set? x set)
  (cond ((null? set) false)
        ((= x (car set)) true)
        ((< x (car set)) false)
        (else (element-of-set? x (cdr set)))))
    
(define (intersection-set set1 set2)
  (if (or (null? set1) (null? set2))
       '()
       (let ((x1 (car set1)) (x2 (car set2)))
         (cond ((= x1 x2)
                (cons x1
                      (intersection-set (cdr set1)
                                        (cdr set2))))
               ((< x1 x2)
                (intersection-set (cdr set1) set2))
               ((< x2 x1)
                (intersection-set set (cdr set2)))))))
```
## 集合作为二叉树
有序二叉树
```lisp
(define (entry tree) (car tree))

(define (left-branch tree) (cadr tree))

(define (right-branch tree) (caddr tree))

(define (make-tree entry left right)
  (list entry left right))
  
(define (element-of-set? x set)
  (cond ((null? set) false)
        ((= x (entry set)) true)
        ((< x (entry set))
         (element-of-set? x (left-branch set)))
        ((> x (entry set))
         (element-of-set? x (right-branch set)))))
;插入要找到正确位置
(define (adjoin-set x set)
  (cond ((null? set) (make-tree x '() '()))
        ((= x (entry set)) set)
        ((< x (entry set))
         (make-tree (entry set)
                    (adjoin-set x (left-branch set))
                    (right-branch set)))
        ((> x (entry set))
         (make-tree (entry set)
                    (left-branch set)
                    (adjoin-set x (right-branch set))))))
```
# huffman编码树
```lisp
;树的表示
;leaf 符号 权重

(define (make-leaf symbol weight)
  (list 'leaf symbol weight))
  
(define (leaf? object)
  (eq? (car object) 'leaf))
  
(define (symbol-leaf x) (cadr x))
(define (weight-leaf x) (caddr x))

(define (make-code-tree left right)
  (list left
        right
        (append (symbols left) (symbols right))
        (+ (weight left) (weight right))))
        
(define (left-branch tree) (car tree))
(define (right-branch tree) (cadr tree))

(define (symbols tree)
  (if (leaf? tree)
      (list (symbol-leaf tree))
      (caddr tree)))

(define (weight tree)
  (if (leaf? tree)
      (weight-leaf tree)
      (caddr tree)))
```
## 解码过程
```lisp
(define (decode bits tree)
  (define (decode-1 bits current-branch)
    (if (null? bits)
        '()
        (let ((next-branch
               (choose-branch (car bits) current-branch)))
          (if (leaf? next-branch)
              (cons (symbol-leaf next-branch)
                    (decode-1 (cdr bits) tree))
              (decode-1 (cdr bits) next-branch)))))
  (decode-1 bits tree))

(define (choose-branch bit branch)
  (cond ((= bit 0) (left-branch branch))
        ((= bit 1) (right-branch branch))
        (else (error "bad bit -- CHOOSE-BRANCH" bit))))
```
## 带权重的集合
```lisp
(define (adjoin-set x set)
  (cond ((null? set) (list x))
        ((< (weight x) (weight (car set))) (cons x set))
        (else (cons (car set)
                    (adjoin-set x (cdr set))))))

(define (make-leaf-set pairs)
  (if (null? pairs)
      '()
      (let ((pair (car pairs)))
        (adjoin-set (make-leaf (car pair)
                               (cadr pair))
                    (make-leaf-set (cdr pairs))))))
```

