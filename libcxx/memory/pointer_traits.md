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

