---
layout: post
title: 'C++ short stories: type traits, concepts, and type constraints'
---

# {{ page.title }}

Suppose you find yourself in a hypothetical situation where you are designing a complex algorithm, for example, an algorithm that requires multiple depth-first searches through a complex tree-like structure. In situations like these, while I do appreciate the power of the debugger, if each node in the tree is a compound data structure (a sequence of some sort, say), it becomes somewhat cumbersome to 'step into' a couple hundred times when all you want is to get an idea what the structure looks like, or the path you took, etc. This is especially true when recursive calls/structures are used. Then, the good-old way of printing to screen can be of great help. Not a problem, just need to make sure that every data structure we work with is convertible to string. But what if our algorithm is inherently generic (think templates); i.e., does not care what the primitive data type stored in the tree is until runtime? How do we communicate to the end-developer using our algorithm to ensure that the data types he applies it to are convertible to string? We could test the primitive type at runtime and throw an exception if need be. But runtime errors are difficult to work with and inherently dangerous. We should take a cue from Rust developers and do as much error handling at the compile-time as possible. Furthermore, is it absolutely necessary to require that those data types are convertible to string? After all, it was a requirement only so that we were able to debug the algorithm while developing it, so perhaps the end-developer should not be burdened with it especially if they do not require convertibility to string. So, the puzzle is as follows: given a class template, how do we constrain the template argument to be convertible to string? Do we need to constrain the template argument for the entire definition of the class, or can we limit the constraints to methods that actually require the convertibility to string? Thus, in what follows, we will try to design a class template such that it comprises of a method `to_string()`, and we will see how we can require for the template argument to be convertible to `std::string` for the entire class definition, and limited to the `to_string()` method only.

## Idea 1 -- using type traits
The first approach relies on the use of type traits and, as such, requires a compiler conformant with C++14 standard. In particular, it makes use of the `std::enable_if_t` construct which is a helper construct for `std::enable_if` defined as follows:

{% highlight cpp %}

template<bool B, class T = void>
struct enable_if;

{% endhighlight %}

And as per the [C++ reference](http://en.cppreference.com/w/cpp/types/enable_if), if `B` is `true`, then the struct `std::enable_if` has a public member `std::enable_if::type` equal to `T`. Thus, we can use `std::enable_if` to enable class if, at compile-time, the template argument satisfies some template constraint encoded as the predicate `B`. This suggests the first solution to our problem to look as follows:

{% highlight cpp %}

template<typename T, typename Enable = void>
class A {};

template<typename T>
class A<T, typename std::enable_if_t<std::is_convertible<T, std::string>::value>> {
public:
  std::string to_string() const {
    return "Class A<>";
  }
};

{% endhighlight %}

In the snippet above, we have defined a generic class template `A` with the second template argument defaulting to `void`. This class will act as a fallback should our constraint be unmet. The second definition of `A` acts as a specialisation of the class for all types which are convertible to `std::string`. This is guaranteed by the call:

{% highlight cpp %}

std::enable_if_t<std::is_convertible<T, std::string>::value>

{% endhighlight %}

Here, `std::is_convertible` evaluates whether `T` can be implicitly cast to `std::string`, and returns `true` if this is the case. This then becomes the predicate `B` for `std::enable_if`, and hence the struct `std::enable_if` has a member `std::enable_if::type`. Therefore, if `T` is convertible to `std::string`, then the second specialisation of `A` will be matched by the compiler. Otherwise, if `T` is not convertible to `std::string`, then `std::enable_if::type` is undefined, and hence, the first definition of `A` will be matched. This approach can be somewhat improved by moving the `std::enable_if_t` construct into the signature of the method requiring `std::string` convertibility; namely, the `to_string()` method. This is precisely what is demonstrated in the snippet below: 

{% highlight cpp %}

template<typename T>
class B {
public:
  template<typename ToString = T>
  typename std::enable_if_t<std::is_convertible<ToString, std::string>::value, std::string>
  to_string() const {
    return "Class B<>";
  }
};

{% endhighlight %}

Firstly, note that we have created a new template argument `ToString` that defaults to `T`. This is to ensure that only `B::to_string()` is affected by the constraint, and not the entire class `B` (try it with the class-wide template argument `T` and see what happens). We have also changed the call to `std::enable_if_t` a little bit:

{% highlight cpp %}

std::enable_if_t<std::is_convertible<ToString, std::string>::value, std::string>

{% endhighlight %}

This call now implies that if `ToString` is convertible to `std::string`, then `std::enable_if::type` will equate to `std::string`, and hence, the entire thing will evaluate to the correct return type for the `B::to_string()` method; namely, to `std::string`. Ultimately, if `T` is not convertible to `std::string` then rather than not being able to create an object of type `B<T>`, we simply forbid the caller from using the `B<T>::to_string()` method.

Bringing it all together, we end up with the short example program:

{% highlight cpp %}
#include <iostream>
#include <string>
#include <type_traits>

template<typename T, typename Enable = void>
class A {};

template<typename T>
class A<T, typename std::enable_if_t<std::is_convertible<T, std::string>::value>> {
public:
  std::string to_string() const {
    return "Class A<>";
  }
};

template<typename T>
class B {
public:
  template<typename ToString = T>
  typename std::enable_if_t<std::is_convertible<ToString, std::string>::value, std::string>
  to_string() const {
    return "Class B<>";
  }
};

struct Size_t {
  Size_t(size_t v) : value(v) {}

  operator std::string() const {
    return std::to_string(value);
  }

  size_t value;
};

int main() {
  // we can create objects of type A and B from a type not convertible to std::string
  // but we are not allowed to call to_string() method both calls below will error
  // out at compile time
  A<size_t> a1;                             // OK
  B<size_t> b1;                             // OK
  std::cout << a1.to_string() << std::endl; // ERROR!
  std::cout << b1.to_string() << std::endl; // ERROR!

  // however, no problem with Size_t type that is a simple encapsulation of size_t with
  // implemented implicit std::string cast operator
  A<Size_t> a2;                             // OK
  B<Size_t> b2;                             // OK
  std::cout << a2.to_string() << std::endl; // OK
  std::cout << b2.to_string() << std::endl; // OK

  return 0;
}

{% endhighlight %}

As discussed above, while `A<size_t>` and `B<size_t>` both compile fine, we are not allowed to call `A<size_t>::to_string()` and `B<size_t>::to_string()` since `size_t` is not convertible to `std::string`. However, everything works OK for type `Size_t` which we defined as a dummy convenience struct which encapsulates `size_t` and defines the conversion to `std::string` (via the `operator std::string() const`).

## Idea 2 -- using concepts and type constraints
While Idea 1 is functional and will get the job done, it could be simplified substantially with the use of concepts. [Concepts](http://en.cppreference.com/w/cpp/language/constraints) are an upcoming feature of C++20 standard. They allow us to define certain constraints on the type itself which can then be caught at compile-time (essentially, `std::enable_if` on steroids and more). Since concepts are a work-in-progress, if you want to try out the code described below (and I strongly encourage you to), make sure you have the latest GCC and compile the snippets presented below with `-fconcepts` flag.

Firstly, we need to define an appropriate concept which will constrain a type `T` to be castable to `std::string`. This can be done as follows:

{% highlight cpp %}

template<typename T>
concept bool CastableToString = requires(T a) {
  { a } -> std::string;
};

{% endhighlight %}

Note that each concept is a predicate (hence, `bool` in the signature). In the body of the concept, we only require that, given an instance of type `T`, it is possible to convert it (cast it) to `std::string`. With the concept specified, we can use it to constrain the type for either the entire class or for a subset of methods that would require it. This can be done with the `requires` keyword. And so, in the former case, this can be done as follows:

{% highlight cpp %}

template<typename T> requires CastableToString<T>
class C {
public:
  std::string to_string() const {
    return "Class C<>";
  }
};

{% endhighlight %}

Here, when instantiating class `C<T>`, `T` has to satisfy our concept `CastableToString`. In order to constrain only some methods within the class, we could do so as follows:

{% highlight cpp %}

template<typename T>
class D {
public:
  std::string to_string() const requires CastableToString<T> {
    return "Class D<>";
  }
};

{% endhighlight %}

In this case, even if `T` is not convertible to `std::string`, `D<T>` can be instantiated just fine; however, `D<T>::to_string()` will be undefined, and calling it will yield a compile-time error.

Bringing it all together, we end up with the short example program:

{% highlight cpp %}
#include <iostream>
#include <string>
#include <type_traits>

template<typename T>
concept bool CastableToString = requires(T a) {
  { a } -> std::string;
};

template<typename T> requires CastableToString<T>
class C {
public:
  std::string to_string() const {
    return "Class C<>";
  }
};

template<typename T>
class D {
public:
  std::string to_string() const requires CastableToString<T> {
    return "Class D<>";
  }
};

struct Size_t {
  Size_t(size_t v) : value(v) {}

  operator std::string() const {
    return std::to_string(value);
  }

  size_t value;
};

int main() {
  // errors out since size_t violates the constraint CastableToString
  // but D does not require the same constraint; only D::to_string() call does
  C<size_t> c1;                             // ERROR!
  D<size_t> d1;                             // OK
  std::cout << d1.to_string() << std::endl; // ERROR!

  C<Size_t> c2;                             // OK
  D<Size_t> d2;                             // OK
  std::cout << c2.to_string() << std::endl; // OK
  std::cout << d2.to_string() << std::endl; // OK

  return 0;
}

{% endhighlight %}

As outlined above, since `size_t` is not convertible to `std::string`, `C<size_t>` yields an immediate compile-time error. `D<size_t>` compiles fine; however, we are not allowed to call `D<size_t>::to_string()`. Finally, everything works OK for type `Size_t` which we defined as a dummy convenience struct which encapsulates `size_t` and defines the conversion to `std::string` (via the `operator std::string() const`).

