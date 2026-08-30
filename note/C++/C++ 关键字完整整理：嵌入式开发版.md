
> 适用场景：STM32、FreeRTOS、嵌入式 Linux、驱动开发；面向底层硬件、资源受限 MCU 环境
## 一、先建立一个总体认识

C++ 中涉及的 "特殊词" 可以先分成四类，不要全部混为传统保留关键字：

```
C++ 特殊词
│
├── ① 保留关键字 Keywords
│      ├── int
│      ├── class
│      ├── if
│      ├── template
│      └── ...
│
├── ② 替代标记 Alternative Tokens
│      ├── and
│      ├── or
│      ├── not
│      └── ...
│
├── ③ 特殊标识符 Special Identifiers
│      ├── override
│      ├── final
│      ├── import
│      └── module
│
└── ④ 预处理器关键字/指令
       ├── #if
       ├── #define
       ├── #include
       ├── #ifdef
       └── ...
```

> 💡 提示：看到网上的 C++ 关键字表，注意区分上面 4 个分类，不是全部都是真正的保留关键字。
## 二、基本数据类型

表格

| 关键字        | 作用          | 标准版本  |
| ---------- | ----------- | ----- |
| `bool`     | 布尔类型        | C++98 |
| `char`     | 字符类型        | C++98 |
| `wchar_t`  | 宽字符类型       | C++98 |
| `char16_t` | UTF‑16 字符类型 | C++11 |
| `char32_t` | UTF‑32 字符类型 | C++11 |
| `char8_t`  | UTF‑8 字符类型  | C++20 |
| `short`    | 短整型         | C++98 |
| `int`      | 整型          | C++98 |
| `long`     | 长整型         | C++98 |
| `signed`   | 有符号类型修饰     | C++98 |
| `unsigned` | 无符号类型修饰     | C++98 |
| `float`    | 单精度浮点       | C++98 |
| `double`   | 双精度浮点       | C++98 |
| `void`     | 无类型         | C++98 |

### 嵌入式重点代码示例

```cpp
uint32_t reg;
uint16_t adc;
uint8_t data;
bool enable;
```

> ⚠重要提醒：`uint8_t`、`uint16_t`、`uint32_t`**不是 C++ 关键字**，来自头文件 `<cstdint>` / `<stdint.h>` 的 typedef 类型别名。

## 三、流程控制

表格

|关键字|作用|标准版本|
|---|---|---|
|`if`|条件判断|C++98|
|`else`|条件分支 - 否则|C++98|
|`switch`|多分支判断|C++98|
|`case`|switch 分支标签|C++98|
|`default`|switch 默认分支|C++98|
|`for`|循环|C++98|
|`while`|while 循环|C++98|
|`do`|do‑while 循环|C++98|
|`break`|跳出循环 /switch|C++98|
|`continue`|直接进入下一轮循环|C++98|
|`goto`|无条件跳转|C++98|
|`return`|函数返回|C++98|

### C++11 范围 for 循环（range‑based for）

```cpp
int array[] = {1,2,3,4};
for (auto value : array)
{
    // 遍历每一个元素
}
```

## 四、面向对象

表格

|关键字|作用|标准版本|
|---|---|---|
|`class`|定义类|C++98|
|`struct`|定义结构体|C++98|
|`public`|公有访问权限|C++98|
|`private`|私有访问权限|C++98|
|`protected`|保护访问权限|C++98|
|`this`|当前对象指针|C++98|
|`friend`|友元|C++98|
|`virtual`|虚函数，实现多态|C++98|
|`explicit`|禁止隐式类型转换|C++98|
|`mutable`|成员绕过 const 限制|C++98|
|`operator`|运算符重载|C++98|

> `override`、`final`：**特殊标识符，不是传统保留关键字 (C++11)** ⭐⭐⭐⭐⭐嵌入式必掌握

### override 强制校验重写虚函数

```cpp
class Base
{
public:
    virtual void run();
};
class Derived : public Base
{
public:
    void run() override; // 编译器强制检查基类是否存在该虚函数
};
```

### final 禁止重写 / 禁止继承

```
// 禁止子类重写该虚函数
class Base
{
public:
    virtual void run() final;
};

// final修饰类，禁止被继承
class Motor final
{
};
```

## 五、动态内存与对象生命周期

表格

|关键字|作用|
|---|---|
|`new`|创建对象、分配堆内存|
|`delete`|销毁对象、释放堆内存|
|`sizeof`|获取类型 / 对象占用字节大小|

```
// 单个对象
int *p = new int(10);
delete p;

// 数组对象
int *p_arr = new int[10];
delete [] p_arr;
```

> ✅匹配规则：`new` ↔ `delete`；`new[]` ↔ `delete[]`，不能混用。

> 💡现代 C++ 嵌入式推荐优先智能指针，减少裸 new/delete： `std::unique_ptr`、`std::shared_ptr`、`std::make_unique`、`std::make_shared`

## 六、类型转换

C++ 四种强制转换（嵌入式重点关注`reinterpret_cast`寄存器地址映射）

表格

|关键字|主要用途|风险等级|
|---|---|---|
|`static_cast`|常规安全类型转换|⭐|
|`dynamic_cast`|多态运行时类型转换|⭐⭐|
|`const_cast`|修改 const/volatile 限定符|⭐⭐⭐|
|`reinterpret_cast`|底层二进制重新解释，硬件寄存器操作高频|⭐⭐⭐⭐⭐|

### static_cast 常规转换

```
int a = 10;
double b = static_cast<double>(a);
```

### dynamic_cast 多态向下转型

```
Base *p = nullptr;
Derived *d = dynamic_cast<Derived *>(p);
```

### const_cast 去掉 const 修饰

```
const int *p;
int *q = const_cast<int *>(p);
```

> ⚠注意：如果原始对象本身就是 const 常量，修改后属于**未定义行为**。

### reinterpret_cast 嵌入式硬件高频用法（寄存器映射）

```
uint32_t addr = 0x40000000;
volatile uint32_t *reg = reinterpret_cast<volatile uint32_t *>(addr);
```

## 七、函数相关

表格

|关键字|作用|标准版本|
|---|---|---|
|`inline`|内联函数提示|C++98|
|`const`|常量、只读限定|C++98|
|`noexcept`|声明函数不会抛出异常|C++11|
|`throw`|抛出异常|C++98|
|`try`|异常捕获块|C++98|
|`catch`|捕获异常|C++98|

### const 多重用法（嵌入式高频）

1. 普通常量

```
const int a = 10;
```

2. const 修饰指针

```
const int *p; // 指针指向内容不可修改
```

3. const 修饰成员函数

```
class A
{
public:
    int get() const; // 函数内部不能修改普通成员变量
};
```

## 八、命名空间、类型别名、链接属性

表格

|关键字|作用|标准版本|
|---|---|---|
|`namespace`|命名空间，防止命名冲突|C++98|
|`using`|引入命名空间、类型别名|C++98|
|`typedef`|旧版类型别名|C++98|
|`extern`|外部链接声明|C++98|
|`static`|静态变量、内部链接、静态类成员|C++98|
|`auto`|自动类型推导|C++11|
|`decltype`|获取表达式的类型|C++11|
|`typename`|模板中标记类型|C++98|

> 💡现代 C++ 优先使用`using`做类型别名，`typedef`大量存在于老工程、Linux 驱动内核代码，必须看得懂。

```
using uint = unsigned int;   // 现代写法
typedef unsigned int uint;   // 老式写法
```

## 九、模板

表格

|关键字|作用|标准版本|
|---|---|---|
|`template`|定义模板|C++98|
|`typename`|模板参数标记类型|C++98|
|`concept`|模板概念约束|C++20|
|`requires`|模板约束条件|C++20|

基础模板示例

```
template <typename T>
T add(T a, T b)
{
    return a + b;
}
```

C++20 concept requires

```
template <typename T>
requires SomeConstraint<T>
void foo(T value)
{
}
```

## 十、常量与编译期

现代 C++ 编译期计算，嵌入式非常适合做静态断言校验硬件参数

表格

|关键字|作用|标准版本|
|---|---|---|
|`const`|只读限定|C++98|
|`constexpr`|编译期常量表达式|C++11|
|`consteval`|强制编译期执行函数|C++20|
|`constinit`|强制静态初始化|C++20|
|`static_assert`|编译期断言|C++11|

### constexpr 编译期函数

```
constexpr int square(int x)
{
    return x * x;
}
constexpr int a = square(10);
```

### consteval 必须编译期求值

```
consteval int square(int x)
{
    return x * x;
}
```

### static_assert 嵌入式硬件校验神器

```
static_assert(sizeof(int) == 4);                // 校验int字节大小
static_assert(sizeof(PacketHeader) == 8);       // 校验协议结构体大小
```

## 十一、枚举、结构体、联合体

表格

|关键字|作用|
|---|---|
|`enum`|C 风格枚举|
|`struct`|结构体|
|`union`|联合体|
|`class`|类|

C++11 强类型枚举（嵌入式状态机首选）

```
enum class MotorState
{
    STOP,
    RUN,
    ERROR
};

MotorState state = MotorState::STOP;
```

## 十二、RTTI 运行时类型信息

表格

|关键字|作用|
|---|---|
|`typeid`|获取运行时类型信息|
|`dynamic_cast`|运行时多态转换|

```
if (typeid(obj) == typeid(MyClass))
{
}
```

> 💡MCU 嵌入式工程通常编译选项关闭 RTTI：`-fno‑rtti`，了解即可，不建议大量使用。

## 十三、线程局部存储 C++11

`thread_local`：每个线程拥有独立变量实例

```
thread_local int counter;
```

## 十四、内存对齐 C++11

`alignas` 指定对齐；`alignof` 查询对齐；DMA、Cache、硬件协议开发高频

```
// 指定结构体32字节对齐
struct alignas(32) Buffer
{
    uint8_t data[64];
};

// 查询类型对齐字节
constexpr size_t a = alignof(Buffer);
```

## 十五、C++20 新增关键字

表格

|关键字|功能|
|---|---|
|`char8_t`|UTF‑8 字符类型|
|`concept`|模板概念约束|
|`consteval`|立即函数，强制编译期执行|
|`constinit`|强制静态初始化|
|`co_await`|协程等待|
|`co_return`|协程返回|
|`co_yield`|协程产出值|
|`requires`|模板约束条件|

分组记忆：

1. 字符：`char8_t`
2. 编译期：`consteval`、`constinit`
3. Concepts：`concept`、`requires`
4. 协程 Coroutine：`co_await`、`co_return`、`co_yield`

## 十六、C++20 Modules 模块

> `module` / `import` / `export` 属于特殊标识符，不等同传统关键字，替代`#include`机制

```
export module my_module;   // 导出模块

import my_module;          // 导入模块
```

## 十七、Alternative Tokens 替代标记

等价于符号，工程一般直接写符号，很少使用单词形式

表格

|替代标记|等价运算符|
|---|---|
|`and`|`&&`|
|`or`|`\|`|
|`not`|`!`|
|`bitand`|`&`|
|`bitor`|`\|`|
|`xor`|`^`|
|`compl`|`~`|
|`and_eq`|`&=`|
|`or_eq`|`\|=`|
|`xor_eq`|`^=`|
|`not_eq`|`!=`|

示例：

```
if (a > 0 and b > 0) {}
//等价
if (a > 0 && b > 0) {}
```

## 十八、嵌入式重中之重：volatile

```
volatile uint32_t reg;
```

作用：告诉编译器**不要对该变量做优化**，用于硬件寄存器、中断标记、DMA 映射内存。

```
volatile uint32_t *GPIO_REG = reinterpret_cast<volatile uint32_t *>(0x40020000);
```

> ⚠重大误区：**volatile != 线程安全、不等于原子操作** `volatile int counter; counter++;` 在中断 / 多线程环境依然会出现竞态问题。FreeRTOS、多核并发要注意这点。

## 十九、static：多语义关键字

1. **局部静态变量**：函数内，生命周期全局，只初始化一次

```
void foo()
{
    static int count;
}
```

2. **全局 static 变量**：内部链接，仅当前`.cpp`翻译单元可见，其他文件访问不到

```
static int value;
```

3. **类 static 成员**：属于类本身，不属于对象实例，所有对象共享同一份

```
class A
{
public:
    static int count;
};
```

## 二十、extern 外部链接

声明变量 / 函数定义在别的翻译单元，用于跨文件共享全局变量

```
// file1.cpp
int g_value = 10;

// file2.cpp
extern int g_value; // 告诉编译器这个变量定义在别处
```

> 和 C 语言的声明、定义、内部链接、外部链接、翻译单元概念完全打通。

## 二十一、容易混淆的特殊标识符汇总

表格

|名称|类型|C++ 版本|
|---|---|---|
|`override`|特殊标识符|C++11|
|`final`|特殊标识符|C++11|
|`nullptr`|关键字|C++11|
|`auto`|关键字，语义更新|C++11|
|`constexpr`|关键字|C++11|
|`concept`|关键字|C++20|
|`requires`|关键字|C++20|
|`import`|模块特殊标识符|C++20|
|`module`|模块特殊标识符|C++20|

## 二十二、嵌入式 C++ 关键字学习优先级

### ⭐⭐⭐⭐⭐ 必须吃透（日常写代码高频）

```
const
static
extern
volatile
class
struct
public
private
protected
this
virtual
override
final
new
delete
sizeof
enum
using
namespace
auto
nullptr
decltype
template
typename
constexpr
static_cast
reinterpret_cast
```

### ⭐⭐⭐⭐ 熟练掌握，看懂工程源码

```
explicit
operator
friend
inline
noexcept
static_assert
alignas
alignof
dynamic_cast
const_cast
thread_local
```

### ⭐⭐⭐ 看得懂即可，按需使用

```
typeid
mutable
union
throw
try
catch
concept
requires
consteval
constinit
co_await
co_return
co_yield
```

### ⭐⭐ 了解，极少使用

```
goto
register
asm
export
module
import
and or not xor ...(替代标记)
```

## 二十三、嵌入式 C++ 知识体系思维导图（文本版）

```
                 C++
                  │
    ┌─────────────┼─────────────┐
    ↓             ↓             ↓
  类型系统       面向对象       泛型
    │             │             │
const/auto     class       template
decltype       this        typename
enum           virtual
using          override
               final
│ │ │
└─────────────┼─────────────┘
↓
生命周期
│
┌───────┴───────┐
↓               ↓
栈对象        动态对象
│               │
RAII         new/delete
│               │
└───────┬───────┘
↓
底层系统
│
┌───────────┼───────────┐
↓           ↓           ↓
volatile   alignas     cast
↓           ↓           ↓
寄存器     DMA        地址
中断      Cache      内存
│
↓
STM32 / FreeRTOS
│
↓
Linux / Driver
│
↓
Embedded Linux
```

> 📌学习路线建议：优先打通 `class →构造析构 → this →继承 →virtual →override →多态 →RAII →template →STL`主线；补齐`constexpr/auto/using`现代特性；C++20 concept、协程、modules 可以延后学习，对嵌入式 MCU 收益较低。

---

下面是完整可直接复制保存为 `cpp_embed_keyword.md` 的原始 markdown 文本（复制全部到记事本保存即可）：

```
# C++ 关键字完整整理：嵌入式开发版
> 适用场景：STM32、FreeRTOS、嵌入式Linux、驱动开发；面向底层硬件、资源受限MCU环境

## 一、先建立一个总体认识
C++ 中涉及的"特殊词"可以先分成四类，不要全部混为传统保留关键字：
```

C++ 特殊词 │ ├── ① 保留关键字 Keywords │ ├── int │ ├── class │ ├── if │ ├── template │ └── ... │ ├── ② 替代标记 Alternative Tokens │ ├── and │ ├── or │ ├── not │ └── ... │ ├── ③ 特殊标识符 Special Identifiers │ ├── override │ ├── final │ ├── import │ └── module │ └── ④ 预处理器关键字 / 指令 ├── #if ├── #define ├── #include ├── #ifdef └── ...

````

> 💡 提示：看到网上的C++关键字表，注意区分上面4个分类，不是全部都是真正的保留关键字。

## 二、基本数据类型
|关键字|作用|标准版本|
|---|---|---|
|`bool`|布尔类型|C++98|
|`char`|字符类型|C++98|
|`wchar_t`|宽字符类型|C++98|
|`char16_t`|UTF‑16 字符类型|C++11|
|`char32_t`|UTF‑32 字符类型|C++11|
|`char8_t`|UTF‑8 字符类型|C++20|
|`short`|短整型|C++98|
|`int`|整型|C++98|
|`long`|长整型|C++98|
|`signed`|有符号类型修饰|C++98|
|`unsigned`|无符号类型修饰|C++98|
|`float`|单精度浮点|C++98|
|`double`|双精度浮点|C++98|
|`void`|无类型|C++98|

### 嵌入式重点代码示例
```cpp
uint32_t reg;
uint16_t adc;
uint8_t data;
bool enable;
````

> ⚠重要提醒：`uint8_t`、`uint16_t`、`uint32_t`**不是 C++ 关键字**，来自头文件 `<cstdint>` / `<stdint.h>` 的 typedef 类型别名。

## 三、流程控制

表格

|关键字|作用|标准版本|
|---|---|---|
|`if`|条件判断|C++98|
|`else`|条件分支 - 否则|C++98|
|`switch`|多分支判断|C++98|
|`case`|switch 分支标签|C++98|
|`default`|switch 默认分支|C++98|
|`for`|循环|C++98|
|`while`|while 循环|C++98|
|`do`|do‑while 循环|C++98|
|`break`|跳出循环 /switch|C++98|
|`continue`|直接进入下一轮循环|C++98|
|`goto`|无条件跳转|C++98|
|`return`|函数返回|C++98|

### C++11 范围 for 循环（range‑based for）

```
int array[] = {1,2,3,4};
for (auto value : array)
{
    // 遍历每一个元素
}
```

## 四、面向对象

表格

|关键字|作用|标准版本|
|---|---|---|
|`class`|定义类|C++98|
|`struct`|定义结构体|C++98|
|`public`|公有访问权限|C++98|
|`private`|私有访问权限|C++98|
|`protected`|保护访问权限|C++98|
|`this`|当前对象指针|C++98|
|`friend`|友元|C++98|
|`virtual`|虚函数，实现多态|C++98|
|`explicit`|禁止隐式类型转换|C++98|
|`mutable`|成员绕过 const 限制|C++98|
|`operator`|运算符重载|C++98|

> `override`、`final`：**特殊标识符，不是传统保留关键字 (C++11)** ⭐⭐⭐⭐⭐嵌入式必掌握

### override 强制校验重写虚函数

```
class Base
{
public:
    virtual void run();
};
class Derived : public Base
{
public:
    void run() override; // 编译器强制检查基类是否存在该虚函数
};
```

### final 禁止重写 / 禁止继承

```
// 禁止子类重写该虚函数
class Base
{
public:
    virtual void run() final;
};

// final修饰类，禁止被继承
class Motor final
{
};
```

## 五、动态内存与对象生命周期

表格

|关键字|作用|
|---|---|
|`new`|创建对象、分配堆内存|
|`delete`|销毁对象、释放堆内存|
|`sizeof`|获取类型 / 对象占用字节大小|

```
// 单个对象
int *p = new int(10);
delete p;

// 数组对象
int *p_arr = new int[10];
delete [] p_arr;
```

> ✅匹配规则：`new` ↔ `delete`；`new[]` ↔ `delete[]`，不能混用。

> 💡现代 C++ 嵌入式推荐优先智能指针，减少裸 new/delete： `std::unique_ptr`、`std::shared_ptr`、`std::make_unique`、`std::make_shared`

## 六、类型转换

C++ 四种强制转换（嵌入式重点关注`reinterpret_cast`寄存器地址映射）

表格

|关键字|主要用途|风险等级|
|---|---|---|
|`static_cast`|常规安全类型转换|⭐|
|`dynamic_cast`|多态运行时类型转换|⭐⭐|
|`const_cast`|修改 const/volatile 限定符|⭐⭐⭐|
|`reinterpret_cast`|底层二进制重新解释，硬件寄存器操作高频|⭐⭐⭐⭐⭐|

### static_cast 常规转换

```
int a = 10;
double b = static_cast<double>(a);
```

### dynamic_cast 多态向下转型

```
Base *p = nullptr;
Derived *d = dynamic_cast<Derived *>(p);
```

### const_cast 去掉 const 修饰

```
const int *p;
int *q = const_cast<int *>(p);
```

> ⚠注意：如果原始对象本身就是 const 常量，修改后属于**未定义行为**。

### reinterpret_cast 嵌入式硬件高频用法（寄存器映射）

```
uint32_t addr = 0x40000000;
volatile uint32_t *reg = reinterpret_cast<volatile uint32_t *>(addr);
```

## 七、函数相关

表格

|关键字|作用|标准版本|
|---|---|---|
|`inline`|内联函数提示|C++98|
|`const`|常量、只读限定|C++98|
|`noexcept`|声明函数不会抛出异常|C++11|
|`throw`|抛出异常|C++98|
|`try`|异常捕获块|C++98|
|`catch`|捕获异常|C++98|

### const 多重用法（嵌入式高频）

1. 普通常量

```
const int a = 10;
```

2. const 修饰指针

```
const int *p; // 指针指向内容不可修改
```

3. const 修饰成员函数

```
class A
{
public:
    int get() const; // 函数内部不能修改普通成员变量
};
```

## 八、命名空间、类型别名、链接属性

表格

|关键字|作用|标准版本|
|---|---|---|
|`namespace`|命名空间，防止命名冲突|C++98|
|`using`|引入命名空间、类型别名|C++98|
|`typedef`|旧版类型别名|C++98|
|`extern`|外部链接声明|C++98|
|`static`|静态变量、内部链接、静态类成员|C++98|
|`auto`|自动类型推导|C++11|
|`decltype`|获取表达式的类型|C++11|
|`typename`|模板中标记类型|C++98|

> 💡现代 C++ 优先使用`using`做类型别名，`typedef`大量存在于老工程、Linux 驱动内核代码，必须看得懂。

```
using uint = unsigned int;   // 现代写法
typedef unsigned int uint;   // 老式写法
```

## 九、模板

表格

|关键字|作用|标准版本|
|---|---|---|
|`template`|定义模板|C++98|
|`typename`|模板参数标记类型|C++98|
|`concept`|模板概念约束|C++20|
|`requires`|模板约束条件|C++20|

基础模板示例

```
template <typename T>
T add(T a, T b)
{
    return a + b;
}
```

C++20 concept requires

```
template <typename T>
requires SomeConstraint<T>
void foo(T value)
{
}
```

## 十、常量与编译期

现代 C++ 编译期计算，嵌入式非常适合做静态断言校验硬件参数

表格

|关键字|作用|标准版本|
|---|---|---|
|`const`|只读限定|C++98|
|`constexpr`|编译期常量表达式|C++11|
|`consteval`|强制编译期执行函数|C++20|
|`constinit`|强制静态初始化|C++20|
|`static_assert`|编译期断言|C++11|

### constexpr 编译期函数

```
constexpr int square(int x)
{
    return x * x;
}
constexpr int a = square(10);
```

### consteval 必须编译期求值

```
consteval int square(int x)
{
    return x * x;
}
```

### static_assert 嵌入式硬件校验神器

```
static_assert(sizeof(int) == 4);                // 校验int字节大小
static_assert(sizeof(PacketHeader) == 8);       // 校验协议结构体大小
```

## 十一、枚举、结构体、联合体

表格

|关键字|作用|
|---|---|
|`enum`|C 风格枚举|
|`struct`|结构体|
|`union`|联合体|
|`class`|类|

C++11 强类型枚举（嵌入式状态机首选）

```
enum class MotorState
{
    STOP,
    RUN,
    ERROR
};

MotorState state = MotorState::STOP;
```

## 十二、RTTI 运行时类型信息

表格

|关键字|作用|
|---|---|
|`typeid`|获取运行时类型信息|
|`dynamic_cast`|运行时多态转换|

```
if (typeid(obj) == typeid(MyClass))
{
}
```

> 💡MCU 嵌入式工程通常编译选项关闭 RTTI：`-fno‑rtti`，了解即可，不建议大量使用。

## 十三、线程局部存储 C++11

`thread_local`：每个线程拥有独立变量实例

```
thread_local int counter;
```

## 十四、内存对齐 C++11

`alignas` 指定对齐；`alignof` 查询对齐；DMA、Cache、硬件协议开发高频

```
// 指定结构体32字节对齐
struct alignas(32) Buffer
{
    uint8_t data[64];
};

// 查询类型对齐字节
constexpr size_t a = alignof(Buffer);
```

## 十五、C++20 新增关键字

表格

|关键字|功能|
|---|---|
|`char8_t`|UTF‑8 字符类型|
|`concept`|模板概念约束|
|`consteval`|立即函数，强制编译期执行|
|`constinit`|强制静态初始化|
|`co_await`|协程等待|
|`co_return`|协程返回|
|`co_yield`|协程产出值|
|`requires`|模板约束条件|

分组记忆：

1. 字符：`char8_t`
2. 编译期：`consteval`、`constinit`
3. Concepts：`concept`、`requires`
4. 协程 Coroutine：`co_await`、`co_return`、`co_yield`

## 十六、C++20 Modules 模块

> `module` / `import` / `export` 属于特殊标识符，不等同传统关键字，替代`#include`机制

```
export module my_module;   // 导出模块

import my_module;          // 导入模块
```

## 十七、Alternative Tokens 替代标记

等价于符号，工程一般直接写符号，很少使用单词形式

表格

|替代标记|等价运算符|||
|---|---|---|---|
|`and`|`&&`|||
|`or`|`||`|
|`not`|`!`|||
|`bitand`|`&`|||
|`bitor`|`|`||
|`xor`|`^`|||
|`compl`|`~`|||
|`and_eq`|`&=`|||
|`or_eq`|`|=`||
|`xor_eq`|`^=`|||
|`not_eq`|`!=`|||

示例：

```
if (a > 0 and b > 0) {}
//等价
if (a > 0 && b > 0) {}
```

## 十八、嵌入式重中之重：volatile

```
volatile uint32_t reg;
```

作用：告诉编译器**不要对该变量做优化**，用于硬件寄存器、中断标记、DMA 映射内存。

```
volatile uint32_t *GPIO_REG = reinterpret_cast<volatile uint32_t *>(0x40020000);
```

> ⚠重大误区：**volatile != 线程安全、不等于原子操作** `volatile int counter; counter++;` 在中断 / 多线程环境依然会出现竞态问题。FreeRTOS、多核并发要注意这点。

## 十九、static：多语义关键字

1. **局部静态变量**：函数内，生命周期全局，只初始化一次

```
void foo()
{
    static int count;
}
```

2. **全局 static 变量**：内部链接，仅当前`.cpp`翻译单元可见，其他文件访问不到

```
static int value;
```

3. **类 static 成员**：属于类本身，不属于对象实例，所有对象共享同一份

```
class A
{
public:
    static int count;
};
```

## 二十、extern 外部链接

声明变量 / 函数定义在别的翻译单元，用于跨文件共享全局变量

```
// file1.cpp
int g_value = 10;

// file2.cpp
extern int g_value; // 告诉编译器这个变量定义在别处
```

> 和 C 语言的声明、定义、内部链接、外部链接、翻译单元概念完全打通。

## 二十一、容易混淆的特殊标识符汇总

表格

|名称|类型|C++ 版本|
|---|---|---|
|`override`|特殊标识符|C++11|
|`final`|特殊标识符|C++11|
|`nullptr`|关键字|C++11|
|`auto`|关键字，语义更新|C++11|
|`constexpr`|关键字|C++11|
|`concept`|关键字|C++20|
|`requires`|关键字|C++20|
|`import`|模块特殊标识符|C++20|
|`module`|模块特殊标识符|C++20|

## 二十二、嵌入式 C++ 关键字学习优先级

### ⭐⭐⭐⭐⭐ 必须吃透（日常写代码高频）

```
const
static
extern
volatile
class
struct
public
private
protected
this
virtual
override
final
new
delete
sizeof
enum
using
namespace
auto
nullptr
decltype
template
typename
constexpr
static_cast
reinterpret_cast
```

### ⭐⭐⭐⭐ 熟练掌握，看懂工程源码

```
explicit
operator
friend
inline
noexcept
static_assert
alignas
alignof
dynamic_cast
const_cast
thread_local
```

### ⭐⭐⭐ 看得懂即可，按需使用

```
typeid
mutable
union
throw
try
catch
concept
requires
consteval
constinit
co_await
co_return
co_yield
```

### ⭐⭐ 了解，极少使用

```
goto
register
asm
export
module
import
and or not xor ...(替代标记)
```

## 二十三、嵌入式 C++ 知识体系思维导图（文本版）

```
                 C++
                  │
    ┌─────────────┼─────────────┐
    ↓             ↓             ↓
  类型系统       面向对象       泛型
    │             │             │
const/auto     class       template
decltype       this        typename
enum           virtual
using          override
               final
│ │ │
└─────────────┼─────────────┘
↓
生命周期
│
┌───────┴───────┐
↓               ↓
栈对象        动态对象
│               │
RAII         new/delete
│               │
└───────┬───────┘
↓
底层系统
│
┌───────────┼───────────┐
↓           ↓           ↓
volatile   alignas     cast
↓           ↓           ↓
寄存器     DMA        地址
中断      Cache      内存
│
↓
STM32 / FreeRTOS
│
↓
Linux / Driver
│
↓
Embedded Linux
```

> 📌学习路线建议：优先打通 `class →构造析构 → this →继承 →virtual →override →多态 →RAII →template →STL`主线；补齐`constexpr/auto/using`现代特性；C++20 concept、协程、modules 可以延后学习，对嵌入式 MCU 收益较低。