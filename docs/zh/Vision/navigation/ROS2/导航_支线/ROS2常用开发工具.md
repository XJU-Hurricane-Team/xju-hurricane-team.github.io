**修改时间:2026.1.26**

**参与者:刘志钰**

# ROS2 常用开发工具

在机器人的开发过程中，坐标变换非常重要且不好处理，ROS 2 就基于话题通信设计了一套库和工具，用于管理机器人坐标变换。

## 1. 坐标变换工具介绍（TF)

### (1) 通过命令行使用 TF

我们可以使用 `static_transform_publisher` 来发布两个坐标系之间的静态变换：

```bash
ros2 run tf2_ros static_transform_publisher --x 0.1 --y 0.0 --z 0.2 --roll 0.0 --pitch 0.0 --yaw 0.0 --frame-id base_link --child-frame-id base_laser
```

**参数详解：**

- `--x 0.1 --y 0.0 --z 0.2`：指定 `base_link` 到 `base_laser` 的平移量。其中 x、y、z 分别代表子坐标系在父坐标系下的 x、y、z 坐标轴上的平移距离，单位为 **m**。
- `--roll 0.0 --pitch 0.0 --yaw 0.0`：指定子坐标系相对于父坐标系的旋转量。roll、pitch、yaw 分别代表绕子坐标系的 x、y、z 轴旋转的欧拉角，单位为 **rad**。
- `--frame-id` 和 `--child-frame-id`：分别用于指定父坐标系和子坐标系的名称。

除了四元数和欧拉角，还有其他用于表示姿态的方式，通过 ROS 2 中的 `mrpt2` 工具，可以方便地获取不同姿态表示之间的对应关系和可视化。

> **四元数 (Quaternion)**
>
> 四元数是一种用于标识三维空间中姿态的数学工具，在 ROS2 中经常被用于机器人的姿态描述等方面。
>
> 四元数由一个实部和三个虚部组成，通常表示为 $q = w + xi + yj + zk$。

**安装与使用姿态转换工具：**

```
# 下载工具
sudo apt install ros-humble-mrpt-*

# 打开工具
3d-rotation-converter
```

![](./picture/rotation_tool.png = 600x400)

**四元数在 TF 中的应用示例：**

```
ros2 run tf2_ros static_transform_publisher --x 0.3 --y 0.0 --z 0.0 --roll 0.0 --pitch 0.0 --yaw 0.0 --frame-id base_laser --child-frame-id wall_point
```

输出结果示例：

```
rotation: ('0.000000', '0.000000', '0.000000', '1.000000')
```

- 最后一个 `1.000000` 是四元数的实部。
- 前面三个是虚部。

> **注意：**
>
> 在真实的机器人中，固定不变的坐标关系才会使用静态坐标变换，而对于障碍物信息应该使用动态的坐标变换。
>
> 改变实部会改变四元素所代表的旋转状态：**实部与旋转角度有关，虚部与旋转轴有关**。

**查看坐标关系 (tf2_echo):**

`tf2_echo` 用于输入两个坐标之间的平移和旋转关系，第一个参数是父坐标系名称，第二个参数是子坐标系名称。

```
ros2 run tf2_ros tf2_echo base_link wall_point
```

**可视化坐标系连接关系 (view_frames):**

除了可以通过命令行计算坐标之间的关系外，还可以使用工具查看所有坐标系之间的连接关系。

```
# 需要将坐标转换的代码一直开着才能看到
ros2 run tf2_tools view_frames
```

该命令会将当前所有广播的坐标关系通过图形的方式表示出来，并在当前目录生成一个 PDF 文件和 GV 格式文件。

![](./picture/view_frames.png = 800x500)

> **注意：** 统一参考系，统一以机器人的基坐标系为标准。

### (2) 对 TF 原理的简单探究

当发布静态广播时，广播的数据就会通过话题通信发布到 `/tf_static` 话题上。

```
# 查看话题列表
ros2 topic list

# 查看话题具体信息
ros2 topic info /tf_static

# 查看该话题的消息接口类型 (是 tf2_msgs/msg/TFMessage)
ros2 interface show tf2_msgs/msg/TFMessage
```

**原理总结：**

需要注意当发布动态 TF 时，数据将发布到名称为 `/tf` 的话题上。当需要查询坐标变换关系时，则会订阅 `/tf` 和 `/tf_static` 话题，通过数据中坐标之间的关系计算要查询的坐标之间的关系，这就是 TF 的工作原理。

### (3) C++ 中的地图坐标系变换

**创建功能包：**

```
ros2 pkg create demo_cpp_tf --build-type ament_cmake --dependencies rclcpp tf2 tf2_ros geometry_msgs tf2_geometry_msgs --license Apache-2.0
```

**通过 C++ 发布静态 TF:**

可以使用 `tf2_echo` 来查看坐标关系：

```
ros2 run tf2_ros tf2_echo map target_point
```

**通过 C++ 发布动态 TF:**

发布动态 TF 和发布静态 TF 相比，最大的不同在于动态 TF 需要不断向外发布。

**通过 C++ 查询 TF 关系:**

需要三个节点都开启才有结果。

------

## 2. 常用可视化工具 rqt 与 RViz

### (1) GUI 框架 rqt

我们在前面章节中使用过 rqt，比如使用它查看节点关系、请求服务等。在任意终端输入 `rqt` 命令就可以启动 rqt。

rqt 是一个 GUI 框架，可以将各种插件工具加载为可停靠的窗口。目前没有选择插件，要添加插件，请从 **Plugins** 菜单中选择项目。

**安装 rqt 插件 (以 rqt-tf-tree 为例):**

```
# 安装插件
sudo apt install ros-humble-rqt-tf-tree -y
```

该工具将安装到 ROS 2 的默认安装目录下，安装完成后需要删除 rqt 的默认配置文件，才能让 rqt 重新扫描和加载到这个工具。

```
# 删除配置文件
rm -rf ~/.config/ros.org/rqt_gui.ini
```

*执行所有操作后，可能没有任何响应（需要重启电脑，然后重新执行一次）。*

**使用插件：**

重新启动 `rqt`，然后选择 `Plugins` → `Visualization` → `TF Tree` 选项，即可打开 TF Tree 插件。运行 TF 发布节点，单击左上角的 Refresh 按钮。

![](./picture/rqt_tf.png = 600x400)

如果需要安装 rqt 所有相关组件：

```
sudo apt install ros-$ROS_DISTRO-rqt-*
```

### (2) 数据可视化工具 RViz

在学习 TF 时，虽然可以通过 `tf2_tools` 或 `rqt-tf-tree` 查看 TF 数据帧之间的关系，但并不能直观地看到它们在空间中的关系。**RViz** 不仅可以帮助我们实现坐标变换可视化，还可以实现机器人的传感器数据、3D 模型、点云、激光雷达数据等数据的可视化与交互。

**启动 RViz:**

在任意终端输入以下命令即可打开：

```
rviz2
```

**配置步骤：**

1. RViz 窗口左侧的 **Displays** 部分用于控制显示的数据。当需要显示某个数据时，可以单击 Displays 窗口下方的 **Add** 按钮添加要显示的视图。
2. 在 **By display type** 选项卡中选择 **TF**，然后单击右下方的 OK 按钮。
3. 接着观察中间的网格，即可看到 TF 的显示结果，此时可以使用鼠标拖动显示视图，调整观察角度。
4. 选中 TF 视图选项下的 `Show Names`，在坐标系中添加对应的坐标系名称，接着修改 `Marker Scale` 选项为 6，放大坐标轴和名字。

![](./picture/rviz_tf.png = 800x600)

**设置观察视角：**

如果想从机器人的视角 `base_link` 来观察目标点 `target_point`，只需要将机器人固定在原点即可。Displays 窗口中全局选项 **Global Options** 下的 **Fixed Frame** 就是用于显示右侧视图原点的坐标系名称的，我们修改为 `base_link`。

- **修改网格：** 默认的宽度为 1m，修改 Grid 下 Cell Size 的值，就可以修改网格宽度。

**保存与加载配置：**

RViz 还支持将当前的配置保存到文件中，方便下次直接加载使用。

- **保存：** 单击标题栏的 `File` → `Save Config As` 或者直接按 `Ctrl+Shift+S` 键。

- **加载：** 启动 RViz 并指定配置文件：

  ```
  rviz2 -d ~/chapt5/rviz_tf.rviz
  ```

### (3) 数据记录工具 ros2 bag

**录制数据：**

首先在新的终端运行海龟模拟器，接着再打开新的终端，启动海龟键盘控制节点。

```
# 录制特定话题
ros2 bag record /turtle1/cmd_vel
```

> **建议：** 如果直接使用 `ros2 bag record` 将录制所有的话题数据，不过我并不推荐这样做，有些数据并不是必需的，我们只需要将自己关心的话题名称依次放到命令后，进行录制即可。

接着在键盘控制节点窗口使用方向箭头移动海龟，然后在话题录制终端按 `Ctrl+C` 键打断录制，`ros2 bag record` 在收到打断指令后就会停止录制并将数据保存到文件中。

**查看录制文件：**

在终端 bags 目录下使用 `ls` 命令，可以看到录制数据存放的目录名。这个文件夹是按照时间命名的，文件夹中存放了两个文件：

- `.db3` 结尾：存储话题数据的数据库文件。
- `metadata.yaml`：记录的描述文件。

使用 `cat` 查看描述文件内容，该文件中描述了被记录的话题名称和类型等信息，还保存了开始记录时间、持续时间、消息的数量等信息。

**回放数据：**

关闭键盘控制节点，然后重启海龟模拟器，现在来重新播放海龟的控制命令，对话题数据进行重播。

```
# 播放数据 (请替换为实际的绝对路径)
ros2 bag play /home/user/rosbag2_2023_12_11-01_57_18/
```

`bag play` 加文件夹名称，用于播放对应文件夹里的话题数据。运行完命令观察海龟窗口，海龟按照刚刚的控制轨迹移动。

**回放控制操作：**

在重新播放话题时还有很多其他操作，例如：

- **空格键**：暂停和继续播放。
- **上下键**：加快和减慢播放速度。
- **右键**：播放下一个消息。
- 可以使用 `--help` 来查看更多选项。

