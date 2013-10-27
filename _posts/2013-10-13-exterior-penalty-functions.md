---
layout: post
title: 'Constrained nonlinear programming: exterior penalty methods'
---

# {{ page.title }}

Recently, my PhD research has taken me on an exciting journey through the world of nonlinear programming. Nonlinear programming is a subfield of mathematical optimisation that deals with optimisation problems which are nonlinear in nature. For example, minimising a quadratic function of one real variable is considered as nonlinear since the quadratic function itself is nonlinear.

In this blog post, I thought I'm going to share my thoughts and recent discoveries when it comes to solving constrained nonlinear problems using unconstrained programming. In particular, I want to demonstrate a class of methods falling under the name *exterior penalty methods*.

## The problem

Suppose you want to minimise the function $$f(x) = x^2$$ subject to the constraint $$x \geq 1$$. The function attains its minimum at $$x = 1$$; however, if considered without the constraint, the minimum (clearly) shifts to $$x = 0$$. Since we can't enforce the feasible region (in our case, this would correspond to all $$x$$s larger or equal than $$1$$) on the unconstrained optimisation method, it is necessary to modify the problem in a way that tricks the unconstrained method into finding the minimum within the feasible region.

## Exterior penalty function

This can be achieved using the so-called exterior penalty function [1]. The function's aim is to penalise the unconstrained optimisation method if it converges on a minimum that is outside the feasible region of the problem. Applied to our example, the exterior penalty function modifies the minimisation problem like so:

$$
\begin{equation}
F(x,\rho^k) = x^2 + \frac{1}{\rho^k}\left[\min(0, x-1)\right]^2
\end{equation}
$$

Here, $$\rho^k > 0$$ for all $$k\in\mathbb{N}_+$$ quantifies the penalty. Note that if $$x$$ lies inside the feasible region, then the problem reduces to the original problem; that is, $$F(x, \rho^k) = f(x) = x^2$$. It can be shown that as the sequence $$(\rho^k), k\in\mathbb{N}_+$$ approaches $$0$$, the solution to the modified problem approaches the solution to the original problem *but* from the outside of the feasible region.

This method can readily be converted into a numerical algorithm that uses an unconstrained optimisation method. Pick a number $$\rho$$ such that $$0 < \rho < 1$$. Starting from $$k = 1$$, minimise $$F(x, \rho^k)$$ using any unconstrained optimisation method. Feed in the results of the minimisation as the new starting point to the minimisation method, and increment $$k$$.

## Cython & Python implementation

To finish off this blog post, I've included an exemplary implementation of the exterior penalty method for the presented problem in Cython and Python. Why Cython? With Cython, it is very easy to hook into the C-based GNU Scientific Library (which in turn provides the most popular unconstrained optimisation algorithms), and then execute the code from the Python level as a C extension to said language.

The objective function to be minimised writted in Cython:

{% highlight cython %}
from cython_gsl cimport *

from libc.math cimport pow

cdef double min_f(double x, void * params) nogil:
    # Extract params
    cdef double * ps = <double *> params
    # Compute penalty
    cdef double penalty = pow(ps[0], ps[1])

    cdef double f = pow(x, 2)

    # If outside feasible region, add the penalty
    if 0 > (x - 1):
        f = f + 1 / penalty * pow(x - 1, 2)

    return f
{% endhighlight %}

And C extension to Python that uses Brent's minimisation method provided by the GNU Scientific Library:

{% highlight cython %}
def solve(initial_point, penalty, k):
    """
    Minimises Cython function min_f using Brent's method, and
    returns the result of the minimisation.

    Arguments:
    initial_point -- initial guess at the minimum
    penalty -- the penalty parameter (rho)
    k -- nonnegative natural number
    """
    cdef int status
    # Initialise counter
    cdef int iter = 0
    cdef int max_iter = 100

    # Initial region of convergence
    cdef double a = -100.0
    cdef double b = 100.0

    # Initialise the minimisation problem
    cdef gsl_function F
    cdef double * params = [penalty, k]
    F.function = &min_f
    F.params = params

    # Initialise Brent's method
    cdef gsl_min_fminimizer_type * T
    cdef gsl_min_fminimizer * s
    T = gsl_min_fminimizer_brent
    s = gsl_min_fminimizer_alloc(T)
    gsl_min_fminimizer_set(s, &F, initial_point, a, b)

    status = GSL_CONTINUE

    # Minimise...
    while (status == GSL_CONTINUE and iter < max_iter):
        iter = iter + 1
        status = gsl_min_fminimizer_iterate(s)

        minimum = gsl_min_fminimizer_x_minimum(s)
        a = gsl_min_fminimizer_x_lower(s)
        b = gsl_min_fminimizer_x_upper(s)

        status = gsl_min_test_interval(a, b, 0.001, 0.0)

        if status == GSL_SUCCESS:
            break

    return minimum
{% endhighlight %}

Assuming the Cython module containing `min_f` and `solve` functions is called `exterior`, the following Python script demonstrates it in action:

{% highlight python %}
from exterior import solve

# Rho (penalty)
rho = 0.1
# Initial minimum
minimum = 0.0

for k in range(1, 15):
    # Minimise the modified problem, and feed in the result as
    # the new starting point
    minimum = solve(minimum, rho, k)
    
    print("k={}, minimum={}".format(k, minimum))


{% endhighlight %}

With the following output generated:

{% highlight console %}
k=1, minimum=0.9090909090909095
k=2, minimum=0.990099009900986
k=3, minimum=0.9990009990009904
k=4, minimum=0.9999000099989902
k=5, minimum=0.9999900001000043
k=6, minimum=0.999999000000997
k=7, minimum=0.9999999000000095
k=8, minimum=0.9999999899999891
k=9, minimum=1.0000000049011502
k=10, minimum=1.0000000049011502
k=11, minimum=1.0000000049011502
k=12, minimum=1.0000000049011502
k=13, minimum=1.0000000049011502
k=14, minimum=1.0000000049011502
{% endhighlight %}

It is clear that the algorithm yields the correct solution to the minimisation problem, and the precision of the solution increases with each iteration until the saturation level is reached (after 9th iteration).

## References

[1] Avriel, M., "Nonlinear Programming: Analysis and Methods", chapter 12, Dover Publications, 2003.
