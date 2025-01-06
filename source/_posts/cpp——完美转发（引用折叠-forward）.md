---
title: C++——完美转发（引用折叠+forward）
date: 2024-11-07 15:38:35
categories:
- C++
- C++知识
tags: 
- forward
- 引用折叠
typora-root-url: ./.. 
---

# 1. 引用折叠

> 其实在学习并发编程——thread原理的时候，就说过引用折叠这件事，还记得我是怎么说的吗？

**引用折叠**问题，即

- 左值引用+左值引用->左值引用
- 左值引用+右值引用->左值引用
- 右值引用+右值引用->右值引用

> **凡是折叠中出现左值引用，优先将其折叠为左值引用**

在类型推断中，如果传入的是一个左值，模板类型会自动将其**推断**为一个左值引用；而传入右值，模板类型会将其**推断**为右值：

```cpp
template <class F, class... Args>
auto commit(F&& f, Args&&... args) -> std::future<decltype(f(args...))> {}

int m = 5;
commit([](int& m){}, m); // 1
commit([](int& m){}, 2); // 2
commit([](int& m){}, std::ref(m)); // 3 
commit([](int& m){}, std::move(m)); // 4
```

- 对于1：`Args`推断m是`int&`类型，经过折叠后，`int& &&->int&`，仍然是`int&`。（注意，不会将其推断为int，虽然m确实是int类型，但是**左值的类型在模板参数中会被视为它本身的引用类型**）
- 对于2：`Args`推断m是`int`类型，经过折叠后，`int&&->int&&`，是`int&&`，右值引用。（注意，**右值会被推断为int类型而不是int&&**，int&&是右值引用类型而不是右值类型）
- 对于3：`Args`推断m是`int&`类型，并且经过ref包装后，thread和async内部不会对其使用delay解除cv修饰符和引用。
- 对于4：`Args`推断m是`int&&`类型，经过折叠后，`int&& &&->int&`，仍然是`int&&`。

------

综上，我们可以用模板定义一个左值引用：

```C++
//接受左值引用的模板函数
template <typename T>
void f1(T &t)
{}
```

也可以用模板类型定义一个**右值引用**时，但是传递给该类型的实参类型，会根据C++标准进行**引用折叠**：

```C++
//接受右值引用的模板函数
template <typename T>
void f2(T &&t)
{}
```

简而言之，当**模板函数**（或者**模板类**）的实参是一个**T**类型的**右值引用**：

1. 传递给该参数的实参是一个右值（42）时， T就是该右值类型
2. 传递给该参数的实参是一个左值（int a = 42）时， T就是该左值引用类型。

------

我们可以根据这个规律，实现一个类似`STL`的`move操作：

```c++
template<typename T>
typename remove_reference<T>::type && my_move(T&& t){
    return static_cast<typename remove_reference<T>::type &&>(t);
}
```

该代码用于定义了一个自定义的 `my_move` 函数，类似于标准库的 `std::move`。它将参数 `T` **强制转换**为右值引用，以实现引用折叠。通过 `my_move` 函数，我们可以将一个左值或右值**强制转换**为右值引用，从而允许调用方进行移动语义优化。

- `T&& t`：实现**引用折叠**；
- `remove_reference<T>::type`：移除类型 `T` 上的引用（如果有的话），无论 `T` 是左值引用、右值引用，还是非引用类型，`remove_reference<T>::type` 得到的都是原始的类型；
  - 例如，如果 `T` 是 `int&` 或 `int&&`，那么 `remove_reference<T>::type` 都会得到 `int`（**类型变成了int，但是值类型并不会改变**）
- `typename remove_reference<T>::type&&`：移除引用后，将该类型强制转换为右值引用（类型是右值引用），以便允许移动语义的优化
- `static_cast`：`static_cast<typename remove_reference<T>::type&&>(t)` 将参数 `t` 转换为右值引用。这是核心操作，使得 `t` 可以被移动，而不是拷贝。
  - 它会将参数`t`的类型转换为右值引用，但是它的值类型仍然不变。如果`t`的值类型本来是左值，那么转换后仍然是左值，只不过参数类型是右值引用。

若我们在函数中作如下调用：

```C++
void use_tempmove()
{
    int i = 100;
    my_move(i);
    //推断规则
    /*
    1  T被推断为int &，而值类型也是左值
    2  remove_reference<int &>的type成员是int（去引用）
    3  my_move 的返回类型是int&&（强制加&&）
    4  推断t类型为int& && 通过折叠规则t为int&类型
    5  最后这个表达式变为 int && my_move(int &t)
    */

    auto rb = my_move(43);
    //推断规则
    /*
    1  T被推断为int
    2  remove_reference<int>的type成员是int
    3  my_move 的返回类型为int&&
    4  my_move 的参数t类型为int &&
    5  最后这个表达式变为 int && my_move(int && t)
    */
}
```

# 2. forward原样转发

> 我们同样在学习引用折叠的同时，简单了解过forward原样转发。

比如，在thread的模板类声明中：

```cpp
template <class _Fn, class... _Args, enable_if_t<!is_same_v<_Remove_cvref_t<_Fn>, thread>, int> = 0>
    _NODISCARD_CTOR_THREAD explicit thread(_Fn&& _Fx, _Args&&... _Ax) {
        _Start(_STD forward<_Fn>(_Fx), _STD forward<_Args>(_Ax)...);
    }
```

- `_Fn&&` 和 `_Args&&` 被称为**转发引用**，它们会根据传入参数的类型自动推断为左值引用或右值引用
  - 如果传入的参数是右值（比如使用 `std::move`），则 `_Fn&&` 和 `_Args&&` 会因为**引用折叠**被推导为右值引用；如果是左值，则会被推导为左值引用。
    - 当传入`_Args`的类型是`int&`（左值）时，后面加`&&->int& &&`折叠为`int&`
      - 如果传递的是左值，类型 `T` 会被推导为**左值引用**类型，即 `int&`。这是因为左值的类型在模板参数中会被视为它本身的引用类型。
    - 当传入`_Args`的类型是`int&`（左值引用）时，后面加`&&->int& &&`折叠`为int&`
    - 当传入`_Args`的类型是`int`（右值）时，后面加`&&->int&&`折叠为`int&&`
    - 当传入`_Args`的类型是`int&&`（右值引用）时，后面加`&&->int&&`折叠为`int&&`
- `std::forward<Fn>(Fx)` 和 `std::forward<Args>(Ax)...` 会保留参数的值类别（左值或右值），确保可以进行适当的移动或拷贝。

------

> STL 的 `std::move`其实是“去引用”的实现，为了将传入参数转换为右值引用，实现移动拷贝。但有时我们也需要**保留传入参数的原本类型不变**（左值、右值或引用），进行原样转发，`std::forward`就是为了实现此功能而定义的。

**举例说明：**

若我们定义如下模板函数：

```C++
template <typename F, typename T1, typename T2>
void flip1(F f, T1 t1, T2 t2)
{
    f(t2, t1);
}
```

其中，`F`是模板函数对传入可调用对象类型自动推断的参数模板类型，`f`就相当于传入的可调用对象，`t1`和`t2`是可调用对象的参数。

如果我们想要将可调用对象传入模板函数`flip1`，通过对模板函数`flip1`内部的修改，间接影响到外部传入的值。这样可行吗？尝试一下：

```C++
void ftemp(int v1, int &v2)
{
    cout << v1 << " " << ++v2 << endl;
}

void use_ftemp(){
    int j = 100;
    int i = 99;
    flip1(ftemp, j, 42);
    cout << "i is " << i << " j is " << j << endl;
}
```

假如我们定义了一个`ftemp`函数，其`v2`参数是一个**左值引用**，那我们将参数`v2`传入`ftemp`函数后，如果对传入参数`v2`进行修改，想必一定会影响到外部传入的值吧？

如果我们不将`ftemp`函数传入模板函数`flip1`中，那么对传入参数`v2`进行修改，确实会影响到外部传入的值。但是，如果我们将`ftemp`函数传入模板函数`flip1`中，就不会影响了。

我们打印上面代码的输出：

```
42 101
i is 99 j is 100
```

明明在`ftemp`函数中，我们将 v2++ 变为101，但为什么在外部的传入形参`v2`的实参`j`仍然是100？

因为`ftemp`的`v2`参数虽然是引用，但其实是`flip1`的形参`t1`的引用。`t1`只是形参（t1的类型是int而不会推断为int&，这和后面有区别。为什么后面传入左值会将其推断为int&，但这里却推断为了int，为什么？其实这涉及到值传递和引用传递的区别），修改`t1`并不能影响外边的实参j。想要达到修改实参的目的，需要将`flip1`的参数修改为引用。我们先实现修改后的版本`flip2`:

```C++
template <typename F, typename T1, typename T2>
void flip2(F f, T1 &&t1, T2 &&t2)
{
    f(t2, t1);
}
```

将参数传入测试：

```C++
int j = 100;
int i = 99;
flip2(ftemp, j, 42);
cout << "i is " << i << " j is " << j << endl;
```

输出为：

```
42 101
i is 99 j is 101
```

我们发现，传入的实参`j`确实被修改了。因为`flip2`的`t1`参数类型为`T1`的**右值引用**，当把实参`j`赋值给`flip2`时，`T1`变为`int&`，`t1`的类型就是`int& &&`，通过折叠`t1`变为`int&`类型。这样`t1`就和实参`j`绑定了，在`flip2`内部修改`t1`，就达到了修改j的目的。

------

**但是`flip2`同样存在一个问题**：如果`flip2`的第一个参数`f`是一个**接受右值引用参数的函数**，会出现编译错误。

我们实现一个形参参数为**右值引用**类型的函数`gtemp`：

```c++
void gtemp(int &&i, int &j)
{
    cout << "i is " << i << " j is " << j << endl;
}
```

如果我们将`gtemp`作为参数传递给`flip2`会报错:

```c++
int j = 100;
int i = 99;
// flip2(gtemp, j, 42) 会报错
// 因为42作为右值纯递给flip2，t2会被折叠为int&&类型
cout << "i is " << i << " j is " << j << endl;
```

当我们将实参`42`传递给`flip2`的第二个参数时，`T2`被推断为`int`类型，`t2`经过引用折叠变为`int&&`类型。`t2`作为参数传递给`gtemp`的第一个参数时会报错。`t2` 此时的类型是 `int&&`，而 `gtemp` 函数第一个参数的类型也是  `int&&`，那么为什么会编译报错呢？

因为 `42` 是一个右值常量，当它被传递到 `flip2` 时，它的生命周期只存在于 `flip2` 调用开始的那一瞬间，而在 `flip2` 内部，`42` 被传递到 `f(t2, t1);` 时，已经不能保证其原有的右值特性。这会导致编译器无法确定如何安全地将 `42` 转换为一个右值引用传递给 `gtemp`。

具体来说，`42` 被传递到 `flip2` 时，`T2` 会被推导为 `int&&`，所以 `t2` 的类型变成 `int&&`。但当 `flip2` 内部调用 `f(t2, t1);` 时，`t2` 被直接传递给 `gtemp` 的第一个参数，而 `t2` 已经不再是一个临时的右值表达式，而是一个左值（因为 `t2` 是 `flip2` 的形参）。而`gtemp`第一个参数为右值引用类型，他需要接收右值，导致失败。

> 为什么`t2`被推断为int&&（右值引用），但它却是左值引用呢？

尽管 `t2` 的类型是 `int&&`，但**一旦你在 `flip2` 内部使用 `t2`，它就会变成了一个左值**，因为 `t2` 是一个有名字的变量。C++ 标准中规定，**有名字的右值引用变量会被视为左值**（在 C++ 中，只有**没有名字的对象**才是右值。**有名字的对象**（无论它是左值引用、右值引用，还是普通的值）都能被当作左值来使用）。因此，在 `f(t2, t1);` 中传递 `t2` 时，它实际上是作为一个左值传递的。

也就是说，**虽然 `t2` 的类型是 `int&&`，但在 `f(t2, t1);` 中，`t2` 是一个左值**，而 `gtemp` 的第一个参数期望的是一个右值引用类型（`int&&`），这导致了编译错误。

**举例说明：**

```
int&& x = 5;  // x是一个右值引用绑定到右值 5
```

右值引用绑定到右值时，它本身成为了一个**左值**，可以被引用或传递给其他函数。

在此代码中，`x` 是一个**右值引用**，但它有一个名字，可以通过 `x` 访问它，改变它的值，或传递它给其他函数，它满足左值的特征。因此，尽管`x`最初是一个绑定到右值的引用，但它现在作为一个有名字的对象，就变成了左值。

------

上面的错误可以简化为:

```c++
int i = 100;
int&& m = 200;
int&& k =  m;
```

上面代码会报错，因为m虽然是一个右值引用，并且绑定到了一个右值 200 上，但因为它有了名字，所以成了一个左值。而左值不能绑定到右值引用 k 上。

> 我们如何解决这个问题？

```C++
int i = 100;
int&& m = 200;
int&& k = int(m);
```

通过 int 强制类型转换，`int(m)` 创建了一个临时的右值（拷贝了 `m` 的值），这就相当于将 `m` 转换成了一个临时对象，自然也就可以将m绑定给k。

> 注意：m 本身的类型是右值引用，但是它的值类型是左值（有了名字），我们这里也只是通过in(m)构造了一个临时变量（右值）赋值给右值变量k。
>
> - 类型为**右值引用**的变量，它的值类型**可以是左值也可以是右值**
> - 类型为**左值引用**的变量，它的值类别只能是**左值**

当然也可以通过如下方式：

```cpp
int i = 100;
int&& m = 200;
int&& k = std::move(m);
```

总之就是将m转化为右值即可。大家要清楚的是即使m是一个int&&类型，但是它本身是一个左值（值类型是左值）。

------

综上所述，解决上面问题的办法就是实现一个flip函数，内部实现对T2，T1类型的原样转发。

```cpp
// 修改前
template <typename F, typename T1, typename T2>
void flip(F f, T1 t1, T2 t2) // 默认模板参数
{
    f(t2, t1);
}
// 修改前
template <typename F, typename T1, typename T2>
void flip(F f, T1 &&t1, T2 &&t2) // 右值引用模板参数
{
    f(t2, t1);
}
// 修改后
template <typename F, typename T1, typename T2>
void flip(F f, T1&& t1, T2&& t2) // 右值引用模板参数
{
    f(std::forward<T2>(t2), std::forward<T1>(t1));
}
```

`std::forward` 用于转发传递给一个函数的参数，并确保它的**值类别**（是否是左值或右值）在转发时不发生变化（不仅保持参数的值类别，还保持其类型）。`std::forward`的实现如下：

```cpp
template <class _Ty>
_NODISCARD constexpr _Ty&& forward(
    remove_reference_t<_Ty>& _Arg) noexcept { // forward an lvalue as either an lvalue or an rvalue
    return static_cast<_Ty&&>(_Arg);
}

template <class _Ty>
_NODISCARD constexpr _Ty&& forward(remove_reference_t<_Ty>&& _Arg) noexcept { // forward an rvalue as an rvalue
    static_assert(!is_lvalue_reference_v<_Ty>, "bad forward call");
    return static_cast<_Ty&&>(_Arg);
}
```

- `std::forward`的精髓在配合模板时才能挥发出来，上层函数给个类型T，forward函数返回个`T&&`，这是个万能类型。作为对比, `std::move`函数返回`remove_reference_t<T>&&`，**只能**是个右值引用类型，而不可能能返回左值引用类型。如果程序员将代码写成`std::forward<MyClass>(my_obj)`的形式，完美转发是不发挥作用的，无论`my_obj`的性质是左还是右。
- `forward`的返回值是根据`_Ty`决定的，如果我们`std::forward<Args&>(args)`那么返回的是`Args& &&`，最后肯定还是`Args&`，返回一个左值引用；如果我们`std::forward<Args>(args)`那么返回的其实是`Args&&`，最后还是`Args&&`。所以 forward 才会实现传进来的左值返回也是左值引用，传进来右值返回是右值引用。而且**forward 必须配合引用折叠**（万能引用）才能实现进来的左值返回也是左值，传进来右值返回是右值。
- remove_reference_t 是一个模板类，用于去除变量的引用，它可以接受左值、左值引用、右值引用三类，并将后面两个的引用去除，返回原本的类型：

```cpp
template <class _Ty>
using remove_reference_t = typename remove_reference<_Ty>::type;
```

很简单，我们使用 `remove_reference_t<_Ty>& _Arg` 时，其实是返回 `remove_reference<_Ty>`的成员`type`，`type`会将引用去掉，仅返回推断的原本类型：

```cpp
template <class _Ty>
struct remove_reference {
    using type                 = _Ty;
    using _Const_thru_ref_type = const _Ty;
};

template <class _Ty>
struct remove_reference<_Ty&> {
    using type                 = _Ty;
    using _Const_thru_ref_type = const _Ty&;
};

template <class _Ty>
struct remove_reference<_Ty&&> {
    using type                 = _Ty;
    using _Const_thru_ref_type = const _Ty&&;
};
```

> `std::forward<T>`只是单纯的返回一个T&&，但是T的类型需要上层函数传入，如果T是int&，那么返回的其实也是int&；如果T是int&&或者int（右值引用模板参数中，右值会被推断为int，所以这里的int代表右值），返回的其实还是int&&。

# 3. 模板推导的基本规则

C++ 的模板类型推断在函数调用时通常会根据传入的实参类型来推断模板参数的类型。比如：

```cpp
template <typename T1, typename T2>
void func(T1 a, T2 b);
```

当你调用：

```cpp
int m = 4;
func(m, 3.14);
```

`m` 是一个 **`int` 类型的左值**，而 `3.14` 是一个 **`double` 类型的右值**。`T1`会被推导为`int`，而不是`int&`；`3.14`会被推导为`double`。

模板类型推断的规则如下：

- **类型推断**：模板推导时，C++ 会选择 **按值传递** 类型，而不是推断为左值引用（比如int&），除非模板参数明确要求使用引用类型（如通过引用传递参数或者显式声明为引用类型）。。

> 对于参数 `T&&`，模板推导的行为有一些特殊之处，因为它同时适用于 **左值** 和 **右值**

```cpp
template <typename T1, typename T2>
void func(T1&& a, T2&& b){
    // 使用 t
}

int m = 4;
func(m, 3.14);  // 传入一个左值
```

- 在这里，`T1` 会被推导为 `int&`，因为 `m` 是左值，而左值传递给右值引用 (`T1&&`) 会根据 C++ 引用折叠规则推导出 `T1` 为 `int&`（**int& &&->int&**）。
- 在模板 `T2&& b` 中，`T2` 会被推导为 `double`，因为右值传递给右值引用模板会保留原始类型，最后根据引用折叠规则推导出`double&&`。

这是因为 C++ 中的 **右值引用** 模板推导规则有点特别，尤其是在传递左值时。

- 当你将一个 **左值** 传递给 `T&&` （右值引用）类型的模板时，C++ 的模板推导会选择 **右值引用** 推导为 **左值引用**，也就是 `T` 会被推导为 `T&`，而不是 `T`。
- 因此，在传递一个左值（如 `m`）时，`T&&` 会推导为 `int& &&`  ，而不是 `int&&`，因为左值引用 (`int&`) 是绑定到左值的，此时形参类型被推断为左值引用 (`int&`) ，可以用来接收类型为左值的实参。

**右值引用模板参数（`T&&`）** 是为了能够接受 **左值** 和 **右值**，而对左值使用时，C++ 会根据 **引用折叠规则** 将 `T&&` 推导为 `T&`，这样左值就可以绑定到左值引用上。

但是左值引用模板参数（T&）会推断传入的左值类型为int，因为它本身就可以接受左值，所以不需要推断为int&。但它不能绑定右值，左值引用不会接受一个右值。

**总结：**

- **`T&`（左值引用）** 只能绑定 **左值**，不能绑定右值。
- **`T&&`（右值引用）** 主要用于绑定 **右值**，并且可以通过引用折叠将左值传递给右值引用类型时变为左值引用（`T&`）。
