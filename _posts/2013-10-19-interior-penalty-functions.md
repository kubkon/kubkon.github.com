---
layout: post
title: 'Constrained nonlinear programming: interior penalty methods'
---

# {{ page.title }}

In the previous [post]({% post_url 2013-10-13-exterior-penalty-functions %}), I have briefly outlined how to solve a constrained nonlinear problem using unconstrained nonlinear programming; in particular, using exterior penalty methods. In this post, I'll present an alternative approach called *interior penalty methods* (or barrier function methods).

## The problem revisited

Let's revisit the optimisation problem introduced in the previous post. Suppose you want to minimise the function $$f(x) = x^2$$ subject to the constraint $$x \geq 1$$. The function attains its minimum at $$x = 1$$; however, if considered without the constraint, the minimum (clearly) shifts to $$x = 0$$. Recall that the unconstrained optimisation method will find the latter as the solution to the problem since we can't enforce the feasible region on the unconstrained optimisation algorithm.

## Interior penalty function

However, we can trick the algorithm into converging on the desired solution using the so-called interior penalty function [1]. The function's aim is to penalise the unconstrained optimisation method if it tries to leave (cross the boundary of) the feasible region of the problem. Hence, with this approach, we are required to start searching for the optimal solution of the problem within the feasible region. This is in direct opposition to the exterior penalty methods where we start outside the feasible region and slowly converge on a minimum from the *outside*.

Let's create an interior penalty function for our example:

\begin{equation}
F(x,\rho^k) = x^2 - \rho^k\ln(x-1)
\end{equation}

The first derivative of $$F$$ with respect to $$x$$ looks as follows:

\begin{equation}
\frac{\partial F}{\partial x} = 2x - \rho^k\cdot\frac{1}{x - 1}
\end{equation}

Equating the derivative to zero, and noting that by definition $$x > 1$$ (since we start in the feasible region of the problem), yields the solution to the modified optimisation problem:

\begin{equation}
x = \frac{1}{2} + \frac{\sqrt{1 + 2\rho^k}}{2}
\end{equation}

Note that as $$\rho^k \rightarrow 0$$ with $$k\rightarrow\infty$$, the solution converges to $$x = 1$$. That is, the solution to the original optimisation problem!

This method can readily be converted into a numerical algorithm that uses an unconstrained optimisation method:

1. pick a number $$\rho$$ such that $$0 < \rho < 1$$
2. starting from $$k = 1$$, minimise $$F(x, \rho^k)$$ using any unconstrained optimisation method
3. feed in the results of the minimisation as the new starting point to the minimisation method, and increment $$k$$

## Cython & Python implementation

To finish off this blog post, an exemplary implementation of the interior penalty method in Cython and Python. Like in the previous post, numerical minimisation is performed using algorithms provided by the GNU Scientific Library.

The objective function to be minimised writted in Cython:

{% highlight cython %}
from cython_gsl cimport *

from libc.math cimport pow, log

cdef double min_f(double x, void * params) nogil:
    # Extract params
    cdef double * ps = <double *> params
    # Compute interior penalty function
    cdef double barrier = (-1) * ps[0] * log(x - 1)
    # Compute original function value
    cdef double f = pow(x, 2)

    return f + barrier
{% endhighlight %}

And C extension to Python that uses Brent's minimisation method provided by the GNU Scientific Library:

{% highlight cython %}
def solve(initial_point, penalty):
    """
    Minimises Cython function min_f using Brent's method, and
    returns the result of the minimisation.

    Arguments:
    initial_point -- initial guess at the minimum
    penalty -- the penalty parameter (rho)
    """
    cdef int status
    # Initialise counter
    cdef int iter = 0
    cdef int max_iter = 100

    # Initial region of convergence
    # NOTE that the region of convergence encompasses
    # all x's greated than 1
    cdef double a = 1.0 + 1e-12
    cdef double b = 100.0

    # Initialise the minimisation problem
    cdef gsl_function F
    cdef double * params = [penalty]
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

Assuming the Cython module containing `min_f` and `solve` functions is called `interior`, the following Python script demonstrates it in action:

{% highlight python %}
from interior import solve

# Initial minimum
minimum = 3

# Penalty parameter
penalties = map(lambda x: x/10, range(9, 0, -1))

for penalty in penalties:
    # Minimise the modified problem, and feed in the result as
    # the new starting point
    minimum = solve(minimum, penalty)
    
    print("penalty=%.1f, minimum=%f" % (penalty, minimum))
{% endhighlight %}

With the following output generated:

{% highlight console %}
penalty=0.9, minimum=1.336660
penalty=0.8, minimum=1.306226
penalty=0.7, minimum=1.274597
penalty=0.6, minimum=1.241620
penalty=0.5, minimum=1.207107
penalty=0.4, minimum=1.170820
penalty=0.3, minimum=1.132455
penalty=0.2, minimum=1.091611
penalty=0.1, minimum=1.047722
{% endhighlight %}

Clearly, as the penalty, $$\rho^k$$, decreases to zero, the solution to the modified optimisation problem approaches the solution to the original problem.

## References

[1] Avriel, M., "Nonlinear Programming: Analysis and Methods", chapter 12, Dover Publications, 2003.
