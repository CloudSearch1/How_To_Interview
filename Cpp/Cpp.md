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

## 虚函数

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

### 4、c++的多态性

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

### **5、构造函数与拷贝构造函数**

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

