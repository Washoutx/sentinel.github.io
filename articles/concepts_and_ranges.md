---
layout: default
---

# C++20: Concepts and Ranges

What is concept? It it step forward in metaprogramming. Lets check following example with simple concept 
and few ways how to use it.
```cpp
template <typename T>
concept IsValid = requires(T obj) {
    obj.foo();
    { obj.goo() } -> std::same_as<int>;
} && std::derived_from<T, struct Base>;


struct Base{};
struct Derived1 : Base {
    void foo(){};
    int goo(){ return 2117; };
};

struct Derived2 : Base {
    void foo(){};
    double goo(){ return 2117; };
};

void callObjMethods1(IsValid auto obj) {
    obj.foo();
    obj.goo();
}

template <IsValid T>
void callObjMethods2(T obj) {
    obj.foo();
    obj.goo();
}

template <typename T>
requires IsValid<T>
void callObjMethods3(T obj) {
    obj.foo();
    obj.goo();
}

int main()
{
    callObjMethods1(Derived1());
    // callObjMethods1(Derived2()); Compilation failed

    callObjMethods2(Derived1());
    // callObjMethods2(Derived2()); Compilation failed

    callObjMethods3(Derived1());
    // callObjMethods3(Derived2()); Compilation failed
    return 0;
}
```
Concept `IsValid` verifies if type has two methods, one of them returns int, and the type inherits from Base type.
`Derived2` type doesn't have `goo()` method which returns int, so compilation will fail. Let's check the compiler output.

```
<source>:24:6: note: candidate: 'template<class auto:53>  requires  IsValid<auto:53> void callObjMethods(auto:53)'
   24 | void callObjMethods(IsValid auto obj)
      |      ^~~~~~~~~~~~~~
<source>:24:6: note:   template argument deduction/substitution failed:
<source>:24:6: note: constraints not satisfied
<source>: In substitution of 'template<class auto:53>  requires  IsValid<auto:53> void callObjMethods(auto:53) [with auto:53 = Derived2]':
<source>:33:19:   required from here
<source>:18:9:   required for the satisfaction of 'IsValid<auto:53>' [with auto:53 = Derived2]
<source>:18:19:   in requirements with 'T obj' [with T = Derived2]
<source>:21:14: note: 'obj.goo()' does not satisfy return-type-requirement
   21 |     { obj.goo() } -> std::same_as<int>;
      |       ~~~~~~~^~
```
One of the reasons for introducing concepts was to improve compiler error readability. Raw templates can generate massive amounts of logs, and analyzing them can be a bit of a pain.

Since concepts can verify our argument type more precisly, standard library was extended by new verions of algorithms. For now you can call algorithms with just name of the container.
```cpp
std::vector<int> values{9, 2, 5, 3, 4};
std::sort(values.begin(), values.end()); // old

std::ranges::sort(values); // new
std::ranges::sort(values.begin(), values.end()); // legacy
```

New version of algorithms provides also different cool features called `Projections`. Let`s analyse following code snippet.
```cpp
struct Player
{
    std::string name;
    int type;
    int level;
};

auto print = [](int id, auto v)
{
    std::cout << "#" << id << "\n";
    for(auto &p : v)
    {
        std::cout << "name=" << p.name << " type=" << 
            p.type << " level=" << p.level << std::endl;
    }
};

int main() {

    std::vector<Player> players 
    {
        {"B", 1, 2},
        {"C", 2, 1},
        {"A", 3, 4},
        {"D", 3, 3},
    };

    std::ranges::sort(players, std::less<>{}, &Player::level);
    print(1, players);
    std::ranges::sort(players, std::less<>{}, &Player::name);
    print(2, players);

    // Custom lambda
    std::ranges::sort(players, std::less<>{}, [](const Player& v)
    {
        return std::tie(v.type, v.level);
    });
    print(3, players);   
    return 0;
}
```
```
#1
name=C type=2 level=1
name=B type=1 level=2
name=D type=3 level=3
name=A type=3 level=4
#2
name=A type=3 level=4
name=B type=1 level=2
name=C type=2 level=1
name=D type=3 level=3
#3
name=B type=1 level=2
name=C type=2 level=1
name=D type=3 level=3
name=A type=3 level=4
```
As you can notice we can specify which class attribute should be use to order the values. Also we can make our own lambda to sort based on more than one value.
`std::tie` returns `std::tuple<const int&, const int&>` in this case, tuple has generated comparison operator so it is valid object for sort.

So what is `std::range`? Is an namespace with group of algorithms (like `std::ranges::sort` used above) and classes. Actually there is a lot of things, full list: [ranges on cppref](https://en.cppreference.com/w/cpp/header/ranges.html). Main concept of ranges is `std::ranges::range`. 
```cpp
// https://en.cppreference.com/w/cpp/ranges/range.html
template< class T >
concept range = requires( T& t ) {
    ranges::begin(t);
    ranges::end (t);
};
``` 
We will focus on `lazy evaluated` algorithms. What is lazy avaluated algorithm? It can be easily illustrated by following code snippet:
```cpp
struct Player
{
    std::string name;
    int type;
    int level;
};

int main() {
    std::vector<Player> players 
    {
        {"B", 1, 2},
        {"C", 2, 1},
        {"A", 3, 4},
        {"D", 3, 3},
    }; 

    auto players_view = std::ranges::ref_view(players);
    auto type_3_players = std::ranges::filter_view(players_view, [](auto& p) {
        std::cout << "TYPE 3 FILTER PREDICATE\n";
        return p.type == 3;
    });
    auto double_level_type_3 = std::ranges::transform_view(type_3_players, [](auto& p) {
        std::cout << "DOUBLE LEVEL\n";
        return std::pair(p.name, p.level*=2);
    });

    std::cout << "Main...\n";

    for(const auto &p : double_level_type_3)
    {
        std::cout << p.first << "=" << p.second <<  std::endl;
    }

    return 0;
}
```
```
Main...
TYPE 3 FILTER PREDICATE
TYPE 3 FILTER PREDICATE
TYPE 3 FILTER PREDICATE
DOUBLE LEVEL
A=8
TYPE 3 FILTER PREDICATE
DOUBLE LEVEL
D=6
```
In the code above we are looking for players type==3 and want to double their level. 
As you can see lambdas are called when we go through the... Wait, what is `double_level_type_3` exactly? Is it new
container or what? Actually it is not. It just a wrapper which contains iterators and callable object like lambda. So `lazy evaluated` means that those lambdas are called on iterators of players when we trigger access to it in the loop. This is why logs from lambdas are printed after the "Main..." string.

Thanks to [cppinsights](https://cppinsights.io/) we can check the exact type of `double_level_type_3`. 
```cpp
std::ranges::transform_view<
    std::ranges::filter_view<
        std::ranges::ref_view<std::vector<Player, std::allocator<Player> > >, __lambda_20_66>, __lambda_24_76>
            double_level_type_3 = 
                std::ranges::transform_view<
                    std::ranges::filter_view<
                        std::ranges::ref_view<std::vector<Player, std::allocator<Player> > >, __lambda_20_66>, __lambda_24_76>(std::ranges::filter_view<std::ranges::ref_view<std::vector<Player, std::allocator<Player> > >, __lambda_20_66>(type_3_players), __lambda_24_76{});
```
Luckily we have `auto` keyword so we don't need to write it ourselfs (Actually it won't be possible to write it manually because lambda type is generated by the compiler). So as you can see `double_level_type_3` is
a nested `view` template classes. So going through `double_level_type_3` object is generally calling two lambdas on the all `players` elements (iterators). To be more specific, second lambda is called only on iterators passed by filter `type_3_players`. This is why `double_level_type_3` lambda is called only twice since there are only two players type 3.

The code above can be written more cleanly using view adaptors. Let's see following code snippet:

```cpp
struct Player
{
    std::string name;
    int type;
    int level;
};

int main() {
    std::vector<Player> players 
    {
        {"B", 1, 2},
        {"C", 2, 1},
        {"A", 3, 4},
        {"D", 3, 3},
    }; 

    auto type_3_filter = [](auto& p) {
        std::cout << "TYPE 3 FILTER PREDICATE\n";
        return p.type == 3;
    };

    auto double_level = [](auto& p) {
        std::cout << "DOUBLE LEVEL\n";
        return std::pair(p.name, p.level*=2);
    };

    auto double_level_type_3 = players | std::views::filter(type_3_filter) 
                                       | std::views::transform(double_level);

    std::cout << "Main...\n";

    for(const auto &p : double_level_type_3)
    {
        std::cout << p.first << "=" << p.second <<  std::endl;
    }

    return 0;
}
```
In both examples original container is untouched. `operator|` is used to chain the operations. 

What if we want to make a new container with those two players?
```cpp
    std::vector<std::pair<std::string,int>> new_players;
    std::ranges::copy(double_level_type_3, std::back_inserter(new_players));
    // auto new_players = double_level_type_3 | std::ranges::to<std::vector>(); // Since C++23
```

Let's see another interesting features of ranges. 
Code to easly parse and convert string to vector of ints:
```cpp
    std::string data = {"0/1/2/3/4/5/6/7/8/9/10"};
    auto parsed_data = data | std::views::split('/') 
                            | std::views::transform([](const auto& substring){
                                return std::stoi(std::string(substring.begin(), substring.end()));})
                            | std::ranges::to<std::vector>();

    for(auto& obj : parsed_data)
    {
        std::cout << obj << "\n";
    } 
```
Ranges also provides api for files:
```cpp
        auto fd = std::ifstream("data.txt");
        for (auto num : std::ranges::istream_view<float>(fd)) {
          std::cout << num << '\n';
        }
```
```
> clang++ -std=c++20 main.cpp
> cat data.txt            
1.223 2.99 3.33
> ./a.out                    
1.223
2.99
3.33
```
Ranges and concepts are a lot more but I think it was a good introduction to the topic.

[back](/)