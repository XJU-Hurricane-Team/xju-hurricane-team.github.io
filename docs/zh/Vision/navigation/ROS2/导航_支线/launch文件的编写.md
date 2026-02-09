**修改时间:2026.1.26**

**参与者:刘志钰**

## Launch文件的编写

这一部分大概讲解一下在ROS2中launch文件的部分具体函数，以及其具有的作用，应该如何去使用。这一部分中所有的代码示例都是以北极熊导航包中的 `rm_navigation_simulation_launch.py` 为例进行说明的，建议对照着看原文全代码效果会更好。

launch文件的头文件调用如下：

```python
import os

from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument, IncludeLaunchDescription, Node, GroupAction, SetEnvironmentVariable
from launch.conditions import IfCondition, LaunchConfigurationEquals, LaunchConfigurationNotEquals
from launch.substitutions import LaunchConfiguration, TextSubstitution
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch.descriptions import ParameterFile
from nav2_common.launch import RewrittenYaml
其中 `os` 一般用于读取文件路径。如使用函数 **`os.path.join()`**
```

在完成了头文件编写后，需要先得到 launch 文件的路径，示例代码如下：

```
bringup_dir = get_package_share_directory("pb2025_nav_bringup")
launch_dir = os.path.join(bringup_dir, "launch")
```

- `bringup_dir` 表示获取功能包的路径
- `launch_dir` 表示将 `bringup_dir` 拼接 `launch`，即在 `pb2025_nav_bringup/launch` 文件夹下。

有了这一部分前置知识，下面开始看各个函数分别表示什么以及如何使用。

------

### 1、`LaunchConfiguration` 获取已声明的参数值

这个函数可以在 launch 文件中引用已声明的参数值，实现动态配置。代码示范如下：

```
namespace = LaunchConfiguration("namespace")  # 获取namespace参数值
```

**数据流示例**：关于其中的`DeclareLaunchArgument`函数在下面会说到

```
用户输入 → DeclareLaunchArgument → LaunchConfiguration → 节点参数/路径配置
```

------

### 2、`DeclareLaunchArgument` 声明启动函数

其功能是定义可以在启动时动态配置的参数，为系统提供灵活的运行配置选项。代码示范如下：

```
declare_slam_cmd = DeclareLaunchArgument(
    "slam",
    default_value="False",
    description="Whether run a SLAM. If True, disable small_gicp and send static tf (map->odom)"
)
```

在这里，声明了一个名为`slam`的布尔型参数，默认的行为是不启动 SLAM 模式。如果想要覆盖，则在启动时进行覆盖即可，覆盖命令即 **`slam:=True`**

------

### 3、`Node` ROS2 节点启动

节点是一个基本的概念，下面说明一下应该如何具体编写，示例代码如下：

```
Node(
    package="ign_sim_pointcloud_tool",               # 节点所属的功能包
    executable="ign_sim_pointcloud_tool_node",       # 可执行文件的名称(即CMake中定义的target)
    namespace=namespace,                             # 命名空间(实现多机器人隔离)
    parameters=[configured_params]                   # 节点参数配置
)
```

在这其中有一个`configured_params`，在下面会写到。

------

### 4、`RewrittenYaml` + `ParameterFile` 动态参数处理

这里以刚刚提到的`configured_params`为例，代码示例如下：

```
configured_params = ParameterFile(
    RewrittenYaml(
        source_file=params_file,
        root_key=namespace,
        param_rewrites={},
        convert_types=True
    ),
    allow_substs=True
)
```

这一部分用于实现参数文件内容运行时动态修改，支持多机器人等复杂场景。`RewrittenYaml`是参数重写，通过在 yaml 文件中添加命名空间前缀实现，这样可以在一个 Yaml 中实现多个工作空间的内容配置。参数重写的示例如下：

```
# 原始内容
controller_server:
  ros__parameters:
    frequency: 20.0
    
# 参数重写（当namespace=robot1时）
robot1:
  controller_server:
    ros__parameters:
      frequency: 20.0
```

原始内容是指如果没有使用参数重写 (`RewrittenYaml`)，在 Yaml 文件中的参数配置的写法。参数重写的部分则表示如果使用了的话，在 Yaml 中应该怎么写参数的配置。

------

### 5、`IncludeLaunchDescription` 子 launch 文件的包含

这一部分较为简单，就是在一个 launch 文件中调用启动另一个 launch 文件，示例代码如下：

```
IncludeLaunchDescription(
    PythonLaunchDescriptionSource(os.path.join(launch_dir, "bringup_launch.py")),
    launch_arguments={
        "slam": slam,
        "map": map_yaml_file
    }.items()
)
```

此处调用的子 launch 文件是`bringup_launch.py`文件，效果则是执行`bringup_launch.py`文件，调用的参数分别是`slam`和`map_yaml_file` ，这两个参数都已经在`LaunchConfiguration`中有配置过。在这里，也可以添加一个 **`condition`** 表示条件执行控制，即根据参数值决定是否执行某个启动动作。其中的`launch_arguments`对应的正是`DeclareLaunchArgument`中声明的参数位置。

------

简单说来就是：你先获取参数值 (即 `LaunchConfiguration` )，获取到的参数值会在下面的 `DeclareLaunchArgument` 中读入具体的内容。而在 `Node` 或 `IncludeLaunchDescription` 都会需要使用你获取到的参数值。

例如这样一个简单流向：

```
world = LaunchConfiguration('world')  # 先声明参数,这里使用了world指代了'world',后面所有用到'world‘的都用world即可

# 下面进行参数声明
declare_world_cmd = DeclareLaunchArgument(              # world参数名称
        "world",
        default_value="space",  # world参数的名称定义为space
        description="map file share the same name as this parameter",
),                                                      # 在这里,将world的值定义为了"space"

# 另一个参数的声明
declare_map_yaml_path_cmd = DeclareLaunchArgument(      # 地图的yaml配置文件位置
        'map', 
        default_value=[
            TextSubstitution(text=os.path.join(nav_bringup_dir, "map", "")),
            world,                                       # 可以看到,在这里正调用了之前命名的world
            TextSubstitution(text=".yaml"),
        ],
        description='Full path to map file to load',
),

ld = LaunchDescription()

# 添加world的声明,结束
ld.add_action(declare_world_cmd)

return ld
```
