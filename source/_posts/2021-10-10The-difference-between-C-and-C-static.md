title: C和C++中static的区别
categories:
  - C/C++
tags:
  - C/C++
  - static
date: 2021-10-10 21:25:38
---
在C/C++中static有三种用法：
1.用來修饰局部变量，形成静态局部函数
2.用來修饰全局变量/函数
3.用來修饰静态数据成员和成员函数
前两种是C/C++共有的，第三种是C++独有的。
***

### 修饰局部变量
在介绍静态局部变量之前，先认识一下关键字auto。
#### auto
auto关键字在C/C++中只有一个作用，就是修饰局部变量(不能用于修饰全局变量)。
被auto修饰的局部变量又称之为自动局部变量。
平时我们定义的局部变量，若没有用static或者register修饰，默认就是auto类型的，例如：
``` c++
#include <stdio.h>
int main()
{
	int i = 0; //等价于 auto int i = 0;
	printf("i = %d \n",i);
    
	return 0;
}
```
自动局部变量的特点：
1.自动局部变量分配在栈上（由于栈内存是脏的，说明在栈上分配的变量，若没有初始化，其值就是随机的）。
2.自动局部变量的生命周期和作用域仅限于定义它的函数，函数运行结束，其生命周期也结束。

#### static
static修饰的局部变量称之为静态局部变量，静态局部变量和自动局部变量（auto）本质区别是变量开辟的空间在内存的位置不同。
静态局部变量的特点：
1.静态局部变量分配在data段或者bss段上。（在程序执行之前BSS段会自动清0，所以未初始化的静态局部变量的值为0）
2.静态局部变量的作用域与自动局部变量（auto）相同，都仅限于定义它的函数。
3.静态局部变量的生命周期与全局变量相同，在程序整个运行期间都不释放。
4.静态局部变量在所处模块在初次运行时进行初始化工作, 且只操作一次，之后操作它都保存着上一次的值，例如：
``` c++
#include <stdio.h>

void test()
{
    static val = 0;
    printf("val = %d",val);

    val++;
}

int main()
{
    test(); //第一次调用输出“val = 0”
    test(); //第二次调用输出“val = 1”
    return 0;
}
```
### 修饰全局变量/函数
当我们使用static来修饰全局变量或函数时，全局变量和函数的作用范围就被限定为本文件了，其他文件在链接时无法使用这些变量和函数，这就是内链接。
使用static修饰之后，可以有效的避免函数和全局变量的命名冲突问题。例如：
a.c文件
``` c++
#include <stdio.h>

int val1 = 1；
static val2 = 2；

void test1（）
{
    ……
}
static void test2（）
{
    ……
}
```
b.c文件
``` c++
#include <stdio.h>

extern val1;  //正确，可以使用a.c文件中的val1
extern val2;  //错误,不可以使用a.c文件中的val2，因为被static修饰了
extern void test1（）； //正确
extern void test2（）； //错误
int main()
{
    printf("val1 = %d \n",val1);

    return 0;
}
```
### 修饰静态数据成员和成员函数
静态数据成员和静态成员函数是C++特有的，被static修饰的类成员或成员函数，称为静态数据成员或静态成员函数，C++引入这一概念的目的是为了在类的范围内实现内存共享。
#### 静态数据成员
静态数据成员实际上是类域中的全局变量。主要特点如下：
* 静态数据成员的定义。举例如下：
xxx.h文件
``` c++
class   base{     
     private:     
     static   const   int   _i;//声明，标准c++支持有序类型在类体中初始化,但vc6不支持。     
};
```
 xxx.c文件
``` c++
const   int   base::_i=10;//定义(初始化)时不受private和protected访问限制.
```
* 静态数据成员被 类 的所有对象所共享，包括该类派生类的对象。即派生类对象与基类对象共享基类的静态数据成员。举例如下：
``` c++
class   base{     
            public   :     
            static   int   _num;//声明     
};     
int   base::_num=0;//静态数据成员的真正定义     
class   derived:public   base{     
};     
int main()     
{     
   base   a;     
   derived   b;     
   a._num++;     
   cout<<"base   class   static   data   number   _num   is"<<a._num<<endl;     
   b._num++;     
   cout<<"derived   class   static   data   number   _num   is"<<b._num<<endl;     
}     
//   结果为1,2;可见派生类与基类共用一个静态数据成员。
```
* 静态数据成员可以成为成员函数的可选参数，而普通数据成员则不可以。举例如下：
``` c++
class   base{     
public   :     
       static   int   _staticVar;     
       int   _var;     
       void   foo1(int   i=_staticVar);//正确,_staticVar为静态数据成员     
       void   foo2(int   i=_var);//错误,_var为普通数据成员     
   };
```
* 静态数据成员的类型可以是所属类的类型，而普通数据成员则不可以。普通数据成员的只能声明为所属类类型的指针或引用。举例如下：
``` c++
class   base{     
public   :     
        static   base   _object1;//正确，静态数据成员     
        base   _object2;//错误     
        base   *pObject;//正确，指针     
        base   &mObject;//正确，引用     
};
```
* 这个特性，我不知道是属于标准c++中的特性，还是vc6自己的特性。 静态数据成员的值在const成员函数中可以被合法的改变。举例如下：
``` c++
class   base{

public:     
        base(){_i=0;_val=0;}     
          mutable   int   _i;     
          static   int   _staticVal;       
          int   _val;     
          void   test()   const{//const   成员函数     

                _i++;//正确，mutable数据成员     
                _staticVal++;//正确，static数据成员     
                _val++;//错误     

        }     
};     
int   base::_staticVal=0;
```
#### 静态成员函数
静态成员函数没有什么太多好讲的。
1.静态成员函数的地址可用普通函数指针储存，而普通成员函数地址需要用类成员函数指针来储存。举例如下：
``` c++
class   base{     
              static   int   func1();     
              int   func2();     
};     

int   (*pf1)()=&base::func1;//普通的函数指针     
int   (base::*pf2)()=&base::func2;//成员函数指针
```
2.静态成员函数不可以调用类的非静态成员。因为静态成员函数不含this指针。
3.静态成员函数不可以同时声明为 virtual、const、volatile函数。举例如下：
``` c++
class   base{     
             virtual   static   void   func1();//错误     
             static   void   func2()   const;//错误     
             static   void   func3()   volatile;//错误     
};
```
最后要说的一点是，静态成员是可以独立访问的，也就是说，无须创建任何对象实例就可以访问。

[关于静态数据成员和成员函数的说明，引用于此处](https://blog.csdn.net/xiajun07061225/article/details/6955226) 
