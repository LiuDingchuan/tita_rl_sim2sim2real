<!--
 * @Author: hilab-workshop-ldc 2482812356@qq.com
 * @Date: 2025-05-17 18:19:28
 * @LastEditors: Please set LastEditors
 * @LastEditTime: 2025-06-23 22:30:58
 * @FilePath: /tita_rl_sim2sim2real/ReadMe.md
 * @Description: 这是默认设置,请设置`customMade`, 打开koroFileHeader查看配置 进行设置: https://github.com/OBKoro1/koro1FileHeader/wiki/%E9%85%8D%E7%BD%AE
-->
持续更新中~~~  
有问题欢迎在Issues中反馈，欢迎大家一起加入学习。


若启动了conda环境先把环境关闭

```bash 
conda deactivate
```


### 安装环境（宿主机host中执行）

安装ros2_control :

https://github.com/DDTRobot/TITA_ROS2_Control_Sim.git

**或者直接使用docker，已配置好tensor\webots\ros2\ros2-control环境**

https://github.com/DDTRobot/webots2023b_ros2_docker

安装ros-humble-pinocchio
```
sudo apt install ros-humble-pinocchio
```

按照以下命令行拉取编译代码  

```bash 
#本仓库
git clone https://github.com/DDTRobot/sim2sim2real.git

cd sim2sim2real

#其他仓库
vcs import < sim2sim2real.repos
```

编译之前先修改这个文件：src/tita_locomotion/tita_controllers/tita_controller/src/fsm/FSMState_RL.cpp
![alt text](/pictures/image.png)
把位置修改为把推理出来的model_gn.engine路径。 **如果是在docker下运行，请将.engine的路径修改为docker中的路径，而不是宿主机的路径！！（一般为/mnt/dev/*.engine）**


### 运行 (docker和 host都可以)

```bash
#编译
source /opt/ros/humble/setup.bash && colcon build --packages-up-to locomotion_bringup webots_bridge gazebo_bridge robot_inertia_calculator template_ros2_controller tita_controller joy_controller keyboard_controller

source install/setup.bash 

#启动webots仿真
source /opt/ros/humble/setup.bash && source install/setup.bash && ros2 launch locomotion_bringup sim_bringup.launch.py
#ros2 launch locomotion_bringup sim_diablopluspro.launch.py
#启动键盘指令输入终端
source /opt/ros/humble/setup.bash && source install/setup.bash && ros2 run keyboard_controller keyboard_controller_node --ros-args -r __ns:=/tita

#如果webots遇到不显示机器人mesh的情况:
sudo mkdir -p /usr/share/robot_description
sudo cp -r src/tita_locomotion/tita_description/tita /usr/share/robot_description/

```

### 实机部分

```bash 
#拷贝文件进入机器
scp -r 你的路径/tita_ros2/src robot@192.168.42.1:~/tita_ros2/

#连接机器：
ssh robot@192.168.42.1
#密码：
apollo

#关掉自启的ros2
systemctl stop tita-bringup.service

cd tita_ros2/

source /opt/ros/humble/setup.bash

#编译
#colcon build --packages-up-to locomotion_bringup robot_inertia_calculator template_ros2_controller tita_controller joy_controller keyboard_controller hw_broadcaster
colcon build --packages-up-to locomotion_bringup robot_inertia_calculator tita_controller keyboard_controller hardware_bridge tita_description joy_controller 
#--symlink-install --parallel-workers 5 --cmake-args -DCMAKE_BUILD_TYPE=RelWithDebInfo

--symlink-install 作用是工作空间（source）中的文件发生变化时，安装目录（build）的文件也会随着改变。这样在调试的时候会高效一些

--cmake-args '-DCMAKE_BUILD_TYPE=RelWithDebInfo' 可以选择 RelWithDebInfo、Debug 或 Release。对应 CMake 的三种编译选项，其中 Release 模式主要用于发布代码，会忽略调试信息；Debug 模式主要用于调试代码，因为需要生成调试信息，所以时间较长；RelWithDebInfo 则在 Release 模式下生成调试信息，也可以用于调试代码。通常建议使用 RelWithDebInfo 即可。

--parallel-workers 5 开启多个线程并行编译

source install/setup.bash

#连接遥控器
crsf-app -bind

#分别运行
nohup ros2 launch locomotion_bringup hw_bringup.launch.py ctrl_mode:=wbc &
nohup ros2 launch joy_controller joy_controller.launch.py & #手柄调试
ros2 run keyboard_controller keyboard_controller_node --ros-args -r __ns:=/tita #键盘调试
```

如果您的tensorrt是10.x版本，参考[这里](https://github.com/DDTRobot/tita_rl_sim2sim2real/issues/1)

### ROS bag相关配置

```bash
source install/setup.bash
ros2 bag record -o <输出数据包名称> <话题名称1> <话题名称2> ... #-o指定输出的数据包名称

#例
ros2 bag record -o tita_controller_log /tita/tita_controller/plan_commands /tita/tita_controller/robot_states /tita/imu_sensor_broadcaster/imu

#查看记录的bag文件内容
ros2 bag info tita_controller_log

#重放bag文件
ros2 bag play tita_controller_log
```