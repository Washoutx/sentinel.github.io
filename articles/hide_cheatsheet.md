---
layout: default
---

# Everything and nothing, personal notes on topic which I always forgot -,-

`using ValueType = std::decay_t<T>` - decay removes CV qualifiers, reference and also convert array/function to raw pointer. 
Struct by default can have maximum alignment equal to `alignof(std::max_align_t)`
Unless it is specified by the coder like this `struct alignas(64) B {}`

Unordered containers increase number of buckets when max_load_factor() (impl defined but mostly equal '1') is achieved.
Load factor is a ratio between number of elements and number of buckets.

`;` - semicolon<br>
`:` - colon<br>
`::` - scope resolution operator in C++<br>

```cpp
class Base {
public:
    virtual void sayHello() {
        cout << "Hello from Base\n";
    }
};

class Derived : public Base {
public:
    void sayHello() override {
        cout << "Hello from Derived\n";
        Base::sayHello();
    }
};

int main() {
    Derived d;

    d.sayHello();
    d.Base::sayHello();
}
```
`std::valarray` is a container optimized for numerical calculations. It should be easier for compiler to use vectorized
operations using SIMD instructions.

`std::reduce` (not good for subtraction since it is not commutative - PL - przemienne) has flags
`std::execution::seq`<br>
`std::execution::par` - on multiple threads<br>
`std::execution::par_unseq` – threads + SIMD if CPU and compiler supports SIMD<br>


Iterators
`Input/Output <- Forward <- bidirectional <- random-access <- contiguous`

Functions like std::unique or std::remove doesn't touch the number of elements in container. So for example unique just moving front all the valid elements
and the rest is moved after the iterator which is new sentinel for the sequence. So the 'bad' values are still in the container. We need then manually erease them
using the returned iterator. Like erease(iterator from unique, end of contrainer).
Standard lib algorithms cannot allocate memory by themselfs so we need to preallocate output container or use method like std::back_inserter
```cpp
const auto square_func = [](int x) { return x * x; };
const auto v = std::vector{1, 2, 3, 4};
auto squared = std::vector<int>{};
squared.resize(v.size());
std::ranges::transform(v, squared.begin(), square_func);
 // OR //
const auto square_func = [](int x) { return x * x; };
const auto v = std::vector{1, 2, 3, 4};
auto squared_vec = std::vector<int>{};
auto dst_vec = std::back_inserter(squared_vec);
std::ranges::transform(v, dst_vec, square_func);
```

"Compare with zero" optimization a very subtle optimization is that counter
is iterated backward in order to compare with zero instead of a value. On some
CPUs, comparing with zero is slightly faster than any other value, as it uses another
assembly instruction (on the x86 platform, it uses test instead of cmp).
Algorithms

The following table shows the difference in assembly output using gcc 9.2:
Action C++ Assembler x86 Compare with zero
```cpp
auto cmp_zero(size_t val) {
 return val > 0;
}
```
```
test edi, edi
setne al
ret
```
```cpp
Compare with the other value
auto cmp_val(size_t val) {
 return val > 42;
}
```
```
cmp edi, 42
setba al
ret
```

ALIASING SHARED_PTR
`-ftrapv` is a compiler flag which will terminate (CORE DUMP) the program when signed integer overflow occurs.
[back](/)