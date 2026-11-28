`extern "C"` 用于指定 C++ 代码采用 C 语言链接约定（Language Linkage），主要解决 **C 和 C++ 混合编程时的符号链接兼容问题**

- C++ 支持函数重载，因此编译器通常会对函数名进行 Name Mangling（名称修饰），将参数类型等信息编码进符号名。
- C 语言通常不使用 C++ 这种函数重载名称修饰规则。
- `extern "C"` 告诉 C++ 编译器使用 C 语言链接约定，使 C 和 C++ 能够通过兼容的符号名称进行链接。

## 举例

```C
// 这是 C 代码
int add(int a, int b) {
    return a + b;
}

// 这是 C 头文件
#ifndef MATH_H
#define MATH_H

int add(int a, int b);

#endif
```

C++调用C库
```c++
#include <iostream>

// 告诉 C++ 编译器，这些函数使用 C 语言链接约定
extern "C" {
    #include "math.h"
}

int main() {
    int result = add(10, 20);

    std::cout << result << std::endl;

    return 0;
}
```

C语言能直接调用C++的代码，但是C 语言不能够完全理解 C++ 语法
它只能调用 C++ 对外暴露的、符合 C ABI 的接口。

ABI（Application Binary Interface，应用程序二进制接口）是指**不同编译单元、库或程序之间，在二进制层面进行交互时遵循的一套约定。**