+++
title = 'Cpp'
date = 2024-03-24T15:22:48+08:00
draft = false
+++

关于Cpp总是有说不完的技巧，海量的教人如何做人的书籍，无穷的最佳实践。但我最近领悟到了一个道理，即万物80/20法则。

比如机器学习，理论上肯定各种数学，优化理论，实践中计算机体系结构，高性能网络等等。但实际上工程实践肯定就那么点事儿。在数学发展早期，解一元二次方程是大数学家的事情，但是现在高中生就教了公式了。所谓伟大的学者总结方法，后人就学着用就行了，而且真正好用的，常用的技巧，必然只有一点点。即万物都是80的深度，20的常用。

当然这篇blog不是为了讲大道理。关于cpp，一个常见的批评就是“心智负担重”，具体来说，当看到一大坨模版代码时，一是涉及的语法知识就很多，未必看得懂语法，还有一点经常是，每行代码都能看懂，但不知道作者为啥这么写，经常读着读着，就怀疑人生，“是不是有什么精深的语法知识点我没掌握？”我感觉这就是所谓的心智负担。

不过自从我认识到80/20原则之后，就开始对cpp有点自信了。最近慢慢也是看懂了模版元编程的一些通用技巧。这里总结下，欢迎提出建议。

## 模版元编程难懂的原因是类型不是一等成员

这其实是历史导致的。比如新出的语言，Zig，type就是first class member，那么你可以写：

```zig
var x: type = int;
```
即，把一个type当成一个变量使用。这个在cpp里很难实现。那么cpp里如何来实现这一行代码呢？其实是如下。

```c++
using x = int
```

那么我们如果要对某个类型做个计算，比如组合一个新类型，Zig里其实非常直观

```Zig
fn Some(comptime InputType: type) type
```

即输入一个类型，输出一个新类型，那么cpp里对应的东西是啥呢？

```c++
template <typename InputType>
struct Some {
  using OutputType = ...
}
```

相比之下， Zig直观太多。那么很自然的，计算一个类型，Zig里就是调用函数，而C++则是模板类实例化，然后访问类成员。

```c++
Some<InputType>::OutputType
```

相当于对于InputType调用一个Some“函数”，然后输出一个OutputType。

## 模版偏特化是编译期的if-else

比如实现一个函数，输入一个bool值，根据bool值，如果为真，那么输出type A，如果为假那么输出type B。

```c++
template <bool, typename A, typename B>
struct Fn {
	using OutputType = A;
};

template<typename A, typename B>
struct Fn<false, A, B> {
	using OutputType = B;
};
```
这样就比较简单了

```c++
Fn<sizeof(A) > sizeof(B), A, B>::OutputType
```
这就是比较类型的size大小，如果A大，OutputType就是A，如果B大，OutputType就是B。

如果用Zig来做，那简直不要太简单。

## 递归推导模版就是编译期的循环

递归最大的用处就是遍历一个列表，当然这里是type的list不是变量的list。其实如果有一点lisp/scheme的基础的话，理解起来就蛮简单。

比如有一堆类型，`typename ...T`，我们怎么遍历这个类型并且选择我们需要的一个类型？返回第N个类型。

首先，我们写出“函数原型”.

```c++
template <int Index, typename ...T>
struct Fn;
```

然后我们递归的基础情况

```c++
template <typename Head, typename ...Tail>
struct Fn<0, Head, Tail...> {
	using Output = Head;
};
```

然后写递归式，

```c++
template<int Index, typename Head, typename ...Tail>
struct Fn<Index, Head, Tail...> : public Fn<Index - 1, Tail...>
{
};
```

这个地方其实稍微有点难理解，上面一个偏特化类似于，

```c++
if (Index == 0)
    return Head;
```

下面一个呢，类似于

```c++
else {
	return Fn(Tail...);
}
```

这里利用的其实是继承，让他一路继承下去，如果Index不等于0，那么这个类其实是空类，即，我们无法继承到`using Output`的这个`Output`。但是index总会等于0，那么到了等于0的那天，递归就终止了，因为，我们不需要继续Index - 1下去了，编译器会选择特化好的`Fn<0, T，Tail...>`这个实例，而不会选择继续递归。

那我能不能不用继承呢？

```c++
template<int Index, typename Head, typename ...Tail>
struct Fn<Index, Head, Tail...> 
{
	using Output = typename Fn<Index - 1, Tail...>::Output;
};
```

这样其实也可以，因为编译器推导到`Fn<0, Tail...>`时，应该就停止继续推导了，因为他能直接找到合适的Output。只是注意，需要在Fn前加一个typename。

## 编译器报警，编译期的printf

编译期编程一个问题就是无法调试。那咋办呢，一个简单的办法就是利用编译器报警，比如如果要证明我上述的代码写的对不对，一个办法就是

```c++
template <typename T>
class TD;
```

只是证明一个类型，但不实现它。然后用下面语句来推导

```c++
TD<Fn<1, int, double, float>::Output> XXX;
```

然后编译，这个时候编译期会帮你推导出类型来，然后报错，因为TD没有实现，所以编译中断。

```
type3.cc:75:41: error: implicit instantiation of undefined template 'TD<double>'
  TD<Fn<1, int, double, float>::Output> b;
                                        ^
type3.cc:64:7: note: template is declared here
class TD;
      ^
```

看上面`TD<double>`，就知道推导的是啥了。

## 有了定义变量，if-else，还有递归

就可以实现很多逻辑。实际上，模板推导是一个图灵完全过程。只是这个写法实在太绕了。
远不如Zig直观。


