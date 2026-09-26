# fuse

The fuse stack provides a general architecture for performing sensor fusion live on a robot. Some possible applications
include state estimation, localization, mapping, and calibration.

## Overview

fuse is a ROS framework for performing sensor fusion using nonlinear least squares optimization techniques. In
particular, fuse provides:

* a plugin-based system for modeling sensor measurements
* a similar plugin-based system for motion models
* a plugin-based system for publishing optimized state values
* an extensible state variable definition
* a "contract" on how an optimizer will interact with the above components
* and some common implementations to get everyone started

(unpresented) ROSCon 2018 Lightning Talk [slides](doc/fuse_lightning_talk.pdf)

Data flows through the system approximately like this:

* A sensor model receives raw sensor data. The sensor model generates a constraint and sends it to the optimizer.
* The optimizer receives the new sensor constraint. A request is sent to each configured motion model to generate
  a constraint between the previous state and the new state involved in the sensor constraint.
* The motion model receives the request and generates the required constraints to connect the new state to the
  previously generated motion model chain. The motion model constraints are sent to the optimizer.
* The optimizer adds the new sensor model and motion model constraints and variables to the graph and
  computes the optimal values for each state variable.
* The optimal state values are sent to each configured publisher (as well as the sensor models and motion models).
* The publishers receive the optimized state values and publish any derived quantities on ROS topics.
* Repeat

It is important to note that much of this flow happens asynchronously in practice. Sensors are expected to operate
independently from each other, so each sensor will be sending constraints to the optimizer at its own frequency. The
optimizer will cache the constraints and process them in small batches on some schedule. The publishers may
require considerable processing time, introducing a delay between the completion of the optimization cycle and the
publishing of data to the ROS topic.

![fuse sequence diagram](doc/fuse_sequence_diagram.png)

## Example

Let's consider a simple robotics example to illustrate this. Assume we have a typical indoor differential-drive robot.
This robot has wheel encoders and a horizontal laser.

The first thing we must do is define our state variables. At a minimum, we want the robot pose at each timestamp.
We model the pose using a 2D position and an orientation. Each 2D pose _at a specific time_ gets a unique variable
name. For ease of notation, let's call the pose variables `X1`, `X2`, `X3`, etc. (In reality, each variable gets
a UUID, but those are much harder to write down.) Each 2D pose is instantiated as a `example_robot::Pose2D` which is
derived from the `fuse_core::Variable` base class. (`fuse` ships with several basic variables, such as 2D and 3D
versions of position, orientation, and velocity variables, but you can derive your own variable types as you need them.)

Next we need to decide how to model our sensors. We can model the wheel encoders as providing an incremental
pose measurement. Given a starting pose, `X1`, and a wheel encoder delta, `z`, we predict the pose `X2'` using some
measurement function `f`.

`X2' = f(X1, z)`

The error term for our constraint is the difference between the predicted pose `X2'` and the actual pose `X2`

`error = X2'^-1 * X2`

where `X2'^-1` is the inverse of the pose `X2'`

We derive a `fuse_core::Constraint` that implements that error function. Similarly, we perform scan-to-scan matching
using out laser data and create an incremental pose constraint between consecutive scans.

In the simplest example, the sensors are synchronized, i.e. the laser and the wheel encoders are sampled at the same
time. This is enough to construct our first `fuse` system. Below is the constraint graph generated from this first
system. The large circles represent state variables at a given time, while the small squares represent measurements.
The graph connectivity indicates which variables are involved in what measurements.

![fuse graph](doc/fuse_graph_1.png)

The two sensor models are configured as plugins to an optimizer implementation. The optimizer performs the required
computation to generate the optimal state variable values based on the provided sensor constraints. We will never be
able to exactly satisfy both the wheel encoder constraints and the laserscan constraints. Instead we minimize the error
of all the constraints using nonlinear least squares optimization.

![fuse optimizer](doc/fuse_optimizer_1.png)

While our `fuse` system is optimizing constraints from two different sensors, it is not yet publishing any data back
out to ROS. In order to publish data to ROS, we derive a `fuse_core::Publisher` class and add it to the
optimizer. Derived publishers have access to the optimized values of all state variables. The specific publisher
implementation determines what type of messages are published and at what frequency. For our example system,
we would like visualize the current pose of the robot in RViz, so we create a `fuse` publisher that finds the most
recent pose and converts it into a `geometry_msgs::PoseStamped` message, then publishes the message to a topic.

![fuse optimizer](doc/fuse_optimizer_2.png)

We finally have something that is starting to be useful.

### Adaptation #1: Asynchronous sensors

Typically the laser measurements and the wheel encoder measurements are not synchronized. The encoder measurements are
sampled faster than the laser, and are sampled at different times using a different clock. If we do not do anything
different in this situation, the constraint graph becomes disconnected.

![fuse graph](doc/fuse_graph_2.png)

This is where motion models come into play. A motion model differs from a sensor model in that constraints can be
generated between any two requested timestamps. Motion model constraints are generated upon request, not due to their
own internal clock. We use the motion model to connect the states introduced by the other sensor measurements. We
derive a class from the `fuse_core::MotionModel` base class and implement a differential drive kinematic
constraint for our robot.

![fuse optimizer](doc/fuse_optimizer_3.png)

The motion models are also configured as plugins to the optimizer. The optimizer requests motion models constraints
from the configured plugins whenever new states are created by the sensor models.

![fuse graph](doc/fuse_graph_3.png)

### Adaptation #2: Full path publishing

Nothing about the `fuse` framework limits you to having a single publisher. What if you want to visualize the entire
robot trajectory, instead of just the most recent pose? Well, we can create a new derived `fuse_core::Publisher` class
that publishes all of the robot poses using a `nav_msgs::Path` message.

![fuse optimizer](doc/fuse_optimizer_4.png)

### Adaptation #3: Changing kinematics

In your spare time, you also build [autonomous power wheels racers](http://www.powerracingseries.org/). But race cars
don't use differential drive; you need a different motion model. Easy enough. We simply derive a new
`fuse_core::MotionModel` class that implements an Ackermann steering model. Everything else can be reused.

![fuse optimizer](doc/fuse_optimizer_5.png)

![fuse graph](doc/fuse_graph_4.png)

### Adaptation #4: Online calibration

Over time you notice that the accuracy of the odometry measurements is decreasing. After some investigation you realize
that the soft rubber racing tires are wearing, decreasing the diameter of the wheels over time. It sure would be nice
if the odometry system could compensate for that automatically. To do that, we derive a new variable type from
`fuse_core::Variable` that holds a single scalar value representing a wheel diameter at a specific point in time. For
ease of notation, we refer to this new variable as `D1, D2, ...`, etc. We also need to derive a new wheel encoder sensor
model from the `fuse_core::SensorModel` base class. This new sensor model involves the previous pose and next pose as
before, but it also involves the previous wheel diameter. Finally, we need a `fuse_core::MotionModel` that describes
how the wheel diameter is expected to change over time. Maybe some sort of exponential decay? And for good measure, we
derive a new publisher plugin from `fuse_core::Publisher` that publishes the current wheel diameter. This allows us to
plot how the wheel diameter changes over the length of the race.

![fuse optimizer](doc/fuse_optimizer_6.png)

![fuse graph](doc/fuse_graph_5.png)

Now our system estimates the wheel diameters at each time step as well as the robot's pose.

## The Math

Internally `fuse` uses Google's [Ceres Solver](http://ceres-solver.org) to perform the nonlinear least squares
optimization, which produces the optimal state variable values. I direct any interested parties to the Ceres Solver
["Non-linear Least Squares"](http://ceres-solver.org/nnls_tutorial.html) tutorial for an excellent primer on the core
concepts and involved math.

## Summary

The purpose of `fuse` is to provide a framework for performing sensor fusion tasks, allowing common components to be
reused between systems, while also allowing components to be customized for different use cases. The goal is to allow
end users to concentrate on modeling the robot, sensor, system, etc. and spend less time wiring the different
sensor models together into runable code. And since all of the models are implemented as plugins, separate plugin
libraries can be shared or kept private at the discretion of their authors.

## API Concepts

* [Variables](doc/Variables.md)
* [Constraints](doc/Constraints.md)
* Sensor Models -- coming soon
* Motion Models -- coming soon
* Publishers -- coming soon
* Optimizers -- coming soon

---

# 中文翻译

## fuse

fuse 软件栈提供了一套通用架构，用于在机器人运行过程中实时执行传感器融合。它可以用于状态估计、定位、建图和标定等任务。

## 概览

fuse 是一个 ROS 框架，使用非线性最小二乘优化技术执行传感器融合。具体来说，fuse 提供：

* 基于插件的传感器测量建模系统；
* 类似的、基于插件的运动模型系统；
* 基于插件的优化状态发布系统；
* 可扩展的状态变量定义；
* 优化器与上述组件交互时所遵循的“契约”；
* 一些通用实现，帮助用户快速开始开发。

（未附上）ROSCon 2018 闪电演讲[幻灯片](doc/fuse_lightning_talk.pdf)

数据在系统中的流动过程大致如下：

* 传感器模型接收原始传感器数据，生成一个约束并将其发送给优化器；
* 优化器接收新的传感器约束，请求每个已配置的运动模型，为传感器约束涉及的前一状态和新状态生成约束；
* 运动模型接收请求，生成所需的约束，将新状态连接到此前生成的运动模型链。运动模型约束随后被发送给优化器；
* 优化器将新的传感器模型约束、运动模型约束和变量加入图中，并计算每个状态变量的最优值；
* 最优状态值被发送给每个已配置的发布器，同时也会发送给传感器模型和运动模型；
* 发布器接收优化后的状态值，在 ROS 话题上发布派生数据；
* 重复上述过程。

需要注意的是，实际运行时其中许多步骤是异步进行的。传感器应当彼此独立运行，因此每个传感器会按照自己的频率向优化器发送约束。优化器会缓存这些约束，并按照一定的调度周期，以小批量方式处理它们。发布器可能需要较长的处理时间，从而使优化周期完成和数据发布到 ROS 话题之间出现延迟。

![fuse 时序图](doc/fuse_sequence_diagram.png)

## 示例

下面通过一个简单的机器人示例说明这一过程。假设我们有一台典型的室内差速驱动机器人，机器人配备轮编码器和水平激光雷达。

首先必须定义状态变量。至少，我们需要定义机器人在每个时间戳对应的位姿。我们使用二维位置和一个方向来表示位姿。每个**特定时间点的二维位姿**都有一个唯一的变量名。为了便于说明，将这些位姿变量称为 `X1`、`X2`、`X3` 等。（实际上，每个变量都会获得一个 UUID，但 UUID 不便于书写。）每个位姿都实例化为 `example_robot::Pose2D`，该类派生自 `fuse_core::Variable` 基类。（`fuse` 自带一些基本变量，例如二维和三维的位置、方向和速度变量；你也可以根据需要派生出自己的变量类型。）

接下来需要决定如何对传感器建模。我们可以将轮编码器建模为提供增量位姿测量。给定初始位姿 `X1` 和轮编码器增量 `z`，通过某个测量函数 `f`，可以预测位姿 `X2'`：

`X2' = f(X1, z)`

约束的误差项是预测位姿 `X2'` 与实际位姿 `X2` 之间的差异：

`error = X2'^-1 * X2`

其中，`X2'^-1` 表示位姿 `X2'` 的逆。

我们派生一个 `fuse_core::Constraint`，实现上述误差函数。同样地，我们使用激光雷达进行帧间匹配，并在相邻扫描之间创建增量位姿约束。

在最简单的示例中，传感器是同步的，也就是说，激光雷达和轮编码器在同一时刻采样。这样就足以构建第一个 `fuse` 系统。下面展示了由这个系统生成的约束图。大圆表示某个时间点的状态变量，小方块表示测量。图中的连接关系表示每个测量涉及哪些变量。

![fuse 图](doc/fuse_graph_1.png)

两个传感器模型被配置为某个优化器实现的插件。优化器根据传入的传感器约束，执行必要的计算，得到最优状态变量值。我们不可能同时完全满足轮编码器约束和激光扫描约束，因此需要使用非线性最小二乘优化，最小化所有约束的总体误差。

![fuse 优化器](doc/fuse_optimizer_1.png)

虽然我们的 `fuse` 系统正在优化两个不同传感器的约束，但它还没有向 ROS 发布任何数据。为了向 ROS 发布数据，我们派生一个 `fuse_core::Publisher` 类，并将其添加到优化器中。派生出的发布器可以访问所有状态变量的优化值。具体的发布器实现决定发布哪种消息以及发布频率。对于当前示例，我们希望在 RViz 中显示机器人的当前位姿，因此创建一个 `fuse` 发布器，找到最新位姿，将其转换为 `geometry_msgs::PoseStamped` 消息，然后发布到一个话题。

![fuse 优化器](doc/fuse_optimizer_2.png)

现在，我们的系统已经开始具备实用价值。

### 改进一：异步传感器

在实际情况下，激光测量和轮编码器测量通常并不同步。编码器的采样频率高于激光雷达，而且两者使用不同的时钟，在不同时间采样。如果对此不做任何处理，约束图就会变得不连通。

![fuse 图](doc/fuse_graph_2.png)

这正是运动模型发挥作用的地方。运动模型与传感器模型不同，它可以在任意两个请求的时间戳之间生成约束。运动模型约束是在收到请求时生成的，而不是由运动模型自身的内部时钟触发。我们使用运动模型连接其他传感器测量引入的状态。具体做法是从 `fuse_core::MotionModel` 基类派生一个类，并为机器人实现差速驱动运动学约束。

![fuse 优化器](doc/fuse_optimizer_3.png)

运动模型同样被配置为优化器插件。当传感器模型创建新状态时，优化器会向已配置的插件请求运动模型约束。

![fuse 图](doc/fuse_graph_3.png)

### 改进二：发布完整轨迹

`fuse` 框架并不限制只能使用一个发布器。如果希望显示整个机器人轨迹，而不是只显示最新位姿，该怎么办？我们可以派生一个新的 `fuse_core::Publisher` 类，使用 `nav_msgs::Path` 消息发布机器人的全部位姿。

![fuse 优化器](doc/fuse_optimizer_4.png)

### 改进三：更换运动学模型

在业余时间，你还制造[自动驾驶动力车赛车](http://www.powerracingseries.org/)。但赛车并不使用差速驱动，因此需要一个不同的运动模型。这很容易实现：只需派生一个新的 `fuse_core::MotionModel` 类，实现 Ackermann 转向模型即可，其他部分都可以复用。

![fuse 优化器](doc/fuse_optimizer_5.png)

![fuse 图](doc/fuse_graph_4.png)

### 改进四：在线标定

随着时间推移，你发现里程计测量的精度在下降。经过调查，你发现软橡胶赛车轮胎正在磨损，导致车轮直径随时间减小。如果里程计系统能够自动补偿这一变化，就很有帮助。

为此，我们从 `fuse_core::Variable` 派生一种新的变量类型，用一个标量表示某个时间点的车轮直径。为了便于说明，将这些新变量称为 `D1`、`D2` 等。我们还需要从 `fuse_core::SensorModel` 基类派生一个新的轮编码器传感器模型。这个新传感器模型与之前一样涉及前一位姿和后一位姿，但还会涉及前一时刻的车轮直径。最后，我们需要一个 `fuse_core::MotionModel`，描述车轮直径随时间的变化规律，例如某种指数衰减模型。作为补充，我们还可以从 `fuse_core::Publisher` 派生一个新的发布器插件，发布当前车轮直径。这样就可以绘制整场比赛中车轮直径的变化曲线。

![fuse 优化器](doc/fuse_optimizer_6.png)

![fuse 图](doc/fuse_graph_5.png)

现在，我们的系统不仅能够估计机器人在每个时间步的位姿，还能估计每个时间步的车轮直径。

## 数学原理

在内部，`fuse` 使用 Google 的 [Ceres Solver](http://ceres-solver.org) 执行非线性最小二乘优化，并得到最优状态变量值。对于感兴趣的读者，建议阅读 Ceres Solver 的[“非线性最小二乘”教程](http://ceres-solver.org/nnls_tutorial.html)，其中对核心概念和相关数学原理进行了很好的介绍。

## 总结

`fuse` 的目标是提供一个用于执行传感器融合任务的框架，使不同系统之间能够复用通用组件，同时允许针对不同使用场景定制组件。它希望用户可以专注于对机器人、传感器和系统进行建模，而不必花费过多时间将不同的传感器模型连接成可运行的代码。由于所有模型都以插件形式实现，插件库的作者可以根据需要共享插件库，也可以将其保持为私有库。

## API 概念

* [变量](doc/Variables.md)
* [约束](doc/Constraints.md)
* 传感器模型——即将推出
* 运动模型——即将推出
* 发布器——即将推出
* 优化器——即将推出
