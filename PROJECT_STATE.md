# Project State

## Current Goal

学习 ROS1 `fuse` 的核心数据流和插件机制，为后续实现二维 `WifiBearingConstraint` 与 `WifiBearingSensorModel` 建立可运行、可解释的基础。

## Current Task

阅读官方 Range Sensor Tutorial，重点追踪：

```text
range measurement
  -> RangeSensorModel
  -> RangeConstraint
  -> RangeCostFunctor
  -> Transaction
  -> FixedLagSmoother
  -> HashGraph / Ceres
```

项目上下文文档和 GitHub 同步通道已经建立，当前回到 Fuse 学习主线。

## Environment

- Host OS: Windows + WSL2
- WSL distribution: Ubuntu 24.04.4 LTS
- Kernel: `6.6.87.2-microsoft-standard-WSL2`
- Docker client: `29.1.3`
- Container base: `ros:noetic-ros-base-focal`
- ROS: ROS1 Noetic
- Container OS: Ubuntu 20.04 Focal（由基础镜像确定）
- Workspace on host: `/home/lenovo/wireless-Kimera-VIO/fuse_ws`
- Workspace in container: `/workspace`
- Repository root on host: `/home/lenovo/wireless-Kimera-VIO/fuse_ws/src/fuse`
- Public repository: `https://github.com/ManJieHongYe/fuse-wifi-slam`
- fuse branch: `devel`
- fuse version/tag: `0.15.0`
- Base commit: `8e3f0a1fef6f3b8af16a50f60bf4c11843bf4d81`
- Language/build: C++17, CMake, catkin tools
- Solver: Ceres Solver
- Current running container status: Codex execution environment cannot access the user's Docker daemon; 待用户终端确认

Important build commands used in the container:

```bash
cd /workspace
source /opt/ros/noetic/setup.bash
catkin config --extend /opt/ros/noetic --cmake-args -DCMAKE_BUILD_TYPE=Release
catkin build --force-cmake -j2 -p2
catkin build --catkin-make-args run_tests -j2 -p2
catkin_test_results --verbose
```

## Completed

- 选择 ROS1 `fuse` 作为无线 SLAM Demo 的图优化基座。
- 建立基于 ROS Noetic/Focal 的 Docker 构建环境。
- 补充构建依赖，包括 Ceres、Eigen、Boost、SuiteSparse、Qt5/Qwt、roslint、RViz、tf2_2d 和 diagnostic_updater。
- 修复本地 Ceres imported target/include path 兼容问题。
- 修复 Fuse 测试目标缺少 Boost Serialization 链接的问题。
- 修复 `stamped.h` 缺少显式 `<string>` include 的 roslint 问题。
- Fuse 的 13 个 catkin 包全部编译成功。
- 已运行测试：668 tests，0 errors，0 failures，0 skipped。
- 已在上游 README 末尾加入中文翻译。
- 已初步梳理 Fuse 的包结构和 Range Sensor Tutorial 数据流。
- 已创建项目上下文文档，并推送到 GitHub public 仓库。

## Current Understanding

- `SensorModel` 接收 ROS 测量并生成 `Transaction`。
- `Transaction` 原子地描述变量与约束的增加或删除，并携带时间戳信息。
- `Variable` 是 Ceres parameter block 对应的待估计状态。
- `Constraint` 保存变量 UUID，并通过 `CostFunction` 计算 residual。
- `MotionModel` 根据传感器事务涉及的时间戳补充时间方向的运动约束。
- `FixedLagSmoother` 缓存事务，周期性更新图、优化、通知插件并边缘化旧状态。
- `HashGraph` 保存变量和约束，并在每次优化时构造 Ceres `Problem`。
- 官方 Range Sensor 示例与未来无线约束在结构上高度相似，可作为第一版实现模板。
- `range_sensor_simulator` 在 `[-50, 50]` 范围内以 20 m 间隔生成 36 个带噪声先验的二维 beacon，发布 IMU、轮速里程计、range、真值和先验话题。
- `RangeSensorModel` 先缓存 `/prior_beacons`，再把每个 `/ranges` 消息转换成一个带机器人位置、地标变量、range 约束和首次地标先验的事务。
- range tutorial 的单次 range 约束只有一个残差，而机器人位置和 beacon 位置共有四个自由度，因此首次事务需要 beacon 先验来避免秩亏。
- 编译和单元测试成功不能替代教程运行验证；Range Sensor Tutorial 是否完整运行仍待确认。

## Modified / Important Files

- `README.md`：保留上游说明，并在末尾增加中文翻译。
- `fuse_core/CMakeLists.txt`：兼容不同 Ceres imported target，并公开链接 Boost Serialization。
- `fuse_variables/include/fuse_variables/stamped.h`：显式包含 `<string>`。
- `docker/Dockerfile.fuse`：可复现的 ROS Noetic/Focal 构建依赖。
- `fuse_tutorials/launch/range_sensor_tutorial.launch`：Range Sensor 示例入口。
- `fuse_tutorials/config/range_sensor_tutorial.yaml`：运动、传感器和发布器插件配置。
- `fuse_tutorials/src/range_sensor_model.cpp`：将 range 消息转换为变量和约束。
- `fuse_tutorials/include/fuse_tutorials/range_cost_functor.h`：range residual 的数学实现。
- `fuse_optimizers/src/fixed_lag_smoother.cpp`：事务队列、优化和边缘化主循环。
- `fuse_graphs/src/hash_graph.cpp`：图存储和 Ceres Problem 构造。

## Commands / Tests

Last verified build result:

```text
All 13 packages succeeded.
```

Last verified test result:

```text
Summary: 668 tests, 0 errors, 0 failures, 0 skipped
```

The `jobserver unavailable: using -j1` message was a parallel-build warning and did not cause test failures.

## Current Problems

- Range Sensor Tutorial 的完整运行和 ROS topic 输出尚未验证。
- 当前 Codex 执行环境没有可用的 ROS 命令，且无法访问用户的 Docker daemon；教程运行需要在用户的 WSL/Docker 终端确认。
- 用户仍在学习 Fuse 的核心对象和数据流，尚未进入 wireless constraint 编码阶段。
- 真实 CSI 数据格式、数据集接入方式和前端输出接口尚未确定。

## Decisions

- 第一版只做二维模型。
- 第一版无线 measurement 使用人为生成的 bearing。
- AP 首先建模为二维 landmark。
- 优先复用 Fuse 的 `SensorModel`、`Constraint`、`Transaction`、optimizer 和 graph。
- 在 fake bearing 约束验证前，不开展复杂 CSI、NLOS、3D 或工业实时部署工作。
- `LEARNING.md` 只在用户明确要求时修改。

## Next Steps

1. 在用户 WSL/Docker 终端运行 Range Sensor Tutorial，确认节点、topic、RViz/输出和优化结果。
2. 按调用顺序阅读 `RangeSensorModel`、`RangeConstraint` 和 `RangeCostFunctor`。
3. 继续追踪 `Transaction` 从 `sendTransaction()` 到 `FixedLagSmoother` 的调用链。
4. 在用户确认已掌握必要概念后，设计最小 fake bearing 消息和约束接口。

## Last Updated

2026-09-26 — 创建项目上下文文档并完成 GitHub public 仓库首次推送。
