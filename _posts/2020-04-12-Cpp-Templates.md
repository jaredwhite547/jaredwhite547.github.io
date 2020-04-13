---
layout: post
title: C++ Templates & Metaprogramming
---

## Purpose
This blog post is intended to serve as a brief introduction to templates in C++. It will show some and explain some basic examples, and provide some reasoning and motivation. It’ll then explain some more advanced concepts, and showcase them.

## What are templates?
Templates are C++’s version of generics. For example, if we want to write a function that can add two parameters together, we could write it like this:
```cpp
template<typename T>
T add(T a, T b){
    return a+b;
}
```


We can call it as we would any normal function. For example:
```cpp
int a = add(1, 2);
```

Let’s break it down bit by bit:
`template<` signifies that this function (or class/struct, more on that later), is a template.

`typename T` says that we are declaring a type called `T`, which will be specified by the user.

`T add(T a, T, b){` says that this function has a return type of `T`, and takes two parameters also of type `T`.

The type of `T` is automatically inferred when we call it, since we use it as parameters.


The key difference between C++ templates and something like Java’s generics, is that all the magic of templates happens at compile-time, and there is no type-erasure. The compiler basically stamps out a copy of the function with `T` replaced with the actual type. This happens once per unique signature. So if we call our add function like this:
```cpp
int a = add(1, 2);
int b = add(2, 1);
float c = add(3.0, 4.0);
```

The compiler will generate code equivalent to this:
```cpp
int add(int a, int b){
    return a+b;
}

float add(float a, float b){
    return a+b;
}
```

Note that templates can also be classes/structs:
```cpp
template <typename T, typename U>
struct pair{
    T first;
    U second;
};
```
Like the functions, a copy of this code(and any class methods), is stamped out per unique signature.

## Benefits
There are numerous benefits to this over run-time, type-erased generics:
* Templates are duck-typed. This means that as long as the types support whatever is done to them(operators, methods, etc), it will compile. This removes the need to inherit from interfaces and override their methods, which incurs a performance penalty.
* Because a unique copy is created for each signature, the compiler can optimize for each specific case, making optimizations not otherwise possible. This also means that layers of templates can be ‘transparent’ to the compiler, making them effectively collapse into very little code.
* Because the types are not erased, we can do interesting things like branch off of the type, running optimized code for special scenarios at no cost.
* Templates are turing complete, allowing for compile-time generation of look-up tables, among other things.

## Drawbacks
* Excessive and irresponsible use of templates can make compile times balloon.
* Error messages are sometimes very hard to decipher
* Some template techniques are confusing and esoteric



## Motivation
Let’s say we have a pair of iterators, and want to find out the distance between them. Here’s how you would write it so that it worked with every iterator from every container:
```cpp
template <typename Iter, typename DiffType = std::iterator_traits<Iter>::difference_type>
DiffType dist(Iter begin, Iter end){
    DiffType count(0);
    for(; begin != end; ++begin)
        ++count;
    return count;
}
```
The `typename DiffType = std::iterator_traits<Iter>::difference_type` just aliases the type so we don’t have to type a long type name twice. See [here](https://en.cppreference.com/w/cpp/iterator/iterator_traits) for more on the actual `iterator_traits` function.

Note that runtime is O(N). This works, but in some cases, we can do better. Iterators from containers such `std::vector` can have their distance computed in constant time, since they are really just pointers. These iterators support the subtraction operator. So for vectors, we want code that looks like this:
```cpp
template <typename Iter, typename DiffType = std::iterator_traits<Iter>::difference_type>
DiffType dist(Iter begin, Iter end){
    return end - begin;
}
```
We can leverage templates to do this. First, we have to discuss `decltype, declval, SFINAE, void_t`, and show some examples.

## decltype & declval
`decltype` Basically extracts the type of the given expression. For example, `decltype(5)` will yield the type `int`.

`declval` takes a type as a template parameter, and produces a non-instantiable version of that object. This is generally used to create an expression from template parameters, and then extract that type. For example, if we want to get the type of adding two `T`s together, we would write:

```cpp
decltype(std::declval<T&>() + std::declval<T&>())
```
## SFINAE, std::enable_if, & void_t
SFINAE stands for Substitution Failure is Not An Error. This rule states that a template that is ill-formed does not generate an error. Instead, the next viable overload is attempted. See [here](https://en.cppreference.com/w/cpp/language/sfinae) for more details.

To leverage this functionality, the standard library provides the function `enable_if`. ([Click here for docs](https://en.cppreference.com/w/cpp/types/enable_if)) 

This function removes a template from the set of possible matches if the first template parameter is false.

`std::void_t` is a utility function that is used to check if a sequence of types is well formed. [Docs](https://en.cppreference.com/w/cpp/types/void_t)


## Part 1
We need to begin by creating a way to test if we can subtract two objects. To do this, we will write some code similar to the ones found in [`type_traits`](https://en.cppreference.com/w/cpp/header/type_traits)

First, we will provide a base template. This template has the lowest priority, so if there is another template, it will take priority (if valid).
```cpp
template <typename T, typename = void>
struct can_subtract : std::false_type{};
```
The `typename = void` is because we will need that parameter later, but don’t need to do anything with it here.

`: std::false_type` makes this class inherit from `std::false_type`, which provides a public boolean named `value`, which is `false`.

Now, we add in our specialization. This is where we test if we can subtract the types.
```cpp
template <typename T>
struct can_subtract<T, std::void_t<decltype(std::declval<T&>() - std::declval<T&>())>> : std::true_type{};
```
This syntax is a specialization. We take the `T` as normal, but instead of having our second parameter be void (`typename = void`), we basically check if the expression `T-T` is valid using `void_t`. If it is, this template will take priority over the other one (because it is specialized), and will be instantiated instead. Now, it will still have a public boolean named `value`, but it will be `true` instead.

We now have a way to test if our objects support subtraction. With this, we can build our distance function.

## Part 2
The non-optimized, normal version will look like this:
```cpp
template <typename T,
        typename diff_type = std::iterator_traits<T>::difference_type,
        std::enable_if_t<!can_subtract<T>::value, int> = 0>
diff_type dist(T begin, T end){
    diff_type count{0};
    for(; begin != end; ++begin, ++count);
    return count;
}
```
The `std::enable_if` line checks if `T` can not be subtracted. If it can’t be, then it evaluates to type `int`. The `= 0` is needed because of a quirk in how redeclarations are recognized (see the `enable_if` docs for more details). Conversely, if `T` CAN be subtracted from itself, the `enable_if` will trigger SFINAE, and this template will be discarded.

The special version looks like this:
```
template<typename T,
        typename diff_type = std::iterator_traits<T>::difference_type,
        std::enable_if_t<can_subtract<T>::value, int> = 0>
diff_type dist(T begin, T end){
    return end - begin;
}
```

The only difference (aside from the function body), is that the `enable_if` fires if we CAN subtract it, and triggers SFINAE if we cannot.

The complete code looks like this:
```
#include <type_traits>
#include <iostream>
#include <vector>
#include <list>
#include <iterator>

template <typename T, typename = void>
struct can_subtract : std::false_type{};

template <typename T>
struct can_subtract<T, std::void_t<decltype(std::declval<T&>() - std::declval<T&>())>> : std::true_type{};

template <typename T,
        typename diff_type = std::iterator_traits<T>::difference_type,
        std::enable_if_t<!can_subtract<T>::value, int> = 0>
diff_type dist(T begin, T end){
    diff_type count{0};
    for(; begin != end; ++begin, ++count);
    std::cout << "Normal Version\n";
    return count;
}

template<typename T,
        typename diff_type = std::iterator_traits<T>::difference_type,
        std::enable_if_t<can_subtract<T>::value, int> = 0>
diff_type dist(T begin, T end){
    std::cout << "Specialized Version\n";
    return end - begin;
}


int main(){
    std::vector<int> vec{1,2,3};
    std::list<int> li{1,2,3};
    std::cout << dist(vec.begin(), vec.end()) << "\n";
    std::cout << dist(li.begin(), li.end()) << "\n";
    return 0;
}
```
Note: You will probably need a newer C++ compiler for this.


## Conclusion


