# TVA 项目详细说明文档

## 一、项目概述

### 1.1 项目简介
TVA (Time-series Verifiable Analytics) 是一个用于**安全时间序列分析**的**多方安全计算 (MPC)** 系统。该项目实现了 USENIX Security'23 会议论文中描述的系统，由 Muhammad Faisal、Jerry Zhang、John Liagouris、Vasiliki Kalavri 和 Mayank Varia 等人开发。

**核心功能：**
- 在多个不信任的计算方之间进行安全的时间序列数据分析
- 保护数据隐私的同时执行复杂的数据查询和分析
- 支持两种安全协议：半诚实的3方协议和恶意安全的4方协议

### 1.2 应用场景
- **医疗数据分析**：多个医院在不泄露患者隐私的情况下联合分析医疗数据
- **能源数据分析**：电力公司安全地分析用户用电数据
- **云计算场景**：在不信任的云服务器上进行安全计算

### 1.3 技术特点
- 使用**复制秘密共享 (Replicated Secret Sharing)** 技术
- 支持两种协议：
  - **Replicated 3PC**：Araki 等人提出的半诚实3方协议
  - **Fantastic Four**：Dalskov 等人提出的恶意安全4方协议
- 提供高级 API，类似 Pandas 的表格操作接口
- 支持向量化操作，性能优化

---

## 二、C++ 项目基础知识

### 2.1 项目构建系统 (CMake)

**什么是 CMake？**
CMake 是一个跨平台的构建系统生成器。它不直接编译代码，而是生成本地构建系统（如 Makefile）所需的配置文件。

**本项目的 CMakeLists.txt 解析：**
```cmake
cmake_minimum_required(VERSION 3.15)  # 要求 CMake 最低版本
project(TVA VERSION 2.0.0 LANGUAGES CXX)  # 项目名称和版本
set(CMAKE_CXX_STANDARD 14)  # 使用 C++14 标准
set(CMAKE_CXX_FLAGS "-O3 -pthread ${PROTOCOL_VAR}")  # 编译优化选项
```

**构建流程：**
1. `mkdir build && cd build` - 创建构建目录
2. `cmake ..` - 生成构建文件
3. `make main` - 编译特定目标
4. `mpirun -np 3 ./main` - 运行程序（3个进程）

### 2.2 头文件和源文件组织

**C++ 项目典型结构：**
- `include/` - 头文件 (.h)，包含类声明、函数原型
- `src/` - 源文件 (.cpp)，包含实现代码
- `tests/` - 测试文件
- `build/` - 编译输出目录

**为什么分离头文件和源文件？**
- 头文件定义接口（"做什么"）
- 源文件实现功能（"怎么做"）
- 便于代码复用和模块化

### 2.3 C++ 模板 (Templates)

本项目大量使用模板编程。**模板是什么？**
模板允许编写泛型代码，可以处理不同的数据类型。

**示例：**
```cpp
template<typename DataType>
class Vector {
    // 可以是 Vector<int>, Vector<long>, Vector<double> 等
};
```

### 2.4 命名空间 (Namespaces)

命名空间用于组织代码，避免命名冲突。

**本项目的命名空间结构：**
```cpp
namespace tva {
    namespace operators { ... }    // 操作符
    namespace service { ... }      // 服务层
    namespace debug { ... }        // 调试工具
}
```

---

## 三、项目架构详解

### 3.1 目录结构

```
tva/
├── include/              # 头文件目录
│   ├── core/            # 核心功能
│   │   ├── containers/  # 数据容器（向量、共享向量等）
│   │   ├── protocols/   # MPC 协议实现
│   │   ├── operators/   # 安全操作符
│   │   ├── communication/ # 通信模块
│   │   └── random/      # 随机数生成
│   ├── relational/      # 关系型数据结构（表、列）
│   ├── service/         # 服务层（运行时、任务管理）
│   ├── debug/           # 调试工具
│   ├── mpc.h           # MPC 总头文件
│   └── tva.h           # TVA 总头文件
├── src/                 # 源代码
│   ├── main.cpp        # 主程序入口
│   ├── micro/          # 微基准测试
│   └── queries/        # 示例查询程序
├── tests/              # 单元测试
├── scripts/            # 构建和运行脚本
└── CMakeLists.txt      # CMake 配置文件
```

### 3.2 核心模块

#### 3.2.1 容器模块 (Containers)

**1. Vector<DataType>** - 明文向量
- 封装 `std::vector`，提供向量化操作
- 支持算术运算：`+`, `-`, `*`, `/`
- 支持位运算：`&`, `|`, `^`, `~`

**2. EVector** - 编码向量（秘密份额的容器）
- 存储复制秘密共享的份额
- 在3方协议中，每方持有2个份额

**3. ASharedVector** - 算术共享向量
- 存储算术秘密共享
- 支持安全算术运算：加、减、乘、除

**4. BSharedVector** - 布尔共享向量
- 存储布尔秘密共享
- 支持安全位运算：AND, OR, XOR
- 支持安全比较操作

#### 3.2.2 协议模块 (Protocols)

**Protocol 基类** - 定义所有协议必须实现的接口

**核心方法：**
- `add_a()`, `sub_a()`, `multiply_a()` - 算术共享的加减乘
- `and_b()`, `or_b()`, `xor_b()` - 布尔共享的逻辑运算
- `a2b()`, `b2a()` - 算术共享与布尔共享之间的转换
- `secret_share_a()`, `secret_share_b()` - 创建秘密共享
- `open_a()`, `open_b()` - 打开（重构）秘密

**Replicated_3PC** - 3方半诚实协议
```cpp
// 3方协议中，每个秘密 x 被分成3个份额：x0, x1, x2
// 满足：x = x0 + x1 + x2
// Party 0 持有 (x0, x1)
// Party 1 持有 (x1, x2)
// Party 2 持有 (x2, x0)
```

**特点：**
- 需要3个计算方
- 假设最多1方是半诚实的（会诚实执行协议但可能窃取信息）
- 通信轮次少，效率高

**Fantastic_4PC** - 4方恶意安全协议
- 需要4个计算方
- 可以抵御恶意攻击者（可能偏离协议）
- 安全性更高，但通信开销更大

#### 3.2.3 操作符模块 (Operators)

**1. 通用操作 (common.h)**
- `multiplex()` - 多路复用器（安全选择）
  ```cpp
  // 根据选择位 sel 选择 a 或 b
  result = sel ? b : a
  ```

**2. 比较和排序 (sorting.h)**
- `compare()` - 安全比较两个向量
- `bitonic_sort()` - 双调排序（适合并行）
- `odd_even_sort()` - 奇偶排序

**3. 聚合操作 (aggregation.h)**
- `sum_aggregation()` - 求和聚合
- `max_aggregation()` - 最大值聚合
- `min_aggregation()` - 最小值聚合
- `odd_even_aggregation()` - 奇偶聚合（用于分组聚合）

**4. 流式操作 (streaming.h)**
- `tumbling_window()` - 滚动窗口
- `session_window()` - 会话窗口
- `threshold_session_window()` - 阈值会话窗口

**5. 关系操作 (relational.h)**
- `distinct()` - 去重
- `join()` - 连接操作

#### 3.2.4 通信模块 (Communication)

**Communicator** - 通信接口
- 定义了计算方之间的通信方法

**MPICommunicator** - 基于 MPI 的通信实现
- 使用 MPI (Message Passing Interface) 进行进程间通信
- 支持点对点通信和集体通信

**MPI 基础知识：**
- MPI 是并行计算的标准接口
- `mpirun -np 3 ./program` 启动3个进程
- 每个进程有唯一的 rank (0, 1, 2, ...)
- 进程间通过消息传递通信

#### 3.2.5 随机数生成 (Random)

**RandomGenerator** - 随机数生成器接口

**PseudoRandomGenerator** - 伪随机数生成器
- 使用 libsodium 库生成密码学安全的随机数
- 用于生成秘密共享的随机份额

---

## 四、关键概念详解

### 4.1 秘密共享 (Secret Sharing)

**什么是秘密共享？**
将一个秘密值分成多个份额，分发给不同的参与方，只有足够多的参与方合作才能重构秘密。

**复制秘密共享示例（3方）：**
```
秘密值：x = 100

生成份额：
- 随机选择 x0 = 30, x1 = 25
- 计算 x2 = x - x0 - x1 = 45

分发：
- Party 0 获得 (x0=30, x1=25)
- Party 1 获得 (x1=25, x2=45)
- Party 2 获得 (x2=45, x0=30)

重构：
- 任意两方合作：x0 + x1 + x2 = 100
```

**安全性：**
- 单个参与方无法获知秘密值
- 需要至少2方合作才能重构

### 4.2 算术共享 vs 布尔共享

**算术共享 (Arithmetic Sharing)：**
- 份额在整数环上相加得到秘密
- 适合算术运算（加、减、乘）
- 示例：`[x]_A = (x0, x1, x2)` 满足 `x = x0 + x1 + x2`

**布尔共享 (Boolean Sharing)：**
- 份额按位异或得到秘密
- 适合位运算（AND, OR, XOR）和比较
- 示例：`[x]_B = (x0, x1, x2)` 满足 `x = x0 ⊕ x1 ⊕ x2`

**为什么需要两种共享？**
- 不同操作在不同共享下效率不同
- 乘法在算术共享下高效
- 比较在布尔共享下高效
- 需要在两种共享间转换（a2b, b2a）

### 4.3 安全计算原语

**加法（算术共享）：**
```cpp
// 本地计算，无需通信
[x]_A + [y]_A = [x+y]_A
每方将自己的份额相加即可
```

**乘法（算术共享）：**
```cpp
// 需要一轮通信
[x]_A * [y]_A = [x*y]_A
1. 本地计算部分乘积
2. 生成随机份额
3. 与相邻方交换份额
```

**比较（布尔共享）：**
```cpp
// 复杂操作，需要多轮通信
[x]_B > [y]_B = [x>y]_B
1. 计算相同位前缀
2. 提取边界位
3. 处理符号位（有符号数）
```

---

## 五、代码示例与解析

### 5.1 基础示例：安全乘法

```cpp
#include "../include/tva.h"

using namespace tva::service::mpi_service::replicated_3pc;

int main(int argc, char** argv) {
    // 1. 初始化 TVA 运行时
    tva_init(argc, argv);

    // 2. 创建明文向量
    Vector<int32_t> a(10), b(10);  // 10个元素的向量

    // 3. 创建算术共享向量
    ASharedVector<int32_t> a_(10), b_(10);

    // 4. 执行安全乘法
    ASharedVector<int32_t> c_ = a_ * b_;

    // 5. 清理
    MPI_Finalize();
    return 0;
}
```

**代码解析：**
1. `tva_init()` - 初始化 MPI 环境和协议
2. `Vector<int32_t>` - 明文向量，用于测试
3. `ASharedVector<int32_t>` - 算术共享向量
4. `a_ * b_` - 调用重载的乘法运算符，执行安全乘法
5. `MPI_Finalize()` - 清理 MPI 资源

### 5.2 医疗数据查询示例

```cpp
// 医疗数据查询：分析患者的胰岛素使用情况
int main(int argc, char** argv) {
    tva_init(argc, argv);

    // 1. 定义表结构
    std::vector<std::string> schema = {
        "[TIMESTAMP]",           // 时间戳（布尔共享）
        "TIMESTAMP",             // 时间戳（算术共享）
        "[PATIENT_ID]",          // 患者ID（布尔共享）
        "[GLUCOSE]",             // 血糖值（布尔共享）
        "INSULIN",               // 胰岛素剂量（算术共享）
        "[THRESHOLD_WINDOW]",    // 阈值窗口（布尔共享）
        "TOTAL_INSULIN_EVENTS"   // 总胰岛素事件（算术共享）
    };

    // 2. 创建秘密共享表
    EncodedTable<int> medical_table = secret_share(medical_data, schema);

    // 3. 排序：按患者ID和时间戳排序
    medical_table.sort(
        {"[PATIENT_ID]", "[TIMESTAMP]"},  // 排序键
        {false, false},                    // 升序
        {"[GLUCOSE]", "INSULIN"}           // 附加列
    );

    // 4. 阈值会话窗口：当血糖超过阈值时创建窗口
    medical_table.threshold_session_window(
        {"[PATIENT_ID]"},      // 分组键
        "[GLUCOSE]",           // 监控列
        "[TIMESTAMP]",         // 时间列
        "[THRESHOLD_WINDOW]",  // 输出窗口ID
        5,                     // 阈值
        false                  // 升序
    );

    // 5. 聚合：计算每个窗口的总胰岛素剂量
    medical_table.odd_even_aggregation(
        {"[PATIENT_ID]", "[THRESHOLD_WINDOW]"},  // 分组键
        {"INSULIN"},                              // 聚合列
        {"TOTAL_INSULIN_EVENTS"},                 // 输出列
        tva::operators::AggregationType::SUM      // 求和
    );

    MPI_Finalize();
    return 0;
}
```

**查询逻辑：**
1. 按患者ID和时间排序数据
2. 识别血糖超过阈值的时间段（会话窗口）
3. 计算每个窗口内的总胰岛素使用量

**隐私保护：**
- 所有数据都是秘密共享的
- 计算过程中不泄露任何明文信息
- 只有授权方才能看到最终结果

### 5.3 测试示例：原语测试

```cpp
// 测试安全计算原语
int main(int argc, char** argv) {
    tva_init(argc, argv);
    auto pID = runTime.getPartyID();

    // 测试数据
    tva::Vector<int> data_a = {111, -4, -17, 2345, 999};
    tva::Vector<int> data_b = {0, -4, -5, -123556, 999};

    // ===== 算术原语测试 =====

    // 秘密共享（Party 0 提供数据）
    ASharedVector<int> a_v1 = secret_share_a(data_a, 0);
    ASharedVector<int> a_v2 = secret_share_a(data_b, 0);

    // 安全加法
    ASharedVector<int> c_add = a_v1 + a_v2;
    auto c_add_open = c_add.open();  // 打开结果
    tva::Vector<int> sum = data_a + data_b;  // 明文计算
    assert(c_add_open.same_as(sum));  // 验证正确性

    // 安全减法
    ASharedVector<int> c_sub = a_v1 - a_v2;
    auto c_sub_open = c_sub.open();
    assert(c_sub_open.same_as(data_a - data_b));

    // 安全乘法
    ASharedVector<int> c_mul = a_v1 * a_v2;
    auto c_mul_open = c_mul.open();
    assert(c_mul_open.same_as(data_a * data_b));

    // 安全除法（除以公开常数）
    ASharedVector<int> c_div = a_v1.div(8);
    auto c_div_open = c_div.open();
    assert(c_div_open.same_as(data_a / 8));

    // ===== 布尔原语测试 =====

    BSharedVector<int> b_v1 = secret_share_b(data_a, 0);
    BSharedVector<int> b_v2 = secret_share_b(data_b, 0);

    // 安全 AND
    BSharedVector<int> c_and = b_v1 & b_v2;
    auto c_and_open = c_and.open();
    assert(c_and_open.same_as(data_a & data_b));

    // 安全 XOR
    BSharedVector<int> c_xor = b_v1 ^ b_v2;
    auto c_xor_open = c_xor.open();
    assert(c_xor_open.same_as(data_a ^ data_b));

    // 安全比较
    BSharedVector<int> c_gt = b_v1 > b_v2;
    auto c_gt_open = c_gt.open();
    tva::Vector<int> gt = data_a > data_b;
    assert(c_gt_open.same_as(gt));

    if (pID == 0) std::cout << "All tests passed!" << std::endl;

    MPI_Finalize();
    return 0;
}
```

---

## 六、高级特性

### 6.1 EncodedTable - 关系表接口

**EncodedTable** 提供类似 Pandas 的高级接口：

```cpp
// 创建表
EncodedTable<int> table("my_table", {"col1", "col2", "[col3]"}, 1000);

// 访问列
table["col1"] = table["col2"] * 2;

// 排序
table.sort({"[col3]"}, {false}, {"col1", "col2"});

// 聚合
table.odd_even_aggregation(
    {"[col3]"},           // 分组键
    {"col1"},             // 聚合列
    {"result"},           // 输出列
    AggregationType::SUM  // 聚合类型
);

// 打开结果
auto result = table.open();
```

**列命名约定：**
- `"column_name"` - 算术共享列
- `"[column_name]"` - 布尔共享列

### 6.2 运行时系统 (Runtime)

**RunTime** 类管理：
- 多线程并行执行
- 任务调度
- 批处理优化

**并行化示例：**
```cpp
// 运行时自动将向量操作分配给多个线程
ASharedVector<int> result = tva::service::runTime.execute_parallel(
    vector,
    &EVector::some_operation,
    args...
);
```

**配置参数：**
```bash
# 运行程序时指定参数
mpirun -np 3 ./program threads_num p_factor batch_size data_size

# 示例：3个进程，16个线程，批大小8192，数据1024行
mpirun -np 3 ./medical 16 1 8192 1024
```

### 6.3 性能优化技术

**1. 批处理 (Batching)**
- 将大向量分成小批次处理
- 减少通信开销
- 提高缓存利用率

**2. 并行化 (Parallelization)**
- 多线程并行处理向量元素
- 利用多核 CPU

**3. 向量化 (Vectorization)**
- 一次操作处理多个元素
- 减少函数调用开销

**4. 通信优化**
- 合并多个小消息
- 使用非阻塞通信
- 重叠计算和通信

---

## 七、编译和运行

### 7.1 依赖安装

**必需依赖：**
```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y \
    build-essential \
    cmake \
    pkg-config \
    libsodium-dev \
    libopenmpi-dev

# 或使用 MPICH
sudo apt-get install -y mpich
```

**依赖说明：**
- **CMake** (≥3.15.0) - 构建系统
- **pkg-config** (≥0.29.2) - 包配置工具
- **libsodium** (≥1.0.18) - 密码学库，用于随机数生成
- **OpenMPI/MPICH** - MPI 实现，用于进程间通信

### 7.2 编译项目

**方法1：使用脚本编译所有测试**
```bash
cd scripts
./run_tests.sh
```

**方法2：手动编译**
```bash
# 创建构建目录
mkdir build && cd build

# 生成构建文件
cmake ..

# 编译特定目标
make main              # 编译主程序
make medical           # 编译医疗查询
make test_primitives   # 编译原语测试

# 编译所有目标
make -j4  # 使用4个并行任务
```

**选择协议：**
```bash
# 使用半诚实3方协议（默认）
cmake ..

# 使用恶意安全4方协议
cmake -DPROTOCOL_VAR="-DUSE_FANTASTIC_FOUR" ..
```

### 7.3 运行程序

**单机运行（模拟多方）：**
```bash
cd build

# 运行主程序（3方）
mpirun -np 3 ./main

# 运行医疗查询（数据大小1024）
mpirun -np 3 ./medical 16 1 8192 1024

# 运行测试
mpirun -np 3 ./test_primitives
```

**参数说明：**
- `-np 3` - 启动3个进程（3个计算方）
- `16` - 线程数
- `1` - 并行因子
- `8192` - 批大小
- `1024` - 数据行数

**集群运行（多机）：**
```bash
# 在3台机器上运行
mpirun --host machine-1,machine-2,machine-3 \
       -np 3 ./medical 16 1 8192 1024
```

### 7.4 运行实验脚本

```bash
cd scripts

# 运行云计算实验（LAN环境，半诚实协议）
./cloud.sh semi lan

# 运行能源查询实验
./energy.sh semi lan

# 运行微基准测试
./micro_primitives.sh semi lan
./micro_sorting.sh semi lan
./micro_aggregation.sh semi lan
```

**结果输出：**
- 结果保存在 `./results/lan/` 或 `./results/wan/`
- 包含运行时间、通信量等性能指标

---

## 八、核心数据结构详解

### 8.1 Vector<DataType>

**定义：**
```cpp
template<typename DataType>
class Vector {
    std::shared_ptr<VectorData<DataType>> data;  // 共享数据指针
    int start, end;  // 批处理边界
};
```

**特点：**
- 浅拷贝：多个 Vector 可以共享底层数据
- 支持批处理：通过 start/end 指定处理范围
- 运算符重载：支持 `+`, `-`, `*`, `/`, `&`, `|`, `^` 等

**常用方法：**
```cpp
Vector<int> v(100);           // 创建100个元素的向量
v.size();                     // 获取大小
v[i];                         // 访问元素
v.subset(start, end);         // 获取子向量
v.same_as(other);             // 比较两个向量
```

### 8.2 EVector (EncodedVector)

**定义：**
```cpp
template<int ReplicationNumber>
class EVector {
    std::vector<Vector> shares;  // 存储多个份额
};
```

**3方协议中的结构：**
```cpp
EVector evec;
evec(0);  // 第一个份额
evec(1);  // 第二个份额
```

### 8.3 SharedVector

**继承关系：**
```
EncodedVector (基类)
    ↓
SharedVector (共享向量基类)
    ↓
    ├── ASharedVector (算术共享)
    └── BSharedVector (布尔共享)
```

**编码类型：**
```cpp
enum class Encoding {
    AShared,  // 算术共享
    BShared,  // 布尔共享
    Public    // 公开值
};
```

### 8.4 ASharedVector - 算术共享向量

**核心操作：**
```cpp
ASharedVector<int> a(100), b(100);

// 本地操作（无通信）
ASharedVector<int> c = a + b;      // 加法
ASharedVector<int> d = a - b;      // 减法
ASharedVector<int> e = -a;         // 取负
ASharedVector<int> f = a * 5;      // 乘以公开常数

// 需要通信的操作
ASharedVector<int> g = a * b;      // 乘法（1轮通信）
ASharedVector<int> h = a.div(8);   // 除以公开常数

// 打开秘密
Vector<int> result = a.open();     // 重构明文值
```

**转换操作：**
```cpp
ASharedVector<int> a_shared(100);
BSharedVector<int> b_shared(100);

// 算术共享 → 布尔共享
b_shared = a_shared.a2b();

// 布尔共享 → 算术共享
a_shared = b_shared.b2a();
```

### 8.5 BSharedVector - 布尔共享向量

**核心操作：**
```cpp
BSharedVector<int> a(100), b(100);

// 位运算（本地操作）
BSharedVector<int> c = a & b;      // AND
BSharedVector<int> d = a | b;      // OR
BSharedVector<int> e = a ^ b;      // XOR
BSharedVector<int> f = ~a;         // NOT

// 比较操作（需要通信）
BSharedVector<int> g = a > b;      // 大于
BSharedVector<int> h = a == b;     // 等于
BSharedVector<int> i = a >= b;     // 大于等于

// 位操作
BSharedVector<int> j = a.bit_right_shift(3);  // 右移3位
BSharedVector<int> k = a.bit_left_shift(2);   // 左移2位
```

**特殊方法：**
```cpp
// 扩展最低有效位到所有位
BSharedVector<int> extended = a.extend_lsb();

// 计算相同位前缀（用于比较）
BSharedVector<int> same = a.bit_same(b);

// 位异或归约
BSharedVector<int> xor_result = a.bit_xor();
```

---

## 九、常见操作模式

### 9.1 秘密共享数据

**方式1：从明文创建**
```cpp
// Party 0 持有数据
Vector<int> data = {1, 2, 3, 4, 5};

// 所有方执行秘密共享（Party 0 提供数据）
ASharedVector<int> shared = secret_share_a(data, 0);
```

**方式2：从随机数创建**
```cpp
ASharedVector<int> shared(100);
shared.populatePRandom();  // 填充伪随机值
```

**方式3：从表创建**
```cpp
std::vector<Vector<int>> table_data(3, 1000);  // 3列，1000行
std::vector<std::string> schema = {"col1", "[col2]", "col3"};
EncodedTable<int> table = secret_share(table_data, schema);
```

### 9.2 执行安全计算

**算术计算：**
```cpp
ASharedVector<int> x = secret_share_a(data_x, 0);
ASharedVector<int> y = secret_share_a(data_y, 0);

// 计算 z = (x + y) * (x - y)
ASharedVector<int> sum = x + y;
ASharedVector<int> diff = x - y;
ASharedVector<int> z = sum * diff;
```

**条件选择：**
```cpp
BSharedVector<int> condition = x > y;
ASharedVector<int> result = multiplex(condition, x, y);
// result[i] = condition[i] ? x[i] : y[i]
```

**聚合计算：**
```cpp
// 分组求和
std::vector<BSharedVector<int>*> keys = {&group_id};
std::vector<ASharedVector<int>*> values = {&amounts};
std::vector<ASharedVector<int>*> results = {&sums};

odd_even_aggregation(
    keys, values, results,
    sum_aggregation<int, EVector>,
    false  // 正向聚合
);
```

### 9.3 打开结果

**打开单个向量：**
```cpp
ASharedVector<int> shared_result = /* ... */;
Vector<int> plaintext = shared_result.open();

// 只有 Party 0 查看结果
if (runTime.getPartyID() == 0) {
    for (int i = 0; i < plaintext.size(); i++) {
        std::cout << plaintext[i] << " ";
    }
}
```

**打开整个表：**
```cpp
EncodedTable<int> table = /* ... */;
auto plaintext_table = table.open();

// plaintext_table 是 std::vector<Vector<int>>
for (auto& column : plaintext_table) {
    // 处理每一列
}
```

---

## 十、调试和测试

### 10.1 调试工具

**打印调试信息：**
```cpp
#include "debug/debug.h"

// 打印向量
Vector<int> v = {1, 2, 3};
if (runTime.getPartyID() == 0) {
    std::cout << "Vector: ";
    for (int i = 0; i < v.size(); i++) {
        std::cout << v[i] << " ";
    }
    std::cout << std::endl;
}
```

**验证正确性：**
```cpp
// 安全计算
ASharedVector<int> secure_result = secure_computation(shared_x, shared_y);
Vector<int> result = secure_result.open();

// 明文计算（用于验证）
Vector<int> expected = plaintext_computation(data_x, data_y);

// 比较结果
assert(result.same_as(expected));
```

### 10.2 单元测试

**测试结构：**
```cpp
int main(int argc, char** argv) {
    tva_init(argc, argv);
    auto pID = runTime.getPartyID();

    // 测试1：加法
    test_addition();
    if (pID == 0) std::cout << "Addition...OK" << std::endl;

    // 测试2：乘法
    test_multiplication();
    if (pID == 0) std::cout << "Multiplication...OK" << std::endl;

    // 测试3：比较
    test_comparison();
    if (pID == 0) std::cout << "Comparison...OK" << std::endl;

    MPI_Finalize();
    return 0;
}
```

**运行测试：**
```bash
cd build
make test_primitives
mpirun -np 3 ./test_primitives
```

### 10.3 性能分析

**计时：**
```cpp
#include <sys/time.h>

struct timeval begin, end;
long seconds, micro;
double elapsed;

// 开始计时
gettimeofday(&begin, 0);

// 执行操作
perform_computation();

// 结束计时
gettimeofday(&end, 0);
seconds = end.tv_sec - begin.tv_sec;
micro = end.tv_usec - begin.tv_usec;
elapsed = seconds + micro * 1e-6;

if (pID == 0) {
    std::cout << "Elapsed time: " << elapsed << " seconds" << std::endl;
}
```

---

## 十一、常见问题和解决方案

### 11.1 编译问题

**问题1：找不到 MPI**
```
CMake Error: Could not find MPI
```
**解决：**
```bash
sudo apt-get install libopenmpi-dev
# 或
sudo apt-get install mpich
```

**问题2：找不到 libsodium**
```
Package libsodium was not found
```
**解决：**
```bash
sudo apt-get install libsodium-dev
```

**问题3：C++ 标准不支持**
```
error: 'auto' not supported in C++98
```
**解决：**
确保 CMakeLists.txt 中设置了 C++14：
```cmake
set(CMAKE_CXX_STANDARD 14)
```

### 11.2 运行时问题

**问题1：进程数不匹配**
```
mpirun -np 2 ./main  # 错误！需要3个进程
```
**解决：**
```bash
mpirun -np 3 ./main  # 3方协议需要3个进程
mpirun -np 4 ./main  # 4方协议需要4个进程
```

**问题2：内存不足**
```
std::bad_alloc
```
**解决：**
- 减小数据大小
- 减小批大小
- 增加系统内存

**问题3：通信超时**
```
MPI_Recv timeout
```
**解决：**
- 检查网络连接
- 确保所有进程都在运行
- 检查防火墙设置

### 11.3 正确性问题

**问题：结果不正确**

**检查清单：**
1. 确保所有方使用相同的数据大小
2. 确保秘密共享的数据方ID正确
3. 检查是否有整数溢出
4. 验证随机数生成器的种子是否正确同步

**调试方法：**
```cpp
// 打开中间结果检查
auto intermediate = intermediate_result.open();
if (pID == 0) {
    std::cout << "Intermediate: ";
    for (int i = 0; i < intermediate.size(); i++) {
        std::cout << intermediate[i] << " ";
    }
    std::cout << std::endl;
}
```

---

## 十二、扩展和定制

### 12.1 添加新的安全操作

**步骤：**

1. **在协议类中添加原语**
```cpp
// 在 include/core/protocols/replicated_3pc.h
EVector my_new_primitive(const EVector &x, const EVector &y) {
    // 实现新的安全原语
    // ...
}
```

2. **在共享向量类中添加接口**
```cpp
// 在 include/core/containers/a_shared_vector.h
ASharedVector my_operation(const ASharedVector &other) {
    return ASharedVector(
        runTime.my_new_primitive<EVector::replicationNumber>(*this, other)
    );
}
```

3. **添加测试**
```cpp
// 在 tests/ 中创建测试文件
void test_my_operation() {
    Vector<int> data_a = {1, 2, 3};
    Vector<int> data_b = {4, 5, 6};

    ASharedVector<int> a = secret_share_a(data_a, 0);
    ASharedVector<int> b = secret_share_a(data_b, 0);

    ASharedVector<int> result = a.my_operation(b);
    Vector<int> opened = result.open();

    // 验证结果
    Vector<int> expected = /* 明文计算 */;
    assert(opened.same_as(expected));
}
```

### 12.2 添加新的查询

**创建新查询文件：**
```cpp
// src/queries/my_query.cpp
#include "../../include/tva.h"

using namespace tva::service::mpi_service::replicated_3pc;

int main(int argc, char** argv) {
    tva_init(argc, argv);

    // 1. 定义数据和表结构
    std::vector<std::string> schema = {"col1", "[col2]", "col3"};
    std::vector<Vector<int>> data(schema.size(), 1000);

    // 2. 秘密共享
    EncodedTable<int> table = secret_share(data, schema);

    // 3. 执行查询操作
    table.sort({"[col2]"}, {false}, {"col1", "col3"});
    table["result"] = table["col1"] * table["col3"];

    // 4. 打开结果
    auto result = table.open();

    MPI_Finalize();
    return 0;
}
```

**在 CMakeLists.txt 中添加：**
```cmake
# 自动包含 src/queries/ 中的所有 .cpp 文件
```

**编译运行：**
```bash
cd build
cmake ..
make my_query
mpirun -np 3 ./my_query
```

---

## 十三、最佳实践

### 13.1 性能优化建议

1. **选择合适的共享类型**
   - 算术运算多 → 使用 ASharedVector
   - 比较操作多 → 使用 BSharedVector
   - 减少 a2b/b2a 转换次数

2. **批处理优化**
   - 增大批大小以减少通信次数
   - 但不要超过缓存大小

3. **并行化**
   - 使用多线程处理大向量
   - 线程数 = CPU 核心数

4. **减少通信**
   - 合并多个操作
   - 使用本地操作（加法、XOR）而非通信操作（乘法、比较）

### 13.2 安全性建议

1. **使用正确的协议**
   - 半诚实环境 → Replicated 3PC
   - 恶意环境 → Fantastic Four

2. **保护随机数生成器**
   - 使用密码学安全的随机数
   - 确保种子的安全性

3. **验证输入**
   - 检查数据范围
   - 防止整数溢出

### 13.3 代码组织建议

1. **模块化**
   - 将复杂查询分解为小函数
   - 重用通用操作

2. **注释**
   - 说明安全属性
   - 标注通信轮次

3. **测试**
   - 为每个操作编写测试
   - 验证正确性和性能

---

## 十四、参考资源

### 14.1 论文和文档

- **TVA 论文**: [USENIX Security'23](https://www.usenix.org/conference/usenixsecurity23/presentation/faisal)
- **Replicated 3PC**: Araki et al. "High-Throughput Semi-Honest Secure Three-Party Computation"
- **Fantastic Four**: Dalskov et al. "Fantastic Four: Honest-Majority Four-Party Secure Computation With Malicious Security"

### 14.2 相关技术

- **MPI 教程**: [Open MPI Documentation](https://www.open-mpi.org/doc/)
- **CMake 教程**: [CMake Tutorial](https://cmake.org/cmake/help/latest/guide/tutorial/index.html)
- **C++ 模板**: [C++ Templates Guide](https://en.cppreference.com/w/cpp/language/templates)
- **libsodium**: [Libsodium Documentation](https://libsodium.gitbook.io/)

### 14.3 项目链接

- **GitHub**: (项目仓库地址)
- **许可证**: GNU AFFERO GENERAL PUBLIC LICENSE (v3.0)
- **注意**: 这是学术原型，不适合生产环境使用

---

## 十五、总结

### 15.1 核心要点

1. **TVA 是什么**
   - 安全多方计算系统
   - 专注于时间序列分析
   - 保护数据隐私

2. **如何使用**
   - 秘密共享数据
   - 执行安全操作
   - 打开结果

3. **关键技术**
   - 复制秘密共享
   - 算术/布尔共享
   - MPC 协议

### 15.2 学习路径

**初学者：**
1. 理解秘密共享基本概念
2. 运行简单测试（test_primitives）
3. 修改示例查询

**进阶：**
1. 实现自定义操作
2. 优化性能
3. 添加新协议

**高级：**
1. 研究协议安全性
2. 扩展到新应用场景
3. 贡献代码

### 15.3 下一步

- 阅读论文了解理论基础
- 运行所有测试熟悉系统
- 尝试实现自己的安全查询
- 参与社区讨论和贡献

---

**文档版本**: 1.0
**最后更新**: 2024
**作者**: TVA 项目文档团队

