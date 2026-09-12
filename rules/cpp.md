---
---
# C++ 规范

## 命名

| 种类 | 风格 | 示例 |
|------|------|------|
| 类/结构体 | `PascalCase` | `UserManager` |
| 公有函数/成员函数 | `PascalCase` | `GetName()` |
| 类私有函数 | `camelCase` | `getName()` |
| 静态变量 | `sPascalCase` | `sGetName()` |
| 变量 | `camelCase` | `userCount` |
| 私有成员变量 | `m_camelCase` | `m_name`、`m_pPointer` |
| 常量 | `kPascalCase` | `kMaxSize` |
| 枚举值 | `PascalCase` | `Red` |
| 类型别名 | `PascalCase` | `HandleId` |
| 命名空间 | `snake_case` | `my_project` |
| 宏 | `UPPER_SNAKE_CASE` | `MY_VERSION` |
| 文件名 | `PascalCase`，头文件与cpp文件同名即可 | `Header.h` 、 `Header.cpp` |

## 头文件

| 规则 | 说明 |
|------|------|
| 头文件保护 | `#pragma once` |
| Include 顺序 | 本项目 `.h` → 本项目 `""` → 第三方 → C++ 标准库 → C 库，组间空行 |

## 类

| 规则 | 说明 |
|------|------|
| 声明顺序 | `public` → `protected` → `private`，成员变量放最后 |
| 变量暴露方式 | 私有变量供外部使用一律使用`getter`方法获取，而非直接public变量 |

## 现代 C++

| 规则 | 说明 |
|------|------|
| C++标准 | C++20 |
| 空指针 | `nullptr`，不用 `NULL`/`0` |
| 常量 | `constexpr`，不用宏 |
| 枚举 | `enum class`，不用裸 enum |
| 类型别名 | `using`，不用 `typedef` |
| 类型转换 | `static_cast`/`dynamic_cast`，不用 C 风格 |
| 返回值不可忽略 | 函数返回值不应被丢弃时使用 `NODISCARD` 宏（定义于 `Utils/Macro.h`，展开为 `[[nodiscard]]`；如错误码、句柄、状态查询等），不直接写 `[[nodiscard]]` 属性字面量 |

## 代码封装抽象
- 魔法数字、重复字符串不散落字面量：定义为 `constexpr` 常量（`kPascalCase`）后复用
  - 反例：`m_capacity = 256;` 与 `new char[256]` 散落，256 含义不明
  - 正例：`constexpr size_t kCapacity = 256;` 定义一次，各处引用 `kCapacity`
- 简短重复的操作（函数/template 无法简洁表达时）定义为宏（`UPPER_SNAKE_CASE`）复用
  - 反例：对齐计算 `(x + 15) & ~15u` 重复写多遍
  - 正例：`#define ALIGN_UP(v, a) (((v) + ((a) - 1)) & ~((a) - 1))` 一处定义、多处调用
- 位操作相关代码都必须封装成宏定义（`UPPER_SNAKE_CASE`）复用，不散落裸位运算
  - 反例：散落各处的 `(value >> 4) & 0xF`、`flags |= 0x80`，魔法位掩码含义不明且易写错
  - 正例：`#define BIT_GET(v, b) (((v) >> (b)) & 1u)`、`#define BIT_SET(v, b) ((v) |= (1u << (b)))`、`#define BIT_CLEAR(v, b) ((v) &= ~(1u << (b)))` 一处定义、多处调用
- 有特殊意义的类型（句柄、ID、索引、计数等语义值）定义为 `using` 别名（`PascalCase`），不直接用裸类型
  - 反例：`uint32_t m_graphicsQueueFamilyIndex;` 与 `uint32_t m_graphicsQueueIndex;` 类型相同，语义混淆编译器无法发现
  - 正例：`using QueueFamilyIndex = uint32_t;`、`using QueueIndex = uint32_t;` 参数与成员的类型即语义，误传一眼可见
  - 边界：`using` 别名只是语义标注，类型系统上仍是原类型；需要编译器强制隔离时改用 `enum class` 或 wrapper 类型
- 函数体的执行内容必须与函数名一致：函数只做名称所表达的一件事，不做名外之事
  - 反例：`Update()` 名为"更新"却顺带做了资源加载、日志落盘、网络同步，名不副实
  - 正例：`Update()` 只更新状态，资源加载/日志/网络分别由 `LoadResource()`、`WriteLog()`、`SyncNetwork()` 承担
- 函数体内若存在适合独立成函数的代码段，优先封装成独立函数，函数体只保留编排逻辑
  - 反例：`Initialize()` 从创建实例到创建管线，把整个初始化流程平铺在一个函数里，包含
    实例创建、扩展校验、设备选择、队列族获取、交换链创建、渲染管线构建……每个环节的
    错误处理都混在一起，函数动辄上百行，无法单独复用或测试
  - 正例：`Initialize()` 只负责按序编排，每个环节拆成独立小函数，职责单一、可复用可单测：
    ```
    void Initialize() {
        m_instance   = CreateInstance();
        m_device     = CreateDevice(m_instance);
        m_swapchain  = CreateSwapchain(m_device);
        m_pipeline   = CreatePipeline(m_device);
    }
    ```
    `CreateInstance()` / `CreateDevice()` / `CreateSwapchain()` / `CreatePipeline()`
    各自只做一件事，错误处理内聚在自己内部

## 善用宏定义
- 枚举 → 值的映射 switch 中，单值映射的 case（`case X: return Y;`）用 `CASE_FROM_TO(X, Y)` 压缩，禁止逐 case 手写两行
  - 反例：`case ElementType::BYTE: return VK_FORMAT_R8_SNORM;` 十六行逐一手写
  - 正例：`CASE_FROM_TO(ElementType::BYTE, VK_FORMAT_R8_SNORM);`（宏来自 `Utils/Macro.h`，需 include）
  - `TO` 可为任意单个返回表达式，条件返回值同样适用，无需拆手写 case：`CASE_FROM_TO(ElementType::BYTE, integer ? VK_FORMAT_R8_SINT : VK_FORMAT_R8_SSCALED);` 等价于 `case ElementType::BYTE: return integer ? VK_FORMAT_R8_SINT : VK_FORMAT_R8_SSCALED;`
- 通用语句型宏（供普通代码调用、内含 `return` 或多条语句）必须用 `do { } while (0)` 包裹展开体，隔离作用域
  - 反例：`#define EARLY_RETURN(cond) if (cond) return;` —— 展开后与调用处外层 `if/else` 错配；条件成立时后续语句的归属被改变
  - 正例：`#define EARLY_RETURN(cond) do { if (cond) { return; } } while (0)` —— 宏整体等价单条语句，尾部可安全加 `;`，宏内变量不泄漏到调用方作用域
  - 含 `return` 的宏仅适用于无返回值语境（void 函数/构造函数/析构函数），使用前需确认；通用早退宏（如 `EARLY_RETURN`）放项目统一 `src/Macro.h` 一处定义
