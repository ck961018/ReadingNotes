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
如果 _Sp 中不包含 rebind<_Up> ，但 _Sp 本身是接受 _Up 作为模板参数的容器，那么将 type 定义为 _Sp<_Up, _Args...> type ；  
否则没有模板匹配。

## 七.__pointer_traits_impl`
```
template <class _Ptr, class = void>
struct __pointer_traits_impl {};

template <class _Ptr>
struct __pointer_traits_impl<_Ptr, __void_t<typename __pointer_traits_element_type<_Ptr>::type> > {
  typedef _Ptr pointer;
  typedef typename __pointer_traits_element_type<pointer>::type element_type;
  typedef typename __pointer_traits_difference_type<pointer>::type difference_type;

  template <class _Up>
  using rebind = typename __pointer_traits_rebind<pointer, _Up>::type;

private:
  struct __nat {};

public:
  _LIBCPP_HIDE_FROM_ABI _LIBCPP_CONSTEXPR_SINCE_CXX20 static pointer
  pointer_to(__conditional_t<is_void<element_type>::value, __nat, element_type>& __r) {
    return pointer::pointer_to(__r);
  }
};
```
pointer_traits 非指针版本的实现。  
利用前面几个萃取工具获取 _Ptr 类型的 element_type, difference_type 和 rebind 成员。  
pointer_to 成员函数用于获取指向引用对象的指针。

## 八.pointer_traits
```
template <class _Ptr>
struct _LIBCPP_TEMPLATE_VIS pointer_traits : __pointer_traits_impl<_Ptr> {};

template <class _Tp>
struct _LIBCPP_TEMPLATE_VIS pointer_traits<_Tp*> {
  typedef _Tp* pointer;
  typedef _Tp element_type;
  typedef ptrdiff_t difference_type;

  template <class _Up>
  using rebind = _Up*;
#endif

private:
  struct __nat {};

public:
  _LIBCPP_HIDE_FROM_ABI _LIBCPP_CONSTEXPR_SINCE_CXX20 static pointer
  pointer_to(__conditional_t<is_void<element_type>::value, __nat, element_type>& __r) _NOEXCEPT {
    return std::addressof(__r);
  }
};

template <class _From, class _To>
using __rebind_pointer_t = typename pointer_traits<_From>::template rebind<_To>;
```
当 _Ptr 非指针时，直接采用 __pointer_traits_impl 的实现；  
当 _Ptr 为指针时，特化 element_type, difference_type 和 rebind 。  
此外，指针版本的 pointer_to 成员函数将直接调用 std::addressof 库函数。

__rebind_pointer_t 是直接获取 rebind 成员的语法糖。

## 九._IsFancyPointer
```
template <class _Pointer, class = void>
struct _HasToAddress : false_type {};

template <class _Pointer>
struct _HasToAddress<_Pointer, decltype((void)pointer_traits<_Pointer>::to_address(std::declval<const _Pointer&>())) >
    : true_type {};

template <class _Pointer, class = void>
struct _HasArrow : false_type {};

template <class _Pointer>
struct _HasArrow<_Pointer, decltype((void)std::declval<const _Pointer&>().operator->()) > : true_type {};

template <class _Pointer>
struct _IsFancyPointer {
  static const bool value = _HasArrow<_Pointer>::value || _HasToAddress<_Pointer>::value;
};
```
这几个模板结构体联系比较紧密，因此放在一起讲解。  

_HasToAddress 与 _HasArrow 都很好理解，分别用来检测 _Pointer 是否包含 to_address 成员函数和重载的 operator-> 成员函数。  
其中 std::declval 在不求值表达式中获取类型对应的对象，避免 _Pointer 类型不包含无参构造函数。  

_IsFancyPointer 通过检查 _HasToAddress 和 _HasArrow 来确定一个类型是否为 Fancy 指针类型。 

## 十.__to_address
```
template <class _Pointer, class = void>
struct __to_address_helper;

template <class _Tp>
_LIBCPP_HIDE_FROM_ABI _LIBCPP_CONSTEXPR _Tp* __to_address(_Tp* __p) _NOEXCEPT {
  static_assert(!is_function<_Tp>::value, "_Tp is a function type");
  return __p;
}

template <class _Pointer, __enable_if_t< _And<is_class<_Pointer>, _IsFancyPointer<_Pointer> >::value, int> = 0>
_LIBCPP_HIDE_FROM_ABI _LIBCPP_CONSTEXPR
    __decay_t<decltype(__to_address_helper<_Pointer>::__call(std::declval<const _Pointer&>()))>
    __to_address(const _Pointer& __p) _NOEXCEPT {
  return __to_address_helper<_Pointer>::__call(__p);
}

template <class _Pointer, class>
struct __to_address_helper {
  _LIBCPP_HIDE_FROM_ABI _LIBCPP_CONSTEXPR static decltype(std::__to_address(
      std::declval<const _Pointer&>().operator->()))
  __call(const _Pointer& __p) _NOEXCEPT {
    return std::__to_address(__p.operator->());
  }
};

template <class _Pointer>
struct __to_address_helper<_Pointer,
                           decltype((void)pointer_traits<_Pointer>::to_address(std::declval<const _Pointer&>()))> {
  _LIBCPP_HIDE_FROM_ABI _LIBCPP_CONSTEXPR static decltype(pointer_traits<_Pointer>::to_address(
      std::declval<const _Pointer&>()))
  __call(const _Pointer& __p) _NOEXCEPT {
    return pointer_traits<_Pointer>::to_address(__p);
  }
};

```
如果 __p 为指针， __to_address 直接返回 __p；  
如果 __p 是 Fancy 指针类型，则调用 __to_address_helper 的 __call 函数来获取指向的地址；  
否则没有模板匹配。

__to_address_helper 会判断 _Pointer 包含 to_address 成员函数还是 operator-> 重载成员函数，在 __call 中分别调用对应的函数。 

## 十一.to_address
```
template <class _Tp>
inline _LIBCPP_HIDE_FROM_ABI constexpr auto to_address(_Tp* __p) noexcept {
  return std::__to_address(__p);
}

template <class _Pointer>
inline _LIBCPP_HIDE_FROM_ABI constexpr auto to_address(const _Pointer& __p) noexcept
    -> decltype(std::__to_address(__p)) {
  return std::__to_address(__p);
}
```
综合以上的实现，获取 __p 指针或对象指向的地址。
