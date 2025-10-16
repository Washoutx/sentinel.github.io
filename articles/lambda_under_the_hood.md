---
layout: default
---

# Why are some lambdas tricky? What is a lambda under the hood?

What is lambda? To be honest it is just syntax sugar. It is implementation defined but
compilers mostly just generate callable object which is class with `operator()` (functor).

Let's check what compiler does with following example:
```cpp
int main()
{
    int i = 10;
    auto lambda = [i](int b){ return i + b; };
    std::cout << lambda(20);

    return 0;
}
```
You can check above example on [cppinsights](https://cppinsights.io).

Possible output:
```cpp
int main()
{
  int i = 10;
    
  class __lambda_6_19
  {
    public: 
    inline /*constexpr */ int operator()(int b) const
    {
      return i + b;
    }
    
    private: 
    int i;
    
    public:
    __lambda_6_19(int & _i)
    : i{_i}
    {}
    
  };
  
  __lambda_6_19 lambda = __lambda_6_19{i};
  std::cout.operator<<(lambda.operator()(20));
  return 0;
}
```

This example is good to show why `mutable` keyword is used in context of lambdas.
As you can see `operator()` is `const`. So I cannot change the generated `__lambda_6_19`
attributes. `i` is pass to lambda via copy so it is part of the generated functor. 
Following code is invalid:

```cpp
int main()
{
    int i = 10;
    auto lambda = [i](int b){ ++i; return i + b; };
    std::cout << lambda(20);

    return 0;
}
```
```
/home/insights/insights.cpp:6:31: error: cannot assign to a variable captured by copy in a non-mutable lambda
    6 |     auto lambda = [i](int b){ ++i; return i + b; };
      |                               ^ ~
1 error generated.
```
Let's add mutable keyword and check what changed.

```cpp
int main()
{
    int i = 10;
    auto lambda = [i](int b) mutable { ++i; return i + b; };
    std::cout << lambda(20);

    return 0;
}
```
```cpp
int main()
{
  int i = 10;
    
  class __lambda_6_19
  {
    public: 
    inline /*constexpr */ int operator()(int b)
    {
      ++i;
      return i + b;
    }
    
    private: 
    int i;
    
    public:
    __lambda_6_19(int & _i)
    : i{_i}
    {}
    
  };
  
  __lambda_6_19 lambda = __lambda_6_19{i};
  std::cout.operator<<(lambda.operator()(20));
  return 0;
}
```
As you can see `const` disappeared from `operator()`. And that is the whole magic behind this.

You need to be carefull if you create lambdas which use references. Check following example of dangling reference:
```cpp
auto genLambda()
{
  int j = 100;
  return [&j]{ return j; };
}

int main()
{
    auto lambda = genLambda();
  	std::cout<< lambda();
    return 0;
}
```
```cpp
__lambda_6_10 genLambda()
{
  int j = 100;
    
  class __lambda_6_10
  {
    public: 
    inline /*constexpr */ int operator()() const
    {
      return j;
    }
    
    private: 
    int & j;
    
    public:
    __lambda_6_10(int & _j)
    : j{_j}
    {}
    
  } __lambda_6_10{j};
  
  return __lambda_6_10;
}

int main()
{
  __lambda_6_10 lambda = genLambda();
  std::cout.operator<<(lambda.operator()());
  return 0;
}
```
Output is UB...
```
/home/insights/insights.cpp:6:12: warning: address of stack memory associated with local variable 'j' returned [-Wreturn-stack-address]
    6 |   return [&j]{ return j; };
      |            ^
/home/insights/insights.cpp:6:12: note: captured by reference here
    6 |   return [&j]{ return j; };
```

This was trivial example, but imagine situation when you copy lambdas between different threads, operations on reference may lead to race conditions and weird results.

[back](/)