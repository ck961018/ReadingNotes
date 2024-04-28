# pointer_traits.h
## 一.__has_element_type
```
template <class _Tp, class = void>
struct __has_element_type : false_type {};

template <class _Tp>
struct __has_element_type<_Tp, __void_t<typename _Tp::element_type> > : true_type {};
```
常见的类型萃取写法，利用了 SFINAE 技术。当 _Tp 类型包含 element_type 时继承自 true_type ，否则继承自 false_type 。

## 二.__pointer_traits_element_type
```
template <class _Ptr, bool = __has_element_type<_Ptr>::value>
struct __pointer_traits_element_type {};

template <class _Ptr>
struct __pointer_traits_element_type<_Ptr, true> {
  typedef _LIBCPP_NODEBUG typename _Ptr::element_type type;
};

template <template <class, class...> class _Sp, class _Tp, class... _Args>
struct __pointer_traits_element_type<_Sp<_Tp, _Args...>, true> {
  typedef _LIBCPP_NODEBUG typename _Sp<_Tp, _Args...>::element_type type;
};

template <template <class, class...> class _Sp, class _Tp, class... _Args>
struct __pointer_traits_element_type<_Sp<_Tp, _Args...>, false> {
  typedef _LIBCPP_NODEBUG _Tp type;
};
```
也是很常见的写法，将 type 定义为 _Ptr 类型的 element_type 。  
如果 _Ptr 是个模板容器且没有 element_type ，则将容器的第一个模板参数类型当作 type 。  
如果 _Ptr 没有 element_type ，也不是模板容器，则 __pointer_traits_element_type 为空。

## 三.__has_difference_type
```
template <class _Tp, class = void>
struct __has_difference_type : false_type {};

template <class _Tp>
struct __has_difference_type<_Tp, __void_t<typename _Tp::difference_type> > : true_type {};
```
与 __has_element_type 一样，判断类型 _Tp 是否包含 difference_type。

## 四.__pointer_traits_difference_type
```
template <class _Ptr, bool = __has_difference_type<_Ptr>::value>
struct __pointer_traits_difference_type {
  typedef _LIBCPP_NODEBUG ptrdiff_t type;
};

template <class _Ptr>
struct __pointer_traits_difference_type<_Ptr, true> {
  typedef _LIBCPP_NODEBUG typename _Ptr::difference_type type;
};
```
若 _Ptr 没有 difference_type ，则将 type 定义为 ptrdiff_t ，64位系统上通常是 long int ；
若 _Ptr 包含 difference_type , 则将 type 定义为 difference_type 。

## 五.__has_rebind
```
template <class _Tp, class _Up>
struct __has_rebind {
private:
  template <class _Xp>
  static false_type __test(...);
  _LIBCPP_SUPPRESS_DEPRECATED_PUSH
  template <class _Xp>
  static true_type __test(typename _Xp::template rebind<_Up>* = 0);
  _LIBCPP_SUPPRESS_DEPRECATED_POP

public:
  static const bool value = decltype(__test<_Tp>(0))::value;
};
```
依旧是 SFINAE ，当 _Tp 中不包含接受 _Up 作为参数的模板成员 rebind<_Up> 时，匹配返回值为 false_type 的 __test ；  
否则匹配返回值为 true_type 的 __test 。  
这里还用到了不求值表达式 decltype , 用于获取 __test 的返回值类型 ，同时避免编译时检查 __test 的实现。

## 六.__pointer_traits_rebind
```
template <class _Tp, class _Up, bool = __has_rebind<_Tp, _Up>::value>
struct __pointer_traits_rebind {
  typedef _LIBCPP_NODEBUG typename _Tp::template rebind<_Up> type;
#endif
};

template <template <class, class...> class _Sp, class _Tp, class... _Args, class _Up>
struct __pointer_traits_rebind<_Sp<_Tp, _Args...>, _Up, true> {
  typedef _LIBCPP_NODEBUG typename _Sp<_Tp, _Args...>::template rebind<_Up> type;
#endif
};

template <template <class, class...> class _Sp, class _Tp, class... _Args, class _Up>
struct __pointer_traits_rebind<_Sp<_Tp, _Args...>, _Up, false> {
  typedef _Sp<_Up, _Args...> type;
};
```
如果 _Tp / _SP 中包含接受 _Up 作为参数的模板成员 rebind<_Up> ，将 type 定义为 rebind<_Up> ；
如果 _Sp 中不包含 rebind<_Up> ，但 _Sp 本身是接受 _Up 作为模板参数的容器，那么将 type 定义为 _Sp<_Up, _Args...> type 。
否则没有模板匹配。
