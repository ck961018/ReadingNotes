# addressof.h
## 一.addressof(_Tp&)
```
template <class _Tp>
inline _LIBCPP_CONSTEXPR_SINCE_CXX17 _LIBCPP_NO_CFI _LIBCPP_HIDE_FROM_ABI _Tp* addressof(_Tp& __x) _NOEXCEPT {
  return __builtin_addressof(__x);
}
```
提供了一种可靠的方式来获取左值对象的地址。  
通过调用编译器内建函数 __builtin_addressof 的方式，来避免调用用户的 & 操作符重载。  
这里声明 _LIBCPP_NO_CFI 主要是出于性能考虑。

## 二.addressof(const _Tp&&)
```
template <class _Tp>
_Tp* addressof(const _Tp&&) noexcept = delete;
```
禁止对右值调用 addressof。
