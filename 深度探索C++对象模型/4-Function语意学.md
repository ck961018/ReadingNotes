# 四、Function语意学
## Member的各种调用方式
#### 非静态成员函数
* 非静态成员函数会被编译器转换为<font color=red>以类对象指针为第一个参数</font>的**非成员函数**。
* 所有的类成员名称都会被进行特殊处理（Mangling）以绑定具体类。
* 成员函数的Name Mangling仅会考虑<font color=green>函数名称</font>、<font color=green>参数个数</font>和<font color=green>参数类型</font>。这就是为什么不能重载仅返回值不同的函数，也说明了为什么声明时的参数变量名可以和定义时不同。

#### 虚拟成员函数
* 虚拟成员函数的调用经由vptr索引。
* 虚拟成员函数若调用同一个类中的其它虚拟成员函数，则不需要vptr索引，类似非静态成员函数直接调用即可。

#### 静态成员函数
本质上就是个非成员函数。
- - - 
## 虚拟成员函数
当类中定义了虚函数，就会生成虚函数指针和虚函数表以支持多态。

#### 多重继承下的Virtual Functions
#### 虚拟继承下的Virtual Functions
*书中并没有详细讲解虚拟继承，因此这里将两节笔记放在一起*

当发生以下三种情况时，需要在<font color=red>执行期</font>根据实际类型来调整指针：  
1. 通过“指向非最左继承链基类”的指针，调用派生类虚函数；
2. 通过“指向派生类”的指针，调用非最左继承链基类的虚函数；
3. 通过“指向非最左继承链基类”的指针，调用修改返回值类型为派生类的虚函数。

调整指针所需要的偏移量都保存在虚函数表中。其中，最左继承链的虚函数表被称为<font color=yellow>主要表格</font>，其它表格都是<font color=yellow>次要表格</font>。

* 次要表格中会放置“次要表格vptr相对主要表格指针”的偏移量（***top_offset***）。
* 当存在虚函数时，表格中会放置“当前表vptr相对于virtual base class”的偏移量（***vbase_offset***）。
* 通过thunk技术实现 **“指向派生类对象的基类指针能够正确调用派生类重写的虚函数”** 。当存在虚拟继承时，又需要“当前表vptr相对于override函数所在表vptr”的偏移量（***vcall_offset***)。

实际虚函数表结构如下：
|虚函数表                 | 
|:-:                      |
|vcall_offset             |
|vbase_offset             |
|top_offset               |
|RTTI information         |
|virtual function pointers|

vcall_offset是为了解决菱形继承中的二义性问题。考虑如下代码：
```cpp
class A {
public:
    virtual void Test() {}
    virtual ~A() = default;
};

class B : virtual public A {
public:
    void Test() override {}
};

class C : virtual public A {
};

class D: public B, public C {
}; 
```
当指向A类的指针调用D类型的Test()函数时，无法从A和D的vptr中获取实际的override函数，因此需要vcall_offset来记录A类vptr相对于B类vptr的偏移量。

对于thunk技术，这里用个简单的案例演示一下：
```cpp
class A {
public:
    virtual ~A() {}
};

class B {
public:
    virtual ~B() {}
};

class C : public A, public B {};

int main() {
    A* c = new C;
    delete c;
}
```
对于以上代码，x86-64平台下的gcc13.2将其编译为如下汇编：
```x86asm
main:
        push    rbp
        mov     rbp, rsp
        push    rbx
        sub     rsp, 24
        mov     edi, 16
        call    operator new(unsigned long)
        mov     rbx, rax
        mov     rdi, rbx
        call    C::C() [complete object constructor]
        mov     QWORD PTR [rbp-24], rbx
        mov     rax, QWORD PTR [rbp-24]
        test    rax, rax
        je      .L9
        mov     rdx, QWORD PTR [rax]
        add     rdx, 8
        mov     rdx, QWORD PTR [rdx]
        mov     rdi, rax
        call    rdx
.L9:
        mov     eax, 0
        mov     rbx, QWORD PTR [rbp-8]
        leave
        ret
```
将cpp代码中对象c的类型改为B后，汇编如下：
```x86asm
main:
        push    rbp
        mov     rbp, rsp
        push    rbx
        sub     rsp, 24
        mov     edi, 16
        call    operator new(unsigned long)
        mov     rbx, rax
        mov     rdi, rbx
        call    C::C() [complete object constructor]
        test    rbx, rbx
        je      .L9
        lea     rax, [rbx+8]
        jmp     .L10
.L9:
        mov     eax, 0
.L10:
        mov     QWORD PTR [rbp-24], rax
        mov     rax, QWORD PTR [rbp-24]
        test    rax, rax
        je      .L11
        mov     rdx, QWORD PTR [rax]
        add     rdx, 8
        mov     rdx, QWORD PTR [rdx]
        mov     rdi, rax
        call    rdx
.L11:
        mov     eax, 0
        mov     rbx, QWORD PTR [rbp-8]
        leave
        ret
```
第二段汇编将第一段汇编中的部分代码封装成了`.L10`。两段代码的区别仅在于第二段进入`.L10`之前的`lea rax, [rbx+8]`指令。  
根据书中的描述，第二段代码在调用析构函数前需要通过thunk技术调整this指针到实际类型的位置。由此，可以看出thunk技术本质上就是通过汇编代码调整this指针。

*有个书中没有提到的知识点：派生类自己定义的虚函数只会保存在最左侧继承链的虚函数表中。*
- - - 
## 函数的效能
单一继承、多重继承、虚拟继承的效率依次降低，但书中说测试多重继承的编译器不支持thunk技术。于是我用msvc14.22进行了测试，结果单一继承和多重继承效率基本一致。
- - -
## 指向Member Function的指针
指向非虚函数的指针是一个函数地址。
#### 支持“指向Virtual Member Functions”的指针
书上说指向虚函数的指针是虚函数表中的索引值，不过如今编译器的实现有所不同。  
在gcc和clang中，输出的是1、9、17等值，推测是地址偏移量加一。  
而在msvc中，输出的值直接是个地址，但在汇编中看到这样的结果：`OFFSET [thunk]:ClassName::"vcall'{0,{flat}}' }'"`，其中的数字应该是地址偏移量，在64位下的排布是：0,8,16,24……
#### 在多重继承之下，指向Member Functions的指针
参考上一节，msvc的实现和书中描述的一样。不过书中提到的额外结构式是否还在被应用，我没有进行过测试。但如今的虚函数表中都是用thunk技术实现虚函数，这个结构体被使用的可能性很低。
#### “指向Member Functions之指针”的效率
和之前的结论一样，只有虚函数和虚拟继承会造成效率的降低。
- - -
## Inline Functions
*如今inline关键字的含义已经完全不同，现代编译器完全不会参考用户定义的inline关键字来进行内联，因此这节略。*

