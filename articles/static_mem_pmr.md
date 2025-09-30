---
layout: default
---

# Static memory pools and PMR(Polymorphic Memory Resource)

Handling heap memory comes with a performance overhead, so if we know how much memory we need, we may prefer to use memory pools.
In general, a `MemoryPool` is a class type that provides a set of methods to manage the memory it encapsulates.
`MemoryPool` can handle either static or dynamic memory. Using dynamic memory pool we can reduce `new` calls to one when we initialize the `MemoryPool` object, otherwise we can use static memory pool and reduce `new` calls to zero. Actually it won't be zero but instead we will use samething called `placement new`. I will back to this later.

The concept of a dynamic memory pool is fairly simple, so we'll focus on the static one instead. Let's define the requirements for this implementation. The code will be simplified, as the goal is just to demonstrate the concept.
1. use static memory only
2. avoid memory fragmentation
3. initialize memory using placement new instead of returning raw memory

The first two points are pretty straightforward, but what exactly is [placement new](https://en.cppreference.com/w/cpp/language/new.html#Placement_new) and why is memory alignment important?
Let’s take a look at the following code snippet.
```cpp
struct Object
{
    Object(int32_t _i) : i(_i) { std::println("Object()"); }
    ~Object() { std::println("~Object()"); }
    int32_t i;
};

int main()
{
    /*1*/ alignas(Object) std::byte buffer[10];
    /*2*/ Object* ptr = new (buffer) Object(1670);
    std::println("ptr->i={}", ptr->i);
    /*3*/ ptr->~Object();
    // std::println("ptr->i={}", ptr->i); Undefined behavior
    return 0;
}
```
```
Object()
ptr->i=1670
~Object()
```
1. Creation of a 10-byte buffer aligned to `alignof(Object)` which is `4` in our case.
Memory alignment ensures that the object is initialized at a memory address that satisfies the type's alignment requirements. For example, the address must fulfill the condition `addr % alignof(Object) == 0` to be valid for an Object. In the context of our example, the alignment condition is guaranteed at the start of the buffer due to `alignas(Object)`, and it will also be satisfied for any subsequent address offset by a multiple of `alignof(Object)` as long as it remains within the bounds of the buffer. If you try to initialize an `Object` at `buffer+1`, the address will likely be misaligned, resulting in undefined behavior. But I should mention here that x86 supports unaligned memory access, and even modern ARM architectures support it as well — although it can cause performance penalties. On the other hand, `buffer+4` is valid because it satisfies the alignment requirement. To safely obtain an aligned address within a buffer, check out `std::align`.

2. Placement new syntax that initializes an Object at the beginning of the buffer.
This is a way to construct an object in pre-allocated memory without allocating new memory from the heap.

3. Manual destructor call, which is required in the case of placement new.
This is one of the rare cases in C++ where the destructor needs to be called explicitly, since there is no such thing as `placement delete`.

Now that we understand how placement new works, we can move on to implementing the `MemoryPool`.

```cpp
template<typename T, std::size_t N>
class MemoryPool {
public:
    MemoryPool() = default;
    MemoryPool(const MemoryPool&) = delete;
    MemoryPool(MemoryPool&&) = delete;
    MemoryPool& operator=(const MemoryPool&) = delete;
    MemoryPool& operator=(MemoryPool&&) = delete;
    ~MemoryPool() {
        for(std::size_t idx = 0; idx < slots.size(); ++idx) {
            if(slots[idx]) {
                T* ptr = reinterpret_cast<T*>(&buffer_[idx * sizeof(T)]);
                ptr->~T();
            }
        }
    }

    template <typename... Args>
    T* allocate(Args&&... args) {
        if (!slots.all()) {
            for(int idx = 0; idx < slots.size(); ++idx) {
                if(!slots[idx]) {
                    slots.set(idx);
                    T* ptr = new (&buffer_[idx * sizeof(T)]) T(std::forward<Args>(args)...);
                    /* std::launder allows the compiler to correctly interpret that a new object 
                    has been created in memory, enabling safe optimizations under the strict aliasing rules. */
                    return std::launder(ptr);
                }
            }
        }
        return nullptr;
    }

    void deallocate(T* ptr) {
        if (!ptr) return;

        if(owns(ptr)) {
            ptr->~T();
            slots.reset((std::uintptr_t(ptr) - std::uintptr_t(buffer_)) / sizeof(T));
        }
        else
        {
            throw std::invalid_argument("Pointer does not belong to this memory pool"); 
        }
    }

private:
    bool owns(const T* ptr) {
        if(!ptr) return false;

        if (std::uintptr_t(ptr) >= std::uintptr_t(buffer_) 
            && std::uintptr_t(ptr) < std::uintptr_t(buffer_) + N * sizeof(T)) {
           if(std::uintptr_t(ptr) % alignof(T) == 0) return true; 
        }
        return false;
    } 

    // std::aligned_storage_t<sizeof(T), alignof(T)> buffer_[N]; deprecated since c++23
    alignas(T) std::byte buffer_[N * sizeof(T)];
    std::bitset<N> slots{};
};

int main() {
    MemoryPool<Object, 10> mem_pool;
    auto* obj1 = mem_pool.allocate(1670);
    auto* obj2 = mem_pool.allocate(1671);

    std::println("obj1={} obj1->i={}", static_cast<void*>(obj1), obj1->i);
    mem_pool.deallocate(obj1);
    std::println("obj2={} obj2->i={}", static_cast<void*>(obj2), obj2->i);

    auto* obj3 = mem_pool.allocate(1673);
    std::println("obj3={} obj3->i={}", static_cast<void*>(obj3), obj3->i);

    return 0;
}
```
```
Object()
Object()
obj1=0x16bc2ed58 obj1->i=1670
~Object()
obj2=0x16bc2ed5c obj2->i=1671
Object()
obj3=0x16bc2ed58 obj3->i=1673
~Object()
~Object()
```
My MemoryPool is divided into memory blocks of size T. A `std::bitset<N> slots{}` is used to track which blocks are currently allocated.
However, this implementation has a performance drawback. The slots bitset must be scanned each time a new object is allocated via the `T* allocate(Args&&... args)`. For example, if there are 100,000 total elements and 50,000 of them are already in use, the allocation loop may need to check up to 50,000 bits before finding a free slot.

The purpose of the bitset is to avoid memory fragmentation. Suppose we had a simpler implementation that used only a buffer and a pointer to the next available slot. If an object were deallocated somewhere in the middle of the buffer, it would leave a "hole" in memory that wouldn't be reused (It can be reuse but it isn't easy). By using slots, we can clear the flag for a specific block when it is deallocated, allowing it to be reused in future allocations. As shown in our example, `obj3` is allocated at the address previously occupied by `obj1`, which was deallocated. Of course, we could extend this implementation by adding a container to track free indexes, but I’ve kept it simple for the purpose of this article.

The `void deallocate(T* ptr)` function first checks whether the pointer is within the buffer's address range. The `bool owns(const T* ptr)` function uses `std::uintptr_t` to compare pointer values. This is an unsigned integer type guaranteed to be large enough to hold any pointer value on the current platform. For example, on a 32-bit architecture, `std::uintptr_t` will typically be 4 bytes, while on a 64-bit architecture, it will be 8 bytes. This ensures that all virtual memory addresses can safely be stored and compared using this type. 

Checking whether the address is within the buffer is not enough. I also verify that the address is properly aligned using the following condition:
`if (std::uintptr_t(ptr) % alignof(T) == 0) return true;`
This ensures that the pointer is not pointing to a byte in the middle of an object. Dereferencing such a misaligned pointer to call the destructor would result in undefined behavior (UB).

As I mentioned earlier, this example is not perfect, but it illustrates the basic idea of a static memory pool.
In a real-world project, we would typically use external resources like `Boost.Pool`, or standard solutions such as `PMR (Polymorphic Memory Resource)` — which will be the focus of the next part of this article.

We will start this part with a custom allocator. It is a class that provides allocate/deallocate methods (optionally also construct/destroy):
```cpp
template<class T>
struct CustomAlloc {
    using value_type = T;

    T* allocate(std::size_t n) {
        std::println("allocate");
        return static_cast<T*>(::operator new(n * sizeof(T)));
    }

    void deallocate(T* p, std::size_t) noexcept {
        std::println("deallocate");
        ::operator delete(p);
    }
};
```
Our example is not doing anything fancy but you can use it to trace allocation and deallocation for example in vector. There is one issue with custom allocators: they change the underlying type of the container.
```cpp
    std::vector<int> v1;
    std::vector<int, CustomAlloc<int>> v2;
    std::pmr::vector<int> v3;
```
The compiler may expand these declarations as follows (you can check this using the excellent tool [cppinsights](https://cppinsights.io/)):
```cpp
  std::vector<int, std::allocator<int> > v1 = std::vector<int, std::allocator<int> >();
  std::vector<int, CustomAlloc<int> > v2 = std::vector<int, CustomAlloc<int> >();
  std::vector<int, std::pmr::polymorphic_allocator<int> > v3 = std::vector<int, std::pmr::polymorphic_allocator<int> >();
```
Because of this, we cannot write a single function like `void foo(std::vector<int>)` that works for both v1 and v2 because v1 uses default `std::allocator<int>` allocator.
Of course, you might have noticed the third example, v3, which belongs to the special `std::pmr::` namespace. Its default allocator is `std::pmr::polymorphic_allocator<int>`. This is a special adaptor that internally holds a pointer to a `std::pmr::memory_resource`.
So what is `std::pmr::memory_resource`? It is an abstract interface with three pure virtual functions:
```cpp
virtual void* do_allocate( std::size_t bytes, std::size_t alignment ) = 0;
virtual void do_deallocate( void* p, std::size_t bytes, std::size_t alignment ) = 0;
virtual bool do_is_equal( const std::pmr::memory_resource& other ) const noexcept = 0;
```
These methods need to be overridden in our custom allocator. Let’s update our code so that `CustomAlloc` can be reused in a `std::pmr::vector<>`
```cpp
template<class T>
struct CustomAlloc : std::pmr::memory_resource {
    virtual void* do_allocate(std::size_t bytes, std::size_t alignment) override {
        std::println("do_allocate");
        void* ptr = std::aligned_alloc(alignment, bytes);
        if (!ptr) {
            throw std::bad_alloc();
        }
        return ptr;
    }
    
    virtual void do_deallocate(void* p, 
                               [[maybe_unused]]std::size_t bytes, 
                               [[maybe_unused]] std::size_t alignment) override {
        std::println("do_deallocate");
        if (p) {
            std::free(p);
        }
    }
    
    virtual bool do_is_equal(const std::pmr::memory_resource& other) const noexcept override {
        return this == &other;
    }
};

int main() {
    CustomAlloc<int> alloc;
    std::pmr::vector<int> vec(&alloc);
    return 0;
}
```
As you can see, I am not changing the base allocator. `std::pmr::polymorphic_allocator` holds a pointer to a `std::pmr::memory_resource`. By inheriting memory_resource, `CustomAlloc` can be passed to the allocator.
We can also create many custom allocators this way, and all vectors using different allocators can still be passed around with the same API, e.g. `void foo(std::pmr::vector<int>)`
The allocator does not change the type of the vector. It will always be `std::vector<T, std::pmr::polymorphic_allocator<T>>`.
And is that all? Not quite. Let’s take a look at what else the standard library provides in the `std::pmr` namespace.

`std::pmr::monotonic_buffer_resource` is a concept similar to our memory pool. We can construct it with an external memory buffer like this:
```cpp
    std::array<std::byte, 1024> buffer;
    std::pmr::monotonic_buffer_resource mbr{buffer.data(), buffer.size()};
    std::pmr::vector<int> list{&mbr};
```
or a variant with a fallback memory resource, which I will describe later
```cpp
    std::array<std::byte, 1024> buffer;
    std::pmr::monotonic_buffer_resource mbr{buffer.data(), buffer.size(), std::pmr::null_memory_resource()};
    std::pmr::vector<int> list{&mbr};
```
Ok, let’s pause for a moment and describe what is going on here. `std::pmr::monotonic_buffer_resource` is useful when we know the amount of data in advance and their lifetime is short, because deallocation only happens when the container using this memory resource is destroyed.

But what happens if the monotonic buffer consumes the entire external buffer? In our static memory pool implementation above, we simply return nullptr. In contrast, `std::pmr::monotonic_buffer_resource` has an additional attribute: a pointer to a fallback mechanism called the upstream resource. By default, it uses `std::pmr::get_default_resource()`.

1. `std::pmr::get_default_resource()` returns `std::pmr::new_delete_resource()` by default, or another resource if `std::pmr::set_default_resource()` has been called.
2. `std::pmr::new_delete_resource()` returns a pointer to the memory resource, which simply uses new and delete in its allocate() and deallocate() functions.
3. `std::pmr::null_memory_resource()` returns a pointer to a memory resource whose allocate() always throws `std::bad_alloc`, while its deallocate() does nothing.

So what does this mean in practice? If a memory resource exhausts its entire external buffer, it will fall back to its upstream resource. That upstream might simply use new/delete to obtain more memory, throw an exception, or behave differently if you implement your own custom upstream resource.

Another things are wrappers to memory resources called pool resource.
1. `std::pmr::unsynchronized_pool_resource`
2. `std::pmr::synchronized_pool_resource`

They work the same way, but the synchronized version is thread-safe. These classes use the upstream resource as a managed buffer. For example, we can pass them a `std::pmr::monotonic_buffer_resource` like this:
```cpp
int main() {
    std::array<std::byte, 512> buffer;
    std::pmr::monotonic_buffer_resource mbr{buffer.data(), 
                                            buffer.size(), 
                                            std::pmr::null_memory_resource()};

    std::pmr::pool_options opts{.max_blocks_per_chunk = 10, 
                                .largest_required_pool_block = 1};
    
    std::pmr::unsynchronized_pool_resource pool(opts, &mbr);
    std::pmr::vector<std::byte> vec(&pool);
    try
    {
        for (int i = 0; i < 10; ++i) {
            vec.push_back(std::byte{});
        }
    }
    catch(std::bad_alloc e)
    {
        std::cout << e.what();
    }
    return 0;
}
```
With this configuration, `std::pmr::unsynchronized_pool_resource` allocates 10-byte blocks per chunk request from the upstream resource. 
A block is returned to the chunk after deallocation, so its behavior is similar to my static memory pool.
The buffer is also used to store internal metadata, so you need to be careful when choosing the buffer size. The example above will still throw an exception, even if I provide a 512-byte buffer. (TODO: Debug it and provide more details about memory usage)

So that’s it for `PMR`. Try experimenting with this yourself. Cheers!

[back](./)