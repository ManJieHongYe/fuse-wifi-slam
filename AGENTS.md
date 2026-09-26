# Fuse WiFi Project Instructions

## Project Goal

本项目同时是学习项目和开发项目。最终目标是在 ROS1 `fuse` 框架上实现一个无线信息辅助状态估计 / SLAM 的演示性 Demo，并研究以下数据链路：

```text
无线信号
  -> 无线空间特征
  -> measurement
  -> constraint / residual
  -> fuse graph
  -> 联合优化
```

第一阶段的重点是理解并实现无线观测进入现有图优化框架的过程，不研究新的通用 SLAM 后端。

## Current Environment

- Host: Windows + WSL2，Ubuntu 24.04.4 LTS
- Container target: Ubuntu 20.04 Focal + ROS1 Noetic
- Build system: catkin tools + CMake
- Main language: C++17
- Optimizer: Ceres Solver
- Repository root: `fuse_ws/src/fuse`
- Host workspace root: `/home/lenovo/wireless-Kimera-VIO`

具体镜像、依赖和已验证命令以 `PROJECT_STATE.md` 为准。

## Main Technical Route

### Phase 1 — Learn the fuse architecture

理解：

- `Variable`
- `Constraint`
- `CostFunction`
- `SensorModel`
- `Transaction`
- `Graph`
- `Optimizer`

### Phase 2 — Study the official range sensor tutorial

追踪：

```text
Range measurement
  -> RangeSensorModel
  -> RangeConstraint
  -> RangeCostFunctor
  -> Transaction
  -> Optimizer
```

### Phase 3 — Implement a fake wireless bearing measurement

实现：

```text
fake bearing
  -> WifiBearingSensorModel
  -> WifiBearingConstraint
  -> fuse graph
```

第一版使用二维机器人状态和二维 AP landmark。

### Phase 4 — Connect a real wireless frontend

在图约束链路验证后，再接入：

```text
CSI
  -> bearing / other spatial feature
  -> Wifi constraint
  -> fuse
```

## Development Principles

1. 把学习目标与代码改动同时记录清楚。
2. 不默认一次实现大量代码；优先采用能够独立解释和验证的小步骤。
3. 修改关键代码前，应先说明修改内容、原因、它在 fuse 架构中的位置以及输入输出。
4. 优先扩展 fuse 的插件接口，避免重新实现一套状态估计框架。
5. 第一阶段优先得到最小可运行 Demo。
6. 在 wireless constraint 验证之前，不优先研究复杂 CSI、NLOS、3D 或工业级实时部署。
7. 实现应尽量简单、透明、容易验证。
8. 每轮实质性阅读、开发、调试或实验完成后，检查并更新 `PROJECT_STATE.md`。
9. 解决具有复用价值的问题时，更新 `DEV_LOG.md`。
10. **只有用户明确要求修改 `LEARNING.md` 时，Codex 才能修改它。** Codex 可以主动提出学习路线调整建议，但不能自行落盘。
11. 只有项目总体目标、长期原则或主技术路线变化时，才更新本文件。
12. 未经验证的状态必须标记为“待确认”，不能凭推测写成已完成。

## Shared Context Files

- `AGENTS.md`：长期目标和工作规则。
- `PROJECT_STATE.md`：项目当前状态的唯一主要记录。
- `LEARNING.md`：由用户控制更新的学习路线图。
- `DEV_LOG.md`：值得长期保留的问题、根因和解决经验。

不要把同一段内容完整复制到多个文件。

## End-of-Task Maintenance

完成一轮实质性工作后：

1. 检查代码和运行结果。
2. 更新 `PROJECT_STATE.md` 中的当前任务、已完成事项、问题、关注文件和下一步。
3. 若本轮解决了可复用的问题，更新 `DEV_LOG.md`。
4. 若学习路线可能需要调整，只向用户提出建议；得到用户明确要求后才能更新 `LEARNING.md`。
5. 提交前检查 diff，确保没有凭据、Token、本机隐私信息或生成文件。
