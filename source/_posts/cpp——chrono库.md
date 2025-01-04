---
title: C++——chrono库
date: 2025-01-04 19:57:13
categories:
- C++
- C++知识
tags: 
- chrono
typora-root-url: ./..
---



# 序言

`chrono`库主要用于处理日期和时间，在多线程编程中常和`thread`的`sleep_for`、`until`以及命名空间`std::this_thread`中的一些公共成员函数组合使用。`chrono`库主要包含三种类型的类：`时间间隔duration`、`时钟clocks`、`时间点time point`。

参考：

[C++ 标准库  | 菜鸟教程](https://www.runoob.com/cplusplus/cpp-libs-chrono.html)

[处理日期和时间的chrono库 | 爱编程的大丙](https://subingwen.cn/cpp/chrono/)

[C++11 std::chrono库详解 - mjwk - 博客园](https://www.cnblogs.com/jwk000/p/3560086.html)

[C++ std::chrono库使用指南 (实现C++ 获取日期,时间戳,计时等功能)-CSDN博客](https://blog.csdn.net/qq_21438461/article/details/131198438)

------

# 1. 时间间隔 duration

## 1.1 介绍

`std::chrono::duration`（时间间隔）是C++中表示时间段的类型。它是一个模板类，可以表示秒、毫秒、微秒等不同的时间单位。它不仅可以表示时间长度，也可以表示特定时间点到另一个时间点的间隔。与人的感知类似，`duration`可以表示"过去多久"（例如5秒前）、“现在”（例如现在到程序开始的时间）、或"未来多久"（例如5秒后）。

`duration` 位于头文件 `<chrono >`中，原型为：

```cpp
template<class Rep, class Period = std::ratio<1>> 
class duration;
```

- `Rep`：表示**时钟数（周期）的类型**（默认为整形，可以自定义为 `double`、`float`等其他类型）。若 `Rep` 是浮点数，则 `duration` 能使用小数描述时钟周期。

- `Period`：表示**时钟的周期**，它的原型如下：

  ```cpp
  template<std::intmax_t Num,std::intmax_t Denom = 1> 
  class ratio;
  ```

  `ratio`类表示每个时钟周期的秒数，第一个模板参数`Num`代表分子，`Denom`代表分母，该分母值默认为1，因此，`ratio`代表的是**分子除以分母的数值**，比如：`ratio<2>`代表一个时钟周期是2秒，`ratio<60>`代表一分钟，`ratio<60*60>`代表一个小时，`ratio<60*60*24>`代表一天，r`atio<1,1000>`代表的是1/1000秒，也就是1毫秒，`ratio<1,1000000>`代表一微秒，`ratio<1,1000000000>`代表一纳秒。

  > 标准库中定义了一些常用的时间间隔，它们都位于`chrono`命名空间下，定义如下：

  | 类型                              | 定义                  |
  | --------------------------------- | --------------------- |
  | 纳秒：`std::chrono::nanoseconds`  | `ratio<1,1000000000>` |
  | 微秒：`std::chrono::microseconds` | `ratio<1,1000000>`    |
  | 毫秒：`std::chrono::milliseconds` | `ratio<1,1000>`       |
  | 秒：`std::chrono::seconds`        | `ratio<1>`            |
  | 分：`std::chrono::minutes`        | `ratio<60>`           |
  | 时：`std::chrono::hours`          | `ratio<3600>`         |

`duration` 可通过以下三种方法构造：

```cpp
// 1. 拷贝构造函数
duration( const duration& ) = default;
// 2. 通过指定时钟周期的类型来构造对象
template< class Rep2 >
constexpr explicit duration( const Rep2& r );
// 3. 通过指定时钟周期类型，和时钟周期长度来构造对象
template< class Rep2, class Period2 >
constexpr duration( const duration<Rep2, Period2>& d );
```

且不同 `duration` 对象可通过运算符进行加减乘除，运算符在 `duration` 中已被重载：

- `operator=`：赋值运算符
- `operator+`、`operator-`
- `operator++`、`operator++(int)`、`operator-–`、`operator-–(int)`
- `operator+=`、`operator-=`、`operator*=`、`operator/=`、`operator%=`

此外，我们也可以通过调用 `count()` 函数获取时间间隔的**时钟周期数**（秒、分、时），得到 `duration` 对象存储的时间，原型如下：

```cpp
constexpr rep count() const;
```

## 1.2 使用

1）在创建 `duration` 对象时，我们既可以通过构造函数定义，也可以通过标准库定义的类型进行定义。

```cpp
#include <chrono>
#include <iostream>
using namespace std;
int main()
{
    chrono::hours h(1);                          // 一小时
    chrono::milliseconds ms{ 3 };                // 3 毫秒
    chrono::duration<int, ratio<1000>> ks(3);    // 3000 秒

    // chrono::duration<int, ratio<1000>> d3(3.5);  // error
    chrono::duration<double> dd(6.6);               // 6.6 秒

    // 使用小数表示时钟周期的次数
    chrono::duration<double, std::ratio<1, 30>> hz(3.5);
}
```

2）我们可以对 `duration` 对象进行一些算术运算（支持不同时间单位之间的运算），包括加法、减法、乘法和除法：

```cpp
#include <iostream>
#include <chrono>
using namespace std;

int main()
{
    chrono::minutes t1(10);
    chrono::seconds t2(60);
    chrono::seconds t3 = t1 - t2;
    cout << t3.count() << " second" << endl; // 540 second
}
```

> 注意事项：`duration`的加减运算有一定的规则：当两个`duration`时钟周期不相同的时候，会先统一成一种时钟，然后再进行算术运算，统一的规则如下：假设有`ratio<x1,y1>` 和 `ratio<x2,y2>`两个时钟周期，首先需要求出`x1`，`x2`的最大公约数`X`，然后求出`y1`，`y2`的最小公倍数`Y`，统一之后的时钟周期`ratio`为`ratio<X,Y>`。

```cpp
chrono::duration<double, ratio<9, 7>> d1(3);
chrono::duration<double, ratio<6, 5>> d2(1);
// d1 和 d2 统一之后的时钟周期
chrono::duration<double, ratio<3, 35>> d3 = d1 - d2;
```

对于分子6,、9最大公约数为3，对于分母7、5最小公倍数为35，因此推导出的时钟周期为`ratio<3,35>`

3）也支持比较操作，包括等于、不等于、小于、大于、小于等于和大于等于：

```cpp
std::chrono::seconds sec1(5);
std::chrono::seconds sec2(3);
if (sec1 > sec2) {
    // do something
}
```

4）通过使用`duration_cast`，我们可以将一个`duration`转换为**不同的单位**。例如：

```cpp
std::chrono::minutes min(1);
auto sec = std::chrono::duration_cast<std::chrono::seconds>(min);  // sec is 60 seconds
```

# 2. 时间点 time point

`chrono`库中提供了一个表示时间点的模板类`std::chrono::system_clock::time_point`，可以表示不同精度的时间，该类的定义如下：

```cpp
template<class Clock,class Duration = typename Clock::duration> 
class time_point;
```

它被实现成如同存储一个 `Duration（std::chrono::duration）` 类型的自 `Clock` （`system_clock`或`steady_clock`）的纪元起始开始的时间间隔的值，通过这个类最终可以得到时间中的某一个时间点。为了更好地理解`time_point`，我们可以将其比喻为一个足球场上的地标。纪元（`epoch`，通常是1970年1月1日午夜）就像球场的一端，而`time_point`就像球场上的一个具体位置，通过度量从球场一端到这个位置的距离（时间），我们可以确定这个位置的确切位置。举例：

```cpp
#include <iostream>
#include <chrono>

int main() {
    // 获取当前的时间点
    std::chrono::system_clock::time_point now = std::chrono::system_clock::now();

    // 转换为时间戳并打印
    std::time_t now_c = std::chrono::system_clock::to_time_t(now);
    std::cout << "Current time: " << std::ctime(&now_c) << std::endl;

    return 0;
}
```

在上述示例中，我们获取了当前的时间点，并将其转换为C时间，然后打印。这只是`time_point`的基本使用，随着我们深入研究`std::chrono`库，我们将发现更多关于`time_point`的有趣和强大的应用。

1）`std::chrono::system_clock::time_point` 可通过以下三种方法构造：

```cpp
// 1. 构造一个以新纪元(epoch，即：1970.1.1)作为值的对象，需要和时钟类一起使用，不能单独使用该无参构造函数
time_point();
// 2. 构造一个对象，表示一个时间点，其中d的持续时间从epoch开始，需要和时钟类一起使用，不能单独使用该构造函数
explicit time_point( const duration& d );
// 3. 拷贝构造函数，构造与t相同时间点的对象，使用的时候需要指定模板参数
template< class Duration2 >
time_point( const time_point<Clock,Duration2>& t );
```

举例：

```
std::chrono::system_clock::time_point epoch_time; // 1
std::chrono::system_clock::time_point custom_time(system_clock::duration(3600)); // 2

// 当前时间点（高精度）
std::chrono::high_resolution_clock::time_point high_res_now = high_resolution_clock::now(); 
// 将高精度时间点转换为系统时钟的时间点
std::chrono::system_clock::time_point sys_now = time_point_cast<system_clock::duration>(high_res_now); // 3

using dday = duration<int, ratio<60 * 60 * 24>>;
// 新纪元1970.1.1时间 + 10天
time_point<system_clock, dday> t(dday(10));  // 4
```

其中，**1** 和 **2** 分别使用了前两个构造函数，而 **3** 通过另一个 `time_point` 的不同时间单位进行构造，可以改变时间点的分辨率；**4** 同样调用了 **2** 的构造函数，只不过是我们显式构造了一个 `time_point` 模板类对象，而不是直接通过 `std::chrono::system_clock::time_point` 调用已经定义好的  `time_point` 模板类对象。

2）除此之外，时间点`time_point`对象和时间段对象`duration`之间还支持直接进行算术运算（即加减运算），时间点对象之间可以进行逻辑运算（大小比较），如下图

> 算术运算支支持`time_point`对象和`duration`对象进行，逻辑运算仅支持两个`time_point`的比较。   
>
> 其中 `tp` 和 `tp2` 是`time_point` 类型的对象， `dtn` 是`duration`类型的对象。

![image-20250104235504271](/images/$%7Bfiilename%7D/image-20250104235504271.png)

<center>图片来源：https://subingwen.cn/cpp/chrono/#4-2-time-point-cast</center>

我们可以对`time_point`进行加减运算来得到新的`time_point`：

```cpp
#include <iostream>
#include <chrono>

int main() {
    // 获取当前时间点
    std::chrono::system_clock::time_point now = std::chrono::system_clock::now();
    // 创建一个1小时的duration对象
    std::chrono::hours one_hour(1);
    // 通过加法运算得到1小时后的时间点
    std::chrono::system_clock::time_point one_hour_later = now + one_hour;

    return 0;
}
```

在上述代码中，我们首先获取了当前的时间点`now`，然后创建了一个表示1小时的`std::chrono::hours`对象`one_hour`。通过将`now`和`one_hour`进行加法运算，我们得到了1小时后的时间点`one_hour_later`。

我们可以使用比较操作符比较两个`time_point`对象：

```cpp
#include <iostream>
#include <chrono>

int main() {
    // 获取当前时间点
    std::chrono::system_clock::time_point now = std::chrono::system_clock::now();

    // 创建一个1秒后的时间点
    std::chrono::system_clock::time_point one_sec_later = now + std::chrono::seconds(1);

    // 比较两个时间点
    if (one_sec_later > now) {
        std::cout << "one_sec_later is later than now.\n";
    } else {
        std::cout << "one_sec_later is not later than now.\n";
    }

    return 0;
}
```

在上述代码中，我们创建了一个1秒后的时间点`one_sec_later`，然后使用大于操作符`>`比较了`one_sec_later`和`now`，并打印了比较结果。

3）`time_point` 有一些公有函数用于获取时间点：

| **方法名称** | **描述**                     | **返回类型**             |
| ------------ | ---------------------------- | ------------------------ |
| min()        | 获取可能的最小时间点         | system_clock::time_point |
| max()        | 获取可能的最大时间点         | system_clock::time_point |
| now()        | 获取从纪元开始到现在的时间点 | system_clock::time_point |

4）`time_since_epoch` 是 C++ 标准库中 `std::chrono::time_point` 的一个成员函数，用来获取当前**时间点**到新纪元（epoch，即 1970 年 1 月 1 日 00:00:00 UTC）之间的时间间隔。原型为：

```cpp
constexpr duration time_since_epoch() const;
```

常用于以下三点：

1. 计算时间点之间的间隔。
2. 获取时间点的绝对偏移量。
3. 用于转换为其他时间表示形式（如秒、毫秒等）。

```cpp
示例 1：获取时间点到 epoch 的秒数
system_clock::time_point now = system_clock::now(); // 获取当前时间点
auto duration_since_epoch = now.time_since_epoch(); // 获取从 epoch 到当前时间的持续时间
auto seconds = duration_cast<std::chrono::seconds>(duration_since_epoch); // 转换为秒
std::cout << "Seconds since epoch: " << seconds.count() << "s" << std::endl;

示例 2：计算两个时间点之间的间隔
std::chrono::system_clock::time_point start = std::chrono::system_clock::now(); // 获取开始时间点
std::this_thread::sleep_for(std::chrono::milliseconds(500)); // 模拟延迟
system_clock::time_point end = std::chrono::system_clock::now(); // 获取结束时间点
auto duration = end - start; // 计算两个时间点之间的间隔，隐式调用了 time_since_epoch
auto milliseconds = duration_cast<milliseconds>(duration);
std::cout << "Elapsed time: " << milliseconds.count() << "ms" << std::endl;

示例 3：时间点偏移
system_clock::time_point now = system_clock::now(); // 获取当前时间点
system_clock::time_point future = now + seconds(10); // 偏移 10 秒后的时间点
auto future_duration = future.time_since_epoch(); // 计算偏移后的时间
auto future_seconds = duration_cast<seconds>(future_duration);
std::cout << "Future time (seconds since epoch): " << future_seconds.count() << "s" << std::endl;
```

# 3. 时钟 clocks

`chrono`库中提供了获取当前的系统时间的时钟类，包含的时钟一共有三种：

- `system_clock`：系统的时钟，系统的时钟可以修改，甚至可以网络对时，因此使用系统时间计算时间差可能不准。
- `steady_clock`：是固定的时钟，相当于秒表。开始计时后，时间只会增长并且不能修改，适合用于记录程序耗时
- `high_resolution_clock`：和时钟类 `steady_clock` 是等价的（是它的别名）

## 3.1 system_clock

我们的生活在时间的流逝中不断推进，这个自然的过程在C++中被封装在了`std::chrono::system_clock`这个类中。它表示当前的墙上时间，从中可以获得当前时间，也可以在时间点上执行算术运算。

`system_clock` 结构体在底层源码中的定义如下：

```cpp
struct system_clock { // wraps GetSystemTimePreciseAsFileTime/GetSystemTimeAsFileTime
    using rep                       = long long;
    using period                    = ratio<1, 10'000'000>; // 100 nanoseconds
    using duration                  = chrono::duration<rep, period>;
    using time_point                = chrono::time_point<system_clock>;
    static constexpr bool is_steady = false;

    _NODISCARD static time_point now() noexcept 
    { // get current time
        return time_point(duration(_Xtime_get_ticks()));
    }

    _NODISCARD static __time64_t to_time_t(const time_point& _Time) noexcept 
    { // convert to __time64_t
        return duration_cast<seconds>(_Time.time_since_epoch()).count();
    }

    _NODISCARD static time_point from_time_t(__time64_t _Tm) noexcept 
    { // convert from __time64_t
        return time_point{seconds{_Tm}};
    }
};
```

- `rep`：时钟周期次数是通过整形来记录的`long long`
- `period`：一个时钟周期是100纳秒`ratio<1, 10'000'000>`
- `duration`：时间间隔为`rep*period`纳秒`chrono::duration<rep, period>`
- `time_point`：时间点通过系统时钟做了初始化`chrono::time_point<system_clock>`，里面记录了新纪元时间点

且 `std::chrono::system_clock` 提供了**三个静态函数**：

```cpp
// 返回表示当前时间的时间点。
static std::chrono::time_point<std::chrono::system_clock> now() noexcept;
// 将 time_point 时间点类型转换为 std::time_t 类型
static std::time_t to_time_t( const time_point& t ) noexcept;
// 将 std::time_t 类型转换为 time_point 时间点类型
static std::chrono::system_clock::time_point from_time_t( std::time_t t ) noexcept;
```

想象 `std::chrono::system_clock` 是一个在墙上的钟表，如果我们想要获取当前时间，我们可以使用 `now()`成员函数：

```cpp
std::chrono::system_clock::time_point now = std::chrono::system_clock::now();
```

这就像是我们在钟表上捕捉到了一个时间点，这个点可以被保存，可以用来与其他时间点做比较，或者用作时间运算。

虽然`system_clock::time_point`表示一个时间点，但是我们通常需要更人性化的表达方式，比如年、月、日、小时、分钟和秒。这时候，我们可以像打开一个礼物盒一样，打开这个`time_point`，取出我们需要的信息：

```cpp
std::chrono::system_clock::time_point now = std::chrono::system_clock::now(); // 获取当前时间点
std::time_t tt = std::chrono::system_clock::to_time_t(now); // 将 time_point 时间点类型转换为 std::time_t 类型
std::tm* ptm = std::localtime(&tt);
std::cout << "Current time is: " << std::put_time(ptm,"%c") << std::endl;
```

其中，`std::localtime` 将 `std::time_t` 类型的时间（通常是从 1970 年 1 月 1 日开始的秒数）转换为当地时间的结构化时间 (`std::tm`)，这个结构体包含了时间的各个组成部分，例如年、月、日、时、分、秒等。且我们可以自行格式化时间输出（如 `std::put_time(ptm, "%Y-%m-%d %H:%M:%S")`）。

```cpp
int main() {
    // 获取当前的时间点
    std::chrono::system_clock::time_point now = std::chrono::system_clock::now();

    // 转换为时间戳并打印
    std::time_t now_c = std::chrono::system_clock::to_time_t(now);
    std::cout << "Current time: " << std::ctime(&now_c) << std::endl;

    return 0;
}
```

而 `std::ctime` 将 `std::time_t` 类型的时间直接格式化为一个字符串，表示当地时间，并返回指向字符串的指针，表示格式化后的时间。例如：`"Thu Jan 5 12:34:56 2025\n"`。

| 特性           | `localtime`                                        | `ctime`                          |
| -------------- | -------------------------------------------------- | -------------------------------- |
| **功能**       | 将时间转为结构化的本地时间 (`std::tm`)。           | 将时间直接转为格式化字符串。     |
| **返回值**     | 指向 `std::tm` 的指针。                            | 指向格式化时间字符串的指针。     |
| **线程安全性** | 非线程安全。                                       | 非线程安全。                     |
| **灵活性**     | 用户可以自行格式化时间输出（如 `std::put_time`）。 | 格式化固定，不能自定义输出格式。 |

示例：

```cpp
#include <chrono>
#include <iostream>
#include <iomanip> // 用于 std::put_time 格式化时间

using namespace std;
using namespace std::chrono;

int main() {
    // 新纪元1970.1.1时间
    system_clock::time_point epoch;

    duration<int, ratio<60 * 60 * 24>> day(1);
    // 新纪元1970.1.1时间 + 1天
    system_clock::time_point ppt(day);

    using dday = duration<int, ratio<60 * 60 * 24>>;
    // 新纪元1970.1.1时间 + 10天
    time_point<system_clock, dday> t(dday(10));

    // 系统当前时间
    system_clock::time_point today = system_clock::now();

    // 转换为 time_t 时间类型
    time_t tm = system_clock::to_time_t(today);
    time_t tm1 = system_clock::to_time_t(today + day);
    time_t tm2 = system_clock::to_time_t(epoch);
    time_t tm3 = system_clock::to_time_t(ppt);
    time_t tm4 = system_clock::to_time_t(t);

    // 使用 localtime 格式化输出
    cout << "今天的日期是:    " << put_time(localtime(&tm), "%Y-%m-%d %H:%M:%S") << endl;
    cout << "明天的日期是:    " << put_time(localtime(&tm1), "%Y-%m-%d %H:%M:%S") << endl;
    cout << "新纪元时间:      " << put_time(localtime(&tm2), "%Y-%m-%d %H:%M:%S") << endl;

    // 使用 ctime 输出
    cout << "新纪元时间+1天:  " << ctime(&tm3);
    cout << "新纪元时间+10天: " << ctime(&tm4);

    return 0;
}
```

输出结果为：

```yaml
今天的日期是:    2025-01-05 14:23:45
明天的日期是:    2025-01-06 14:23:45
新纪元时间:      1970-01-01 00:00:00
新纪元时间+1天:  Thu Jan  1 00:00:00 1970
新纪元时间+10天: Sun Jan 11 00:00:00 1970
```

## 3.2 steady_clock

如果我们通过时钟不是为了获取当前的系统时间，而是进行程序耗时的时长，比如 100m短跑裁判需要使用一个秒表记录运动员的记录，此时使用`syetem_clock`就不合适了，因为这个时间可以跟随系统的设置发生变化。在C++11中提供的时钟类`steady_clock`相当于秒表，只要启动就会进行时间的累加，并且不能被修改，非常适合于进行耗时的统计。

`steady_clock` 时钟类在底层源码中的定义如下：

```cpp
struct steady_clock { // wraps QueryPerformanceCounter
    using rep                       = long long;
    using period                    = nano;
    using duration                  = nanoseconds;
    using time_point                = chrono::time_point<steady_clock>;
    static constexpr bool is_steady = true;

    // get current time
    _NODISCARD static time_point now() noexcept 
    { 
        // doesn't change after system boot
        const long long _Freq = _Query_perf_frequency(); 
        const long long _Ctr  = _Query_perf_counter();
        static_assert(period::num == 1, "This assumes period::num == 1.");
        const long long _Whole = (_Ctr / _Freq) * period::den;
        const long long _Part  = (_Ctr % _Freq) * period::den / _Freq;
        return time_point(duration(_Whole + _Part));
    }
};
```

`std::chrono::steady_clock` 类同样提供了一个静态的`now()`方法，用于得到当前的时间点 `std::chrono::steady_clock::time_point`，函数原型如下：

```cpp
static std::chrono::time_point<std::chrono::steady_clock> now() noexcept;
```

使用如下：

```cpp
std::chrono::steady_clock::time_point start = std::chrono::steady_clock::now();
```

这就像裁判打出发令枪并按下计时器的开始按钮。

在运动员冲刺终点时，你可能会想知道实际的比赛时间。同样，在C++中，我们可以通过减去开始时间来得到一个`std::chrono::steady_clock::duration`，这个`duration`表示了一个时间段。看下面的代码：

```cpp
std::chrono::steady_clock::time_point start = std::chrono::steady_clock::now(); // 比赛开始
// 模拟运动员进行比赛
this_thread::sleep_for(chrono::seconds(10))
std::chrono::steady_clock::time_point end = std::chrono::steady_clock::now(); // 比赛结束
std::chrono::steady_clock::duration elapsed = end - start; // 比赛时间
```

`std::chrono::steady_clock::duration` 给出的默认时间单位可能并不是我们想要的。比如，默认可能给的是微秒级别的时间，但我们可能想要以秒为单位的时间。这时候，就需要转换时间单位。看下面的代码：

```cpp
long long elapsed_seconds = std::chrono::duration_cast<std::chrono::seconds>(elapsed).count();
```

示例：

```cpp
#include <chrono>
#include <iostream>
using namespace std;
using namespace std::chrono;
int main()
{
    // 获取开始时间点
    steady_clock::time_point start = steady_clock::now();
    // 执行业务流程
    this_thread::sleep_for(chrono::seconds(10))
    // 获取结束时间点
    steady_clock::time_point last = steady_clock::now();
    // 计算差值
    auto dt = last - start;
    cout << "总共耗时: " << dt.count() << "纳秒" << endl;
}
```

## 3.3 high_resolution_clock

`high_resolution_clock` 提供的时钟精度比`system_clock`要高，它也是不可以修改的。在底层源码中，这个类其实是`steady_clock`类的别名。使用步骤和 steady_clock 也类似：

```cpp
std::chrono::high_resolution_clock::time_point start = std::chrono::high_resolution_clock::now();
std::chrono::high_resolution_clock::time_point end = std::chrono::high_resolution_clock::now();
std::chrono::high_resolution_clock::duration elapsed = end - start;
```

和其他时钟一样，我们可能需要将`std::chrono::high_resolution_clock::duration`的时间单位进行转换，使其满足我们的需求。例如，我们可以将时间间隔转换为微秒，如下所示：

```cpp
long long elapsed_microseconds = std::chrono::duration_cast<std::chrono::microseconds>(elapsed).count();
```

# 4. duration_cast

我们在上面通过 `std::chrono::duration_cast<>` 将不同 `duration` 类型进行转换。而 `duration_cast` 其实是`chrono`库提供的一个模板函数，这个函数不属于`duration`类。通过这个函数可以对`duration`类对象内部的时钟周期`Period`，和周期次数的类型`Rep`进行修改，该函数原型如下：

```cpp
template <class ToDuration, class Rep, class Period>
  constexpr ToDuration duration_cast (const duration<Rep,Period>& dtn);
```

- **`ToDuration`**：目标 `duration` 类型。
- **`Rep` 和 `Period`**：源 `duration` 类型的模板参数。
- **返回值**：一个转换后的 `ToDuration` 类型的 `duration` 对象。

使用该函数时，需要注意以下要点：

1）**对时钟周期（`Period`）进行转换**：

- 如果转换涉及时钟周期（如从小时转为分钟），源周期必须能整除目标周期。
  - 例如：
    - 小时（`std::ratio<3600>`）能整除分钟（`std::ratio<60>`），可以直接转换。
    - 如果不能整除，则需要显式使用 `duration_cast`。
- 转换后，会自动调整时间值，以保持时间间隔的总量不变。

2）**对时钟周期次数的类型（`Rep`）进行转换**：

- 低精度类型（如 `int`）可以隐式转换为高精度类型（如 `double`）。
- 反之（如 `double` 转 `int`），需要显式使用 `duration_cast`，以避免数据丢失或精度问题。

3）**时钟周期和时钟周期次数类型同时变化**：

- 按照上述规则综合考虑：
  - 如果时钟周期变了，优先满足时钟周期整除关系。
  - 如果计数类型变了，则按类型提升规则处理。

4）**其他情况**：

- 如果转换过程中无法隐式完成（如周期不整除或类型不兼容），必须使用 `duration_cast`。

示例：

```cpp
#include <iostream>
#include <chrono>
using namespace std;
using namespace std::chrono;

int main() {
    // 定义一个以秒为单位的 duration 对象
    duration<int, ratio<1>> seconds(3600); // 3600 秒

    // 隐式转换：秒 -> 双精度秒（int -> double）
    duration<double> double_seconds = seconds; // 隐式转换
    cout << "Double Seconds: " << double_seconds.count() << endl; // 输出 3600.0

    // 隐式转换：int -> double 的类型提升
    duration<double, ratio<60>> double_minutes = seconds; // 隐式转换
    cout << "Double Minutes: " << double_minutes.count() << endl; // 输出 60.0

    // 显式转换：秒 -> 毫秒（周期不兼容，需使用 duration_cast）
    auto milliseconds = duration_cast<duration<int, milli>>(seconds);
    cout << "Milliseconds: " << milliseconds.count() << endl; // 输出 3600000

    // 显式转换：从 double 到 int（避免精度丢失）
    auto int_seconds = duration_cast<duration<int>>(double_seconds);
    cout << "Int Seconds: " << int_seconds.count() << endl; // 输出 3600

    return 0;
}
```

输出：

```yaml
Double Seconds: 3600.0
Double Minutes: 60.0
Milliseconds: 3600000
Int Seconds: 3600
```

# 5. time_point_cast

`time_point_cast`也是`chrono`库提供的一个模板函数，这个函数不属于`time_point`类。函数的作用是对时间点进行转换，因为不同的时间点对象内部的时钟周期`Period`，和周期次数的类型`Rep`可能也是不同的，一般情况下它们之间可以进行隐式类型转换，也可以通过该函数显示的进行转换，函数原型如下：

```cpp
template <class ToDuration, class Clock, class Duration>
time_point<Clock, ToDuration> time_point_cast(const time_point<Clock, Duration> &t);
```

示例：

```cpp
#include <chrono>
#include <iostream>
using namespace std;

using Clock = chrono::high_resolution_clock;
using Ms = chrono::milliseconds;
using Sec = chrono::seconds;
template<class Duration>
using TimePoint = chrono::time_point<Clock, Duration>;

void print_ms(const TimePoint<Ms>& time_point)
{
    std::cout << time_point.time_since_epoch().count() << " ms\n";
}

int main()
{
    TimePoint<Sec> time_point_sec(Sec(6));
    // 无精度损失, 可以进行隐式类型转换
    TimePoint<Ms> time_point_ms(time_point_sec);
    print_ms(time_point_ms);    // 6000 ms

    time_point_ms = TimePoint<Ms>(Ms(6789));
    // error，会损失精度，不允许进行隐式的类型转换
    TimePoint<Sec> sec(time_point_ms);

    // 显示类型转换,会损失精度。6789 truncated to 6000
    time_point_sec = std::chrono::time_point_cast<Sec>(time_point_ms);
    print_ms(time_point_sec); // 6000 ms
}
```

# 6. 应用

**1）网络通信**

在网络通信中，**超时控制**是一项非常重要的机制。例如，在建立连接或发送数据时，如果在规定时间内没有收到响应，就需要进行超时处理。这就要求我们能够精确地表示和计算时间。下面是一个简单的示例：

```cpp
#include <iostream>
#include <chrono>
#include <thread>

// 模拟尝试连接的函数，接受一个超时时间点作为参数
bool try_connect(std::chrono::system_clock::time_point deadline) {
    while (std::chrono::system_clock::now() < deadline) { // 检查是否超时
        // 模拟连接尝试
        bool success = false; // 示例中假设未成功，实际应用中需要调用具体连接函数
        if (success) {
            return true; // 如果连接成功，返回 true
        }

        // 等待一段时间后再次尝试
        std::this_thread::sleep_for(std::chrono::seconds(1));
    }

    // 如果超过超时时间仍未成功，则返回 false
    return false;
}

int main() {
    // 设置超时时间为当前时间加 10 秒
    auto deadline = std::chrono::system_clock::now() + std::chrono::seconds(10);
    
    // 在 10 秒内尝试建立连接
    if (try_connect(deadline)) {
        std::cout << "Successfully connected within the time limit.\n";
    } else {
        std::cout << "Failed to connect within 10 seconds.\n";
    }

    return 0;
}
```

在这个代码中，我们定义了一个超时时间点 `deadline`，表示连接操作的最晚时间。然后，通过循环不断尝试连接。如果在超时时间内建立了连接，就返回 `true`；如果超过时间仍未成功，则返回 `false`。

**2）事件调度**

在很多情况下，我们需要在**某个具体的时间点**触发一个任务或事件。为了实现这种功能，我们需要能够精确表示和计算时间点。下面是一个简单的示例代码：

```cpp
#include <iostream>
#include <chrono>
#include <thread>

// 安排并在指定时间点执行事件的函数
void schedule_event(std::chrono::system_clock::time_point event_time) {
    // 获取当前时间
    auto now = std::chrono::system_clock::now();

    // 如果事件时间晚于当前时间，计算需要等待的时间
    if (event_time > now) {
        auto wait_time = event_time - now; // 计算等待时间
        std::this_thread::sleep_for(wait_time); // 等待指定的时间
    }

    // 执行事件
    std::cout << "Event executed at " 
              << std::chrono::system_clock::to_time_t(std::chrono::system_clock::now()) 
              << std::endl;
}

int main() {
    // 计划在 5 秒后执行事件
    auto event_time = std::chrono::system_clock::now() + std::chrono::seconds(5);
    schedule_event(event_time);

    return 0;
}
```

- **设定事件时间**：在 `main` 函数中，我们通过 `std::chrono::system_clock::now() + std::chrono::seconds(5)` 计算出事件触发时间点，即当前时间延后 5 秒。
- **等待到时间点**：在 `schedule_event` 函数中，首先获取当前时间，然后判断是否需要等待。如果 `event_time` 在未来，则计算需要等待的时间，并调用 `std::this_thread::sleep_for()` 进行等待。
- **触发事件**：当等待结束后，打印出事件执行的时间。
