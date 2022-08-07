# 符合STL标准的C++ HashMap

## 描述

本项目基于Stanford CS106L : Standard C++ Programming，实现了一个简单的符合STL标准的HashMap。它与C++11引入的`std::unordered_map`具有类似的接口和功能，包括`size()`, `empty()`, `at()`, `contains()`, `insert()`, `find()`, `clear()`等

HashMap的核心成员是一个hash函数（默认为std::hash）和一组桶。每一个桶都被组织成一个链表，被映射到特定桶的元素会称为链表新的头节点。类中也提供了`rehash()`方法来指定新的桶数并重新映射

## 主要工作

该项目的主要工作包括

1. 设计若干特殊的构造函数，包括
   - 基于指示范围的迭代器的构造函数
   - 基于`std::initializer_list`的构造函数
2. 对运算符的重载，包括
   - `operator[]`：用于索引，并且在键不存在时自动添加键值对
   - `operator==` 与 `operator!=`
   - `operator<<`：用于将HashMap的内容写入输出流
3. 若干特殊成员函数，包括
   - 拷贝构造函数与拷贝赋值运算符
   - 移动构造函数与移动赋值运算符
4. 为部分成员函数重载了`const`版本，以便`const`的HashMap对象调用

此外，课程提供的原始代码中存在BUG，该项目也对原始的实现进行了检查和修正，详情见备注部分

## 部分思路概述

### 1. 基于迭代器和`std::initializer_list`的构造函数
这一部分的实现最为简单，只需要不断拷贝迭代器指向的元素，并调用`insert`方法插入当前的HashMap即可

基于`std::initializer_list`的构造函数只需要获得其首尾迭代器，然后调用上一个函数即可

### 2. 运算符重载

#### a. `operator[]`
对于传入的参数`key`，我们可以就地调用`insert({key, {}})`。倘若`key`在HashMap中已经存在，那么`insert`会以`std::pair`返回指向该键值对的迭代器和`false`；否则，会插入新的键值对并返回其迭代器与`true`。

我们只需要获取迭代器，并进一步获取指向该键值对中`value`的引用，即可将该引用返回

#### b. `operator==` 与 `operator!=`

首先判断两个HashMap的`size()`结果是否相同。由于HashMap不保证迭代器访问的顺序，我们用`std::is_permutation`来检测自身元素是否为对方的重排

#### c. `operator<<`

这部分的实现本身不难，但我们可以利用`ostringstream`字符串流，先将待输出的内容写入到字符串流中，然后获取其底层字符串，并将字符串和`}`写入流中。这样做能够更简洁地确保输出结果的美观（例如没有额外的空行和`,`）

在初始化`ostringstream`对象时，需要指定参数为`std::ostringstream::ate`以寻位到流的结尾

### 3. 特殊成员函数

#### a. 拷贝构造函数
首先我们需要初始化一个空的HashMap，其桶数与hash函数与对方相同。接下来有两种实现策略
- 遍历对方的每一个键值对，并将其拷贝插入到当前的HashMap
- 直接检查对方的每一个桶，将桶的内容拷贝到自己的桶中，并手动设置`_size`与对方相同

第二种实现中，拷贝桶本质上是拷贝链表，因此实现了一个私有的成员函数`bucket_copy`，它将待拷贝链表的值依次入栈，然后不断弹栈并将新节点插入到新桶的头部

#### b. 拷贝赋值运算符

与拷贝构造函数的实现类似，但是我们需要先判断是否出现自赋值，这时直接返回`*this`即可

#### c. 移动构造函数

我们只需要通过`std::move`将对方的3个成员变量（`_size`, `_hash_function`, `_buckets_array`）转换成右值，然后用于初始化自己的成员变量即可


#### d. 移动赋值运算符

类似地，先排除自赋值的特殊情况，然后调用`clear()`清空当前HashMap，最后再通过`std::move`将对方的成员变量转换为右值并移动赋值给自己的成员变量

### 4. 确保const correctness

首先，对于`begin()`, `end()`, `find()`与`at()`，它们返回一个迭代器，因此我们必须设计`const`与非`const`的两个版本。前者被`const`的HashMap对象调用，并返回`const_iterator`，而后者则返回`iterator`

我们可以利用`static_cast/const_cast`的技巧来简单复用非`const`版本的实现。例如，`const`版本的`begin()`实现只需要一行代码
```
return static_cast<const_iterator>(const_cast<HashMap<K, M, H>*>(this)->begin());
```
其原理是通过`const_cast`去除底层`const`，使其可以调用非`const`版本的成员函数。然后，将返回的`iterator`通过`static_cast`强制转换成`const_iterator`即可



## 文件布局

- `hashmap.{cpp,h}`：HashMap的主体部分，包括原始提供的成员和自己实现的成员
- `hashmap_iterator.h`：提供了HashMap迭代器的接口与实现
- `tests.cpp`与`test_settings.cpp`：提供了对HashMap的大量测试函数及其设置
- `main.cpp`：用于简单测试HashMap，或使用课程提供的复杂测试

## 编译与运行
运行如下指令
```
g++ -Wall -Werror -std=c++17 main.cpp -o main
```
然后在shell中运行`./main`，根据程序的提示信息选择不同的测试

注意，`test.cpp`的`run_test_harness()`函数中注释掉了一部分基础测试，你可以去掉注释以便自行测试。

测试4G即`G_move_ctor_time()`的设计有疏漏，详见备注

## 备注

这里对有关课程原始代码的一些问题进行勘误，不断更新

1. 原版的`hashmap.cpp`中，成员函数`clear`和`erase`都没有释放内存，前者仅遍历了全部链表并将`_size`设为零，后者则仅对擦除对象所在的链表进行了修改。本项目中补充了释放内存的代码。
2. 原版的`tests.cpp`中，测试4G即`G_move_ctor_time()`的设计有疏漏。该测试准备了四个大小依次为10, 100, 1000, 10000的HashMap，并分别将其转为右值后传入移动构造函数，检测所耗费的时间$t_1$到$t_4$（单位ns）。测试通过的标准是$3t_i > t_{i+1}$。然而，在有些测试中，会出现某个$t_i$为0ns的情形，以至于无法通过测试（尽管四个时间在同一数量级）。
