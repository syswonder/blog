# webots 开发日志（一）webots 开发环境配置说明

时间: 2026/01/06

作者: 陈衍廷

联系方式：ytchen25@pku.edu.cn

## webots 简介

  webots 是一款开源的专业级机器人仿真软件，其具有以下特点：
  * **3D 仿真**：提供逼真的物理引擎，支持碰撞检测、动力学模拟。
  * **跨平台**：支持 Windows、Linux、macOS 等跨平台开发。
  * **开源免费**：2018年由 Cyberbotics 开源，采用 Apache 2.0 许可证
  * **多编程语言支持**：可用 C/C++、Python、Java、MATLAB 等编写控制器。
  * **ROS 集成**：原生支持 ROS/ROS2，便于与机器人操作系统对接。

  本文将介绍一下如何安装 webots 以及 webots_ros2 包，从而便于使用 webots 与 ROS2 进行联合仿真。

## 环境说明
 操作系统：ubuntu 22.04
 ROS2：ROS2 humble
 webots：2025a

## ROS2 安装教程
  因为我们通常需要使用 webots 与 ROS2 来进行联合仿真，故需要提前安装好 ROS2 的开发环境。具体安装方式可详见官方的安装教程：https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debs.html。
  也可以在网上搜索其他相关的教程来进行安装，本文不再赘述。本人安装的 ROS2 版本为 ROS2 humble，故以下说明皆基于 ROS2 humble 环境。

## webots 安装教程
#### 下载与安装
  从 webots 的[官方仓库](https://github.com/cyberbotics/webots/releases)上选择合适的版本，并下载 deb 包来进行安装。
  本文选择的版本为 2025a，故下载和安装命令如下
  ```Shell
  wget https://github.com/cyberbotics/webots/releases/download/R2025a/webots_2025a_amd64.deb
  sudo dpkg -i webots_2025a_amd64.deb
  ```
  webots 的默认安装路径为：
  ```Shell
  /usr/local/webots
  ```
#### 安装验证
  安装完成后，就可以打开 webots 并开始创建和运行机器人仿真。在终端中输入以下命令启动 webots：
  ```Shell
  webots
  ```

## webots_ros2 安装
#### webots_ros2 简介
  webots_ros2 是一个软件包，提供了在 Webots 开源 3D 机器人仿真器中模拟机器人所需的接口。它通过 ROS2 的消息（messages）、服务（services）和动作（actions）与 ROS2 进行集成。
#### 安装方式一：包管理工具一键安装
  可以直接使用包管理工具来进行一键安装：
  ``` Shell
  sudo apt-get install ros-humble-webots-ros2
  ```
#### 安装方式二：源码安装
  如果上面的安装方式不适用，也可以从源码进行安装。
  首先创建一个包含 src 目录的 ROS 2 工作空间。
  ```bash
  mkdir -p ~/ros2_ws/src
  ```
  加载 ROS 2 环境。
  ```bash
  source /opt/ros/humble/setup.bash
  ```
  从 Github 获取源代码。
  ```bash
  cd ~/ros2_ws
  git clone --recurse-submodules https://github.com/cyberbotics/webots_ros2.git src/webots_ros2
  ```
  安装软件包依赖项。
  ```bash
  sudo apt install python3-pip python3-rosdep python3-colcon-common-extensions
  sudo rosdep init && rosdep update
  rosdep install --from-paths src --ignore-src --rosdistro humble
  ```
  使用 colcon 构建软件包。
  ```bash
  colcon build
  ```
  加载此工作空间。
  ```bash
  source install/local_setup.bash
  ```
  非 ROS2 humble 版本的 webots_ros2 安装可详见官方教程： https://github.com/cyberbotics/webots_ros2/wiki/Getting-Started
  





