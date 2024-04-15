代码中出现的宏都会在这里进行讲解
```
_LIBCPP_NODISCARD_AFTER_CXX17 : 版本大于C++17时，等价于[[nodiscard]]，否则为空；
_LIBCPP_HIDE_FROM_ABI         : 顾名思义，从ABI中隐藏。扩展为几个clang内部宏，使其不会出现在动态链接库中；
_LIBCPP_CONSTEXPR_SINCE_CXX20 : 版本大于C++20时，等价于constexpr，否则为空。用于编译期的空间分配。
_LIBCPP_NO_CFI                : 禁用控制流完整性检查(CFI)
```
