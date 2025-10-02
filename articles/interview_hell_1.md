---
layout: default
---

# Interview hell #1: What is ADL/SSO/SBO/EBO/ODR/SFINAE/CRTP/CTAD/RTTI/POD?

In thic article we would try to explain all the magic shortcuts which can be asked by the reviewer for C++ position.
Without a special introduction lets begin.

### ADL - Argument-Dependent Lookup
ADL is compiler functionality which helps to find function in the specific namespace based on in which namespace is
parameter. Check the following code snipet:

```cpp
namespace test
{
    struct A{};
    void foo(A a){};
}

int main()
{
    foo(test::A()); // OK - automatically foo is find the same namespace as parameter
    test::foo(test::A()); // OK
    // test::foo(A()); Compiler: error: 'A' was not declared in this scope
    return 0;
}
```
### SSO - Small String Optimization | SBO - Small Buffer Optimization | SOO - Small Object Optimization
This optimizations SOO are implementation defined, but most famous are std::strind(SSO) and std::function(SBO).
Long story short this optimization try to keep the data on stack (inside the object) instead on heap.
How is it possible? std::string needs to have 3 things: Pointer, size, capacity. (Additionaly flag if SSO is used.)
Pointer depends on architecture so the size can be 4 or 8 bytes.
Size and Capacity have type std::size_t which also depends on architecture but let's assume 8 bytes.
So 24 bytes are the minimum which we need even for empty std::string. With the following union trick we can reuse this memory in two ways:

```cpp
struct StringSSO
{
    size_t size;
    // this is why usually 15 characters are stored on stack with SSO, last one is '\0'
    char sso_buffer[16];
    // No capacity, buffer is constant 16
};

struct StringHeap
{
    size_t size;
    size_t capacity;
    char* ptr;
};

union StringData
{
    StringSSO sso;
    StringHeap heap;
};

int main()
{
    std::cout << "StringSSO=" << sizeof(StringSSO) << std::endl;
    std::cout << "StringHeap=" << sizeof(StringHeap) << std::endl;
    std::cout << "StringData=" << sizeof(StringData) << std::endl;
    return 0;
}
```
```
StringSSO=24
StringHeap=24
StringData=24
```
Of course you need to somehow distinguish which mode is used so additional memory is needed for some kind of flag. But as I said this is implementation defined it will depends on standard library implementation (libstdc++(GNU) vs libc++(LLVM)). Above example is just pseudo code to illustrate the idea.

Similar is with std::function. std::function is a wrapper for collable object or just a function. Following example shows that in case of
lambda(which is class object with operator()), the allocation happen when the object will be bigger.

```cpp
void* operator new(size_t size)
{   
    std::cout << "new called\n";
    return std::malloc(size);
}

int main()
{
    std::string str = "Hello world";
    auto lambda1 = [&str](){};
    auto lambda2 = [str](){}; // copy so str is a part of lambda2
    std::cout << "lambda1=" << sizeof(lambda1) << std::endl;
    std::cout << "lambda2=" << sizeof(lambda2) << std::endl;

    std::cout << "fun1\n";
    std::function fun1 = lambda1;
    std::cout << "fun2\n";
    std::function fun2 = lambda2; // BTW: It is example of CTAD 

    return 0;
}
```
```
lambda1=8
lambda2=32
fun1
fun2
new called
```
### EBO/EBCO - Empty Base (Class) optimization
What is the size of empty class? No, it is not 0bytes. It is at least one byte. Why? Because even if you have empty class you need to
have opportunity to get pointer of the object. So if I inherite other class then size is 2 bytes? No because of EBCO.

```cpp
struct Base {
    int getValue() { return 1670; }
};
struct Derived : Base {};
struct Composition
{
    int32_t i;
    Base base; 
};
struct Composition2
{
    int32_t i;
    [[no_unique_address]] Base base; 
};
 
int main()
{
    std::cout << "sizeof(Base)=" << sizeof(Base) << std::endl;
    std::cout << "sizeof(Derived)=" << sizeof(Derived) << std::endl;
    std::cout << "sizeof(Composition)=" << sizeof(Composition) << std::endl;
    std::cout << "sizeof(Composition2)=" << sizeof(Composition2) << std::endl;
}
```
```
sizeof(Base)=1
sizeof(Derived)=1
sizeof(Composition)=8
sizeof(Composition2)=4
```
[[no_unique_address]] is a suggestion for compiler, if is it possible please don't assign new address for base.

### ODR - One Definition Rule
The title says everything :)
```cpp
// a.cpp
void foo(){};
// b.cpp
void foo(){};
// main.cpp
extern void foo(){};

int main() {
    foo();
    return 0;
}
```
```
> clang++ -std=c++20 main.cpp a.cpp b.cpp && ./a.out
duplicate symbol 'foo()' in:
    /private/var/folders/bx/lms2wpfs1890hdnpzv76glzm0000gn/T/b-e93a54.o
    /private/var/folders/bx/lms2wpfs1890hdnpzv76glzm0000gn/T/a-89ff6e.o
    /private/var/folders/bx/lms2wpfs1890hdnpzv76glzm0000gn/T/main-8cce0a.o
ld: 1 duplicate symbols
clang++: error: linker command failed with exit code 1 (use -v to see invocation)
```
There is one workaround. If you specified function as inline, then ODR can be break. And linker just select first found definition.
There is one mit that inline unroll definition in the place of function call. It also works that way but it is just a suggestion for compiler.
Compiler finally decide by himself if do it.
```cpp
// a.cpp
inline void foo(){};
// b.cpp
inline void foo(){};
// main.cpp
extern void foo(){};

int main() {
    foo();
    return 0;
}
```
### SFINAE - Substitution Failure Is Not An Error
It means if compiler try to substitute the type in one template and it fails then it is not compilation error until compiler check all the
template definitions.

```cpp
/*1*/ template<typename T> 
typename std::enable_if_t<std::is_integral_v<T>, void> print(T t) {
    std::cout << t << " is integral\n";
}

/*2*/ template<typename T>
typename std::enable_if_t<!std::is_integral_v<T>, void> print(T t) {
    std::cout << t << " isn't integral\n";
}

int main() {
    print(1);
    print(1.1);
}
```
1. print(1) - compiler try to use first template. Substitute is sucessful.
2. print(1.1) - compiler try to use first template. Substitude is unsucessful so compiler try second template. It is not report compilation error due to SFINAE. 

Lets remove second template then all substitution are failed and following compilation error can occur:

```
<source>:11:5: error: no matching function for call to 'print'
   11 |     print(1.1);
      |     ^~~~~
<source>:5:56: note: candidate template ignored: requirement 'std::is_integral_v<double>' was not satisfied [with T = double]
    5 | typename std::enable_if_t<std::is_integral_v<T>, void> print(T t) {
      |                                                        ^
1 error generated.
```

### CRTP - Curiously Recurring Template Pattern
[back](./)