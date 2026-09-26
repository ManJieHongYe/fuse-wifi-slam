# Learning Roadmap

> Maintenance rule: Codex may suggest changes to this roadmap, but must not modify this file unless the user explicitly requests the modification.

## Current Stage

Stage 1 — Fuse 整体数据流（进行中）

阶段是否完成由用户确认，不能仅根据代码修改或测试结果自动判断。

## Stage 1 — Fuse Overall Data Flow

### Goal

理解：

```text
Sensor
  -> SensorModel
  -> Transaction
  -> Variable + Constraint
  -> Graph
  -> Optimizer
  -> Ceres
```

### Must Understand

- `Variable` 是什么。
- `Constraint` 是什么。
- `SensorModel` 做什么。
- `Transaction` 为什么存在。
- `Graph` 和 `Optimizer` 分别负责什么。

### Not Required Yet

- Ceres 内部数值求解算法。
- ROS 底层通信实现。
- Fuse 的所有插件。

### Completion Criterion

用户能够独立解释一条传感器 measurement 如何进入 Fuse graph。

## Stage 2 — Constraint and Ceres

### Goal

理解：

```text
measurement
  -> prediction h(x)
  -> residual
  -> CostFunction
```

### Must Understand

- parameter block
- residual
- `CostFunction`
- automatic differentiation
- covariance and weighting

### Completion Criterion

看到一个 Fuse `Constraint` 时，能够指出：

- 它连接哪些 `Variable`；
- measurement 是什么；
- residual 是什么；
- `CostFunction` 在计算什么。

## Stage 3 — Range Sensor Tutorial

### Key Files

- `fuse_tutorials/include/fuse_tutorials/range_sensor_model.h`
- `fuse_tutorials/src/range_sensor_model.cpp`
- `fuse_tutorials/include/fuse_tutorials/range_constraint.h`
- `fuse_tutorials/src/range_constraint.cpp`
- `fuse_tutorials/include/fuse_tutorials/range_cost_functor.h`

### Goal

完整追踪：

```text
range measurement
  -> SensorModel
  -> Constraint
  -> CostFunctor
  -> Transaction
  -> Optimizer
```

### Completion Criterion

用户能够不看代码画出 Range Sensor 的完整数据流。

## Stage 4 — Wireless Bearing Constraint

### Fake Input

```text
AP ID
bearing
sigma
timestamp
```

### Components

- `WifiBearingSensorModel`
- `WifiBearingConstraint`
- `WifiBearingCostFunctor`

### 2D Model

Robot state:

```text
(x_r, y_r, theta_r)
```

AP state:

```text
(x_a, y_a)
```

Predicted bearing:

```text
beta_hat = atan2(y_a - y_r, x_a - x_r) - theta_r
```

Weighted residual:

```text
r = wrap(beta_hat - beta_measurement) / sigma
```

### Completion Criterion

Fake bearing 能够通过 Fuse graph 对机器人或 landmark 状态产生合理约束。

## Stage 5 — Wireless SensorModel

### Goal

从 ROS topic 接收无线测量：

```text
ROS message
  -> WifiBearingSensorModel
  -> find/create Variable
  -> WifiBearingConstraint
  -> Transaction
```

### Completion Criterion

系统能够实时向 Fuse graph 添加无线约束。

## Stage 6 — CSI Frontend

只有前面阶段完成后，再研究：

```text
CSI
  -> wireless spatial feature
  -> bearing / range / fingerprint / other
  -> Constraint
```

Possible references:

- P2SLAM / P2SLAM-sim
- WiFi CSI AoA or bearing
- Radio Map
- Channel Knowledge Map
- Other wireless spatial representations
