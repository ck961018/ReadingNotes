# align.h
## align
```
_LIBCPP_EXPORTED_FROM_ABI void* align(size_t __align, size_t __sz, void*& __ptr, size_t& __space); // 声明

void* align(size_t alignment, size_t size, void*& ptr, size_t& space) { // 实现
  void* r = nullptr;
  if (size <= space) {
    char* p1 = static_cast<char*>(ptr);
    char* p2 = reinterpret_cast<char*>(reinterpret_cast<uintptr_t>(p1 + (alignment - 1)) & -alignment);
    size_t d = static_cast<size_t>(p2 - p1);
    if (d <= space - size) {
      r   = p2;
      ptr = r;
      space -= d;
    }
  }
  return r;
}
```
首先解释参数：
```
alignment : 对齐字节数，需要为2的倍数
size      : 需要的内存大小
ptr       : 指向当前可用内存起始地址的指针
space     : 'ptr'指向的内存中可用空间大小
```
再逐行讲解代码:
```
void* r = nullptr;
if (size <= space) {
```
初始化结果指针并判断可用空间是否足够。
```
char* p1 = static_cast<char*>(ptr);
char* p2 = reinterpret_cast<char*>(reinterpret_cast<uintptr_t>(p1 + (alignment - 1)) & -alignment);
```
第一句将``ptr``强制转换为 char* 类型，以保证字节级操作。  
第二句可以分为两个部分：  
``p1 + (alignment - 1))`` 先让``p1``增加``alignment - 1``使得结果在目标对齐点及其下一个对齐点之间；  
``& -alignment`` 再和对齐字节数的负数补码执行"与"操作。由于对齐字节数是2的倍数，其负数补码可以保证只有对齐位为1，其他位都是0。
```
size_t d = static_cast<size_t>(p2 - p1);
if (d <= space - size) {
  r   = p2;
  ptr = r;
  space -= d;
}
```
接着检查偏移后剩下的空间是否可以容纳size大小的数据，是的话就更新指针和剩余空间。
```
return r;
```
对齐成功会返回对齐后的指针地址，否则返回空指针。
