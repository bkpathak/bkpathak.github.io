---
layout: post
title: "Scala Higher Order Functions"
comments: True
permalink: "scala-higher-order-functions"
use_math: true
---

### Introduction

In *Scala* functions are first-class values which means functions are values just like other values in *Scala*. We can pass functions as arguments to other functions, stored the function in values, return the functions as a result from other functions.
So, function which takes other function as parameters or return as result are called `higher-order` functions. Any *Scala* function with following properties can be *higher-order* functions. *Higher-order* functions let us write more general code and reusable code and lead us to `DRY` principle ( *Don't Repeat Yourself* ).

Let's us look the below example:
1. Function to sum all integers between two given numbers a and b:

{% highlight scala %}
def sumInts(a: Int, b: Int): Int =
  if (a > b) 0 else a + sumInts( a + 1, b)
{% endhighlight %}

2. Functions to sum all the integers between two given numbers a and b:

def square(x: Int): Int= x * x
def sumSqaures(a: Int, b:Int): Int =
  if (a > b) return 0 else sqaure(a) + sumSqaures(a + 1, b)
{% highlight scala %}

Above two functions are the instance of $\sum_{a}^{b} f(n)$ for the different function $f$. So the above functions can be factored as follow:

{% highlight scala %}
def sum(f: Int => Int, a: Int, b:Int) =
  if (a > b) 0 else f(a) + sum(a + 1 ,b)
{% endhighlight %}

In above sum function we have parameter *f* which is function with type `Int => Int` which means it takes `Int` and also return `Int`. Since the argument of *sum* is function, sum is *higher-order* function. Now the above function can be refactored using *sum* function:

{% highlight scala %}
def id(x: Int): Int = x
def square(x: Int) = x * x
def sumInts(a: Int, b: Int) = sum(id, a, b)
def sumSqaures(a; Int, b:Int) = sum(sqaure, a, b)
{% endhighlight %}

### Anonymous
Simply the anonymous functions means function without name. These functions are defined as a literals and are syntactic sugar for more tedious long definition. The above *square* and *id* function can be defined with anonymous as:

{% highlight scala %}
(x: Int) => x
(x: Int) = > x * x
{% endhighlight %}

The part before `=>` is function parameter and after that is function body. So the above two function can be written with anonymous function which helps to avoid polluting top-level namespace.

{% highlight scala %}
def sumInts(a: Int, b: Int) = sum((x: Int) => x, a, b)
def sumSqaures(a; Int, b:Int) = sum((x: Int) => x * x, a, b)
{% endhighlight %}  
