---
layout: post
title: Lazy evaluation in Python
---

# {{ page.title }}
Recently, while working on my PhD, I've started exploring different algorithms for computing and approximating Nash equilibria in games (I'm using a really well-written book, "Algorithmic Game Theory" by Nisan *et al.*, and in order to enhance my understanding of the theory, I'm concurrently having a go at implementing some of the algorithms therein presented -- you can find an up-to-date result of my work here: [py-game-theory](https://github.com/kubkon/py-game-theory)). While at it, I've bumped (again) into a concept of lazy evaluation. This has motivated me into checking how 'lazy' the Python family of languages is.

But before going into detail of my little investigation, a quick recap of the concept of lazy evaluation. According to [Wikipedia](http://en.wikipedia.org/wiki/Lazy_evaluation), "(...) lazy evaluation is an evaluation strategy which delays the evaluation of an expression until its value is needed and which also avoids repeated evaluations. (...)" Therefore, in a lazily evaluated language, it is possible to create an array of elements, and evaluate each element only when needed, e.g., when computing a sum of all elements in the array, and not sooner. A good example of a language which features lazy evaluation by default is Haskell.

When investigating Python for lazy evaluation, I've tested both Python 2 and Py3k (recalling that [Py3k has supposedly made some serious improvement in that area](http://www.linux-magazine.com/Issues/2009/107/Python-3)). Indeed, it turns out that Py3k is 'lazier' than Python 2 (IMHO, yet another point in favour of using Py3k rather than Python 2).

For example, in Python 2, a built-in `range` function returns a list:
{% highlight python %}

>>> range(5)
[0, 1, 2, 3, 4]

{% endhighlight %}
While in Py3k, the same built-in function returns an iterable `range` object:
{% highlight python %}

>>> range(5)
range(0, 5)
>>> list(range(5))
[0, 1, 2, 3, 4]

{% endhighlight %}
Therefore, each number in a sequence is evaluated on demand rather than immediately. This is a very useful feature when dealing with large sequences of numbers, and can save valuable execution time. Three other widely-used built-ins, `map`, `filter`, and `zip`, exhibit similar difference in behaviour between the two versions of Python (try it out!).

A really cool implication of lazy evaluation, is the possibility of creating an infinite (yes, infinite) sequence of numbers. For example, the following code shows how to implement an infinite sequence of Fibonacci numbers in Py3k:
{% highlight python %}

def fibonacci():
  a, b = 0, 1
  while True:
    yield a
    a, b = b, a + b

{% endhighlight %}
This construct is called a generator function in Python's nomenclature, and allows for lazy evaluation of the function's definition.

In order to print the first 10 Fibonacci numbers using our generator function, `fibonacci`, we first need to create an iterator (or technically, a generator) object by calling `fibonacci`. Then iterate over some control sequence of 10 elements (e.g., a `range` object of 10 numbers), and use `next` built-in function on the created iterator to return successive Fibonacci numbers:
{% highlight python %}

fibb = fibonacci()
for i in range(10):
  print(next(fibb), end=' ')

{% endhighlight %}
Remember though that after evaluating the first 10 numbers using `fibb`, you won't be able to reprint them with the same generator object, as it resumes where it left off previously. Therefore, you either need to create a new generator object, or save the elements in a container for later use, such as a list; for example, using a list comprehension:
{% highlight python %}

fibb = fibonacci()
fibb_list = [next(fibb) for i in range(10)]
print(fibb_list)

{% endhighlight %}
Happy lazy evaluating!