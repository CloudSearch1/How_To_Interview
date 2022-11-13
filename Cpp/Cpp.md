### 1.Cpp中的内联函数

内联函数是通常与类一起使用。如果一个函数是内联的，那么在编译时，编译器会把该函数的代码副本放置在每个调用该函数的地方（会在此文件中在写一份）。对内联函数进行任何修改，都需要重新编译函数的所有客户端，因为编译器需要重新更换一次所有的代码，否则将会继续使用旧的函数。
如果想把一个函数定义为内联函数，则需要在函数名前面放置关键字inline，在调用函数之前需要对函数进行定义。如果已定义的函数多于一行，编译器会忽略inline限定符。在类定义中定义的函数都是内联函数，即使没有使用 inline 说明符。下面是一个实例，使用内联函数来返回两个数中的最大值。



```c++
inline int Max(int a, int b){
    return (a > b)? a:b;
}
int main(){
    cout << "Max(20, 10): " << Max(20, 10) << endl;
    cout << "Max(0, 100): " << Max(0, 100) << endl;
    cout << "Max(100, 1010): " << Max(100, 1010) << endl;
    return 0;
}
```

内联函数inline: 引入内联函数的目的是为了解决程序中函数调用的效率问题，程序在编译器编译的时候，编译器将程序中出现的内联函数的调用表达式用内联函数的函数体进行替换。而对于其他的函数，都是在运行时候才被替代,这其实就是个空间代价换时间的节省。所以内联函数一般都是1-5行的小函数。在使用内联函数时要留神：
在内联函数内不允许使用循环语句和开关语句;
内联函数的定义必须出现在内联函数第一次调用之前;
类结构中所在的类说明内部定义的函数是内联函数
内联函数使用的注意点:
Tip: 只有当函数只有10行甚至更少时才将其定义为内联函数.
当函数被声明为内联函数之后, 编译器会将其内联展开, 而不是按通常的函数调用机制进行调用
优点:当函数体比较小的时候, 内联该函数可以令目标代码更加高效. 对于存取函数以及其它函数体比较短, 性能关键的函数, 鼓励使用内联.
缺点:滥用内联将导致程序变慢. 内联可能使目标代码量或增或减, 这取决于内联函数的大小. 内联非常短小的存取函数通常会减少代码大小, 但内联一个相当大的函数将戏剧性的增加代码大小. 现代处理器由于更好的利用了指令缓存, 小巧的代码往往执行更快。
结论:一个较为合理的经验准则是, 不要内联超过10行的函数. 谨慎对待析构函数, 析构函数往往比其表面看起来要更长, 因为有隐含的成员和基类析构函数被调用!

虚函数

### 1、虚函数表指针

什么是虚函数表指针？
我们把对象从首地址开始的4个字节或者是8个字节，这个位置我们称之为虚函数表指针（vptr），它里面包含一个地址指向的就是虚函数表（vftable）的地址。

### 2、虚函数表

什么是虚函数表，他又在哪里，有什么用？
虚函数表就是里面是一组地址的数组（函数指针数组），他所在的位置就是虚函数表指针里面所存储的地址，它里面所包含的地址就是我们重写了父类的虚函数的地址（没有重写父类的虚函数那么默认的就是父类的函数地址）。

### 3、虚函数（virtual）可以是内联函数（inline）吗?

虚函数可以是内联函数，内联是可以修饰虚函数的，但是当虚函数表现多态性的时候不能内联。
内联是在编译期建议编译器内联，而虚函数的多态性在运行期，编译器无法知道运行期调用哪个代码，因此虚函数表现为多态性时（运行期）不可以内联。
inline virtual 唯一可以内联的时候是：编译器知道所调用的对象是哪个类（如 Base::who()），这只有在编译器具有实际对象而不是对象的指针或引用时才会发生。

```c++
#include <iostream>  
using namespace std;
class Base
{
public:
  inline virtual void who()
  {
    cout << "I am Base\n";
  }
  //~Base() {} // 如果基类析构函数不是虚函数，此时只会按指针类型调用（此例为Base类型）该类型的析构函数代码，所以只会调用Base的析构函数，不会调用子类的析构函数，导致内存泄露。
  virtual ~Base() {}
};
class Derived : public Base
{
public:
  inline void who()  // 不写inline时隐式内联
  {
    cout << "I am Derived\n";
  }
  ~Derived() {};
};

int main()
{
  // 此处的虚函数 who()，是通过类（Base）的具体对象（b）来调用的，编译期间就能确定了，所以它可以是内联的，但最终是否内联取决于编译器。 
  Base b;
  b.who();

  // 此处的虚函数是通过指针调用的，呈现多态性，需要在运行时期间才能确定，所以不能为内联。  
  Base* ptr = new Derived();
  ptr->who();

  // 因为Base有虚析构函数（virtual ~Base() {}），所以 delete 时，会先调用派生类（Derived）析构函数，再调用基类（Base）析构函数，防止内存泄漏。
  // 如果基类析构函数不是虚函数，此时只会按指针类型调用（此例为Base类型）该类型的析构函数代码，所以只会调用Base的析构函数，不会调用子类的析构函数，导致内存泄露。
  delete ptr;
  ptr = nullptr;

  system("pause");
  return 0;
}

//运行结果：
//I am Base
//I am Derived
```
利用虚函数实现多态，从基类指针调用分配到子类的函数，每个析构函数结束时都会自动调用（隐式）父类的析构函数。

为什么上例中基类析构函数定义为虚函数，析构父类指针指向的子类的对象时，会先调用子类的析构函数，再调用父类的析构函数呢？为什么父类不为虚析构函数的时候，子类就不会被析构呢？

解释构造：父类先构造，vptr指向父类，然后构造子类，vptr此时指向子类；最后构造孙类，vptr最后指向孙类；虚函数表指针指向正确；因此构造函数从父类到子类。

解释析构：析构的时候是，父类指针指向子类，拿到vptr指向的虚函数表，该虚函数表是子类的，因此先调用子类的虚析构函数，每个析构函数结束时都会自动调用（隐式）父类的析构函数。因此析构函数是先调用子类再调用父类的析构函数。

## 4、c++的多态性

- 通俗解释：多态：即多种状态，在面向对象语言中，接口的多种不同的实现方式即为多态。

- C++ 多态有两种：

- -静态多态（早绑定）、动态多态（晚绑定）。静态多态是通过函数重载实现的、模板实现；调用一个函数，在编译阶段就能够确定最终执行。

- -动态多态是通过虚函数实现的，调用一个函数；在运行阶段才能确定最终执行的是哪个函数体。

- 多态是以封装和继承为基础的。

  #### 静态多态

  静态多态的设计思想：对于相关的对象类型，直接实现它们各自的定义，不需要共有基类，甚至可以没有任何关系。只需要各个具体类的实现中要求相同的接口声明，这里的接口称之为隐式接口。客户端把操作这些对象的函数定义为模板，当需要操作什么类型的对象时，直接对模板指定该类型实参即可（或通过实参演绎获得）。

  相对于面向对象编程中，以显式接口和运行期多态（虚函数）实现动态多态，在模板编程及泛型编程中，是以隐式接口和编译器多态来实现静态多态。

  ```c++
  #include<iostream>
  #include<vector>
  
  class Line
  {
  public:
    void Draw()const { std::cout << "Line Draw()\n"; }
  };
  
  class Circle
  {
  public:
    void Draw(const char* name = NULL)const { std::cout << "Circle Draw()\n"; }
  };
  
  class Rectangle
  {
  public:
    void Draw(int i = 0)const { std::cout << "Rectangle Draw()\n"; }
  };
  
  template<typename Geometry>
  void DrawGeometry(const Geometry& geo)
  {
    geo.Draw();
  }
  
  template<typename Geometry>
  void DrawGeometry(std::vector<Geometry> vecGeo)
  {
    const size_t size = vecGeo.size();
    for (size_t i = 0; i < size; ++i)
      vecGeo[i].Draw();
  }
  
  void test_static_polymorphism()
  {
    Line line;
    Circle circle;
    Rectangle rect;
    DrawGeometry(circle);
  
    std::vector<Line> vecLines;
    Line line2;
    Line line3;
    vecLines.push_back(line);
    vecLines.push_back(line2);
    vecLines.push_back(line3);
    //vecLines.push_back(&circle); //编译错误，已不再能够处理异质对象
    //vecLines.push_back(&rect);    //编译错误，已不再能够处理异质对象
    DrawGeometry(vecLines);
  
    std::vector<Circle> vecCircles;
    vecCircles.push_back(circle);
    DrawGeometry(circle);
  }
  
  int main()
  {
    test_static_polymorphism();
    return 0;
  }
  //运行结果
  //Circle Draw()
  //Line Draw()
  //Line Draw()
  //Line Draw()
  //Circle Draw()
  ```

  

静态多态本质上就是模板的具现化。静态多态中的接口调用也叫做隐式接口，相对于显示接口由函数的签名式（也就是函数名称、参数类型、返回类型）构成，隐式接口通常由有效表达式组成。

类型Geometry（T）是在编译期模板进行具现化时才表现出调用不同的函数，此时对接口的调用就表现出了编译期时多态。

优点：

-由于静多态是在编译期完成的，因此效率较高，编译器也可以进行优化；

-有很强的适配性和松耦合性，比如可以通过偏特化、全特化来处理特殊类型；

-最重要一点是静态多态通过模板编程为C++带来了泛型设计的概念，比如强大的STL库。

缺点：

-由于是模板来实现静态多态，因此模板的不足也就是静多态的劣势，比如调试困难、编译耗时、代码膨胀、编译器支持的兼容性，不能够处理异质对象集合。

#### 动态多态

实现思路：

对于一组相关的数据类型，抽象出它们之间共同的功能集合，在基类中将共同的功能声明为多个公共虚函数接口，子类通过重写这些虚函数，实现各自对应的功能。在调用时，通过基类的指针来操作这些子类对象，所需执行的虚函数会自动绑定到对应的子类对象上。

优点：

-实现与接口分离，可复用；

-处理同一继承体系下异质对象集合的强大威力；

缺点：

-运行期绑定，导致一定程度的运行时开销；

-编译器无法对虚函数进行优化；

-笨重的类继承体系，对接口的修改影响整个类层次；

动态多态中接口是显式的，以函数签名为中心，多态通过虚函数在运行期实现，静态多台中接口是隐式的，以有效表达式为中心，多态通过模板具现在编译期完成。

动态多态和静态多态都能够使接口和实现相分离，一个是模板定义接口，类型参数定义实现，一个是基类虚函数定义接口，继承类负责实现；

```c++
#include<iostream>
#include<vector>
using namespace std;

class Geometry
{
public:
  virtual void Draw()const = 0;
};

class Line : public Geometry
{
public:
  virtual void Draw()const { std::cout << "Line Draw()\n"; }
};

class Circle : public Geometry
{
public:
  virtual void Draw()const { std::cout << "Circle Draw()\n"; }
};

class Rectangle : public Geometry
{
public:
  virtual void Draw()const { std::cout << "Rectangle Draw()\n"; }
};

void DrawGeometry(const Geometry* geo)
{
  geo->Draw();
}

void DrawGeometry(std::vector<Geometry*> vecGeo)
{
  const size_t size = vecGeo.size();
  for (size_t i = 0; i < size; ++i)
    vecGeo[i]->Draw();
}

void test_dynamic_polymorphism()
{
  Line line;
  Circle circle;
  Rectangle rect;
  DrawGeometry(&circle);

  std::vector<Geometry*> vec;
  vec.push_back(&line);
  vec.push_back(&circle);
  vec.push_back(&rect);
  DrawGeometry(vec);
}
int main()
{
  test_dynamic_polymorphism();
  return 0;
}
//测试结果
//Circle Draw()
//Line Draw()
//Circle Draw()
//Rectangle Draw()
```

基类通过virtual关键字声明和实现虚函数，此时基类会拥有一张虚函数表，虚函数表会记录其对应的函数指针。当子类继承基类时，子类也会获得一张虚函数表，不同之处在于，子类如果重写了某个虚函数，其在虚函数表上的函数指针会被替换为子类的虚函数指针。

编译器为每一个类维护一个虚函数表，每个对象的首地址保存着该虚函数表的指针，同一个类的不同对象实际上指向同一张虚函数表。通过这种方式，基类指针指向子类时，可以根据指针准确找到该调用哪个函数，从而实现动态多态。

## **5、构造函数与拷贝构造函数**

#### 构造函数

C++中通过构造函数确保对象的初始化，如果类存在构造函数，编译器会在创建对象的时候自动调用该函数。

如果不主动编写[拷贝构造函数](https://baike.baidu.com/item/拷贝构造函数?fromModule=lemma_inlink)和赋值函数，[编译器](https://baike.baidu.com/item/编译器?fromModule=lemma_inlink)将以“位拷贝”的方式自动生成[缺省](https://baike.baidu.com/item/缺省?fromModule=lemma_inlink)的函数。

```c++
//如果一个类没有构造函数、析构函数和赋值函数，则编译器会默认为此类生成4个缺省的函数。
class A
{
public:
  A(void); // 缺省的无参数构造函数
  A(const A& a); // 缺省的拷贝构造函数
  ~A(void);// 缺省的析构函数
  A& operator = (const A&); // 缺省的赋值函数
}
//默认构造函数与默认析构函数仅负责对象的创建和销毁，不做对象的初始化和资源的清理。
```

**1.**如果不主动编写[拷贝构造函数](https://baike.baidu.com/item/拷贝构造函数?fromModule=lemma_inlink)和赋值函数，[编译器](https://baike.baidu.com/item/编译器?fromModule=lemma_inlink)将以“位拷贝”的方式自动生成[缺省](https://baike.baidu.com/item/缺省?fromModule=lemma_inlink)的函数。倘若类中含有[指针变量](https://baike.baidu.com/item/指针变量?fromModule=lemma_inlink)，那么这两个缺省的函数就隐含了错误。以类String的两个对象a,b为例，假设a.m_data的内容为“hello”，b.m_data的内容为“world”。

现将a赋给b，缺省赋值函数的“位拷贝”意味着执行b.m_data = a.m_data。这将造成三个错误：一是b.m_data原有的内存没被释放，造成内存泄露；二是b.m_data和a.m_data指向同一块内存，a或b任何一方变动都会影响另一方；三是在对象被析构时，m_data被释放了两次。

**2.**[拷贝构造函数](https://baike.baidu.com/item/拷贝构造函数?fromModule=lemma_inlink)和赋值函数非常容易混淆，常导致错写、错用。拷贝构造函数是在对象被创建时调用的，而赋值函数只能被已经存在了的对象调用。

```c++
class String
{
public:
  //默认赋值函数
  String& operator =(const String& other)
  {
    // (1) 检查自赋值
    if (this == &other)
      return *this;
    // (2) 释放原有的内存资源
    delete[] m_data;
    // (3)分配新的内存资源，并复制内容
    int length = strlen(other.m_data);
    m_data = new char[length + 1];
    strcpy(m_data, other.m_data);
    // (4)返回本对象的引用
    return *this;
  }
private:
  char * m_data;
};
```

构造函数特征：

-与类同名
-没有返回值（与析构函数一致）
-构造函数可以被重载，由实参决定调用哪个构造函数

```c++
class A
{
private:
  int num_1, num_2;
public:
  A() : num_1(0), num_2(0) {}; //构造函数一：使用构造函数初始化列表
  A(int num1, int num2); //构造函数二：使用参数赋值
};
A::A(int num1, int num2)
{
  num_1 = num1;
  num_2 = num2;
}
```

```c++
//对类中const和引用类型对象的初始化，只能使用初始化列表，不能赋值操作
class A
{
public:
  A(int x);
private:
  int i;
  const int ci;
  int& ri;
};
//错误写法
//A::A(int x)
//{
//  i = x;  //ok
//  ci = x; //const对象不可以赋值
//  ri = i;  //赋值给ri但对象未绑定
//}
//正确写法如下：使用初始化列表对数据成员进行初始化
A::A(int x) : i(x), ci(i), ri(x) {}
```

#### 构造函数能是虚函数吗？

不能

虚函数对应一个vtable，可是这个vtable其实是存储在对象的内存空间的。
所以，如果构造函数是虚函数，就要通过vtable来调用，可是对象空间还没有实例化，也就是内存空间还没有，无法找到vtable，所以构造函数不能是虚函数。

#### 补充：析构函数能是虚函数吗？

把基类析构函数定义为虚函数，在调用析构函数时，会根据指向的对象类型（通常是子类对象）到它的虚函数表中找到对应的虚函数，此时找到的是派生类的析构函数，因此调用该析构函数；而调用派生类析构函数之后会再调用基类的析构函数，因此不会导致内存泄漏。

#### 拷贝构造函数

默认形式：

```c++
class Object
{
    public:
        Object(const Object &other);
};
```

拷贝构造函数的含义：

以一个对象为蓝本，来构造另一个对象
Object b;
Object a(b);//写成Object a=b;
称作：以b为蓝本，创建一个新的对象a；

拷贝构造函数特征：

只有单个形参，形参为对本类类型对象的引用（用const修饰）
(1)如果没有定义拷贝构造函数，编译器会合成拷贝构造函数

函数行为是逐个成员进行初始化，然后将新对象初始化为原对象的副本。数组成员是例外，合成拷贝构造函数会拷贝整个数组。

(2)自定义拷贝构造函数 A(const A&);

大多数类应定义拷贝构造函数和默认构造函数。

(3)禁止拷贝

如果一个类需要禁止赋值，并需显式地声明拷贝构造函数为private，这样就不允许用户代码对该类类型的对象进行复制。但类的友元和成员仍然可以进行复制，也需要禁止的话，就需要声明一个拷贝构造函数为private且对它不进行定义。

在C++中，下面三种对象需要调用拷贝构造函数

1)一个对象作为函数参数，以值传递的方式传入[函数体](https://baike.baidu.com/item/函数体?fromModule=lemma_inlink)；

```c++
Object a;
Object b(a);
```

2)一个对象作为函数返回值，以值传递的方式从函数返回；

```c++
Object a;
Object *p=new Object(a);
```

3)一个对象用于给另外一个对象进行初始化（常称为赋值初始化）；

```c++
Object a;
Object b=a;//或写作 Object b(a);
```

区分构造拷贝函数和普通赋值！

```c++
Object a;
Object b;
b=a;//此为“赋值”，不会调用拷贝构造函数
```

注意事项：

一旦你决定了要添加拷贝构造函数，请仔细检查：
1. 所有的成员变量，要依次拷贝!所有成员变量，不要遗漏
2. 调用父类的拷贝构造函数!（要么不负责，要么负责全部事情）

## 6、智能指针？

### 1、unique_ptr

这是个独占式的指针对象，在任何时间、资源只能被一个指针占有，当unique_ptr离开作用域，指针所包含的内容会被释放。

unique_ptr 是通过指针占有并管理另一对象，并在 unique_ptr 离开作用域时释放该对象的智能指针。在下列两者之一发生时用关联的删除器释放对象：

- 销毁了管理的 unique_ptr 对象
- 通过 operator= 或 reset() 赋值另一指针给管理的 unique_ptr 对象。

使用裸指针时，要记得释放内存。

```c++
{
    int* p = new int(100);
    // ...
    delete p;  // 要记得释放内存
}
```

使用 std::unique_ptr 自动管理内存。

```c
{
    std::unique_ptr<int> uptr = std::make_unique<int>(200);
    //...
    // 离开 uptr 的作用域的时候自动释放内存
}
```

std::unique_ptr 是 move-only 的。

```c
{
    std::unique_ptr<int> uptr = std::make_unique<int>(200);
    std::unique_ptr<int> uptr1 = uptr;  // 编译错误，std::unique_ptr<T> 是 move-only 的

    std::unique_ptr<int> uptr2 = std::move(uptr);
    assert(uptr == nullptr);
}
```

std::unique_ptr 可以指向一个数组。

```c
{
    std::unique_ptr<int[]> uptr = std::make_unique<int[]>(10);
    for (int i = 0; i < 10; i++) {
        uptr[i] = i * i;
    }   
    for (int i = 0; i < 10; i++) {
        std::cout << uptr[i] << std::endl;
    }   
}
```

自定义 deleter。

```c++
{
    struct FileCloser {
        void operator()(FILE* fp) const {
            if (fp != nullptr) {
                fclose(fp);
            }
        }   
    };  
    std::unique_ptr<FILE, FileCloser> uptr(fopen("test_file.txt", "w"));
}
```

使用 Lambda 的 deleter。

```c++
{
    std::unique_ptr<FILE, std::function<void(FILE*)>> uptr(
        fopen("test_file.txt", "w"), [](FILE* fp) {
            fclose(fp);
        });
}
```

### 2、shared_ptr

std::shared_ptr 其实就是对资源做引用计数——当引用计数为 0 的时候，自动释放资源。

```c++
{
    std::shared_ptr<int> sptr = std::make_shared<int>(200);
    assert(sptr.use_count() == 1);  // 此时引用计数为 1
    {   
        std::shared_ptr<int> sptr1 = sptr;
        assert(sptr.get() == sptr1.get());
        assert(sptr.use_count() == 2);   // sptr 和 sptr1 共享资源，引用计数为 2
    }   
    assert(sptr.use_count() == 1);   // sptr1 已经释放
}
// use_count 为 0 时自动释放内存
```

和 unique_ptr 一样，shared_ptr 也可以指向数组和自定义 deleter。

```c++
{
    // C++20 才支持 std::make_shared<int[]>
    // std::shared_ptr<int[]> sptr = std::make_shared<int[]>(100);
    std::shared_ptr<int[]> sptr(new int[10]);
    for (int i = 0; i < 10; i++) {
        sptr[i] = i * i;
    }   
    for (int i = 0; i < 10; i++) {
        std::cout << sptr[i] << std::endl;
    }   
}

{
    std::shared_ptr<FILE> sptr(
        fopen("test_file.txt", "w"), [](FILE* fp) {
            std::cout << "close " << fp << std::endl;
            fclose(fp);
        });
}
```

###### std::shared_ptr 的实现原理

一个 shared_ptr 对象的内存开销要比裸指针和无自定义 deleter 的 unique_ptr 对象略大。

```c++
  std::cout << sizeof(int*) << std::endl;  // 输出 8
  std::cout << sizeof(std::unique_ptr<int>) << std::endl;  // 输出 8
  std::cout << sizeof(std::unique_ptr<FILE, std::function<void(FILE*)>>)
            << std::endl;  // 输出 40

  std::cout << sizeof(std::shared_ptr<int>) << std::endl;  // 输出 16
  std::shared_ptr<FILE> sptr(fopen("test_file.txt", "w"), [](FILE* fp) {
    std::cout << "close " << fp << std::endl;
    fclose(fp);
  }); 
  std::cout << sizeof(sptr) << std::endl;  // 输出 16
```

无自定义 deleter 的 unique_ptr 只需要将裸指针用 RAII 的手法封装好就行，无需保存其它信息，所以它的开销和裸指针是一样的。如果有自定义 deleter，还需要保存 deleter 的信息。

shared_ptr 需要维护的信息有两部分：

1. 指向共享资源的指针。
2. 引用计数等共享资源的控制信息——实现上是维护一个指向控制信息的指针。

所以，shared_ptr 对象需要保存两个指针。shared_ptr 的 deleter 是保存在控制信息中，所以，是否有自定义 deleter 不影响 shared_ptr 对象的大小。

当我们创建一个 shared_ptr 时，其实现一般如下：

```c++
std::shared_ptr<T> sptr1(new T);
```

![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/24/172e5c0fee43dfd6~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.awebp)



复制一个 shared_ptr ：

```c++
std::shared_ptr<T> sptr2 = sptr1;
```

![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/24/172e5c0ff4c0d06a~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.awebp)

为什么控制信息和每个 shared_ptr 对象都需要保存指向共享资源的指针？可不可以去掉 shared_ptr 对象中指向共享资源的指针，以节省内存开销？

答案是：不能。 因为 shared_ptr 对象中的指针指向的对象不一定和控制块中的指针指向的对象一样。

来看一个例子。

```c++
struct Fruit {
    int juice;
};

struct Vegetable {
    int fiber;
};

struct Tomato : public Fruit, Vegetable {
    int sauce;
};

 // 由于继承的存在，shared_ptr 可能指向基类对象
std::shared_ptr<Tomato> tomato = std::make_shared<Tomato>();
std::shared_ptr<Fruit> fruit = tomato;
std::shared_ptr<Vegetable> vegetable = tomato;
```



![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/24/172e5c0fef0e205b~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.awebp)



另外，std::shared_ptr 支持 aliasing constructor。

```c++
template< class Y >
shared_ptr( const shared_ptr<Y>& r, element_type* ptr ) noexcept;
```

Aliasing constructor，简单说就是构造出来的 shared_ptr 对象和参数 r 指向同一个控制块（会影响 r 指向的资源的生命周期），但是指向共享资源的指针是参数 ptr。看下面这个例子。

```c++
using Vec = std::vector<int>;
std::shared_ptr<int> GetSPtr() {
    auto elts = {0, 1, 2, 3, 4};
    std::shared_ptr<Vec> pvec = std::make_shared<Vec>(elts);
    return std::shared_ptr<int>(pvec, &(*pvec)[2]);
}

std::shared_ptr<int> sptr = GetSPtr();
for (int i = -2; i < 3; ++i) {
    printf("%d\n", sptr.get()[i]);
}
```

![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/24/172e5c0ff6f5a6d6~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.awebp)

看上面的例子，使用 std::shared_ptr 时，会涉及两次内存分配：一次分配共享资源对象、一次分配控制块。C++ 标准库提供了 std::make_shared 函数来创建一个 shared_ptr 对象，只需要一次内存分配。

![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/24/172e5c0ff8efb560~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.awebp)

这种情况下，不用通过控制块中的指针，我们也能知道共享资源的位置——这个指针也可以省略掉。

![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/24/172e5c0ffa000552~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.awebp)



### 3、weak_ptr

std::weak_ptr 要与 std::shared_ptr 一起使用。 一个 std::weak_ptr 对象看做是 std::shared_ptr 对象管理的资源的观察者，它不影响共享资源的生命周期：

1. 如果需要使用 weak_ptr 正在观察的资源，可以将 weak_ptr 提升为 shared_ptr。
2. 当 shared_ptr 管理的资源被释放时，weak_ptr 会自动变成 nullptr。\

```c
void Observe(std::weak_ptr<int> wptr) {
    if (auto sptr = wptr.lock()) {
        std::cout << "value: " << *sptr << std::endl;
    } else {
        std::cout << "wptr lock fail" << std::endl;
    }
}

std::weak_ptr<int> wptr;
{
    auto sptr = std::make_shared<int>(111);
    wptr = sptr;
    Observe(wptr);  // sptr 指向的资源没被释放，wptr 可以成功提升为 shared_ptr
}
Observe(wptr);  // sptr 指向的资源已被释放，wptr 无法提升为 shared_ptr
```

![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/24/172e5c100e1d768b~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.awebp)

当 shared_ptr 析构并释放共享资源的时候，只要 weak_ptr 对象还存在，控制块就会保留，weak_ptr 可以通过控制块观察到对象是否存活。

![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/24/172e5c1020142816~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.awebp)

### 4、enable_shared_from_this

一个类的成员函数如何获得指向自身（this）的 shared_ptr？ 看看下面这个例子有没有问题？

```c++
class Foo {
 public:
  std::shared_ptr<Foo> GetSPtr() {
    return std::shared_ptr<Foo>(this);
  }
};

auto sptr1 = std::make_shared<Foo>();
assert(sptr1.use_count() == 1);
auto sptr2 = sptr1->GetSPtr();
assert(sptr1.use_count() == 1);
assert(sptr2.use_count() == 1);
```

上面的代码其实会生成两个独立的 shared_ptr，他们的控制块是独立的，最终导致一个 Foo 对象会被 delete 两次。

![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/24/172e5c102982f9ba~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.awebp)

成员函数获取 this 的 shared_ptr 的正确的做法是继承 std::enable_shared_from_this。

```c++
class Foo : public std::enable_shared_from_this<Foo> {
public:
  std::shared_ptr<Foo> GetSPtr() {
    return shared_from_this();
  }
};
int main()
{
  auto sptr1 = std::make_shared<Foo>();
  assert(sptr1.use_count() == 1);
  auto sptr2 = sptr1->GetSPtr();
  assert(sptr1.use_count() == 2);
  assert(sptr2.use_count() == 2);
}
```

一般情况下，继承了 std::enable_shared_from_this 的子类，成员变量中增加了一个指向 this 的 weak_ptr。这个 weak_ptr 在第一次创建 shared_ptr 的时候会被初始化，指向 this。

![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/24/172e5c10342401b8~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.awebp)



似乎继承了 std::enable_shared_from_this 的类都被强制必须通过 shared_ptr 进行管理。

```c++
auto b = new Foo;
auto sptr = b->shared_from_this();
```

在我的环境下（gcc 7.5.0）上面的代码执行的时候会直接 coredump，而不是返回指向 nullptr 的 shared_ptr：

```c
terminate called after throwing an instance of 'std::bad_weak_ptr'
 what():  bad_weak_ptr
```

转载from：https://juejin.cn/post/6844904198962675719#comment

## 7、内存泄露

###### 什么是内存泄露？

在计算机科学中，内存泄漏指由于疏忽或错误造成程序未能释放已经不再使用的内存。内存泄漏并非指内存在物理上的消失，而是应用程序分配某段内存后，由于设计错误，导致在释放该段内存之前就失去了对该段内存的控制，从而造成了内存的浪费。

在C++中出现内存泄露的主要原因就是程序猿在申请了内存后(`malloc(), new`)，没有及时释放没用的内存空间，甚至消灭了指针导致该区域内存空间根本无法释放。

###### 内存泄漏可能会导致严重的后果：

- 程序运行后，随着时间占用了更多的内存，最后无内存可用而崩溃；
- 程序消耗了大量的内存，导致其他程序无法正常使用；

###### 如何知道自己的程序存在内存泄露？

根据内存泄露的原因及其恶劣的后果，我们可以通过其主要表现来发现程序是否存在内存泄漏：程序长时间运行后内存占用率一直不断的缓慢的上升，而实际上在你的逻辑中并没有这么多的内存需求。

###### 如何定位到泄露点呢？

1. 根据原理，我们可以先review自己的代码，利用"查找"功能，查询`new`与`delete`，看看内存的申请与释放是不是成对释放的，可以迅速发现一些逻辑较为简单的内存泄露情况。
2. 如果依旧发生内存泄露，可以通过记录申请与释放的对象数目是否一致来判断。在类中追加一个静态变量 `static int count;`在构造函数中执行`count++;`在析构函数中执行`count--;`，通过在程序结束前将所有类析构，之后输出静态变量，看count的值是否为0，如果为0,则问题并非出现在该处，如果不为0，则是该类型对象没有完全释放。
3. 检查类中申请的空间是否完全释放，尤其是存在继承父类的情况，看看子类中是否调用了父类的析构函数，有可能会因为子类析构时没有释放父类中申请的内存空间。
4. 对于函数中申请的临时空间，认真检查，是否存在提前跳出函数的地方没有释放内存。

## 8、静态链接与动态链接

### 静态链接：

静态链接会将库中的代码直接复制一份到可执行文件中。优势是可移植性好，因为需要的代码段已经复制到了可执行文件里去。缺点是更新静态库的时候需要重新编译，而且会存在将同一份代码拷贝多次在内存中多次载入的问题。

### 静态链接库(*.lib)

如果在程序中使用静态链接库，那么链接器在链接的过程中会将**.obj文件**和**.lib文件**组织成**可执行exe文件**，也就是将.lib中的代码链接到可执行文件中，因此**生成的exe文件比较大**。 程序运行时，将全部数据加载到内存。如果程序体积较大，功能较为复杂，那么加载到内存中的时间就会比较长，最直接的一个例子就是双击打开一个软件，要很久才能看到界面。这是静态链接库的一个弊端。 但程序在发行时不需要提供库文件。

## 静态库优点总结

1. 代码装载速度快，执行速度略比动态链接库快
2. 程序在发行时不需要提供库文件，不存在兼容性问题

#### 静态库缺点总结

1. 使用静态链接生成的可执行文件体积较大，包含相同的公共代码，造成浪费
2. 程序可扩展性差

### 动态链接库(*.dll)

动态链接库又分为两种加载方式：**隐式加载**和**显式加载**

#### 隐式加载

**隐式加载**，也称**载入时加载**，是程序载入内存时加载所需的dll文件，且该dll随主进程始终占用内存。在编码时需要使用`#pragma comment(lib,"myDll.lib")`获得所需函数的入口。**注意该.lib与静态链接库的.lib文件不同，静态链接库的.lib中包含了所需函数的代码，动态链接库的.lib仅指示函数在dll文件中的入口**。

隐式加载也会有静态链接库的问题，**如果程序稍大，加载时间就会过长**。

#### 显式加载

**显式加载**，也称**运行时加载**，是在程序运行过程中加载，不需要该dll时则将其释放。在需要时使用`LoadLibrary`加载，不需要时使用`FreeLibrary`释放。如果在LoadLibrary时该dll已经在内存，则只需将其引用计数加1，如果其引用计数减为0则移出内存。

使用动态链接库的程序在**发行时**需要提供**dll文件**。在编译时，如果使用隐式链接则需要提供**.lib文件**，生成可执行文件后则不再需要该.lib。如果使用显式链接，在编译时不需提供.lib文件。

显式加载将较大的程序分开加载的，程序运行时只需要将主程序载入内存，**软件打开速度快**，用户体验好。

#### 动态库优点总结

1. 更加节省内存并减少页面交换
2. dll文件与exe文件独立，只要输出接口不变（即名称、参数、返回值类型和调用约定不变），更换dll文件不会对exe文件造成任何影响，因而极大地提高了可维护性和可扩展性
3. 不同编程语言编写的程序只要按照函数调用约定就可以调用同一个dll函数
4. 适用于大规模的软件开发，使开发过程独立、耦合度小，便于不同开发者和开发组织之间进行开发和测试

#### 动态库缺点总结

1. 使用动态链接库的应用程序不是自完备的，它依赖的dll模块也要存在，如果使用**载入时加载（隐式加载）**，程序启动时发现dll不存在，系统将终止程序并给出错误信息。而使用**运行时加载（显式加载）**，系统不会终止，但由于dll中的导出函数不可用，程序会加载失败
2. 性能损失
3. 存在兼容性问题

#### 动态链接库使用示例

dll源码：

```c++
#include <Windows.h>  
#include <stdio.h>  
 
extern "C" {
    _declspec(dllexport) int testAdd(int x, int y)
    {
        return x + y;
    }
}
```

使用隐式链接：

```c++
#include"stdafx.h"
using namespace std;
 
#pragma comment(lib,"myDll.lib")//加载动态库
 
extern "C" {
    _declspec(dllimport) int testAdd(int x, int y);
}
int   main(int   argc, TCHAR  *argv[]) {
    _tprintf(_T("sum is %d\n"), testAdd(2,5));
    system("pause");
    return 0;
}
```

使用显示链接：

```c++
#include"stdafx.h"
using namespace std;
 
int   main(int   argc, TCHAR  *argv[]) 
{
    HINSTANCE h = LoadLibrary(_T("myDLL"));//加载动态库
    GetErrorMessage(GetLastError(), 0);
    int(*pAdd)(int, int) = NULL;
    pAdd = (int(__cdecl *)(int, int))(GetProcAddress(h, "testAdd"));
    GetErrorMessage(GetLastError(), 0);
    if (pAdd == NULL) {
        FreeLibrary(h);
        return 0;
    }
    int sum = pAdd(239, 23);
    _tprintf(_T("sum is %d\n"), sum);
    FreeLibrary(h);
    system("pause");
    return 0;
}
```



## 9、内存分布

#### 在C++中，内存分成5个区

1. **栈区**（stack） 由**编译器**自动分配释放 ，存放函数的参数值，局部变量的值等。其操作方式类似于数据结构中的栈。
2. **堆区**（heap） 就是那些由 new 分配的内存块，一般一个 new 就要对应一个 delete。一般由**程序员**分配释放， 若程序员不释放，程序结束时可能由OS回收 。注意它与数据结构中的堆是两回事，分配方式倒是类似于链。堆可以动态地扩展和收缩。
3. **自由存储区** 就是那些由 malloc 等分配的内存块，他和堆是十分相似的，不过它是用 free 来结束自己的生命的。
4. **全局区（静态区）**（static）全局变量和静态变量的存储是放在一块的，初始化的全局变量和静态变量在一块区域data段， 未初始化的全局变量和未初始化的静态变量在相邻的另一块区域bss段。程序结束后由系统释放
5. **常量存储区** 存放的是常量，不允许修改。常量字符串就是放在这里的。常量字符串不能修改, 否则程序会在运行期崩溃（当然，你要通过非正当手段也可以修改，而且[方法](http://blog.csdn.net/cherylnatsu/article/details/6277274)很多）。程序结束后由系统释放。

​	6.**程序代码区** 存放函数体的二进制代码。

**申请后系统的响应** 

- 栈：只要栈的剩余空间大于所申请空间，系统将为程序提供内存，否则将报异常提示栈溢出。
- 堆：操作系统有一个记录空闲内存地址的链表，当系统收到程序的申请时， 会遍历该链表，寻找第一个空间大于所申请空间的堆结点，然后将该结点从空闲结点链表中删除，并将该结点的空间分配给程序。另外，对于大多数系统，会在这块内存空间中的首地址处记录本次分配的大小，这样，代码中的delete语句才能正确的释放本内存空间。另外，由于找到的堆结点的大小不一定正好等于申请的大小，系统会自动的将多余的那部分重新放入空闲链表中。

**申请大小的限制**

- 栈：在Windows下,栈是向低地址扩展，从上到下生长的扩展的数据结构，是一块连续的内存的区域。这句话的意思是栈顶的地址和栈的最大容量是系统预先规定好的，在WINDOWS下，栈的大小是2M（也有的说是1M，总之是一个编译时就确定的常数），如果申请的空间超过栈的剩余空间时，将提示overflow。因此，能从栈获得的空间较小。
- 堆：堆是向高地址扩展的数据结构，是不连续的内存区域。这是由于系统是用链表来存储的空闲内存地址的，自然是不连续的，而链表的遍历方向是由低地址向高地址。堆的大小受限于计算机系统中有效的虚拟内存。由此可见，堆获得的空间比较灵活，也比较大。

**生长方向**：对于堆来讲，生长方向是向上的，是向着内存地址增加的方向；对于栈来讲，它的生长方向是向下的，是向着内存地址减小的方向增长。

**申请效率的比较** 

- 栈：由系统自动分配，速度较快。但程序员是无法控制的。
- 堆：由new分配的内存，一般速度比较慢，而且容易产生内存碎片,不过用起来最方便.

**分配方式**

- 堆都是动态分配的，没有静态分配的堆。
- 栈有2种分配方式：静态分配和动态分配。静态分配是编译器完成的，比如局部变量的分配。动态分配由malloc函数进行分配，但是栈的动态分配和堆是不同的，他的动态分配是由编译器进行释放，无需我们手工实现。

**案例**

```
void f()
{ 
    int* p=new int[5];
}
```

   这条短短的一句话就包含了堆与栈，看到new，我们首先就应该想到，我们分配了一块堆内存，那么指针p呢？他分配的是一块栈内存，所以这句话的意思就是：**在栈内存中存放了一个指向一块堆内存的指针p**。在程序会先确定在堆中分配内存的大小，然后调用operator new分配内存，然后返回这块内存的首地址，放入栈中，

### 10、指针越界如何检测

c++内存 指针越界检测机制_CrtMemBlockHeader
c++的new和delete操作，可以让我们在程序运行中动态的开辟内存空间，但是我们知道，一旦处理不好就会造成内存泄漏。一直有一个疑问，c++为防止（或者说检测）指针越界做了哪些工作？

#### struct _CrtMemBlockHeader

_CrtMemBlockHeader ：这个结构体，存放了动态申请得到的内存块的各种信息，并且返回到你的指针上面。具体内容如下：

```c++
typedef struct _CrtMemBlockHeader 
{      
　　// 指向下一块数据块的指针       
　　struct _CrtMemBlockHeader *pBlockHeaderNext;        
    // 指向前一块数据块的指针   
    struct _CrtMemBlockHeader *pBlockHeaderPrev;    
// File name：请求内存分配操作的那行代码所在的文件的路径和名称，但实际上是空指针  
char *szFileName;  
// Line number：行号，请求内存分配操作的那行代码的行号   
int nLine;            

// 请求分配的大小    size_t nDataSize;     

// Type of block 类型  
int nBlockUse;      
// 请求号 
long lRequest;        
// 这个gap，正是cpp中对于指针的界限 
unsigned char gap[nNoMansLandSize];
} _CrtMemBlockHeader;
```

我们来说一说gap[]的作用，你所申请的空间中的内容我们假定为<tdata>。在<tdata>的前后各有4个B（字节）的gap[]，他在内存中的值为0xFD。这样系统只需要检测你的<tdata>前后的数据是否为0xFD就可知道你有没有越界。

在实际的测试中，我们申请创建一个int类型的p指针，把内存的显示改为4列16进制（以下图片来自vs2019的debug模式）

我们可以看到，0x00FC05A0是我们的p指针所指向的内容，因为我们并没有创建内容所以在内存中使用0xcd进行填充，前后各有4B的0xfd来进行限定。

如何检测内存泄漏
上面我们知道了gap[]以及他的作用，那么我们顺利成章地就会想到_CrtDumpMemoryLeaks()这个函数，他就是通过检查内存分配链表（pBlockHeaderNext和pBlockHeaderPrev为双链表的双链）, 来查找是否有泄漏。
原文链接：https://blog.csdn.net/TalesOV/article/details/107320124/



