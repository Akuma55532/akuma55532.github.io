---
title: ego-planner源码阅读
date: 2026-09-17 15:24:39
tags:
---

今天来读一下ego-planner的源码，这个导航算法我觉得不好用，读懂了之后看看能不能改出自己的一版。

# launch 文件

分析一个ros工程首先从launch文件开始，那么ego-planner的launch文件夹如下。

~~~
launch
├─ advanced_param.xml
├─ default.rviz
├─ run_in_sim.launch
├─ rviz.launch
├─ simple_run.launch
└─ simulator.xml
~~~

### 各文件的分工

| 文件 | 作用 | 如何使用 |
|---|---|---|
| `run_in_sim.launch` | 启动仿真，手动指定目标 | 主入口 |
| `simple_run.launch` | 启动仿真，执行预设航点，同时打开 RViz | 主入口 |
| `advanced_param.xml` | 启动规划器，设置规划、建图和优化参数 | 被两个主入口包含 |
| `simulator.xml` | 启动地图、动力学、控制器、局部感知等节点 | 被两个主入口包含 |
| `rviz.launch` | 启动 RViz 并加载显示配置 | 独立启动或被包含 |
| `default.rviz` | 保存 RViz 的显示项、话题、视角和工具配置 | 由 RViz 加载，并非 launch 文件 |

#### run_in_sim.launch：手动目标仿真入口

它完成四件事：

1. 包含 `advanced_param.xml`，启动 `ego_planner_node`
2. 启动轨迹服务器 `traj_server`
3. 启动目标转换节点 `waypoint_generator`
4. 包含 `simulator.xml`，启动仿真环境。

主要配置如下：

| 参数 | 当前值 | 含义 |
|---|---|---|
| 地图尺寸 | `40 × 40 × 3 m` | 三维地图范围 |
| 里程计话题 | `/visual_slam/odom` | 规划器使用的无人机状态 |
| 最大速度 | `2.0 m/s` | 规划速度约束 |
| 最大加速度 | `3.0 m/s²` | 规划加速度约束 |
| 局部规划距离 | `7.5 m` | 沿全局参考轨迹选取局部目标的距离尺度 |
| `flight_type` | `1` | 手动目标模式 |

实际工程中最常用的参数变量：

| 参数 | 默认值 | 含义 |
|---|---|---|
| `grid_map/resolution` | 0.1 | 栅格地图的分辨率，ego将激光雷达的点云数据经过体素滤波之后被转换成一个个体素块，也就是说点云中的每一个点是以一个正方体来参与到后面的路径规划中的，这里的分辨率即是正方体的大小，单位是m |
| `grid_map/map_size_x` `grid_map/map_size_y` | 40 | 栅格地图的大小，如果要飞更远的目标点就要增大这个地图边界值，单位是m |
| `grid_map/obstacles_inflation` | 0.099 | 障碍物膨胀距离，单位为 m。由于规划时通常将无人机简化为一个点，而实际无人机具有一定体积，因此需要将障碍物向外膨胀，为机身预留避障空间。膨胀距离通常以无人机的外包络半径为基础，并增加一定的安全余量，具体数值应结合定位、感知和控制误差，通过实际测试调整。本次测试初步将膨胀距离设为无人机直径，以采用较保守的避障配置。 |
| `grid_map/ground_height` | -0.01 | 栅格地图底部在世界坐标系中的 Z 坐标，它设置的是地图底部边界，并不会生成地面障碍，所以通过提高这个参数来提高无人机飞行高度是不正确的。 |
| `grid_map/virtual_ceil_height` | 2.5 | 虚拟天花板，ego为了不让无人机太高，会给设置一个虚拟的天花板障碍来限制无人机规划的路径，和上面`ground_height`不同，这个参数是真的会生成障碍物，可以限制无人机的路径。 |
| `grid_map/visualization_truncate_height` | 2.4 | 地图可视化的高度截断阈值，单位是 m，使用世界坐标系中的 Z 坐标。一般将这个值设置在虚拟天花板的下面一点，这样可以隐藏高处（虚拟天花板）障碍在RViz中的显示，方便在 RViz 中观察内部轨迹。 |


那么其中最主要的节点就是`ego_planner_node`,`traj_server`,`waypoint_generator`这三个。

