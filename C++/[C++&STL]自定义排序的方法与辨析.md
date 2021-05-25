# 关于map自定义排序的问题

在使用C++刷Leetcode时，常常会使用有序容器，如map、set等。同时也会用到例如sort这类排序函数。

通常来说，我们知道写lambda表达式或者函数来自定义sort。也会写struct并重载调用运算符来自定义map，set。但是它们究竟有什么区别，又有什么联系？

本文会简析常见的自定义排序方法，并说明它们的区别和联系。

## 1. std::sort()如何自定义排序

为了弄清楚有多少种方式，首先去[cplusplus.com中关于sort的文档](https://www.cplusplus.com/reference/algorithm/sort/)去看一看具体的描述：

![image-20210314141252452](https://gitee.com/molinchn/BlogImage/raw/master/img/image-20210314141252452.png)

sort是一个模板函数，传进去的首先是对相应的容器迭代器，然后是有关自定义排序的`Compare comp`，我们来看对它的描述：

> comp
>
> Binary function that accepts two elements in the range as arguments, and returns a value convertible to `bool`. The value returned indicates whether the element passed as first argument is considered to go before the second in the specific **strict weak ordering** it defines.
> The function shall not modify any of its arguments.
> **This can either be a function pointer or a function object.**

这里就也很明确的说：comp是一个**函数指针**或**函数对象**。

官方给出的代码：

```C++
// sort algorithm example
#include <iostream>     // std::cout
#include <algorithm>    // std::sort
#include <vector>       // std::vector

bool myfunction (int i,int j) { return (i<j); }

struct myclass {
  bool operator() (int i,int j) { return (i<j);}
} myobject;

int main () {
  int myints[] = {32,71,12,45,26,80,53,33};
  std::vector<int> myvector (myints, myints+8);               // 32 71 12 45 26 80 53 33

  // using default comparison (operator <):
  std::sort (myvector.begin(), myvector.begin()+4);           //(12 32 45 71)26 80 53 33

  // using function as comp
  std::sort (myvector.begin()+4, myvector.end(), myfunction); // 12 32 45 71(26 33 53 80)

  // using object as comp
  std::sort (myvector.begin(), myvector.end(), myobject);     //(12 26 32 33 45 53 71 80)

  // print out content:
  std::cout << "myvector contains:";
  for (std::vector<int>::iterator it=myvector.begin(); it!=myvector.end(); ++it)
    std::cout << ' ' << *it;
  std::cout << '\n';

  return 0;
}
```

这里面比较重要的是：

1. 6行定义的函数，在20行被直接传进去参数使用了。
2. 8-10行定义的结构体`myclass`受限实例化了一个对象`myobject`。然后在23行时，`myobject`被拿来作为参数传进了sort里。

也就是说，自定义sort有两种方法，可以自定义一个bool型函数，里面的大小规则要是严格弱序的，然后把这个函数名作为参数传给sort。

除此之外，我们还可以写一个struct或者class，注意要在这个结构体or类中重载调用运算符。然后利用这个结构体or类去实例化一个对象，也就是所谓的函数对象。把这个函数对象传递给sort也可以完成自定义排序。

那我们常见的lambda表达式自定义排序的方式属于哪一类呢？实际上lambda表达式本身就是一个函数对象，它免去了建立结构体或类，然后实例化的过程。通常为了自定义排序所建立的结构体或类也不被调用多次，而lambda表达式大大简化了利用函数对象的步骤，所以常常在一些自定义排序的过程中见到。

## 2. std::map<>如何自定义排序：

为了说清楚map能如何自定义排序，我们还是要去看一下详细的文档。在[cplusplus.com中map容器模板](http://www.cplusplus.com/reference/map/map/)的描述如下：

![image-20210314233655526](https://gitee.com/molinchn/BlogImage/raw/master/img/image-20210314233655526.png)

首先需要注意到，前面的std:sort()是一个模板函数，而这个std::map是一个模板类。这是本质区别，同时也导致了自定义排序的一些不同。在模板参数中，有一个重要的class Compare是需要关注的重点，文档关于它的描述如下：

>Compare
>
>A binary predicate that takes two element keys as arguments and returns a `bool`. **The expression `comp(a,b)`, where *comp* is an object of this type and *a* and *b* are key values, shall return `true` if *a* is considered to go before *b* in the *strict weak ordering* the function defines.**
>The `map` object uses this expression to determine both the order the elements follow in the container and whether two element keys are equivalent (by comparing them reflexively: they are equivalent if `!comp(a,b) && !comp(b,a)`). No two elements in a `map` container can have equivalent keys.
>This can be a function pointer or a function object (see [constructor](http://www.cplusplus.com/map::map) for an example). This defaults to `less<T>`, which returns the same as applying the *less-than operator* (`a<b`).
>Aliased as member type `map::key_compare`.

看粗体部分可以发现，我们需要一个comp(a, b)来控制数据的顺序，而这个comp就是Compare类的对象。看到这里，我们就发现了前后的联系：**又是函数对象**！

接着来看一下[map的构造函数](http://www.cplusplus.com/reference/map/map/map/)怎么声明的：

![image-20210314135225224](https://gitee.com/molinchn/BlogImage/raw/master/img/image-20210314135225224.png)

在map容器的构造函数中又一次出现了`comp`，对于这个形参的描述，cplusplus.com文档继续描述如下：

> comp：
>
> Binary predicate that, taking two *element keys* as argument, returns `true` if the first argument goes before the second argument in the ***strict weak ordering*** it defines, and `false` otherwise. **This shall be a function pointer or a function object.** Member type `key_compare` is the internal comparison object type used by the container, defined in [map](https://www.cplusplus.com/map) as an alias of its third template parameter (`Compare`). **If `key_compare` uses the default [less](https://www.cplusplus.com/less) (which has no state), this parameter is not relevant.**

看到这里，我们就明确了comp可以是两种类型：**（1）函数指针，（2）函数对象**。

这就很巧了，前面std::sort()也是这两点，那它们实际使用的时候会有什么区别呢，还是说完全一样？下面来看cplusplus给出的两种方法的实例：

```c++
// constructing maps
#include <iostream>
#include <map>

bool fncomp (char lhs, char rhs) {return lhs<rhs;}

struct classcomp {
  bool operator() (const char& lhs, const char& rhs) const
  {return lhs<rhs;}
};

int main ()
{
  std::map<char,int> first;

  first['a']=10;
  first['b']=30;
  first['c']=50;
  first['d']=70;

  std::map<char,int> second (first.begin(),first.end());

  std::map<char,int> third (second);

  std::map<char,int,classcomp> fourth;                 // class as Compare

  bool(*fn_pt)(char,char) = fncomp;
  std::map<char,int,bool(*)(char,char)> fifth (fn_pt); // function pointer as Compare

  return 0;
}
```

上面的代码中，比较重要的是：

1. 第5行的函数，在27行被用函数指针指向。紧接着，在28行这个函数指针被用来构造名为`fifth`的map。(实际上直接传函数名也是可以的)
2. 第7-10行的结构体，在25行被用来构造名为`fourth`的map。

通过上面的代码来看，确实是两种自定义的方式，一个是利用函数指针，另一个是用函数对象。那为什么没见过用lambda表达式来自定义排序的呢？根本原因在于，map不仅需要函数对象or函数指针，还需要模板参数（函数对象的结构体or类，或者函数指针的类型）。

具体原因是，map底层是红黑树实现高效的排序。在map实现时，需要传进去那个“重载了调用运算符的结构体or类”来去构造红黑树，而红黑树本身也是一个模板类。因此，自定义排序要在模板参数里传入**已经重载调用运算符的结构体或者类**（而不是在某个地方传入实例化的函数对象）。使用“函数对象”这种方式时，就不需要再传入其他参数了，因为构造函数可以直接默认构造一个对象去使用。

而如果使用函数指针，则需要**在模板参数和构造函数参数里都传入参数**，前者是函数指针的类型，后者是函数指针。原因是编译器没办法通过一个函数指针的类型，来推断这个函数的内容。

那我们不使用lambda表达式自定义map排序的原因就是，lambda表达式是一个临时的函数对象，它没有上层的类实现，因此没办法传进去一个模板参数（但是其实也可以用lambda自定义map顺序，后面会说明）。



## 3. 使用lambda表达式自定义std::map<>排序

由于sort()不需要使用自定义类，而是直接使用函数对象就可以了。所以我们使用lambda表达式会很方便。lambda表达式写出来就已经是函数对象了，而重载调用运算符后的struct（或class）还需要实例化后才是函数对象。

不过，使用lambda表达式来自定义map的排序规则也是可以的，因为可以把函数对象当做函数指针来用。lambda表达式本身是一个函数对象，如果对map<int, int>来排序，它的类型就是bool (*)(int, int)，类似函数指针的写法，如下例：

```C++
auto cmp = [](int lhs, int rhs) { return lhs > rhs; };
map<int, string, bool (*)(int, int)> mp(p_cmp_func);
```

实际上也是可以编译通过的。稍微测试一下：

```C++
#include <iostream>
#include <map>
#include <string>
using namespace std;
int main() {
    auto cmp = [](int lhs, int rhs) { return lhs > rhs; };
    map<int, string, bool (*)(int, int)> mp(cmp);
    mp[1] = "abc";
    mp[4] = "def";
    mp[2] = "qqqq";
    for (auto i = mp.begin(); i != mp.end(); ++i) {
        cout << i->first << " " << i->second << endl;
    }
}
```

运行结果：

![image-20210315134557151](https://gitee.com/molinchn/BlogImage/raw/master/img/image-20210315134557151.png)

所以也是能运行的。

## 4. 小结

**下面进行总结：**

1. **函数指针可以用来为sort或map的自定义排序。sort只需要函数指针（或函数名），map不仅需要函数指针（或函数名），还需要在模板参数里写明函数指针的类型。**
2. **函数对象可以用来为sort或map进行自定义排序。sort只需要传函数对象，map需要的则是构建函数对象所需的类或结构体。**
3. **可以用lambda表达式为sort或map进行自定义排序。sort的自定义很简单，直接传入lambda表达式即可。为map自定义排序时不仅需要lambda表达式，还需要像函数指针一样，说明其类型。**



## 5. 题外话：std::map<>如何按value排序

前面所提到的map排序都是按照key排序，那怎么样才能按照value排序呢。

**思路很简单：可以把map装进vector<pair<>>里**，然后对这个vector自定义sort就可以了。

举个简单的例子：

```C++
#include <iostream>
#include <map>
#include <vector>
using namespace std;
int main() {
    map<int, string> mp;
    mp[1] = "abc";
    mp[4] = "def";
    mp[2] = "qqqq";
    cout<<"排序前"<<endl;
    for (auto i = mp.begin(); i != mp.end(); ++i) {
        cout << i->first << " " << i->second << endl;
    }
    cout<<"排序后"<<endl;
    vector<pair<int, string>> vec(mp.begin(), mp.end());
    sort(vec.begin(), vec.end(), [](pair<int, string> lhs, pair<int, string> rhs){
        return lhs.second > rhs.second;
    });
    for (auto x : vec){
        cout<<x.first << " " << x.second << endl;
    }
}
```

运行结果：

![image-20210315202041180](https://gitee.com/molinchn/BlogImage/raw/master/img/image-20210315202041180.png)

