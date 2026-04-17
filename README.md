# 从零配置 ROS Noetic

# Ubuntu20.04 虚拟机准备

## 创建 Ubuntu20.04 虚拟机

**从Vmware Workstation Pro中新建虚拟机，并加载**​**​`Ubuntu20.04-iso`​**​**镜像文件；**

**硬件配置：**

* 保守配置：**8GB RAM + 6 vCPU（1*6）**
* 激进配置：**10GB RAM + 8 vCPU（1*8）**

![Ubuntu20.04 虚拟机配置](assets\image-20260417122030-moo8sop.png)

> 我的笔记本本身就只有**一颗物理 CPU**，VMware 模拟出多颗 CPU（多 Socket）没有任何性能提升，反而会增加系统调度开销。所以配置为 1*N 的模式。


**全英文下安装Ubuntu20.04系统；**

> Windows 主机开启Clash Verge的系统代理，不会影响 VMware 里 Ubuntu 的网络访问。

## 配置必要的工具

**如果主机和虚拟机之间无法互通复制粘贴，需要下载 VM tools：**

```bash
sudo apt update
sudo apt install -y open-vm-tools open-vm-tools-desktop

# 重启
sudo reboot
```

**配置SSH（Windows终端直连Ubuntu，更加方便）：**

```bash
sudo apt install openssh-server
```

## 更换Ubuntu系统软件源

```bash
# 查看目前软件源的配置
king@king:~$ cat /etc/apt/sources.list

# 备份原始源
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak

# 换为清华源，适用于Ubuntu20.04/22.04/24.04全新安装、官方默认软件源
sudo sed -i -e 's|http://cn.archive.ubuntu.com/ubuntu|http://mirrors.tuna.tsinghua.edu.cn/ubuntu|g' -e 's|http://security.ubuntu.com/ubuntu|http://mirrors.tuna.tsinghua.edu.cn/ubuntu|g' /etc/apt/sources.list

# 刷新生效
sudo apt update # 换源之后的第一次运行是 Get 而不是 Hit，会稍慢！之后的运行会很快

# 最后清一下缓存
sudo apt clean
```

> 此时可以创建一个Ubuntu20.04-base的快照，方便回滚。

# 配置 **ROS Noetic**

## 安装 **ROS**

```bash
# 安装必备依赖工具
sudo apt install -y curl gnupg lsb-release ca-certificates

# 创建密钥存储目录
sudo mkdir -p /etc/apt/keyrings

# 下载并导入 ROS 官方信任的 GPG 密钥
tmpdir=$(mktemp -d) && \
GNUPGHOME="$tmpdir" gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C1CF6E31E6BADE8868B172B4F42ED6FBAB17C654 && \
GNUPGHOME="$tmpdir" gpg --export C1CF6E31E6BADE8868B172B4F42ED6FBAB17C654 | sudo tee /etc/apt/keyrings/ros-archive-keyring.gpg > /dev/null && \
rm -rf "$tmpdir"

# 添加 ROS1 的清华镜像源
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/ros-archive-keyring.gpg] https://mirrors.tuna.tsinghua.edu.cn/ros/ubuntu/ $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/ros1.list > /dev/null

# 检查源配置
cat /etc/apt/sources.list.d/ros1.list
# deb [arch=amd64 signed-by=/etc/apt/keyrings/ros-archive-keyring.gpg] https://mirrors.tuna.tsinghua.edu.cn/ros/ubuntu/ focal main

# 刷新生效
sudo apt update

# 进行官方二进制安装
sudo apt install ros-noetic-desktop-full

# 写入配置文件，每次开终端自动加载ROS环境
echo "source /opt/ros/noetic/setup.bash" >> ~/.bashrc && source ~/.bashrc

# 检查
echo $ROS_DISTRO
# noetic
```

## 初始化rosdep

> ​`rosdep`​ 是 ROS 官方自带的依赖自动管理工具，接下来需要用国内镜像替代官方源，让 rosdep 在国内也能正常初始化。

```bash
# 先安装对应的包
sudo apt install python3-rosdep

# 手动模拟 rosdep init
sudo mkdir -p /etc/ros/rosdep/sources.list.d && \
sudo curl -o /etc/ros/rosdep/sources.list.d/20-default.list -L \
https://mirrors.tuna.tsinghua.edu.cn/github-raw/ros/rosdistro/master/rosdep/sources.list.d/20-default.list

# 配置清华源镜像
echo 'export ROSDISTRO_INDEX_URL=https://mirrors.tuna.tsinghua.edu.cn/rosdistro/index-v4.yaml' >> ~/.bashrc
# source一下
source ~/.bashrc

# 更新 rosdep 数据库
rosdep update
```

## 安装必要的**工具链**

```bash
sudo apt install -y python3-rosinstall python3-rosinstall-generator python3-wstool build-essential
```

## 启动 ROS Master 并自检

```bash
# 终端1：启动
roscore

# 终端2：
rosrun turtlesim turtlesim_node

# 终端3：控制小海龟运动
rosrun turtlesim turtle_teleop_key
```

> 此时 **Ubuntu 20.04+ROS Noetic+rosdep+开发工具链** 的环境配置完成。可以做一个快照。

# 编译 ROS 工作空间

## 安装缺失的包

```bash
sudo apt install ros-noetic-object-recognition-msgs ros-noetic-move-base-msgs ros-noetic-moveit-msgs ros-noetic-moveit
```

## 编译

```bash
cd ~/projects/catkin_ws
catkin_make
```

## 实验验证

**每一个步骤都需要打开一个新的终端**，并记得先刷新环境变量。

无论做哪个实验，第一步永远是在新终端中 source 工作空间

```bash
cd ~/projects/catkin_ws

# 系统级setup已经写入.bashrc文件：source /opt/ros/noetic/setup.bash
# 还需要source用户级 setup.bash：
source devel/setup.bash
```

### 1. 话题通信 (Topic) —— 异步单向数据流

**功能包：**  `learning_topic`​

这是最常用的通信方式，发布者（Publisher）不停发消息，订阅者（Subscriber）接收。

**步骤：**

1. **终端 1 (启动核心)：**

    ```bash
    roscore
    ```
2. **终端 2 (运行发布者)：**

    ```bash
    cd ~/projects/catkin_ws
    source devel/setup.bash

    rosrun learning_topic person_publisher  
    ```
3. **终端 3 (运行订阅者)：**

    ```bash
    cd ~/projects/catkin_ws
    source devel/setup.bash

    rosrun learning_topic person_subscriber
    ```

> 下面的实验中，有服务端和客户端的，**先启动服务端**！

### 2. 服务通信 (Service) —— 同步双向请求/响应

**功能包：**  `learning_service`​

这是一问一答的模式，客户端（Client）发请求，服务器（Server）给结果。

**步骤：**

1. **终端 1 (如果已关了重开)：**

    ```bash
    roscore
    ```
2. **终端 2 (运行服务端 Server)：**

    ```bash
    cd ~/projects/catkin_ws
    source devel/setup.bash

    rosrun learning_service person_server
    ```
3. **终端 3 (运行客户端 Client)：**

    ```bash
    cd ~/projects/catkin_ws
    source devel/setup.bash

    rosrun learning_service person_client
    ```

---

### 3. 动作通信 (Action) —— 带反馈的异步任务

**功能包：**  `learning_communication`​

动作通信适合耗时任务（如机械臂抓取），它不仅有结果，还有过程反馈。

**步骤：**

1. **终端 1：**

    ```bash
    roscore
    ```
2. **终端 2 (运行 Action 服务端)：**

    ```bash
    cd ~/projects/catkin_ws
    source devel/setup.bash

    rosrun learning_communication server
    ```
3. **终端 3 (运行 Action 客户端)：**

    ```bash
    cd ~/projects/catkin_ws
    source devel/setup.bash

    rosrun learning_communication client 123 456
    ```

### 4. 参数服务器 (Parameter Server) —— 全局共享数据

**功能包：**  `learning_parameter`​ 

参数服务器就像一个公共字典，所有节点都可以往里面存数据、读数据。

这个 `parameter_config.cpp`​ 是一个非常经典的参数服务器验证例程，它是配合 **小乌龟仿真 (turtlesim)**  一起使用的。

这个程序的逻辑是：读取小乌龟的背景颜色 -> 改成白色 -> 再读取确认 -> 刷新界面。

**步骤：**

1. 终端 1：

    ```bash
    roscore
    ```

2. 终端 2：先运行参数节点：

    ```bash
    rosrun learning_parameter parameter_config
    ```
3. 终端 3：启动小乌龟

    ```bash
    rosrun turtlesim turtlesim_node
    ```

### 5. Learning_tf 坐标变换实验

**功能包：**  `learning_tf`​

TF 是 ROS 中管理坐标系变换的核心库，通常用小乌龟（turtlesim）来演示。

**步骤：**

1. 发布launch文件，启动多个节点

    ```bash
    roslaunch learning_tf start_demo_with_listener.launch
    ```

2. 启动键盘控制小海龟的节点

    ```bash
    rosrun turtlesim turtle_teleop_key
    ```

> 如果在运行 `rosrun`​ 时忘记了节点名，可以使用以下命令查看功能包里编译好的可执行文件：
>
> ```bash
> cd ~/projects/catkin_ws
>
> # 查看功能包下有哪些节点
> ls devel/lib/learning_topic/
> ls devel/lib/learning_service/
> ```

‍
