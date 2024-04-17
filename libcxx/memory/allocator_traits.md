# allocator_traits.h
## 一._LIBCPP_ALLOCATOR_TRAITS_HAS_XXX(NAME, PROPERTY)
```
#define _LIBCPP_ALLOCATOR_TRAITS_HAS_XXX(NAME, PROPERTY)                                                               \
  template <class _Tp, class = void>                                                                                   \
  struct NAME : false_type {};                                                                                         \
  template <class _Tp>                                                                                                 \
  struct NAME<_Tp, __void_t<typename _Tp::PROPERTY > > : true_type {}
```
一个用于定义类型萃取辅助类的宏，该类的功能是判断模板参数类中是否定义 PROPERTY 成员。

to be continue
