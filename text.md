

## 第六章 代码总结与3D建图

### 6.1 代码的全面总结

### 6.2 Cartographer的优缺点分析

#### 6.2.1 优点

代码架构十分优美
各个模块独立性很强, 可以很方便的进行修改, 或则是单独拿出来做其他应用
代码鲁棒性非常高, 很少出现莫名崩掉的情况, 错误提示很好
代码命名非常规范, 能够清楚的根据变量名与函数名判断其代表的含义

总之, cartographer的代码十分值得学习与借鉴.

#### 6.2.2 缺点

**点云的预处理**
发生的拷贝次数太多
自适应体素滤波如果参数不好时计算量太大

**位姿推测器**

可能有问题的点

- 计算pose的线速度与角速度时, 是采用的数据队列开始和末尾的2个数据计算的
- 计算里程计的线速度与角速度时, 是采用的数据队列开始和末尾的2个数据计算的
- 使用里程计, 不使用imu时, **计算里程计的线速度方向**和**姿态的预测**时, 用的是里程计数据队列开始和末尾的2个数据的平均角速度计算的, **时间长了就不准**
- 不使用里程计, 不使用imu时, 用的是pose数据队列开始和末尾的2个数据的平均角速度计算的, **时间长了就不准**
- **添加位姿时, 没有用pose的姿态对imu_tracker_进行校准, 也没有对整体位姿预测器进行校准, 只计算了pose的线速度与角速度**
- 从代码上看, cartographer认为位姿推测器推测出来的位姿与姿态是准确的

可能的改进建议

- pose的距离越小, 匀速模型越能代替机器人的线速度与角速度, 计算pose的线速度与角速度时, 可以考虑使用最近的2个数据进行计算

- 里程计距离越短数据越准, 计算里程计的线速度与角速度时, 可以考虑使用最近的2个数据进行计算

- 使用里程计, 不使用imu时, 计算里程计的线速度方向时, 可以考虑使用里程计的角度进行计算

- 使用里程计, 不使用imu时, 进行姿态的预测时, 可以考虑使用里程计的角度进行预测

- 不使用里程计, 不使用imu时, 可以考虑用最近的2个pose计算线速度与角速度

- 使用pose对imu_tracker_的航向角进行校准

**基于Ceres的扫描匹配**

可能有问题的点

- 平移和旋转的残差项是逼近于先验位姿的, 当先验位姿不准确时会产生问题

可能的改进建议

- 先将地图的权重调大, 平移旋转的权重调小, 如 1000, 1, 1, 或者 100, 1, 1
- 调参没有作用的时候可以将平移和旋转的残差项注释掉

**后端优化**

优化时的计算量太大, 可以根据自己需求调整参数, 或者增加计算前的过滤.

在计算子图间约束的时候, 目前cartographer是根据节点个数来做的, 定位时又根据时间来决定是否进行全子图的匹配, 这部分计算的判断可以根据自己的需求增加一些, 以减少计算量.

### 6.3 TSDF地图

#### TSDF地图与ProbabilityGrid地图的区别

TSDF2D类继承了Grid2D类

具体的栅格值保存在Grid2D里的correspondence_cost_cells_中, 只不过这里保存的不再是空闲的概率了. 而是tsd值转成的value.

```c++
ProbabilityGrid::ProbabilityGrid(const MapLimits& limits,
                                 ValueConversionTables* conversion_tables)
    : Grid2D(limits, kMinCorrespondenceCost, kMaxCorrespondenceCost,
             conversion_tables),
      conversion_tables_(conversion_tables) {}

/**
 * @brief 构造函数
 * 
 * @param[in] limits 地图坐标信息
 * @param[in] truncation_distance 0.3
 * @param[in] max_weight 10.0
 * @param[in] conversion_tables 转换表
 */
TSDF2D::TSDF2D(const MapLimits& limits, float truncation_distance,
               float max_weight, ValueConversionTables* conversion_tables)
    : Grid2D(limits, -truncation_distance, truncation_distance,
             conversion_tables),
      conversion_tables_(conversion_tables),
      value_converter_(absl::make_unique<TSDValueConverter>(
          truncation_distance, max_weight, conversion_tables_)),
      weight_cells_(
          limits.cell_limits().num_x_cells * limits.cell_limits().num_y_cells,
          value_converter_->getUnknownWeightValue()) {}
```

可以看到, ProbabilityGrid地图的栅格值的最大最小分别是 0.9 与 0.1, 而 TSDF地图的上遏制的最大最小分别是 0.3 与 -0.3.

TSDF地图保存tsd值的同时还保存了权重值, 权重值保存在TSDF2D类的weight_cells_中.

获取TSDF地图栅格值是通过TSDF2D::GetTSDAndWeight获取栅格值的, 同时获取到TSD值与权重值.

#### 栅格值更新的方式

新的权重 = 之前的weight + 新的weight
新的tsd值 = (之前的tsd值 * 之前的weight + 新的tsd值 * 新的weight) / (新的权重)

```c++
// TSDF地图栅格的更新, 分别更新tsd值与权重值
void TSDFRangeDataInserter2D::UpdateCell(const Eigen::Array2i& cell,
                                         float update_sdf, float update_weight,
                                         TSDF2D* tsdf) const {
  if (update_weight == 0.f) return;
  // 获取TSD值与权重值
  const std::pair<float, float> tsd_and_weight = tsdf->GetTSDAndWeight(cell);
  float updated_weight = tsd_and_weight.second + update_weight;
  float updated_sdf = (tsd_and_weight.first * tsd_and_weight.second +
                       update_sdf * update_weight) /
                      updated_weight;
  updated_weight =
      std::min(updated_weight, static_cast<float>(options_.maximum_weight()));
  tsdf->SetCell(cell, updated_sdf, updated_weight);
}
```

#### 相关性扫描匹配时使用TSDF计算得分

#### TSDF地图的扫描匹配
InterpolatedTSDF2D
CreateTSDFMatchCostFunction2D

### 6.4 3D网格地图

ActiveSubmaps3D
Submap3D
HybridGrid

就是用三维网格替换了二维网格, 其余是差不多的.

和2D不同的是, 地图里保存的是odd, 而不是costodd, 即是占用的概率, 而不是miss的概率.


### 6.5 3D扫描匹配

#### LocalTrajectoryBuilder3D

#### RealTimeCorrelativeScanMatcher3D
首先分别对 xyz 与 绕xyz的旋转 这6个维度进行遍历, 生成所有的可能解
对所有的可能解进行打分, 选出最高分的解

#### CeresScanMatcher3D

基本上与2D是一样的, 只是地图这的残差变了.

OccupiedSpaceCostFunction3D
地图变成2个了, 一个高分辨率地图, 一个低分辨率地图.

地图残差的计算基本也是一样的, 就是拿点云对应的栅格值当做残差, 只不过作为残差的是(1. - probability).

InterpolatedGrid
手动实现了双三次插值

#### 将点云插入到三维网格地图里

旋转直方图

推荐2个文章

cartographer 3D scan matching 理解
[https://www.cnblogs.com/mafuqiang/p/10885616.html](https://www.cnblogs.com/mafuqiang/p/10885616.html)

Cartographer源码阅读3D-Submap创建 
[https://blog.csdn.net/yeluohanchan/article/details/109462508?spm=1001.2014.3001.5501](https://blog.csdn.net/yeluohanchan/article/details/109462508?spm=1001.2014.3001.5501)


插入器 RangeDataInserter3D

插入的时候, 不是调用的RayCasting方法, 更简单粗暴, 直接计算光线的向量, 沿着向量搜索经过的网格, 并更新经过的网格的概率

### 6.6 3D后端优化

#### PoseGraph3D

基本一样, 只不过位姿是6维的了, 不需要再去与重力对齐向量相乘了, 直接获取local_pose, 不用进行旋转变换了.

#### ConstraintBuilder3D
基本一样, 只不过调用的是FastCorrelativeScanMatcher3D.

#### FastCorrelativeScanMatcher3D

将高分辨率地图弄成多分辨率地图
保存低分辨率地图

#### OptimizationProblem3D

根据参数选择是否对节点与子图的pose的z坐标进行优化

优化时第一个子图固定了xyz, 旋转固定了yaw, 只优化绕xy的旋转, 因为绕xy的旋转可以通过重力的方向进行约束.

由于旋转是通过四元数表示, 所以在ceres中添加了QuaternionParameterization, 以对四元数进行更新.

多了根据imu计算的残差, 分为加速度的残差与旋转的残差

其余的残差基本一样.

## 第七章 地图保存与纯定位模式

### 7.1 Submap与ROS格式地图间的格式转换

### 7.2 ROS地图的发布

### 7.3 纯定位模式

---




## 第八章 调参总结与工程化建议

### 8.1 调参总结

#### 8.1.1 降低延迟与减小计算量

**前端部分**

- 减小 max_range, 减小了需要处理的点数, 在雷达数据远距离的点不准时一定要减小这个值

- 增大 voxel_filter_size, 相当于减小了需要处理的点数

- 增大 submaps.resolution, 相当于减小了匹配时的搜索量

- 对于自适应体素滤波 减小 min_num_points与max_range, 增大 max_length, 相当于减小了需要处理的点数


**后端部分**

- 减小 optimize_every_n_nodes, 降低优化频率, 减小了计算量

- 增大 MAP_BUILDER.num_background_threads, 增加计算速度

- 减小 global_sampling_ratio, 减小计算全局约束的频率

- 减小 constraint_builder.sampling_ratio, 减少了约束的数量

- 增大 constraint_builder.min_score, 减少了约束的数量

- 减小分枝定界搜索窗的大小, 包括linear_xy_search_window,inear_z_search_window, angular_search_window

- 增大 global_constraint_search_after_n_seconds, 减小计算全局约束的频率

- 减小 max_num_iterations, 减小迭代次数

#### 8.1.2 降低内存

增大子图的分辨率 submaps.resolution

#### 8.1.3 常调的参数

```lua
 TRAJECTORY_BUILDER_2D.min_range = 0.3
 TRAJECTORY_BUILDER_2D.max_range = 100.
 TRAJECTORY_BUILDER_2D.min_z = 0.2 -- / -0.8
 TRAJECTORY_BUILDER_2D.voxel_filter_size = 0.02

 TRAJECTORY_BUILDER_2D.ceres_scan_matcher.occupied_space_weight = 10.
 TRAJECTORY_BUILDER_2D.ceres_scan_matcher.translation_weight = 1.
 TRAJECTORY_BUILDER_2D.ceres_scan_matcher.rotation_weight = 1.

 TRAJECTORY_BUILDER_2D.submaps.num_range_data = 80.
 TRAJECTORY_BUILDER_2D.submaps.grid_options_2d.resolution = 0.1 -- / 0.02

 POSE_GRAPH.optimize_every_n_nodes = 160. -- 2倍的num_range_data以上
 POSE_GRAPH.constraint_builder.sampling_ratio = 0.3
 POSE_GRAPH.constraint_builder.max_constraint_distance = 15.
 POSE_GRAPH.constraint_builder.min_score = 0.48
 POSE_GRAPH.constraint_builder.global_localization_min_score = 0.60
```



### 8.2 工程化建议

#### 8.2.1 工程化的目的

根据机器人的**传感器硬件**, 最终能够实现**稳定地**构建一张**不叠图**的二维栅格地图.

由于cartographer代码十分的优秀, 所以cartographer的稳定性目前已经很好了, 比目前的大部分slam的代码都稳定, 很少会出现崩掉的情况, 最多就是会由于某些原因提示错误.

#### 8.2.2 如何提升建图质量

最简单的一种方式, 选择好的传感器. 选择频率高(25hz以上), 精度高的雷达, 精度高的imu, 这样的传感器配置下很难有建不好的地图.

##### 如果只能用频率低的雷达呢


由于频率低时的叠图基本都是在旋转时产生的, 所以推荐使用一个好的imu, 然后建图的时候让机器人的**移动与旋转速度慢一点**(建图轨迹与建图速度十分影响建图效果), 这时候再看建图效果.

如果效果还不行, 调ceres的匹配权重, 将地图权重调大, 平移旋转权重调小. 

如果效果还不行, 可以将代码中平移和旋转的残差注释掉.

如果效果还不行, 那就得改代码了, 去改位姿推测器那部分的代码, 让预测的准一点.

##### 里程计

为什么一直没有说里程计, 就是由于cartographer中对里程计的使用不太好.

cartographer中对里程计的使用有2部分, 一个是前端的位姿推测器, 一个是后端根据里程计数据计算残差. 后端部分的使用是没有问题的.

如果想要在cartographer中使用里程计达到比较好的效果, 前端的位姿推测器这部分需要自己重写. 

可以将karto与gmapping的使用里程计进行预测的部分拿过来进行使用, 改完了之后就能够达到比较好的位姿预测效果了.

##### 粗匹配

cartographer的扫描匹配中的粗匹配是一种暴力匹配的方法, 目的是对位姿预测出的位姿进行校准, 但是这个扫描匹配的计算量太大了, 导致不太好用.

这块可以进行改进, 可以将karto的扫描匹配的粗匹配放过来, karto的扫描匹配的计算量很小, 当做粗匹配很不错.

##### 地图

有时前端部分生成的地图出现了叠图, 而前端建的地图在后端是不会被修改的, 后端优化只会优化节点位姿与子图位姿.

同时cartographer_ros最终生成的地图是将所有地图叠加起来的, 就会导致这个叠图始终都存在, 又或者是后边的地图的空白部分将前边的地图的边给覆盖住了, 导致墙的黑边消失了.

后端优化会将节点与子图的位姿进行优化, 但是不会改动地图, 所以可以在最终生成地图的时候使用后端优化后的节点重新生成一次地图, 这样生成的地图的效果会比前端地图的叠加要好很多.

这块的实现可以参考一下我写的实时生成三维点云地图部分的代码.

##### 更极致的修改

后端优化后的节点与子图位姿是不会对前端产生影响的, 这块可以进行优化一下, 就是前端匹配的时候, 不再使用前端生成的地图进行匹配, 而是使用后端生成的地图进行匹配, 这样就可以将后端优化后的效果带给前端. 但是这要对代码进行大改, 比较费劲.


#### 8.2.3 降低计算量与内存

- 体素滤波与自适应体素滤波的计算量(不是很大)

- 后端进行子图间约束时的计算量很大

- 分支定界算法的计算量很大
- 降低内存, 内存的占用基本就是多分辨率地图这, 每个子图的多分辨率地图都进行保存是否有必要

#### 8.2.4 纯定位的改进建议

目前cartographer的纯定位和正常的建图是一样的, 只是仅保存3个子图, 依然要进行后端优化.


这就导致了几个问题:

第一个: 前端的扫描匹配, 是当前的雷达点云与当前轨迹的地图进行匹配, 而不是和之前的地图进行匹配, 这就导致了定位时机器人当前的点云与之前的地图不一定能匹配很好, 就是因为当前的点云是匹配当前轨迹的地图的, 不是与之前的地图进行匹配.

第二个: 纯定位其实就是建图, 所以依然会进行回环检测与后端优化, 而后端优化的计算在定位这是没有必要的, 带来了额外的计算量.

第三个: 纯定位依然会进行回环检测, 回环检测有可能导致机器人的位姿发生跳变.



**改进思路**

将纯定位模式与建图拆分开, 改成读取之前轨迹的地图进行匹配.

新的轨迹只进行位姿预测, 拿到预测后的位姿与之前轨迹的地图进行匹配, 新的轨迹不再进行地图的生成与保存. 同时将整个后端的功能去掉.

去掉了后端优化之后, 会导致没有重定位功能, 这时候可以将cartographer的回环检测(子图间约束的计算)部分单独拿出来, 做成一个重定位功能. 通过服务来调用这个重定位功能, 根据当前点云确定机器人在之前地图的位姿.

这样才是一个比较好的定位功能的思路.

#### 8.2.5 去ros的参考思路

有一些公司不用ros, 所以就要进行去ros的开发.

咱讲过数据是怎么通过cartographer_ros传到cartographer里去的, 只要仿照着cartographer_ros里的操作, 获取到传感器数据, 将数据转到tracking_frame坐标系下并进行格式转换, 再传入到cartographer里就行了.

cartographer_ros里使用ros的地方比较少, 只有在node.cc, sensor_bridge等几个类中进行使用, 只需要改这个类接受数据的方式以及将ros相关的格式修改一下就行了.




##
4.4 生成3D局部地图
  a. 查找表的实现
  b. 3D网格地图的实现
  c. 3D网格地图的更新方式
  d. 如何将点云插入到3D网格地图中
4.5 3D情况下的基于局部地图的扫描匹配
  a. 基于IMU与里程计的先验位姿估计
  b. 将点云分别进行高分辨率与低分辨率的体素滤波
  c. 使用实时的相关性扫描匹配对高分辨率体素滤波后的点云进行粗匹配
  d. 进行基于图优化的扫描匹配实现精匹配
  e. 旋转直方图的作用
  f. 基于ceres-solver的优化模型搭建
5.3 3D情况下的后端优化
  a. 基于ceres-solver的后端位姿图优化模型的搭建
  b. 向位姿图中添加基于GPS的2个连续位姿间的约束
  c. 向位姿图中添加基于IMU的角速度和线性加速度的约束
  d. 残差项雅克比矩阵的计算与优化模型的求解
6.3 3D情况下的回环检测
  a. 多分辨率网格地图的生成
  b. 计算点云在低分辨率地图下的得分作为初值
  c. 通过旋转直方图对点云所有可能的旋转角度进行评分
  d. 根据评分是否大于阈值生成按照可能角度旋转后的点云
  e. 生成最低分辨率地图下的所有的可能解
  f. 对所有的可能解进行评分与排序
  g. 使用分支定界算法找到最优解