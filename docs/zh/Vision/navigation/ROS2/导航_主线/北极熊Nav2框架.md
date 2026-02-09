**修改时间:2026.1.26**

**参与者:刘志钰**

# 北极熊Nav2框架



### 一.框架的搭建

[SMBU-PolarBear-Robotics-Team/pb2025_sentry_nav: Shenzhen MSU-BIT University PolarBear Team's Sentry Navigation Sim2Real Package for RoboMaster2025. QQ group: 932119307](https://github.com/SMBU-PolarBear-Robotics-Team/pb2025_sentry_nav?tab=readme-ov-file)



### 二.北极熊代码调用

##### rmu_gazebo_simulator（配套的仿真包）

- colcon build

  - ```
    rosdep install -r --from-paths src --ignore-src --rosdistro $ROS_DISTRO -y
    ```

  - ```
    colcon build --symlink-install --cmake-args -DCMAKE_BUILD_TYPE=release
    ```

- 启动仿真环境

  - ```
    ros2 launch rmu_gazebo_simulator bringup_sim.launch.py
    ```

- 控制机器人移动（只有导航模式能使用）

  - ```
    ros2 run rmoss_gz_base test_chassis_cmd.py --ros-args -r __ns:=/red_standard_robot1/robot_base -p v:=0.3 -p w:=0.3
    #根据提示进行输入，支持平移与自旋
    #速度可以自己调节
    ```

- 机器人云台

  - ```
    ros2 run rmoss_gz_base test_gimbal_cmd.py --ros-args -r __ns:=/red_standard_robot1/robot_base
    #根据提示进行输入，支持绝对角度控制
    ```

- 机器人射击

  - ```
    ros2 run rmoss_gz_base test_shoot_cmd.py --ros-args -r __ns:=/red_standard_robot1/robot_base
    #根据提示进行输入
    ```

- 切换仿真世界

  - 修改 [gz_world.yaml](https://github.com/SMBU-PolarBear-Robotics-Team/rmu_gazebo_simulator/blob/main/rmu_gazebo_simulator/config/gz_world.yaml) 中的 `world`。当前可选: `rmul_2024`, `rmuc_2024`, `rmul_2025`, `rmuc_2025`



##### pd2025_sentry_nav

- 导航

  - ```
    ros2 launch pb2025_nav_bringup rm_navigation_simulation_launch.py \
    world:=rmuc_2025 \
    slam:=False
    ```

- 建图

  - ```
    ros2 launch pb2025_nav_bringup rm_navigation_simulation_launch.py \
    slam:=True
    ```

  - 用Nav2 Goal来跑图

  - ```
    ros2 run nav2_map_server map_saver_cli -f <YOUR_MAP_NAME>  --ros-args -r __ns:=/red_standard_robot1
    ```

- 两辆车

  - ```
    ros2 launch pb2025_nav_bringup rm_multi_navigation_simulation_launch.py \
    world:=rmul_2024 \
    robots:=" \
    red_standard_robot1={x: 0.0, y: 0.0, yaw: 0.0}; \
    blue_standard_robot1={x: 5.6, y: 1.4, yaw: 3.14}; \
    "
    ```

- 可视化机器人模型（在ROS_WS里面）

  - ```
    ros2 launch pb2025_robot_description robot_description_launch.py
    ```

  - 查看TF树

  - ```
    ros2 run rqt_tf_tree rqt_tf_tree --ros-args -r /tf:=tf -r /tf_static:=tf_static
    ```


- 建图指南

  - 启动建图模式

  - 保存初版地图

    - SLAM边建图边导航（**先不要 Ctrl+C 终止建图程序**，而是新开一个终端，运行以下命令保存地图）

      - ```
        ros2 run nav2_map_server map_saver_cli -f <YOUR_MAP_NAME>
        ```

    - pcd2pgm建图

      - 读取指定的.pcd文件

      - 使用 Pass Through 滤波器过滤点云

      - 使用 Radius Outlier 滤波器进一步处理点云

      - 将处理后的点云转换为占据栅格地图

      - 将转换后的地图发布到指定ROS话题上

      - ---

      - 启动pcd2pgm节点，可在RViz中预览滤波后点云和栅格地图

      - ```
        ros2 launch pcd2pgm pcd2pgm_launch.py
        ```

      - 保存栅格地图

      - ```
        ros2 run nav2_map_server map_saver_cli -f <YOUR_MAP_NAME>
        ```




### 三.北极熊导航介绍

下面是北极熊导航包的各个文件及作用说明。

```
.
├── fake_vel_transform                  # 虚拟速度参考坐标系，以应对云台扫描模式自旋，详见子仓库 README
├── ign_sim_pointcloud_tool             # 仿真器点云处理工具
├── livox_ros_driver2                   # Livox 驱动
├── loam_interface                      # point_lio 等里程计算法接口
├── pb_teleop_twist_joy                 # 手柄控制
├── pb2025_nav_bringup                  # 启动文件
├── pb2025_sentry_nav                   # 本仓库功能包描述文件
├── pb_omni_pid_pursuit_controller      # 路径跟踪控制器
├── point_lio                           # 里程计
├── pointcloud_to_laserscan             # 将 terrain_map 转换为 laserScan 类型以表示障碍物（仅 SLAM 模式启动）
├── sensor_scan_generation              # 点云相关坐标变换
├── small_gicp_relocalization           # 重定位
└── terrain_analysis                    # 分割出非地面障碍物点云
```

​	在这里面，诸如 livox_ros_driver2, point_lio 等功能包都是可以通过 git clone 到工作空间的src下的。而像其他部分如pointcloud_to_laserscan, pb_teleop_twist_joy 等暂时先不说明。只说明其中的一部分代码。



##### 1. loam_interface说明

​	如上所述，这是一个point_lio等里程计的算法接口，该代码是用来做LOAM(激光里程计与建图)算法与ROS2导航系统之间的接口模块。主要实现的功能包括：

- **坐标系转换**：将LOAM输出的`lidar_odom`（激光雷达局部坐标系）的里程计和点云数据，转换到导航系统所需的`odom`（全局里程计坐标系）框架。

- **数据重发布**：将转换后的点云和里程计数据发布到标准ROS2导航话题，供下游模块（如SLAM、路径规划）使用。

    

> ​	***定位如 amcl 提供的是 map->odom 坐标系的转换***
>
> ​	***里程计则提供的是 odom->base_link 坐标系的转换***

​	

​	下面是对部分代码进行解析与说明。其中LOAM原始里程计和点云的输入数据都是订阅的Pointlio发布的话题，具体内容在nav2_params.yaml 中即可进行配置。

```
// 参数声明（用于动态配置）
this->declare_parameter<std::string>("state_estimation_topic", "");  # LOAM原始里程计输入话题
this->declare_parameter<std::string>("registered_scan_topic", "");   # LOAM注册后的点云输入话题
this->declare_parameter<std::string>("odom_frame", "odom");			 # 导航系统使用的全局里程计坐标系
this->declare_parameter<std::string>("base_frame", "");				 # 机器人基坐标系话题
this->declare_parameter<std::string>("lidar_frame", "");			 # 激光雷达坐标系话题

// 参数更新
this->get_parameter("state_estimation_topic", state_estimation_topic_);
this->get_parameter("registered_scan_topic", registered_scan_topic_);
this->get_parameter("odom_frame", odom_frame_);
this->get_parameter("base_frame", base_frame_);
this->get_parameter("lidar_frame", lidar_frame_);
```

​	接下来是关于数据收发部分的代码。声明两个发布器，发布的话题名称分别是"registered_scan"和"lidar_odometry"，发布的消息类型分别是PointCloud2和Odometry。

```
// 声明用于发送点云和里程计数据的发布器
pcd_pub_ = this->create_publisher<sensor_msgs::msg::PointCloud2>("registered_scan", 5);
odom_pub_ = this->create_publisher<nav_msgs::msg::Odometry>("lidar_odometry", 5);
```

​	在完成了声明之后，对话题数据进行订阅并绑定回调函数。

```
// 订阅话题,并在回调函数中对订阅的话题进行操作
pcd_sub_ = this->create_subscription<sensor_msgs::msg::PointCloud2>(
	registered_scan_topic_, 5,
	std::bind(&LoamInterfaceNode::pointCloudCallback, this, std::placeholders::_1));
	
odom_sub_ = this->create_subscription<nav_msgs::msg::Odometry>(
	state_estimation_topic_, 5,
	std::bind(&LoamInterfaceNode::odometryCallback, this, std::placeholders::_1));
}
```

​	以点云数据为例，读取的真实话题名称并不是“registered_scan_topic_”，这个话题会在launch文件中进行配置，以loam_interface 文件夹下的loam_interface_launch.py为例。读取的里程计数据真实话题是 "aft_mapped_to_init"，读取的点云数据的真实话题是"cloud_registered"，剩余部分同理。

> ​	**需要注意的是，这两个数据话题本质上是 point_lio 进行转换后发布的话题，与FAST_LIO 发布的话题 /Odometry 不同，如果想要调用的功能包是FAST_LIO，那么就需要对这两个部分进行修改。**

```
start_loam_interface = Node(
        package="loam_interface",
        executable="loam_interface_node",
        name="loam_interface",
        namespace=namespace,
        remappings=remappings,
        output="screen",
        parameters=[
            {
                "state_estimation_topic": "aft_mapped_to_init",
                "registered_scan_topic": "cloud_registered",
                "odom_frame": "odom",
                "base_frame": "gimbal_yaw",
                "lidar_frame": "front_mid360",
            }
        ],
    )
```

​	接下来就是在回调函数中对读取到的数据进行处理，这里以point_cloud 点云数据为例，代码如下

```
void LoamInterfaceNode::pointCloudCallback(const sensor_msgs::msg::PointCloud2::ConstSharedPtr msg)
{
  // NOTE: Input point cloud message is based on the `lidar_odom`
  // Here we transform it to the REAL `odom` frame
  auto out = std::make_shared<sensor_msgs::msg::PointCloud2>();
  // 转换odom_frame_成tf_odom_to_lidar_odom_
  pcl_ros::transformPointCloud(odom_frame_, tf_odom_to_lidar_odom_, *msg, *out);
  // 转到pcd_pub_发布
  pcd_pub_->publish(*out);
}
```

​	在loam_interface代码中，实际运行的数据流如下(仍然以点云数据为例)，其中/cloud_registered替换成真实的雷达点云话题：

```
/cloud_registered → pcd_sub_ → pointCloudCallback → pcd_pub_ → /registered_scan
```

​	在代码结尾的部分，再将这个loam_interface.cpp处理成一个Node节点，用于后续launch.py文件的调用

```
#include "rclcpp_components/register_node_macro.hpp"

RCLCPP_COMPONENTS_REGISTER_NODE(loam_interface::LoamInterfaceNode)
```



##### 2. pb_nav2_plugins说明

​	在这个功能包的src文件夹中有两个模块的代码，分别是 back_up_free_space.cpp 与 intensity_voxel_layer.cpp 。这两个模块分别继承的是 nav2_core中 的 Behavior 基类与 nav2_costmap_2d 中的 Layer 基类。这

​	nav2_costmap_2d的工作流程如下：

```
[传感器数据] → [Layer] → [LayeredCostmap] → [Costmap2D]  
                               ↓  
                   [Planner] 生成路径 → [Controller] 控制运动
                               ↓  
                   [Behavior] 处理异常情况
```



a.  intensity_voxel_layer

​	这一部分是Nav2代价地图的一个自定义层，这里自定义这一部分内容主要用于：基于点云强度值的障碍物检测，三维体素化处理以及动态代价更新。主要包括有以下几个部分的核心代码。在完成基本的初始化内容之后，下面需要对代价地图进行更新，对代价地图的更新主要依赖于updateBounds。

​	updateBounds ，主要用于处理点云数据，并将符合条件的点云数据存入到地图中(即完成一次代价地图的更新)。首先是对其进行初始设置。

```
// 滚动窗口原点更新,确保代价地图的原点(左下角)始终保持在机器人当前位置的中心
if (rolling_window_) {
	updateOrigin(robot_x - getSizeInMetersX() / 2, robot_y - getSizeInMetersY() / 2);
}

// 每次都清空上一帧的代价地图和体素网格数据
resetMaps();

// 如果图层被禁止启用(如临时禁止使用激光雷达),则直接跳过后续处理
if (!enabled_) {
	return;
}

// 扩展更新边界
useExtraBounds(min_x, min_y, max_x, max_y);

// 获取激光雷达的观测数据
bool current = true;
std::vector<Observation> observations;
current = getMarkingObservations(observations) && current;
```

​	下面是对点云数据的循环处理，也是 intensity_voxel_layer 核心部分。通过对三维世界观测到的点云进行遍历，确定其是否符合障碍物的范围内，之后将其更新到二维的代价地图中，并将其标注为致命障碍让路径规划避开障碍物。其中有结构体Observation，定义在nav2_costmap_2d(Nav2自带的)功能包中。结构体包括内容有：传感器数据(如点云)，传感器原点，障碍物检测的最大最小范围。

```
for (const auto & obs : observations) {
double sq_obstacle_max_range = obs.obstacle_max_range_ * obs.obstacle_max_range_;
double sq_obstacle_min_range = obs.obstacle_min_range_ * obs.obstacle_min_range_;

// 读取点云数据的xyz坐标和反射强度,此处只展示读取点云数据x坐标,后续代码略
sensor_msgs::PointCloud2ConstIterator<float> it_x(*obs.cloud_, "x");
......  

// 遍历点云坐标(即将每一个观测到的点云数据都做处理)
for (; it_x != it_x.end(); ++it_x, ++it_y, ++it_z, ++it_i) {
  double px = *it_x, py = *it_y, pz = *it_z;
  
  ......  // 此处代码略,包括障碍物高度是否在区间内,障碍物强度是否在区间内
  
  // 计算观测到的点云距离和传感器的距离
  double sq_dist = (px - obs.origin_.x) * (px - obs.origin_.x) +
                   (py - obs.origin_.y) * (py - obs.origin_.y) +
                   (pz - obs.origin_.z) * (pz - obs.origin_.z);

  // 计算距离是否在区间内
  if (sq_dist <= sq_obstacle_min_range || sq_dist >= sq_obstacle_max_range) {
    continue;
  }
  
  // 将三维世界的坐标转换为网格坐标
  unsigned int mx, my, mz;
  if (pz < origin_z_) {
    if (!worldToMap3D(px, py, origin_z_, mx, my, mz)) {
      continue;
    }
  } else if (!worldToMap3D(px, py, pz, mx, my, mz)) {
    continue;
  }
  // 体素标记与代价更新
  if (voxel_grid_.markVoxelInMap(mx, my, mz, mark_threshold_)) {
    unsigned int index = getIndex(mx, my);
	
	// 代价地图对应网格标记为LETHAL_OBSTACLE(致命障碍),路径规划将避开该区域
    costmap_[index] = LETHAL_OBSTACLE;
    
    // 更新min_x等边界值,确保后续图层处理覆盖该区域
    touch(static_cast<double>(px), static_cast<double>(py), min_x, min_y, max_x, max_y);
    }
  }
}
```

​	得到一个更新后的代价地图后，需要将其发布出去。

```
if (publish_voxel_) {
    nav2_msgs::msg::VoxelGrid grid_msg;
    // 填充部分代码略
    voxel_pub_->publish(grid_msg); //发布
  }
  
// 更新机器人足迹区域
updateFootprint(robot_x, robot_y, robot_yaw, min_x, min_y, max_x, max_y);
```



b.  back_up_free_space

​	这一部分是Nav2行为树中的自定义恢复行为插件，核心功能是当机器人被困住时，自动寻找最大自由空间并执行避障操作。因此，这是一个避障用插件。显然其处于恢复器服务器中。

​	关于声明参数和获取参数部分不做过多介绍，主要的参数声明与包括有全局坐标系，最大检测半径(m)。除此之外在初始化部分，还包括有创建Costmap服务客户端。可视化初始化。

​	机器人执行避障的行为逻辑通过函数onRun实现。具体关键代码如下：

```
// 等待costmap服务可用(该服务是通过local_costmap/get_costmap获取的)
while (!costmap_client_->wait_for_service(std::chrono::seconds(1))) {
	......  // 此处略
}

// 请求并获取costmap
auto request = std::make_shared<nav2_msgs::srv::GetCostmap::Request>();
auto result = costmap_client_->async_send_request(request);
auto costmap = result.get()->map;

...... // 此处略

// 获取当前位姿
geometry_msgs::msg::Pose2D pose;
pose.x = initial_pose_.pose.position.x;
pose.y = initial_pose_.pose.position.y;
pose.theta = tf2::getYaw(initial_pose_.pose.orientation);

// 找到最佳后退方向
float best_angle = findBestDirection(costmap, pose, -M_PI, M_PI, max_radius_, M_PI / 32.0);

// 设置运动参数
twist_x_ = std::cos(best_angle) * command->speed;
twist_y_ = std::sin(best_angle) * command->speed;
command_x_ = command->target.x;
command_time_allowance_ = command->time_allowance;

......  // 此处略
}
```

​	基本的逻辑就是 获得代价地图 ——> 获取当前位姿 ——> 找到后退方向 ——> 设置运动参数

​	接下来则是重点内容，如何找到最佳的后退方向？这里通过函数 findBestDirection 实现。

```
float findBestDirection(...) {

  ......  // 此处略,设置初始状态

  for (float angle = start; angle <= end; angle += increment) {
  
    // 沿角度方向检测半径内所有点
    for (float r = 0; r <= radius; r += resolution) {
      ......  // 此处略,坐标转换与障碍判断
    }
    
    // 记录最大连续安全角度区间
    if (is_safe) {
      if (first_safe_angle == -1) first_safe_angle = angle;
    } else {
      if (first_safe_angle != -1) last_unsafe_angle = angle;
      
    // 更新最佳区间
    if (
      last_unsafe_angle - first_safe_angle > final_unsafe_angle - final_safe_angle &&
      first_safe_angle != -1.0f && last_unsafe_angle != -1.0f) {
      final_safe_angle = first_safe_angle;
      final_unsafe_angle = last_unsafe_angle;
      first_safe_angle = -1.0f;
      last_unsafe_angle = -1.0f;
      }
    }
  
  // 最佳角度
  best_angle = (final_safe_angle + final_unsafe_angle) / 2.0f;
  
  return best_angle
}
```

​	backup的行为通过函数 onCycleUpdate 进行循环控制。其实现函数大致内容如下：

```
Status BackUpFreeSpace::onCycleUpdate() {
  // 超时检测
  if (time_remaining.seconds() < 0.0) {
    stopRobot();
    return FAILED;
  }

  // 计算已移动距离
  float diff_x = initial_pose_.pose.position.x - current_pose.pose.position.x;
  float diff_y = initial_pose_.pose.position.y - current_pose.pose.position.y;
  float distance = hypot(dx, dy);

  // 到达目标距离判断
  if (distance >= command_x_) {
    stopRobot();
    return SUCCEEDED;
  }
  
  // 速度指令
  auto cmd_vel = std::make_unique<geometry_msgs::msg::Twist>();
  cmd_vel->linear.y = twist_y_;
  cmd_vel->linear.x = twist_x_;

  // 更新位姿
  geometry_msgs::msg::Pose2D pose;
  pose.x = current_pose.pose.position.x;
  pose.y = current_pose.pose.position.y;
  pose.theta = tf2::getYaw(current_pose.pose.orientation);

  // 碰撞检测
  if (!isCollisionFree(...)) {
    stopRobot();
    return FAILED;
  }

  // 发布速度指令
  vel_pub_->publish(cmd_vel);
  
  return RUNNING;
}
```

​	

​	总结：在这一个功能包中，主要内容是作者自己编写新的规划器插件，其实也可以不自己定义规划器，因为Nav2框架已经提供了规划器，例如关于恢复行为的部分，Nav2默认的恢复行为是Backup，其会通过固定的直线进行后退，但是自定义的back_up_free_space则是自定义了动态选择最优的后退方向。

​	

##### 3. pb_omni_pid_pursuit_controller说明

​	这是一个路径跟踪控制器。在Nav2导航框架下，同样还有多种路径跟踪控制器，包括有Pure pursuit，DWB等等，只不过与自己编写规划器类似，这也是一个由导航包作者自行编写的一个 omni(麦克纳姆轮) 小车的跟踪控制器。该路径跟踪控制器的功能即为驱动小车按照发布的路径进行移动。

​	首先来看声明参数部分，这一部分的代码内容极多，仅介绍其中一小部分，具体代码如下：

```
declare_parameter_if_not_declared(
    node, plugin_name_ + ".translation_kp", rclcpp::ParameterValue(3.0));
declare_parameter_if_not_declared(
	node, plugin_name_ + ".translation_ki", rclcpp::ParameterValue(0.1));
declare_parameter_if_not_declared(
	node, plugin_name_ + ".translation_kd", rclcpp::ParameterValue(0.3));
......  // 后略
```

​	在这一部分中，声明了几个关键参数，包括有：

​	1、PID参数：`translation_kp/ki/kd`（平移PID）、`rotation_kp/ki/kd`（旋转PID）

​	2、预瞄控制：`lookahead_dist`（静态预瞄距离）、`lookahead_time`（速度相关预瞄时间）

​	3、曲率限制：`curvature_min/max`（最小/最大曲率阈值）

​	4、安全限制：`v_linear_max`（最大线速度）、`v_angular_max`（最大角速度）

​	下面是速度指令生成模块，代码及注释如下所示：

```
geometry_msgs::msg::TwistStamped computeVelocityCommands(...) {
  //===== 第一阶段：数据准备 =====
  // 1.1 加锁保证线程安全
  std::lock_guard<std::mutex> lock_reinit(mutex_);
  
  // 1.2 获取代价地图并加锁
  nav2_costmap_2d::Costmap2D * costmap = costmap_ros_->getCostmap();
  std::unique_lock<nav2_costmap_2d::Costmap2D::mutex_t> lock(*(costmap->getMutex()));

  //===== 第二阶段：路径处理 =====
  // 2.1 将全局路径转换到机器人坐标系
  auto transformed_plan = transformGlobalPlan(pose);
  
  // 2.2 计算动态预瞄距离
  double lookahead_dist = getLookAheadDistance(velocity);
  
  // 2.3 获取预瞄点（胡萝卜点）
  auto carrot_pose = getLookAheadPoint(lookahead_dist, transformed_plan);
  carrot_pub_->publish(createCarrotMsg(carrot_pose)); // 可视化发布

  //===== 第三阶段：误差计算 =====
  // 3.1 计算线距离误差（机器人到预瞄点的直线距离）
  double lin_dist = hypot(carrot_pose.pose.position.x, carrot_pose.pose.position.y);
  
  // 3.2 计算角度误差（机器人朝向与目标方向的偏差）
  double theta_dist = atan2(carrot_pose.pose.position.y, carrot_pose.pose.position.x);
  
  // 3.3 计算终点角度误差（针对最终朝向调整）
  double angle_to_goal = tf2::getYaw(carrot_pose.pose.orientation);

  //===== 第四阶段：PID控制 =====
  // 4.1 平移PID计算线速度
  auto lin_vel = move_pid_->calculate(lin_dist, 0); // 目标误差为0
  
  // 4.2 旋转PID计算角速度（可选启用）
  auto angular_vel = enable_rotation_ ? heading_pid_->calculate(angle_to_goal, 0) : 0.0;

  //===== 第五阶段：安全调整 =====
  // 5.1 曲率限速（急转弯时降速）
  applyCurvatureLimitation(transformed_plan, carrot_pose, lin_vel);
  
  // 5.2 接近终点减速
  applyApproachVelocityScaling(transformed_plan, lin_vel);

  //===== 第六阶段：碰撞检测 =====
  // 6.1 构建检测路径（采样10个点）
  nav_msgs::msg::Path costmap_frame_local_plan;
  for (int i = 0; i < sample_points; ++i) {
    // 路径点坐标转换到全局坐标系
    transformPose(costmap_ros_->getGlobalFrameID(), transformed_plan.poses[index], map_pose);
    costmap_frame_local_plan.poses.push_back(map_pose);
  }

  // 6.2 生成速度指令
  geometry_msgs::msg::TwistStamped cmd_vel;
  cmd_vel.header = pose.header;
  
  // 6.3 碰撞检测通过则发布速度
  if (!isCollisionDetected(costmap_frame_local_plan)) {
    // 全向速度分解
    cmd_vel.twist.linear.x = lin_vel * cos(theta_dist);  // X方向分量
    cmd_vel.twist.linear.y = lin_vel * sin(theta_dist);  // Y方向分量（全向特有）
    cmd_vel.twist.angular.z = angular_vel;               // 旋转分量
  } else {
    throw nav2_core::PlannerException("Collision detected!");
  }

  return cmd_vel;
}
```

​	在这里面中，有几个在函数中提到的关键子模块，其代码与作用分别如下：

​	1、**`transformGlobalPlan`** ：路径转换

```
nav_msgs::msg::Path OmniPidPursuitController::transformGlobalPlan(...) 
{
  // 将全局路径转换到机器人坐标系的核心逻辑
  // ------------------------------------------------------
  // 输入：全局路径（地图坐标系），当前机器人位姿
  // 输出：机器人坐标系下的局部路径（仅保留附近有效部分）

  // 步骤1：获取机器人当前在全局路径坐标系下的位姿
  geometry_msgs::msg::PoseStamped robot_pose;
  transformPose(global_plan_.header.frame_id, pose, robot_pose);

  // 步骤2：裁剪路径，仅保留机器人当前位置前后一定范围内的路径点
  auto closest_pose_upper_bound = ...; // 搜索半径限制（max_robot_pose_search_dist_）
  auto transformation_end = ...;       // 过滤超出代价地图范围的远端路径点

  // 步骤3：坐标转换（全局坐标系→机器人基坐标系）
  auto transform_global_pose_to_local = [&](...) {
    geometry_msgs::msg::PoseStamped transformed_pose;
    tf_->transform(global_pose, transformed_pose, costmap_ros_->getBaseFrameID());
    return transformed_pose;
  };

  // 步骤4：转换路径并发布局部路径可视化
  std::transform(...); // 批量转换坐标
  local_path_pub_->publish(transformed_plan);

  return transformed_plan;
}
```

​	2、**`getLookAheadDistance`** ：动态预瞄距离

```
double OmniPidPursuitController::getLookAheadDistance(...)
{
  // 动态计算预瞄距离的核心逻辑
  // ------------------------------------------------------
  // 输入：当前机器人线速度
  // 输出：动态调整后的预瞄距离

  // 基础预瞄距离（静态配置或速度相关）
  double lookahead_dist = lookahead_dist_; 

  // 如果启用速度相关预瞄
  if (use_velocity_scaled_lookahead_dist_) {
    // 计算速度相关预瞄距离 = 速度 × 预瞄时间
    lookahead_dist = std::hypot(speed.linear.x, speed.linear.y) * lookahead_time_;
    
    // 钳制在[min,max]范围内防止极端值
    lookahead_dist = std::clamp(lookahead_dist, min_lookahead_dist_, max_lookahead_dist_);
  }

  return lookahead_dist;
}
```

​	3、**`isCollisionDetected`** ：碰撞检测

```
bool OmniPidPursuitController::isCollisionDetected(...)
{
  // 沿路径检测障碍物的核心逻辑
  // ------------------------------------------------------
  // 输入：转换到代价地图坐标系的局部路径
  // 输出：是否检测到碰撞（true=存在障碍物）

  // 遍历路径上的采样点（此处采样10个点）
  for (const auto & pose_stamped : path.poses) {
    // 将世界坐标转换为代价地图网格坐标
    unsigned int mx, my;
    if (costmap->worldToMap(pose.position.x, pose.position.y, mx, my)) {
      // 获取代价值并判断是否超过安全阈值
      if (costmap->getCost(mx, my) >= nav2_costmap_2d::INSCRIBED_INFLATED_OBSTACLE) {
        return true; // 发现障碍物
      }
    } else {
      // 路径点超出代价地图范围（视为不安全）
      return true;
    }
  }
  return false; // 路径安全
}
```

​	4、**`applyCurvatureLimitation`** ：曲率限速

```
void OmniPidPursuitController::applyCurvatureLimitation(...)
{
  // 根据路径曲率动态调整线速度的核心逻辑
  // ------------------------------------------------------
  // 输入：局部路径，预瞄点，当前计算的线速度（引用方式直接修改）
  
  // 步骤1：计算当前路径段的曲率
  double curvature = calculateCurvature(...);

  // 步骤2：根据曲率值调整速度
  if (curvature > curvature_min_) {
    // 计算速度缩减比例（曲率越大缩减越多）
    double reduction_ratio = 1.0 - ...; // 基于曲率区间插值计算
    
    // 渐进式调整速度，避免突变
    scaled_linear_vel = last_velocity_scaling_factor_ + ...; // 带速率限制的变化
    
    // 最终线速度取较小值（保证安全性）
    linear_vel = std::min(linear_vel, scaled_linear_vel);
  }
  
  // 步骤3：可视化曲率计算点
  visualizeCurvaturePoints(...); // 发布RViz标记
}
```



​	上面的代码只是一个简单的介绍，真正代码的实现还是需要个人自己到代码中去一行行语句看，这也同样有助于你学习如何用C++编写一个个ROS2功能包，并且真正系统性地将它们连接起来。这一部分代码的工作流程图如下所示：

```
[全局路径] → transformGlobalPlan() → [局部路径]
                   ↓
          getLookAheadDistance() → [动态预瞄点]
                   ↓
             PID控制器计算 → [基础速度指令]
                   ↓
         applyCurvatureLimitation() → [曲率安全速度]
                   ↓
   applyApproachVelocityScaling() → [终点减速速度]
                   ↓
          isCollisionDetected() → [最终速度指令]
```







# 北极熊代码跑通的版本



https://github.com/XJU-Hurricane-Team/Vision_coding
