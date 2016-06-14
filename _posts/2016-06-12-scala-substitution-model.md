---
layout: post
title: "Scala Substitution Model"
comments: True
permalink: "scala-substitution-model"
use_math: true
---

### Introduction

`Scala` model of expression evaluation is based on the principle of substitution
model, in which variable names are replaced by the values they are bound to.The underlying
idea is that all evaluation does is reduce an expression to a value, which is a term that
does not need further evaluation. Values includes not only constants, but tuple, constructors, and
functions. The substitution model is formalized in the $\lambda-calculus$, which gives foundation for functional programming.

In `Scala` program expressions are evaluated in the same way we would evaluate a mathematical expression. For example we know the expression $(2 * 2) + (3 * 4)$ will be evaluated by first evaluating $2 * 2$ and $3 * 4$ and finally $4 + 12$. `Scala` evaluation works in same way. Similarly in `Scala` non-primitive expression is evaluated as follows.

1. Take the leftmost operator.
2. Evaluate its operands from left to right.
3. Apply the operator to operand values.

During the process, `Scala` evaluator *rewrites* the program expression to another expression. The evaluation process finally converts the expression into value and the evaluation stops: assuming evaluation terminates else evaluation fails to reduce to final value and we have *infinite loop*.

### Expression Evaluation
Expression evaluation works by rewriting the original expression. Rewriting works by performing simple steps called *reductions*. For example, the evaluation of arithmetic expression is:

{% highlight scala %}
  scala> def x = 2
  scala> def y = 5
  scala> (2 * x) + (4 * y)
{% endhighlight %}

$\to (2 * 2) + (4 * y)$

$\to 4 + (4 * 5)$

$\to 4 + 20$

$\to 24$

### Function Evaluation
Function with parameters are evaluated similar to the operators in expressions. Following are the rules for function parameters evaluation:

1. Evaluates all the function arguments from left-to-right.
2. Replace the function application by the functions right-hand side, and, at the same time
3. Replace all the formal parameters of the function by the actual arguments.

Below is the evaluation of the function parameters:
{% highlight scala %}
  scala> def square(x: Double) = x * x
  scala> square(3 + 3)
{% endhighlight %}

$\to square(6)$

$\to 6 * 6$

$\to 36$

### Function Evaluation Strategy
There are two evaluation strategy for function with parameters namely `call-by-value` and `call-by-name`. For expressions that use only pure function and can be reduced with substitution model, both yields the same final result. Let's define the function `sumofSqaures` and evaluate it with both `call-by-value` and `call-by-name`.

{% highlight scala %}
  scala> def sumofSqaures(x: Int, y: Int) = square(x) + square(y)
  scala> sumofSqaures(2, 2 + 3)
{% endhighlight %}

1. *call-by-value*: *Call-by-value* evaluates every function argument only once thus it avoids the repeated evaluation of arguments. Example:
$\to sumofSqaures(2, 5)$

    $\to square(2) + square(5)$

    $\to 2*2 + 5 * 5$

    $\to 4 + 25$

    $\to 29$
2. *call-by-name*: *Call-by-name* avoids evaluation of parameters if it is not used in the function body. Example:

    $\to square(2) + sqaure(2 + 3)$

    $\to square(2) + square( 2 + 3)$

    $\to 2 * 2 + square(2 + 3)$

    $\to 4 + (2 + 3) * (2 + 3)$

    $\to 4 + 5 * (2 * 3)$

    $\to 4 + 5 * 5$

    $\to 4 + 25$

    $\to 29$

### Difference between *Call-by-value* and  *Call-by-name*
  *Call-by-value* is more efficient then *call-by-name* . If *call-by-value* evaluation of expression terminates then *call-by-name* evaluation also terminates, too but the other direction is not true since *call-by-value*  might loop infinitely but *call-by-name* would terminate. Example:

  {% highlight scala %}
    scala> def loop:Int = loop
    scala> def test(x: Int, y: Int) = x {% endhighlight %}

  Then the evaluation of function `test(1, loop)` is:


  1. *Call-by-name* evaluation reduces to $1$.

     $\to 1$

  2. *Call-by-value* evaluation leads to infinite loop.

     $\to test(1, loop)$

     $\to test(1, loop)$

     $\to ...$

 `Scala` uses *call-by-value* as a default since *call-by-value* is often exponentially efficient then *call-by-name*. We can force it to use *call-by-name* by preceding the parameter types by $\Rightarrow$. Example:

 {% highlight scala %}
  scala> def constOne(x: Int, y: => Int) = 1 {% endhighlight %}

Following function call reduces to 1.

{% highlight scala %}
  scala> constOne(1, loop)
  unnamed0: Int = 1
{% endhighlight %}

And the below function call leads to infinite loop.

{% highlight scala %}
  scala> constOne(loop, 1) // Goes to infinite loop.
{% endhighlight %}

### Conclusion
The actual operation of `Scala` interpretor and compiler is more complex under the hood. Nevertheless, understanding the evaluation model helps us to know what the programming is doing and reason behind its correctness. The evaluation model is abstraction that hides the compiler details and help us understands the function parameters evaluations.

#### Reference
1. [Functional Programming Principles in Scala](https://www.coursera.org/learn/progfun1/home/welcome)
