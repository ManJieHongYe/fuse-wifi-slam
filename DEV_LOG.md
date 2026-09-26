# Development Log

本文件只记录具有复用价值的问题、根因和解决过程，不作为每日流水账。

## 2026-09-26 — Build fuse 0.15.0 in a ROS Noetic/Focal container

### Context

在 WSL2 的 Ubuntu 24.04 主机上，通过基于 Ubuntu 20.04 Focal 的 ROS Noetic Docker 环境编译和测试 Fuse 0.15.0。

### Symptom

构建过程中依次出现：

- `rosdep`/APT 找不到 `libbenchmark-dev`、`qtbase5-dev`、`libqwt-qt5-dev`。
- CMake 找不到 `roslint`。
- CMake 找不到 `tf2_2d`。
- CMake 找不到 `diagnostic_updater`。
- `fuse_core` 引用 `Ceres::ceres` 或 Ceres include path 时出现兼容问题。
- 测试目标链接时报 `libboost_serialization` 的 `DSO missing from command line`。
- roslint 报告 `stamped.h` 使用 `string` 时缺少显式 `#include <string>`。

### Initial Hypothesis

最初怀疑主要是依赖没有完整安装。补齐依赖后，仍存在 CMake target、传递链接依赖和头文件自包含问题。

### Investigation

检查了：

- ROS Noetic Docker 基础镜像对应的 Ubuntu 版本和 APT 仓库。
- 各包的 `package.xml` 和 `CMakeLists.txt`。
- Ceres 导出的 imported target 名称。
- `fuse_core` 测试程序的链接错误。
- `catkin build` 日志和 `catkin_test_results --verbose`。

### Root Cause

问题由多个独立因素共同造成：

1. 基础镜像没有安装 Fuse 全部构建、测试和可视化依赖。
2. 不同 Ceres 版本可能导出 `ceres` 或 `Ceres::ceres`，源码假设不完全兼容当前环境。
3. Boost Serialization 只作为私有链接依赖时，部分下游测试目标没有得到所需符号。
4. `stamped.h` 依赖其他头文件间接包含 `<string>`，roslint 要求头文件自身声明其直接依赖。

### Solution

- 在 Dockerfile 中启用 Ubuntu `universe`，并安装完整依赖。
- 在 `fuse_core/CMakeLists.txt` 中检测 `Ceres::ceres` 和 `ceres` 两种 target。
- 将 Boost Serialization 作为 `fuse_core` 的 public link dependency。
- 在 `fuse_variables/include/fuse_variables/stamped.h` 中显式加入 `<string>`。
- 重新强制运行 CMake、构建所有包并执行测试。

### Result

```text
All 13 packages succeeded.
Summary: 668 tests, 0 errors, 0 failures, 0 skipped
```

### Lesson

- ROS1 工程在容器中构建时，要同时固定 ROS 发行版、Ubuntu 版本和系统库版本。
- imported target 名称和传递链接属性是跨版本 CMake 兼容的常见故障点。
- 编译成功之后仍需运行测试和 lint；二者能够发现不同类型的问题。
- 每次新开容器 shell 后，应重新 source ROS 和 catkin workspace 环境。

### Related Files

- `docker/Dockerfile.fuse`
- `fuse_core/CMakeLists.txt`
- `fuse_variables/include/fuse_variables/stamped.h`
