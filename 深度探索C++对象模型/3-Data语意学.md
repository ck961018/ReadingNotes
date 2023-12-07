# 三、Data语意学
* 空的class大小为1byte，以便其对象在内存中获得独一无二的地址
* 一个virtual base class subobject只会在derived class中存在一份实例
- - - 
## Data Member的绑定
对成员函数的分析在整个class的声明都出现后才开始，但参数列表除外。因此，诸如`typdef`、`using`的嵌套类型声明应放在class起始处。
- - -
## Data Member的布局
* static成员变量不保存在对象布局之中
* nonstatic成员变量<font color=red>在每个access section中</font>按声明顺序排列
- - -
## Data Member的存取
#### Static Data Members
对静态成员变量的调用与对象无关，类似于全局变量（只在class生命范围内可见），从数据段直接存取。
#### Nonstatic Data Members
非静态成员变量的地址为类对象地址加上成员变量的偏移地址。
当成员变量是virtual base class的member时，需要推迟到执行期才能得知偏移位置。
其它情况下，在编译期就能解析出偏移地址，执行效率完全相同。
- - -
## “继承”与Data Member
#### 只要继承不要多态
在仅继承的情况下，子类的空间占用是自身的大小加上基类的大小，而不会填补基类的对齐空间。  
这样设计是为了防止将基类对象赋值给子类对象时影响到子类对象的成员。
#### 加上多态
每个对象中需要引入一个vptr。
#### 多重继承
若将vptr放在对象的最后，可以保持“自然多态”：子类转换为多态指针只需要复制地址。
多重继承中，将对象转换为“最左端”基类指针时，与单一继承相同具有“自然多态”。而其它情况则需要在执行期计算偏移。
***（然而现在大多数编译器都将vptr放在对象最前端）***
#### 虚拟继承
虚拟继承中会将菱形继承来的virtual base class和其它部分分开布局，然后用virtual base class指针指向virtual base class，现在编译器通常将virtual base class放在最后。  
为了保证固定存取时间，编译器会将所有嵌套virtual base class指针拷贝下来。但这又会导致每个对象对每个virtual base class都需要背负一个额外指针。  
书中提出了两种解决方法，不过现在通常使用第二种：在vptr所指位置的前面放对象地址相对virtual base class的偏移量。这样只需要用类似`&obj + __vptr__class[-1]`的方式就能获得实际地址。
- - -
## 对象成员的效率
**<font color=red>普通封装和继承在编译器优化后不会带来任何成本，只有虚拟继承会影响成员变量的存取效率。</font>**
- - -
## 指向Data Members的指针
指向数据成员的指针中存的是成员变量在对象中的偏移位置。书中说会将所有指向数据成员的指针加一，以区分“没有指向任何data member”的指针和“指向第一个data member”的指针。  
**不过当前的编译器通常是将“没有指向任何data member”的指针减一以进行区分。**  
*当多重继承时，只有声明了指针的实际类型，编译器才会给指向Data Members的指针复制：*
```cpp
class A
{
public:
    int x;
};

class B
{
public:
    int y;
};

class C : public A, public B
{
public:
    int z;
};

int main()
{
    auto p0 = &B::y;     // p0 == 0
    int C::* p1 = &B::y; // p0 == 4
}
```
#### “指向Member的指针”的效率问题
和之前的结论一样：经过优化后，只有虚拟继承会对成员变量的存取效率产生影响。
