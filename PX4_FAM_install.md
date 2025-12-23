# 配置 PX4 Fully-Actuated-Multiroter 教程

## 前置条件
1. Ubuntu 20.04 LTS已安装
2. Localhost能够访问Github

## 配置基本环境
```
sudo apt install git curl htop
sudo apt update
sudo apt upgrade -y
```

### 网络配置:

虚拟机：主机需翻墙，网络模式设为镜像并在虚拟机中设置端口号，使之与主机代理服务器端口号一致
```
# WSL2 Setting Network Mirror 模式设置代理
export http_proxy=http://localhost:7890
export https_proxy=http://localhost:7890
export all_proxy=socks5://localhost:7890
```

原生Ubuntu：翻墙或更改源

## 根据 ROS2 Foxy 官网进行 ROS2 Foxy 安装
参考：https://docs.ros.org/en/foxy/Installation/Ubuntu-Install-Debians.html
```
locale  # check for UTF-8

sudo apt update && sudo apt install locales
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8

locale  # verify settings

sudo apt install software-properties-common
sudo add-apt-repository universe

sudo apt update && sudo apt install curl -y
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null

sudo apt update
sudo apt upgrade

sudo apt install ros-foxy-desktop python3-argcomplete  
sudo apt install ros-dev-tools
```
	

### 将 ROS2 Foxy 路径放入 ~/.bashrc 中
```
# ROS2 Foxy源
source /opt/ros/foxy/setup.bash
```


### 测试 ROS2 Foxy是否安装成功
```
ros2 run demo_nodes_cpp talker
ros2 run demo_nodes_py listener
```

## 安装Python依赖
```
sudo apt install python3-pip
pip install --user "setuptools<65"
pip install --user testresources
pip install --user -U empy==3.3.4 pyros-genmsg setuptools  
```

## 安装Uxrce-DDS agent 
参考：https://github.com/scarforhere/px4-fully-actuated-multirotor/blob/main/media/uxrce_dds_setup.md
```
cd ~
export ROS_DISTRO=foxy 
source /opt/ros/$ROS_DISTRO/setup.bash
mkdir -p microros_ws/src
cd microros_ws
git clone -b $ROS_DISTRO https://github.com/micro-ROS/micro_ros_setup.git src/micro_ros_setup

sudo rosdep init
rosdep update
rosdep update && rosdep install --from-paths src --ignore-src -y
python3 -m pip install --upgrade pip setuptools importlib_metadata --user
colcon build

source install/local_setup.bash
ros2 run micro_ros_setup create_agent_ws.sh
ros2 run micro_ros_setup build_agent.sh

cd ~/microros_ws    
source install/local_setup.sh
ros2 run micro_ros_agent micro_ros_agent udp4 --port 8888
```

### 创建uXRCE-DDS_Agent自动化：

#### 在用户根目录创建uXRCE-DDS_Agent_start.sh写入:
```
#!/bin/bash

cd microros_ws

export ROS_DISTRO=foxy 
source /opt/ros/$ROS_DISTRO/setup.bash
source install/local_setup.bash
source install/local_setup.sh
ros2 run micro_ros_agent micro_ros_agent udp4 --port 8888
```

#### 使之可执行
```
chmod +x ~/uXRCE-DDS_Agent_start.sh
```

#### 测试Uxrce-DDS agent是否安装成功
```
bash ~/uXRCE-DDS_Agent_start.sh
```

## 安装PX4 v1.15版本 
```
cd ~
mkdir -p ~/PX4
cd ~/PX4
git clone https://github.com/PX4/PX4-Autopilot.git --recursive PX4-FAM 

cd PX4-FAM
git checkout release/1.15 
make submodulesclean
make clean
make distclean
	
pip3 install --upgrade numpy
python3 -m pip install pip==19.3.1 --user
bash ~/PX4/PX4-FAM/Tools/setup/ubuntu.sh
```

### 验证是否安装配置成功
```
make px4_sitl gazebo-classic <-------------------------------
```

## 配置PX4-FAM适配Fullu-actuated-multiroter
```
cd ~/PX4
git clone https://github.com/scarforhere/px4-fully-actuated-multirotor.git     

cp -rf /home/fam/PX4/px4-fully-actuated-multirotor/px4_changes/Tools/* /home/fam/PX4/PX4-FAM/Tools/
cp -rf /home/fam/PX4/px4-fully-actuated-multirotor/px4_changes/ROMFS/* /home/fam/PX4/PX4-FAM/ROMFS/
```

### 模型添加至 ROMFS/px4fmu_common/init.d-posix/airframes/CMakeLists.txt
```
  # [22000, 22999] Reserve for custom models
  6020_gazebo-classic_hex_x
  6021_gazebo-classic_tilted_hex
  6022_gazebo-classic_tilted_hex_arm
```

### 添加 gazebo targets 在 src/modules/simulation/simulator_mavlink/sitl_targets_gazebo-classic.cmake
```
set(models
	advanced_plane
	believer
	boat
     .
     .
     .
  # Modification for Fully-Actuated-Multiroter
	hex_x
	tilted_hex
	tilted_hex_arm
)
```

### 对照px4-changes/dds_topics.yaml补充src/modules/uxrce_dds_client/dds_topics.yaml的缺少项
```
# Custom for Fully Actuated Multiroter
  - topic: /fmu/out/hover_thrust_estimate
    type: px4_msgs::msg::HoverThrustEstimate
    
  - topic: /fmu/out/vehicle_torque_setpoint
    type: px4_msgs::msg::VehicleTorqueSetpoint
    
  - topic: /fmu/out/vehicle_thrust_setpoint
    type: px4_msgs::msg::VehicleThrustSetpoint
    
  - topic: /fmu/out/actuator_motors
    type: px4_msgs::msg::ActuatorMotors
    
  - topic: /fmu/out/vehicle_rates_setpoint
    type: px4_msgs::msg::VehicleRatesSetpoint
    
  - topic: /fmu/out/vehicle_attitude_setpoint
    type: px4_msgs::msg::VehicleAttitudeSetpoint
    
  - topic: /fmu/out/vehicle_acceleration
    type: px4_msgs::msg::VehicleAcceleration
```

### 该项目取消注释
```
- topic: /fmu/out/vehicle_angular_velocity
  type: px4_msgs::msg::VehicleAngularVelocity		
```

### 验证是否改动成功
```
make clean
# 绝对不能执行make distclean!!!会重置操作
sudo apt install ros-foxy-gazebo-ros-pkgs
make px4_sitl gazebo-classic_tilted_hex_arm
```

## 安装QGrandControl 
参考：https://docs.qgroundcontrol.com/Stable_V5.0/en/qgc-user-guide/getting_started/download_and_install.html
```
cd ~
mkdir ~/App
cd App

sudo usermod -a -G dialout $USER
sudo apt-get remove modemmanager -y
sudo apt install gstreamer1.0-plugins-bad gstreamer1.0-libav gstreamer1.0-gl -y
sudo apt install libfuse2 -y
sudo apt install libxcb-xinerama0 libxkbcommon-x11-0 libxcb-cursor-dev -y

wget https://github.com/mavlink/qgroundcontrol/releases/download/v4.2.8/QGroundControl.AppImage

chmod +x QGroundControl.AppImage
```

## 安装PlotJuggler 
参考： https://github.com/facontidavide/PlotJuggler
```
sudo snap install plotjuggler
sudo apt update
sudo apt install ros-foxy-plotjuggler-ros
ros2 run plotjuggler plotjuggler
```

## 构建ROS2 工作空间
参考： https://github.com/TareqAlqutami/px4-fully-actuated-multirotor/blob/main/media/ros2_workspace_setup.md
```
cd ~
mkdir ros2_rode
cd ros2_code
git clone https://github.com/PX4/px4_msgs.git src/px4_msgs
cd src/px4_msgs
git checkout bd9dc0fae0162960a31af8f232a9e9f85522ae73
cd ../..

cd ros2_code
git clone https://github.com/TareqAlqutami/traj_gen.git src/traj_gen

cp -r /home/fam/PX4/px4-fully-actuated-multirotor/ros2_code/src/* \
      /home/fam/ros2_code/src/
cp -r /home/fam/PX4/px4-fully-actuated-multirotor/scripts /home/fam/ros2_code/

sudo apt install ros-foxy-joint-state-publisher-gui
sudo apt install ros-foxy-xacro

source ~/ros2_code/scripts/source_ros2.bash

colcon build --symlink-install

python3 -m pip install scipy
```
导入 ros2_code 源
```
# 导入 ros2_code 源
source /home/fam/ros2_code/install/setup.bash
```

### 验证 ROS2 工作空间是否工作正常
```
ros2 launch am_description display.launch.py urdf_model:=src/am_description/urdf/tilted_hex_arm.urdf   
```


## 安装 ROS2 bridge 将 gazebo 中 map 转化为 ROS2 中 TF 
参考： https://docs.px4.io/main/en/ros2/user_guide#foxy
```
cd ~/ros2_code/src
git clone https://github.com/PX4/px4_ros_com.git
cd ~/ros2_code
rm -rf build install log

colcon build
source install/local_setup.bash

# 测试
ros2 launch px4_ros_com sensor_combined_listener.launch.py
```

	

	