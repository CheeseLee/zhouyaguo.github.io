---
layout: post
title: "Emacs Lisp"
comments: true
categories:
- Emacs
---

简介
-----

一般人认为lisp是门很奇怪的语言，到处都是括号。

Lisp stands for LISt Processing，这门编程语言主要就是对list进行操作（将list放入括号内，括号作为list的边界）

list是lisp语言的基石。

一个直观的list例子：

```
’(rose violet daisy buttercup)
```

- 以单引号开头
- 在括号内
- 元素（atom）间用空格分隔

在lisp语言里，data和programs采用了同样的方式来描述：都是用空格分隔，然后用括号包起来

evaluate
---------

当evaluate一段lisp程序时，会有三种处理方式：

- 什么都不做，直接返回list本身
- 报error message
- 将list中的第一个atom当做function，后面的atoms当做参数

前面的例子中的单引号(')，就是告诉lisp什么都不做，直接返回这个list，如果没有这个单引号，lisp会将第一个atom当做function，然后报错。

emacs debugger
------------

在emacs中的`*scratch*`中输入 (1 2 3 4) ，然后按C-x C-e执行，会出现`*Backtrace*`，其中显示：

```
Debugger entered--Lisp error: (invalid-function 1)
  (1 2 3 4)
  eval((1 2 3 4) nil)
  eval-last-sexp-1(nil)
  eval-last-sexp(nil)
  call-interactively(eval-last-sexp nil nil)
  command-execute(eval-last-sexp)
```

这个error信息是由内置的emacs debugger产生的，按 *q* 退出debugger，error message要从下向上读，代表了emacs的执行顺序

p后缀
-----

当输入 (+ 2 ’hello) ，然后按C-x C-e执行，会报错：

```
Wrong type argument: number-or-marker-p, hello
```

其中的p的用法开始于Lisp programming早期，意味着“predicate”，意思是：是true还是false。比如zerop意思为是否为0，
listp意思就是是否为list


为变量设值，set与setq的区别
---------------

(set 'a 1)   --  _需要单引号_

(setq a 1)   -- _setq中的q含义是quote，可以省略set中的单引号_

要点
----

- Lisp programs are made up of expressions, which are lists or single atoms.
- Lists are made up of zero or more atoms or inner lists, separated by whitespace and surrounded by parentheses. A list can be empty.
- Atoms are multi-character symbols, like forward-paragraph, single character symbols like +, strings of characters between double quotation marks, or numbers.
- A number evaluates to itself.
- A string between double quotes also evaluates to itself.
- When you evaluate a symbol by itself, its value is returned.
- When you evaluate a list, the Lisp interpreter looks at the first symbol in the list and then at the function definition bound to that symbol. Then the instructions in the function definition are carried out.
- A single quotation mark, ’ , tells the Lisp interpreter that it should return the following expression as written, and not evaluate it as it would if the quote were not there.
- Arguments are the information passed to a function. The arguments to a function are computed by evaluating the rest of the elements of the list of which the function is the first element.
- A function always returns a value when it is evaluated (unless it gets an error); in addition, it may also carry out some action called a “side effect”. In many cases, a function’s primary purpose is to create a side effect.

emacs的工作原理
-------------

**Whenever you give an editing command to Emacs Lisp, such as the command to move the cursor or to scroll the screen, you are evaluating an expression, the first element of which is a function. This is how Emacs works**

how to write function
----------------------

```
(defun function-name (arguments...)
  "optional-documentation..."
  (interactive argument-passing-info) ; optional
  body...)
```

例如：

```
(defun addOne (number)
  "this is my test function"
  (+ 1 number))

(addOne 2)
```


let
----

let creates a name for a local variable that overshadows any use of the same name outside the let expression.
Another way to think about let is that it is like a setq that is temporary and local

```
(let VARLIST BODY...)

Bind variables according to VARLIST then eval BODY.
The value of the last form in BODY is returned.
Each element of VARLIST is a symbol (which is bound to nil)
or a list (SYMBOL VALUEFORM) (which binds SYMBOL to the value of VALUEFORM).
All the VALUEFORMs are evalled before any symbols are bound.
```

举例：

```
(let ((zebra ’stripes)
           (tiger ’fierce))
       (message "One kind of animal has %s and another is %s."
                zebra tiger))
```

if-then-else
-------------

(if true-or-false-test
  action-to-carry-out-if-the-test-returns-true
  action-to-carry-out-if-the-test-returns-false)

TODO
