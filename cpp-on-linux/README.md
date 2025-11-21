# Linux环境下的C++开发
## 一、环境配置与基础工具
### 1. 核心开发环境安装（支持 C++20+）
```bash
# 1. 更新软件源（Ubuntu/Debian 系）
sudo apt update && sudo apt upgrade -y

# 2. 安装编译器（gcc-11+ 支持 C++20 完整特性）
sudo apt install -y gcc-11 g++-11

# 3. 安装构建/版本控制/调试工具链
sudo apt install -y cmake git gdb valgrind perf

# 4. 验证环境（确保 C++20 支持）
g++-11 --version  # 需显示版本 ≥11.0
g++-11 --std=c++20 -o cpp20_test cpp20_test.cpp && ./cpp20_test
```

### 2. 基础环境验证代码（cpp20_test.cpp）
```cpp
#include <iostream>
#include <ranges>  // C++20 特性
int main() {
    auto nums = std::views::iota(1, 6) | std::views::transform([](int x) { return x * 2; });
    for (int x : nums) std::cout << x << " ";  // 输出：2 4 6 8 10
    return 0;
}
```

## 二、编译与构建实战
### 1. 手动编译（多文件项目入门）
```bash
# 假设仓库项目结构（以简易日志系统为例）
# logger_project/
#   ├── main.cpp
#   ├── logger.h
#   └── logger.cpp

# 编译命令（指定 C++20 标准 + 链接线程库）
g++-11 -std=c++20 main.cpp logger.cpp -o logger_app -lpthread -g

# 运行程序
./logger_app

# 清理编译产物
rm -rf logger_app
```

### 2. CMake 自动化构建
#### 2.1 项目根目录 CMakeLists.txt（适配 C++20）
```cmake
cmake_minimum_required(VERSION 3.20)
project(LoggerProject LANGUAGES CXX)

# 指定 C++ 标准（强制要求 C++20）
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# 指定编译器（可选，确保使用 g++-11）
set(CMAKE_CXX_COMPILER g++-11)

# 收集源文件
file(GLOB SOURCES "*.cpp")

# 生成可执行文件
add_executable(logger_app ${SOURCES})

# 链接依赖库（如线程库）
target_link_libraries(logger_app PRIVATE pthread)

# 启用调试信息
set(CMAKE_BUILD_TYPE Debug)
```

#### 2.2 CMake 构建命令
```bash
# 创建构建目录（分离源码与构建产物）
mkdir -p build && cd build

# 生成 Makefile（指定源码目录为上级目录）
cmake ..

# 并行编译（-j 后接 CPU 核心数，加速构建）
make -j4

# 运行程序
./logger_app

# 清理构建产物（保留 CMakeCache.txt）
make clean

# 彻底清理（删除所有构建文件）
cd .. && rm -rf build
```

## 三、调试与排错
### 1. GDB 调试 C++ 核心用法
#### 1.1 编译带调试信息的程序
```bash
g++-11 -std=c++20 -g main.cpp logger.cpp -o logger_app -lpthread
```

#### 1.2 GDB 常用命令实战
```bash
# 启动 GDB
gdb ./logger_app

# 1. 设置断点（支持函数、行号、类成员函数）
(gdb) break main                  # 主函数断点
(gdb) break logger.h:25           # 头文件第 25 行断点
(gdb) break Logger::log           # 类成员函数断点
(gdb) break Node::height          # 适配仓库 deducing this 示例

# 2. 运行程序
(gdb) run  # 可带命令行参数：run --config config.json

# 3. 单步执行
(gdb) next  # 下一步（不进入函数）
(gdb) step  # 下一步（进入函数）
(gdb) continue  # 继续运行到下一个断点

# 4. 查看变量与内存
(gdb) print msg                   # 打印变量值
(gdb) print self->left            # 查看 this 指针成员（适配 deducing this）
(gdb) x/10xw 0x7fffffffddd0       # 查看内存地址（10 个 32 位十六进制值）

# 5. 调用栈分析（排查崩溃问题）
(gdb) backtrace  # 简写 bt，查看函数调用链
(gdb) bt full    # 查看调用栈+局部变量

# 6. 条件断点（仅满足条件时触发）
(gdb) break Logger::log if msg.find("error") != std::string::npos

# 7. 退出 GDB
(gdb) quit
```

### 2. 内存泄漏检测（valgrind）
```bash
# 检测内存泄漏（--leak-check=full 显示详细泄漏信息）
valgrind --leak-check=full --show-leak-kinds=all ./logger_app

# 检测内存越界/使用未初始化变量
valgrind --tool=memcheck --track-origins=yes ./logger_app
```

## 四、性能分析与优化
### 1. 基础性能统计
```bash
# 统计程序运行时间、CPU 占用
time ./logger_app

# 统计程序资源占用（内存、CPU、IO 等）
/usr/bin/time -v ./logger_app
```

### 2. Perf 热点分析
```bash
# 1. 采样程序运行（-g 记录调用栈）
perf record -g ./logger_app

# 2. 查看性能报告（交互式界面，按方向键导航）
perf report

# 3. 常用操作（在 report 界面）
# - 按 Enter 查看函数调用栈
# - 按 Ctrl+C 退出
# - 筛选关键词：/std::ranges::filter

# 4. 生成火焰图（需安装 flamegraph 工具）
perf script | ./flamegraph.pl > perf.svg
```

### 3. 仓库特性性能对比（flat_map vs map）
```bash
# 编译性能测试程序
g++-11 -std=c++20 flat_map_benchmark.cpp -o flat_map_demo -O2

# 运行并统计时间
time ./flat_map_demo

# 对比 std::map 性能
g++-11 -std=c++20 map_benchmark.cpp -o map_demo -O2
time ./map_demo
```

## 五、常用开发效率工具
### 1. 代码搜索与文件操作
```bash
# 1. 递归搜索代码（查找 std::expected 用法）
grep -r "std::expected" ./awesome-modern-cpp-2025 --include="*.cpp" --include="*.h"

# 2. 查找文件（在项目中找日志相关文件）
find ./ -name "*logger*" -type f

# 3. 批量替换代码（将 old_func 替换为 new_func）
sed -i 's/old_func/new_func/g' $(grep -r "old_func" ./ --include="*.cpp" --include="*.h" | cut -d: -f1 | uniq)
```

### 2. 远程开发与部署
```bash
# 1. 远程登录服务器
ssh username@server_ip

# 2. 上传本地项目到服务器
scp -r ./awesome-modern-cpp-2025 username@server_ip:/home/username/projects/

# 3. 从服务器下载文件
scp username@server_ip:/home/username/projects/logger_app ./
```
