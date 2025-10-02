---
layout: default
---

# Interview hell #1: What is ADL/SSO/SBO/EBO/ODR/SFINAE/CRTP/CTAD/RTTI/POD?

In this article, we will try to explain some common shortcuts that may be asked about during a C++ interview. Without further introduction, let’s begin.

### ADL - Argument-Dependent Lookup
ADL is a compiler feature that helps the compiler find functions in the namespace associated with the type of a function’s parameter. Check the following code snippet:

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
These optimizations are implementation-defined, but the most well-known examples are std::string (SSO) and std::function (SBO).
In short, these optimizations attempt to keep data on the stack (inside the object) instead of allocating it on the heap.
How is this possible? A std::string typically needs to store three things: a pointer, a size, and a capacity. (Additionally, some implementations include a flag to indicate whether SSO is used others looks on capacity.)
The pointer size depends on the architecture, so it can be either 4 or 8 bytes. Both size and capacity are of type std::size_t, which also depends on the architecture, but let’s assume 8 bytes each.
That means at least 24 bytes are needed even for an empty std::string. With the following union trick, this memory can be reused in two different ways:
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
As mentioned earlier, this behavior is implementation-defined and depends on the standard library (e.g., libstdc++ in GNU vs. libc++ in LLVM). The above example is only pseudocode intended to illustrate the idea.

A similar concept applies to std::function. std::function is a type-erasing wrapper that can store either a callable object or a function pointer. In the case of a lambda (which is essentially a class object with an operator()), allocation occurs only if the callable object is too large to fit into the small buffer.

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
    auto lambda2 = [str](){}; // str pass by copy so str is a part of lambda2
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
What is the size of an empty class? No, it is not 0 bytes. It is at least one byte. Why? Because even an empty class object must have a unique address, so you must be able to take its pointer.
So if I inherit from another empty class, will the size be 2 bytes? No, because of EBCO (Empty Base Class Optimization).

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
[[no_unique_address]] is a hint to the compiler. It tells the compiler that, if possible, it should not assign a separate address for this member (often used with empty types), allowing for optimizations similar to EBCO.

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
There is a common workaround. If you specify a function as inline, it can be defined in multiple translation units without violating ODR. The linker then resolves these definitions safely.
There is a common myth that inline forces the compiler to substitute the function body at the call site. While this can happen, it is only a suggestion for the compiler. The compiler ultimately decides whether to perform inlining.
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
This means that if the compiler tries to substitute a type into a template and it fails, it is not treated as a compilation error. The compiler will continue checking other template definitions to find a match.

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
1. print(1) - the compiler tries to use the first template. Substitution is successful.
2. print(1.1) - the compiler tries to use the first template. Substitution is unsuccessful, so the compiler tries the second template. No compilation error is reported due to SFINAE.

Let’s remove the second template. Then all substitutions fail, and the following compilation error can occur:

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
It is a programming idiom. The base class is a template that takes the derived class as a parameter.
You can find it in the `std::enable_shared_from_this<T>` class, which enables creating a `shared_ptr` from the `this` pointer.
```cpp
template <typename CRTP>
class Y {
public:
    void get() {
        static_cast<CRTP*>(this)->foo();
    }
};

class X : public Y<X> {
public:
    void foo() {
        std::cout << "Hi!";
    }
};

int main() {
    X x;
    x.get();
    return 0;
}
```

### CTAD - Class template argument deduction (since C++17)
When a template class is instantiated, you usually need to specify template arguments, like this:
```cpp
std::pair<std::string, int> p{"one", 1};
```
Since C++17, for certain classes, their template arguments can be deduced:
```cpp
std::pair p{"one", 1};
```
### Extra: Structured binding declaration (since C++17)
cppref definition: Binds the specified names to subobjects or elements of the initializer.
```cpp
std::pair<std::string, int> p{"one", 1};
auto [str, i] = p; // Structured binding declaration
std::cout << str << ", " << i;
```
```
one, 1
```
### RTTI - Run-time type information
When doing metaprogramming, we rely on type information at compile time. C++ also provides a mechanism for obtaining type information at run time, called RTTI, which includes features like typeid and dynamic_cast.
```cpp
struct Base{};
struct BaseVirt{
    virtual void foo (){};
};

struct A : public Base{};
struct B: public BaseVirt {};

int main() {
    A a;
    B b;
    Base& base = a;
    BaseVirt& baseVirt = b;
    const std::type_info& ti_a = typeid(a);
    const std::type_info& ti_b = typeid(b);
    const std::type_info& ti_base = typeid(base);
    const std::type_info& ti_baseVirt = typeid(baseVirt);

    std::cout << "typeid(a)=" << ti_a.name() << std::endl;
    std::cout << "typeid(b)=" << ti_b.name() << std::endl;
    std::cout << "typeid(base)=" << ti_base.name() << std::endl;
    std::cout << "typeid(baseVirt)=" << ti_baseVirt.name() << std::endl;

    dynamic_cast<B&>(baseVirt); // OK

    try {
        // Unrelated classes, for references dynamic_cast throws, for pointers just returns nullptr
        dynamic_cast<A&>(baseVirt);
    }
    catch(std::bad_cast& ex) {
        std::cout << ex.what();
    }
    return 0;
}
```
```
typeid(a)=1A
typeid(b)=1B
typeid(base)=4Base
typeid(baseVirt)=1B
std::bad_cast
```
Number before the name is how long the name is. dynamic_cast based on RTTI. As you can notice RTTI
distinguish polimorphic types.

### POD - Plain Old Data (also known as aggregate) (deprecated in C++20)
A POD type is compatible with C: it has no special functions, no polymorphism that affects memory layout, and behaves in a simple, predictable way.
C++11 introduced the following type traits to describe these properties more precisely:

1) std::is_trivial – the type has no user-defined or non-trivial special functions (constructors, destructors, or copy/move operators)
2) std::is_standard_layout – the memory layout is predictable and not affected by implicit changes like vtables
3) std::is_trivially_copyable – the type can be copied bitwise

Thanks to these traits, we have more precise control over types. In C++20, the concept of POD is deprecated, because we can now specify type properties more precisely using the above type traits.

```cpp
struct Base{};
struct A : Base { ~A() = default; };
struct B{ ~B(){}; };
struct C{ C(){}; };
struct D{ virtual void foo(){} };
int main()
{
    std::cout << std::boolalpha;
    std::cout << "std::is_trivial_v<A>=" << std::is_trivial_v<A> << std::endl;
    std::cout << "std::is_standard_layout_v<A>=" <<  std::is_standard_layout_v<A> << std::endl;
    std::cout << "std::is_trivially_copyable_v<A>=" <<  std::is_trivially_copyable_v<A> << std::endl;
    std::cout << std::endl;
    std::cout << "std::is_trivial_v<B>=" << std::is_trivial_v<B> << std::endl;
    std::cout << "std::is_standard_layout_v<B>=" <<  std::is_standard_layout_v<B> << std::endl;
    std::cout << "std::is_trivially_copyable_v<B>=" <<  std::is_trivially_copyable_v<B> << std::endl;
    std::cout << std::endl;
    std::cout << "std::is_trivial_v<C>=" << std::is_trivial_v<C> << std::endl;
    std::cout << "std::is_standard_layout_v<C>=" <<  std::is_standard_layout_v<C> << std::endl;
    std::cout << "std::is_trivially_copyable_v<C>=" <<  std::is_trivially_copyable_v<C> << std::endl;
    std::cout << std::endl;
    std::cout << "std::is_trivial_v<D>=" << std::is_trivial_v<D> << std::endl;
    std::cout << "std::is_standard_layout_v<D>=" <<  std::is_standard_layout_v<D> << std::endl;
    std::cout << "std::is_trivially_copyable_v<D>=" <<  std::is_trivially_copyable_v<D> << std::endl;
    return 0;
}
```

```
1) struct A{ ~A() = default; };
std::is_trivial_v<A>=true
std::is_standard_layout_v<A>=true
std::is_trivially_copyable_v<A>=true

2) struct B{ ~B(){}; };
std::is_trivial_v<B>=false
std::is_standard_layout_v<B>=true
std::is_trivially_copyable_v<B>=false

3) struct C{ C(){}; };
std::is_trivial_v<C>=false
std::is_standard_layout_v<C>=true
std::is_trivially_copyable_v<C>=true

4) struct D{ virtual void foo(){} };
std::is_trivial_v<D>=false
std::is_standard_layout_v<D>=false
std::is_trivially_copyable_v<D>=false
```
Note: Good example which proves that `~A() = default;` is not equal `~A(){};`

1.  - Destructor is = default → trivial.
    - No virtual functions or special constructors → standard layout.
    - Can be copied bitwise → trivially copyable.
    - A POD type remains POD under inheritance, unless virtual inheritance is used.
2.  - Explicit destructor {} → not trivial.
    - No vtable → standard layout.
    - Non-trivial destructor → not trivially copyable. A non-trivial destructor prevents the compiler from implicitly generating the copy constructor and copy assignment operator.
3.  - Explicit default constructor → not trivial.
    - No virtuals → standard layout.
    - Can still be copied bitwise → trivially copyable.
4.  - Has virtual function → non-trivial, non-standard layout, not trivially copyable.

[back](/)