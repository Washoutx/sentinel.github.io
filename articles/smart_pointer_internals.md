---
layout: default
---

# Let's check Smart Pointers internals

Template TemplateTemplateTemplateTemplateTemplateTemplateTemplateTemplateTemplate
TemplateTemplateTemplateTemplateTemplateTemplateTemplate
TemplateTemplateTemplateTemplate

```cpp
#include <cmath>

int main()
{
    auto process = [](std::string_view str)
    {
        std::cout << "str: " << std::quoted(str) << ", ";
    };
 
    for (auto src : {"42", "42abc", "meow", "inf"})
        process(src);
}
```

[back](./)