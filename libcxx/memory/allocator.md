# allocator.h
## 一.__non_trivial_if
继承它的类可以根据条件_Cond来决定自身的平凡性(triviality)。  
```
template <bool _Cond, class _Unique>
struct __non_trivial_if {};
```
当_Cond为false时，__non_trivial_if为空类，继承它的类会保持自身的平凡性。

```
template <class _Unique>
struct __non_trivial_if<true, _Unique> { // 模板特化
  _LIBCPP_HIDE_FROM_ABI _LIBCPP_CONSTEXPR __non_trivial_if() _NOEXCEPT {}
};
```
而当_Cond为true时，__non_trivial_if会包含一个空构造函数，使得它的派生类非平凡；  

具体到Allocator模块中，allocator类会根据_Tp的类型来决定自身的平凡性。  
当_Tp为void时，allocator\<void\>将是平凡的以保证兼容之前的版本；否则，allocator\<_Tp\>将是非平凡的。

## 二.allocator<_Tp>
C++20中，allocator类仅包含两个成员函数：allocate和deallocate。
### 1.allocate函数
```
_LIBCPP_NODISCARD_AFTER_CXX17 _LIBCPP_HIDE_FROM_ABI _LIBCPP_CONSTEXPR_SINCE_CXX20 _Tp* allocate(size_t __n) {
  if (__n > allocator_traits<allocator>::max_size(*this))
    __throw_bad_array_new_length();
  if (__libcpp_is_constant_evaluated()) {
    return static_cast<_Tp*>(::operator new(__n * sizeof(_Tp)));
  } else {
    return static_cast<_Tp*>(std::__libcpp_allocate(__n * sizeof(_Tp), _LIBCPP_ALIGNOF(_Tp)));
  }
}
```
逐行进行讲解：
```
if (__n > allocator_traits<allocator>::max_size(*this))
  __throw_bad_array_new_length();

```
这一句判断请求的大小是否超过分配器允许的大小，超过大小时抛出异常。
```
if (__libcpp_is_constant_evaluated()) {
  return static_cast<_Tp*>(::operator new(__n * sizeof(_Tp)));
}

```
然后判断当前是否在编译期，若在编译期，则直接调用全局new操作符进行内存分配。
```
else {
  return static_cast<_Tp*>(std::__libcpp_allocate(__n * sizeof(_Tp), _LIBCPP_ALIGNOF(_Tp)));
}
```
否则调用std::__libcpp_allocate函数分配内存，该函数会在最底层调用编译器内建函数__builtin_operator_new，以实现内存对齐及避免用户自定义的new操作符。

### 2.deallocate函数
```
_LIBCPP_HIDE_FROM_ABI _LIBCPP_CONSTEXPR_SINCE_CXX20 void deallocate(_Tp* __p, size_t __n) _NOEXCEPT {
  if (__libcpp_is_constant_evaluated()) {
    ::operator delete(__p);
  } else {
    std::__libcpp_deallocate((void*)__p, __n * sizeof(_Tp), _LIBCPP_ALIGNOF(_Tp));
  }
}
```
deallocate函数更为简洁明了。
```
if (__libcpp_is_constant_evaluated()) {
  ::operator delete(__p);
}
```
若在编译期，直接调用全局delete操作符。
```
else {
  std::__libcpp_deallocate((void*)__p, __n * sizeof(_Tp), _LIBCPP_ALIGNOF(_Tp));
}
```
否则调用std::__libcpp_deallocate释放内存，该函数同样会在最底层调用编译器内建函数__builtin_operator_delete，以避免调用用户自定义的delete操作符。

## 三.allocator<const _Tp>
allocator的const特化，与allocator\<_Tp\>大同小异，仅在函数形参、模板参数、返回值处将*_Tp更换为了const *_Tp，这里就不再赘述。

## 四.operator==
```
template <class _Tp, class _Up>
inline _LIBCPP_HIDE_FROM_ABI _LIBCPP_CONSTEXPR_SINCE_CXX20 bool
operator==(const allocator<_Tp>&, const allocator<_Up>&) _NOEXCEPT {
  return true;
}
```
两个allocator对象调用==操作符进行比较，永远返回true。
这个设计反映了标准库中认为所有allocator都是等价的，可以互换使用。
