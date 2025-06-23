---
title: ArkTs与C++之间的类型转换
date: 2025-06-23 16:27:03
categories:
- ArkTS
tags: 
- ohos
typora-root-url: ./..
---

[toc]

# ArkTS与C++的跨语言交互

Ohos 上的应用一般都是用 arkts 语言编写的，而 arkts 语言是无法直接调用 C/C++ 接口的，如果我们的应用需要调用 C/C++ 第三方库，需要在 arkts 和 C/C++ 之间建立一个可以互通的桥梁，Node-API 正是搭建这个桥梁的主要关键。

HarmonyOS 提供了 Node-API 框架用于实现 ArkTS/JS 与 C/C++ 模块之间的跨语言交互，Node-API规范封装了I/O、CPU密集型、OS底层等能力并对外暴露 ArkTS/JS 接口，从而实现ArkTS/JS和C/C++的交互。主要应用场景如下：

- 系统可以将框架层丰富的模块功能通过ArkTS/JS接口开放给上层应用。
- 应用开发者也可以选择将一些对性能、底层系统调用有要求的核心功能用C/C++封装实现，再通过ArkTS/JS接口使用，提高应用本身的执行效率。

# 架构组成

![img](/images/$%7Bfiilename%7D/0000000000011111111.20240828155403.22037363316058508197625782091681500012310000002800753446C7440769FE0DE886BE4BB1AD78C702731E21B71332542F59F12F2BD5DE.png)

如上图所示，整体架构分为 C++、ArkTS和工具链三部分：

- C++：包含各种文件的引用、C++或者C代码、Native项目必需的配置文件等。
- ArkTS：包含界面UI、自身方法、调用引用包的方法等。
- 工具链：包含CMake编译工具在内的系列工具

C++ 部分中，`.cpp` 文件实现 C++ 方法，并通过`NAPI`将C++方法与ArkTS方法关联，C++代码通过CMake编译工具编译成动态链接库so文件，使用`index.d.ts`文件对外提供接口，ArkTS引入`.so`文件后调用其中的接口，即可实现 ArkTS 对 C++ 对跨语言调用。

# 代码结构

```
├──entry/src/main
│  ├──common      
│  │  └──CommonContants.ets            // 常量定义文件
│  ├──cpp                              // C++代码区
│  │  ├──CMakeLists.txt                // CMake编译配置文件
│  │  ├──hello.cpp                     // C++源代码
│  │  ├──common
│  │  │  └──common.h                   // C++常量文件
│  │  └──types                         // 接口存放文件夹
│  │     └──libhello
│  │        ├──index.d.ts              // 接口文件
│  │        └──oh-package.json5        // 接口注册配置文件
│  └──ets                              // 代码区
│     ├──entryability
│     │  └──EntryAbility.ets           // 程序入口类
│     └──pages
│        └──Index.ets                  // 主界面
└──entry/src/main/resources            // 资源文件目录
```

我们需要在 `hello.cpp` 中通过 `node-API` 实现接口函数，并在 `index.d.ts`  中声明相应的接口函数，最后编译生成`lbhello.so` ，在 `Index.ets` 中导入 `import lbhello.so` 进行调用。

# C++返回ArkTS的自定义类型

假如定义如下 ArkTS 类型：

```TypeScript
export interface weChatDailyType {
  img: string // 头像
  name: string // 名称
  content: string // 聊天内容
  bottom: string // 背景颜色
  time: string // 上次聊天时间
}
```

我们希望 ArkTS 调用 C++ 接口函数返回类型为 `weChatDailyType[]` 的数组，以供内部需求。

首先，在文件 `Index.d.ts` 中声明接口：

```TypeScript
// entry/src/main/cpp/types/libentry/index.d.ts
export const returnContacts: () => weChatDailyType[];
```

然后，在文件 `napi_init.cpp` 中配置模块描述信息,设置 Init 方法为 napi_module 的入口方法。`__attribute__((constructor))` 修饰的方法由系统自动调用，使用 NAPI 接口 `napi_module_register()` 传入模块描述信息进行模块注册，属性`.nm_register_func`、`.nm_modname` 分别是模块初始化函数和模块名称，分别用于将我们写的 ArkTS 接口与具体的 C++ 实现进行绑定和映射，以及用作 ArkTS 侧引入的 so 库的名称，模块系统会根据后者的名称来区分不同的 so。

```cpp
// entry/src/main/cpp/napi_init.cpp

// 准备模块加载相关信息，将上述Init函数与本模块名等信息记录下来。
static napi_module nativeModule = {
    .nm_version = 1,
    .nm_flags = 0,
    .nm_filename = nullptr,
    .nm_register_func = Init,
    .nm_modname = "entry",
    .nm_priv = ((void *)0),
    .reserved = {0},
};

// 加载so时，该函数会自动被调用，将上述demoModule模块注册到系统中。
extern "C" __attribute__((constructor)) void RegisterEntryModule(void) { napi_module_register(&nativeModule); }

```

注：以上代码无须复制，创建Native C++工程以后在 `napi_init.cpp` 代码中已配置好。

`Init()` 函数用于实现 ArkTS 接口与 C++ 实现函数的绑定与映射，`napi_env env` 用于提供 ArkTS 运行时的上下文，`api_value exports` 作为目标对象，接收通过 `napi_property_descriptor` 定义的属性，最后返回 `exports`，确保初始化后的导出对象被正确设置.

`napi_property_descriptor` 定义了 C++ 暴露给 ArkTS 的属性和方法，在下面的示例代码中：

- `returnContacts`：暴露了一个名为 `returnContacts` 的方法，对应的 C++ 实现是`ReturnContacts` 函数

```cpp
// entry/src/main/cpp/napi_init.cpp
EXTERN_C_START
static napi_value Init(napi_env env, napi_value exports) {
    napi_property_descriptor desc[] = {
        {"returnContacts", nullptr, ReturnContacts, nullptr, nullptr, nullptr, napi_default, nullptr},
    napi_define_properties(env, exports, sizeof(desc) / sizeof(desc[0]), desc);
    return exports;
}
EXTERN_C_END
```

`returnContacts `接口已经在文件 `Index.d.ts` 声明，我们需要实现该接口映射的 C++ 实现 `ReturnContacts`:

```cpp
const std::vector<weChatDailyType> contacts = {
    {"http://yjy-teach-oss.oss-cn-beijing.aliyuncs.com/HeimaCloudMusic/1.jpg", "林小宇", "会议地点在3楼会议室",
     "#EBEBEB", "上午 10:25"},
    {"http://yjy-teach-oss.oss-cn-beijing.aliyuncs.com/HeimaCloudMusic/2.jpg", "王晓峰", "明天上午进行代码 review",
     "#EBEBEB", "昨天 19:47"},
    {"http://yjy-teach-oss.oss-cn-beijing.aliyuncs.com/HeimaCloudMusic/3.jpg", "陈思远", "打完球一起吃饭", "#EBEBEB",
     "周五 15:30"},
    {"http://yjy-teach-oss.oss-cn-beijing.aliyuncs.com/HeimaCloudMusic/4.jpg", "张雨欣", "是一个包裹和一封信",
     "#FFFFFF", "周四 11:20"},
    {"http://yjy-teach-oss.oss-cn-beijing.aliyuncs.com/HeimaCloudMusic/5.jpg", "李浩然", "新的会议议程已发群里",
     "#FFFFFF", "周三 16:15"},
    {"http://yjy-teach-oss.oss-cn-beijing.aliyuncs.com/HeimaCloudMusic/6.jpg", "赵雨晴", "记得查收一下微信红包",
     "#FFFFFF", "周二 09:05"},
    {"http://yjy-teach-oss.oss-cn-beijing.aliyuncs.com/HeimaCloudMusic/7.jpg", "吴思成", "有任何问题随时联系我",
     "#FFFFFF", "周一 18:40"},
    {"http://yjy-teach-oss.oss-cn-beijing.aliyuncs.com/HeimaCloudMusic/8.jpg", "郑雅琪", "需要我提前预定会议室吗",
     "#FFFFFF", "上周日 14:30"},
    {"http://yjy-teach-oss.oss-cn-beijing.aliyuncs.com/HeimaCloudMusic/9.jpg", "孙天宇", "可以给你一些参考意见？",
     "#FFFFFF", "上周六 10:15"},
    {"http://yjy-teach-oss.oss-cn-beijing.aliyuncs.com/HeimaCloudMusic/10.jpg", "周嘉怡", "我买好票叫你", "#FFFFFF",
     "上周四 21:35"},
    {"http://yjy-teach-oss.oss-cn-beijing.aliyuncs.com/HeimaCloudMusic/11.jpg", "黄俊杰", "有新的需求随时告诉我",
     "#FFFFFF", "上周三 08:50"}};

napi_value loadContactsFromDatabase(napi_env env, napi_callback_info info) {
    napi_value resultArray;
    napi_create_array(env, &resultArray);

    // 将每个元素转为 napi_value 并添加到 resultArray 中
    for (std::size_t i = 0; i < contacts.size(); ++i) {
        napi_value contactObject;
        napi_create_object(env, &contactObject);

        napi_value imgKey, nameKey, contentKey, bottomKey, timeKey;
        napi_create_string_utf8(env, "img", NAPI_AUTO_LENGTH, &imgKey);
        napi_create_string_utf8(env, "name", NAPI_AUTO_LENGTH, &nameKey);
        napi_create_string_utf8(env, "content", NAPI_AUTO_LENGTH, &contentKey);
        napi_create_string_utf8(env, "bottom", NAPI_AUTO_LENGTH, &bottomKey);
        napi_create_string_utf8(env, "time", NAPI_AUTO_LENGTH, &timeKey);


        napi_value imgValue, nameValue, contentValue, bottomValue, timeValue;
        napi_create_string_utf8(env, contacts[i].img.c_str(), contacts[i].img.size(), &imgValue);
        napi_create_string_utf8(env, contacts[i].name.c_str(), contacts[i].name.size(), &nameValue);
        napi_create_string_utf8(env, contacts[i].content.c_str(), contacts[i].content.size(), &contentValue);
        napi_create_string_utf8(env, contacts[i].bottom.c_str(), contacts[i].bottom.size(), &bottomValue);
        napi_create_string_utf8(env, contacts[i].time.c_str(), contacts[i].time.size(), &timeValue);

        napi_set_property(env, contactObject, imgKey, imgValue);
        napi_set_property(env, contactObject, nameKey, nameValue);
        napi_set_property(env, contactObject, contentKey, contentValue);
        napi_set_property(env, contactObject, bottomKey, bottomValue);
        napi_set_property(env, contactObject, timeKey, timeValue);

        napi_set_element(env, resultArray, i, contactObject);
    }

    return resultArray;
}

static napi_value ReturnContacts(napi_env env, napi_callback_info info) { return loadContactsFromDatabase(env, info); }
```

函数主要功能是在 C++ 构造一个在 ArkTS 中可直接使用的 weChatDailyType 数组对象。对应 API 的介绍参考链接：[>>>Node-API使用方法<<<](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/use-napi-about-array)

在接口配置文件 `oh-package.json5 `中将 `index.d.ts `与CMake编译的so文件关联起来。模块级目录下`oh-package.json5`文件添加so文件依赖。

```json
// index.d.ts
export const returnContacts: () => weChatDailyType[];

// oh-package.json5
{
  "name": "libentry.so",
  "types": "./Index.d.ts",
  "version": "1.0.0",
  "description": "Please describe the basic information."
}

// entry/oh-package.json5
{
  "name": "entry",
  "version": "1.0.0",
  "description": "Please describe the basic information.",
  "main": "",
  "author": "",
  "license": "",
  "dependencies": {
    "libentry.so": "file:./src/main/cpp/types/libentry"
  }
}
```

上面所述流程完成后，编译，我们即可在 ArkTS 进行调用：

```typescript
import libentry from 'libentry.so'
init() {
    this.currentContact.contactList = libentry.returnContacts();
}
```

`libentry.so` 是 cpp 编译后的 so 文件名。

[>>>实现使用NAPI调用C标准库的功能<<<](https://developer.huawei.com/consumer/cn/codelabsPortal/carddetails/tutorials_NEXT-NativeTemplateDemo)
[>>>使用Node-API实现跨语言交互<<<](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/using-napi-interaction-with-cpp)

# C++解析ArkTS的类型

## 基本类型

NAPI 提供了相应的接口将 ArkTS 的基本类型转换为 C++ 的基本类型：

- `napi_get_value_uint32`：将ArkTS环境中number类型数据转为Node-API模块中的uint32类型数据。
- `napi_get_value_int32`：将ArkTS环境中获取的number类型数据转为Node-API模块中的int32类型数据。
- `napi_get_value_int64`：将ArkTS环境中获取的number类型数据转为Node-API模块中的int64类型数据。
- `napi_get_value_double`：将ArkTS环境中获取的number类型数据转为Node-API模块中的double类型数据。
- `napi_create_int32`：将Node-API模块中的int32_t类型转换为ArkTS环境中number类型。
- `napi_create_uint32`：将Node-API模块中的uint32_t类型转换为ArkTS环境中number类型。
- `napi_create_int64`：将Node-API模块中的int64_t类型转换为ArkTS环境中number类型。
- `napi_create_double`：将Node-API模块中的double类型转换为ArkTS环境中number类型。

示例，解析从 ArkTS 获取的 uint32 类型值：

```cpp
#include "napi/native_api.h"

static napi_value GetValueUint32(napi_env env, napi_callback_info info)
{
    // 获取传入的数字类型参数
    size_t argc = 1;
    napi_value argv[1] = {nullptr};
    // 解析传入的参数
    napi_get_cb_info(env, info, &argc, argv, nullptr, nullptr);

    uint32_t number = 0;
    // 获取传入参数的值中的无符号32位整数
    napi_status status = napi_get_value_uint32(env, argv[0], &number);
    // 如果传递的参数不是数字,将会返回napi_number_expected，设置函数返回nullptr
    if (status == napi_number_expected) {
        return nullptr;
    }
    napi_value result = nullptr;
    // 创建传入参数无符号32位整数，并传出
    napi_create_uint32(env, number, &result);
    return result;
}
```

[>>>使用Node-API接口创建基本数据类型<<<](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/use-napi-basic-data-types)

## string 字符串

NAPI 提供了相应的接口将 ArkTS 的字符串类型转换为 C++ 的字符串类型：

- napi_get_value_string_utf8: 需要将ArkTS的字符类型的数据转换为utf8编码的字符时使用这个函数。
- napi_create_string_utf8: 需要通过UTF8编码的C字符串创建ArkTS string值时使用这个函数。
- napi_get_value_string_utf16: 需要将ArkTS的字符类型的数据转换为utf16编码的字符时使用这个函数。
- napi_create_string_utf16: 需要通过UTF16编码的C字符串创建ArkTS string值时使用这个函数。
- napi_get_value_string_latin1: 需要将ArkTS的字符类型的数据转换为ISO-8859-1编码的字符时使用这个函数。
- napi_create_string_latin1: 需要通过ISO-8859-1编码的字符串创建ArkTS string值时使用这个函数。

对字符串对象的处理相比基本类型稍微复杂些，需要通过 napi_get_value_string_utf8 将 ArkTS 的字符串对象转换为 C++ 的 std::string `对象，但需要传入字符串的长度，napi_get_value_string_utf8` 的原型如下：

```cpp
napi_status napi_get_value_string_utf8(napi_env env,
                                       napi_value value,
                                       char* buf,
                                       size_t bufsize,
                                       size_t* result)

```

- napi_value value：要转换的 ArkTS 字符串值
- char* buf：目标缓冲区，用于存储转换后的 UTF-8 字符串
- size_t bufsize：目标缓冲区的大小（以字节为单位）
- size_t* result：可选参数，用于存储转换后的字符串长度（不包括终止符）

函数返回 `napi_status` 枚举类型，参考：[napi_status 类型说明](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/napi-data-types-interfaces#napi_status)

我们需要先创建一个 char 类型的空间去接收转换后的字符串，创建 char 数组需要指定大小，可以先调用一次`napi_get_value_string_utf8`，buf传入空获取到传入的数据大小，然后创建对应大小buf，再次调用`napi_get_value_string_utf8`获取转换后的字符串，示例如下：

```cpp
#include "napi/native_api.h"
#include <cstring>

static napi_value GetValueStringUtf8(napi_env env, napi_callback_info info) 
{
    size_t argc = 1;
    napi_value args[1] = {nullptr};

    napi_get_cb_info(env, info, &argc, args, nullptr, nullptr);
    // 获取字符串的长度
    size_t length = 0;
    napi_status status = napi_get_value_string_utf8(env, args[0], nullptr, 0, &length);
    // 传入一个非字符串 napi_get_value_string_utf8接口会返回napi_string_expected
    if (status != napi_ok) {
        return nullptr;
    }
    // 创建 char 数组用于接收字符串
    char* buf = new char[length + 1];
    std::memset(buf, 0, length + 1);
    napi_get_value_string_utf8(env, args[0], buf, length + 1, &length);
    napi_value result = nullptr;
    status = napi_create_string_utf8(env, buf, length, &result);
    delete buf;
    if (status != napi_ok) {
        return nullptr;
    };
    return result;
}
```

## object 类型

解析object对象参数和前面参数一样，通过 `napi_get_cb_info` 转换为 `napi_value` 对象后，通过 `napi_get_named_property` 获取对象中的属性值：

```cpp
//arkts对象：
export class Task {  
  public id: number; //unique task identify  
  public channel?: number;   
  //...
}

napi_value Demo::startTask(napi_env env, napi_callback_info info){
    size_t argc = 1;
    napi_value js_cb;
    napi_get_cb_info(env, info, &argc, &js_cb, nullptr, nullptr);

    napi_value taskIdNapiValue;
    napi_get_named_property(env, js_cb, "id", &taskIdNapiValue);
    int32_t taskid;
    napi_get_value_int32(env, taskIdNapiValue, &taskid);

    napi_value channelSelectNapiValue;
    napi_get_named_property(env, js_cb, "channelSelect", &channelSelectNapiValue);
    int32_t channel_select;
    napi_get_value_int32(env, channelSelectNapiValue, &channel_select);

	//....

	//调用业务逻辑
    return nullptr;
}
```

## 数组类型

和 string 类似，数组类型参数需要先通过 `napi_get_array_length` 参数获取数组长度，再通过 `napi_get_element` 获取对应每个 `napi_value` 对象，通过上述基本类型转换为对应C++对象：

```cpp
napi_value Demo::setArrays(napi_env env, napi_callback_info info){
	std::vector<std::string> backupip_list;

    size_t argc = 1;
    napi_value args[1] = {nullptr};
    napi_get_cb_info(env, info, &argc, args, nullptr, nullptr);
    bool is_array;
    //判断是否为数组
    napi_status status = napi_is_array(env, args[0], &is_array);
    if (!is_array) {
        return nullptr;
    }
    napi_value array = args[0];
    // 获取数组长度
    uint32_t length;
    napi_get_array_length(env,array,&length);
    for(int i = 0; i < length; i++){
        napi_value element;
        napi_get_element(env, array, i, &element); // 获取数组的每个元素
        std::string ipStr;
        NapiUtil::JsValueToString(env, element, ipStr);
        backupip_list.push_back(ipStr);
    }   
    //调用业务逻辑
    return nullptr;
}
```

## ArrayBuffer 类型

如果ArkTS向C++传输二进制流，需要用到 `ArrayBuffer` 类型接收，在C++侧通过 `napi_get_arraybuffer_info` 转换成C++字节流，接口说明：

```cpp
napi_status napi_get_arraybuffer_info(napi_env env,
                                      napi_value arraybuffer,
                                      void** data,
                                      size_t* byte_length)
```

- napi_value arraybuffer：要查询的 ArkTS ArrayBuffer 对象
- void** data：输出参数，指向 ArrayBuffer 底层数据的指针
- size_t* byte_length：输出参数，指向 ArrayBuffer 的字节长度

第三个参数传入空时，只会获取字节流大小。第三个参数是指向指针的指针，我们不需要创建空间，函数内部会创建空间：

```cpp
napi_value Demo::setArrayBufferData(napi_env env, napi_callback_info info){
    size_t argc = 1;
    napi_value js_cb;
    napi_get_cb_info(env, info, &argc, &js_cb, nullptr, nullptr);

    // 获取 ArrayBuffer 对象的指针和长度 
	void* buffer; 
	size_t length; 
	napi_get_arraybuffer_info(env, arrayBuffer, &buffer, &length); 
	// 打印 ArrayBuffer 中的数据 ，也可以修改ArrayBuffer的值
	uint32_t* data = (uint32_t*) buffer;

	//调用业务逻辑
    return nullptr;
}
```

## 综合案例

最后附上一个综合案例，包含大部分的类型转换：
[>>>北向应用集成三方库——应用如何调用C/C++三方库<<<](https://cloud.tencent.com/developer/article/2447589?policyId=1004)
