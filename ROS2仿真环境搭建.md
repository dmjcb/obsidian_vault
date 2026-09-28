## 一、安装基础包

```sh
sudo apt update

sudo apt install -y ubuntu-drivers-common linux-headers-$(uname -r)
```

## 二、安装NVIDIA驱动

```sh
sudo ubuntu-drivers install
```

## 三、安装docker

```sh
sudo apt install -y \
    docker-ce \
    docker-ce-cli \
    containerd.io \
    docker-buildx-plugin \
    docker-compose-plugin
```

## 四、NVIDIA Container Toolkit

目标

```
Ubuntu
    ↓
NVIDIA Driver
    ↓
NVIDIA Container Toolkit
    ↓
Docker
    ↓
RTX 3060
```

安装

```sh
sudo apt-get update

sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg2
    
    
curl -fsSL \
https://nvidia.github.io/libnvidia-container/gpgkey \
| sudo gpg --dearmor \
-o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L \
https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
| sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
| sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```


```sh
sudo apt update

sudo apt install -y nvidia-container-toolkit
```

把 NVIDIA Runtime 接入 Docker

```sh
sudo nvidia-ctk runtime configure --runtime=docker
```

```sh
sudo systemctl restart docker
```

检查

```sh
sudo docker info | grep -i runtime
```

应该出现

```sh
 Runtimes: runc io.containerd.runc.v2 nvidia

 Default Runtime: runc
```

#### 验证 Docker GPU

```sh
sudo docker run --rm --gpus all ubuntu:22.04 nvidia-smi
```

## 五、ROS 2 + Go2 推荐方案

```
Go2
 ↓
ROS2
 ↓
Gazebo
 ↓
LiDAR / IMU
 ↓
SLAM
 ↓
Nav2
 ↓
自主导航
```

## 六、项目

宿主机：

```
mkdir -p ~/robotics/go2_docker
cd ~/robotics/go2_docker

mkdir -p ws/src
```

以后：

```
~/robotics/go2_docker/
│
├── Dockerfile
├── compose.yaml
├── .env
│
└── ws/
    ├── src/
    ├── build/
    ├── install/
    └── log/
```

全部都是你自己的

### 创建 ROS2 + CUDA + Gazebo Dockerfile

创建：

```
cd ~/robotics/go2_docker

nano Dockerfile
```

内容：

```dockerfile
FROM nvidia/cuda:12.6.3-devel-ubuntu22.04

ARG USER_NAME=rosdev
ARG USER_UID=1000
ARG USER_GID=1000

ENV DEBIAN_FRONTEND=noninteractive
ENV LANG=en_US.UTF-8
ENV LC_ALL=en_US.UTF-8
ENV ROS_DISTRO=humble

# ----------------------------
# 基础工具
# ----------------------------
RUN apt-get update && apt-get install -y \
    locales \
    curl \
    wget \
    gnupg2 \
    lsb-release \
    ca-certificates \
    software-properties-common \
    git \
    vim \
    nano \
    sudo \
    build-essential \
    cmake \
    pkg-config \
    python3-pip \
    python3-dev \
    python3-venv \
    mesa-utils \
    xauth \
    && locale-gen en_US en_US.UTF-8 \
    && update-locale LANG=en_US.UTF-8 \
    && rm -rf /var/lib/apt/lists/*

# ----------------------------
# ROS repository
# ----------------------------
RUN add-apt-repository universe

RUN curl -sSL \
    https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
    -o /usr/share/keyrings/ros-archive-keyring.gpg

RUN echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] \
    http://packages.ros.org/ros2/ubuntu \
    $(. /etc/os-release && echo $UBUNTU_CODENAME) main" \
    > /etc/apt/sources.list.d/ros2.list

# ----------------------------
# ROS2 Humble + 开发工具
# ----------------------------
RUN apt-get update && apt-get install -y \
    ros-humble-desktop \
    ros-dev-tools \
    python3-rosdep \
    python3-colcon-common-extensions \
    python3-vcstool \
    && rm -rf /var/lib/apt/lists/*

# ----------------------------
# Go2 / Gazebo / ros2_control
# ----------------------------
RUN apt-get update && apt-get install -y \
    ros-humble-gazebo-ros2-control \
    ros-humble-xacro \
    ros-humble-robot-localization \
    ros-humble-ros2-control \
    ros-humble-ros2-controllers \
    ros-humble-controller-manager \
    ros-humble-teleop-twist-keyboard \
    ros-humble-rmw-cyclonedds-cpp \
    ros-humble-velodyne \
    ros-humble-velodyne-description \
    ros-humble-velodyne-gazebo-plugins \
    && rm -rf /var/lib/apt/lists/*

# ----------------------------
# 创建普通用户
# 避免编译产生 root-owned 文件
# ----------------------------
RUN groupadd --gid ${USER_GID} ${USER_NAME} \
    && useradd \
        --uid ${USER_UID} \
        --gid ${USER_GID} \
        -m ${USER_NAME} \
        -s /bin/bash \
    && echo "${USER_NAME} ALL=(ALL) NOPASSWD:ALL" \
        > /etc/sudoers.d/${USER_NAME}

RUN rosdep init || true

USER ${USER_NAME}

RUN echo "source /opt/ros/humble/setup.bash" >> /home/${USER_NAME}/.bashrc

WORKDIR /workspace/ws

CMD ["/bin/bash"]
```

这里最关键的是：

```
FROM nvidia/cuda:12.6.3-devel-ubuntu22.04
```

说明 CUDA 只存在于镜像内部。

---

### 创建 `.env`

先执行：

```
cd ~/robotics/go2_docker
```

创建：

```
cat > .env <<EOF
USER_NAME=$(id -un)
USER_UID=$(id -u)
USER_GID=$(id -g)
ROS_DOMAIN_ID=42
EOF
```

查看：

```
cat .env
```

例如：

```
USER_NAME=dmjcb
USER_UID=1001
USER_GID=1001
ROS_DOMAIN_ID=42
```

---

ROS_DOMAIN_ID 为什么特别重要

假设实验室服务器上：

```
张三 → ROS2
李四 → ROS2
你   → ROS2
```

ROS2 默认使用 DDS 自动发现。

如果所有人：

```
ROS_DOMAIN_ID=0
```

就可能出现非常迷惑的情况：

你执行：

```
ros2 topic list
```

突然发现：

```
/别人的机器人/cmd_vel
/别人的/lidar
```

甚至你的节点和别人的仿真互相通信。

因此给自己分配一个：

```
ROS_DOMAIN_ID=42
```

其他用户用：

```
41
43
44
...
```

这样逻辑上隔离。

---

仿真期间进一步禁止 ROS 跑到实验室局域网

如果现在只是 Docker 内本地仿真，可以使用：

```
ROS_LOCALHOST_ONLY=1
```

这样 ROS2 DDS 不需要去实验室 LAN 里广播发现信息。

以后真正连接 Go2：

```
ROS_LOCALHOST_ONLY=0
```

再开放网络。

这是公共环境中非常实用的设置。

---

### 创建 compose.yaml

```json
services:
  go2:
    build:
      context: .
      args:
        USER_NAME: ${USER_NAME}
        USER_UID: ${USER_UID}
        USER_GID: ${USER_GID}

    image: go2-humble-cuda12.6:${USER_NAME}

    container_name: ${USER_NAME}_go2_humble

    network_mode: host

    gpus: all

    environment:
      DISPLAY: ${DISPLAY}
      XAUTHORITY: /tmp/.docker.xauth

      NVIDIA_VISIBLE_DEVICES: all
      NVIDIA_DRIVER_CAPABILITIES: compute,utility,graphics,display

      ROS_DOMAIN_ID: ${ROS_DOMAIN_ID}
      ROS_LOCALHOST_ONLY: "1"

    volumes:
      - ./ws:/workspace/ws
      - /tmp/.X11-unix:/tmp/.X11-unix:rw
      - ${HOME}/.docker.xauth:/tmp/.docker.xauth:ro

    working_dir: /workspace/ws

    shm_size: "2gb"

    stdin_open: true
    tty: true
```

这里 NVIDIA 对 OpenGL/GUI 容器明确要求相关场景加入：

```
graphics
display
```

capability；CUDA/NVML 则需要：

```
compute
utility
```

所以这里写：

```
compute,utility,graphics,display
```

比较适合 Gazebo/RViz

### 构建 ROS Docker 镜像

现在`~/robotics/go2_docker` 执行

```
sudo docker compose build
```

第一次会下载：

```
Ubuntu
CUDA
ROS
Gazebo
```

体积比较大，这是正常的

```sh
sudo docker images
```

创建容器

```sh
sudo docker compose up -d
```

验证

终端A进入容器执行

```sh
ros2 run demo_nodes_py listener
```

终端B进入容器执行

```sh
ros2 run demo_nodes_cpp talker
```

## 下载项目

进入`/workspace/ws/src`

```sh
git clone https://github.com/anujjain-dev/unitree-go2-ros2.git
```

项目目前明确提供 ROS2 Humble 下的：

````
Go2 description
CHAMP
Gazebo
RViz
teleop
IMU
2D LiDAR
Velodyne
````

#### rosdep 是什么

现在执行：

```sh
rosdep update
```

然后`/workspace/ws`执行

```sh
rosdep install --from-paths src --ignore-src -r -y
```

你以后会经常看到这个命令。

可以理解成：

```
扫描 ROS package.xml

          ↓

看看项目需要哪些依赖

          ↓

调用 apt 安装缺失 ROS/system package
```

而且这里的：

```
sudo apt
```

只发生在Docker里面

不会污染宿主机。

## 理解 ROS2 Workspace

现在目录类似：

```
ws/
├── src/
│   └── unitree-go2-ros2
│
├── build/
├── install/
└── log/
```

含义：

### src

源码：

```
C++
Python
URDF
launch
yaml
```

 - build

编译临时文件

-  install

最终 ROS 包环境

- log

编译日志

以后开发你主要改：

```
src/
```

因此 Docker 即使删掉：

```
docker rm ...
```

代码仍然在宿主：

```
~/robotics/go2_docker/ws/src
```

不会丢。

---

### 编译 Go2

```
cd /workspace/ws
```

然后：

```
source /opt/ros/humble/setup.bash
```

编译：

```
colcon build --symlink-install
```

### 加载工作空间

编译后：

```
source /workspace/ws/install/setup.bash
```

检查：

```
ros2 pkg list | grep go2
```

应该能发现 Go2 package。

### 第一次启动 Go2 Gazebo

按照该项目当前文档：

```
ros2 launch go2_config gazebo.launch.py
```

应该出现 Gazebo。

里面应该能看到：

```
Unitree Go2
```

该仓库明确说明这个 demo 不需要真实机器人

### Gazebo + RViz 一起运行

```
ros2 launch go2_config gazebo.launch.py rviz:=true
```

如果成功：

```
Gazebo
+
RViz2
+
Go2
```

就全部起来了。

到这一步你的第一阶段环境基本完成。

### 控制 Go2 行走

新开一个终端：

```
cd ~/robotics/go2_docker
sudo docker compose exec go2 bash
```

进入：

```
source /opt/ros/humble/setup.bash
source /workspace/ws/install/setup.bash
```

启动键盘控制：

```
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

一般使用：

```
i → 前进
, → 后退
j → 左转
l → 右转
k → 停止
```

你应该看到：

```
keyboard
   ↓
teleop_twist_keyboard
   ↓
/cmd_vel
   ↓
CHAMP controller
   ↓
腿部轨迹
   ↓
ros2_control
   ↓
Gazebo joints
   ↓
Go2运动
```

这条数据链非常值得理解。

---

# 三十六、第一次学习 ROS，不要直接看代码

先执行：

```
ros2 node list
```

看看有哪些节点。

再：

```
ros2 topic list
```

应该看到类似：

```
/cmd_vel
/joint_states
/tf
/tf_static
/imu
...
```

然后：

```
ros2 topic info /cmd_vel
```

再：

```
ros2 interface show geometry_msgs/msg/Twist
```

你会看到：

```
Vector3 linear
Vector3 angular
```

这就解释了：

```
/cmd_vel
```

实际上是在给机器人发送：

```
vx
vy
vz
wx
wy
wz
```

## 不用键盘，直接测试控制

例如：

```
ros2 topic pub \
    -r 10 \
    /cmd_vel \
    geometry_msgs/msg/Twist \
    "{linear: {x: 0.2}, angular: {z: 0.0}}"
```

含义：

```
线速度 x = 0.2 m/s
角速度 z = 0
```

即：

> 向前走。

停止：

```
Ctrl+C
```

然后发送一次零速度：

```
ros2 topic pub \
    --once \
    /cmd_vel \
    geometry_msgs/msg/Twist \
    "{linear: {x: 0.0}, angular: {z: 0.0}}"
```

如果 Go2 可以响应这一套命令，那么：

```
ROS2 → controller → Gazebo
```

控制链就是通的。

---

## 查看机器人关节

执行：

```
ros2 topic echo /joint_states
```

你会看到类似：

```
name:
  FL_hip_joint
  FL_thigh_joint
  FL_calf_joint
  ...

position:
velocity:
effort:
```

Go2 四条腿，每条腿三个主要自由度，合计：

```
4 × 3 = 12 DOF
```

ROS 中：

```
joint_states
```

就是非常重要的机器人状态数据。

---

## 查看 TF

执行：

```
ros2 run tf2_tools view_frames
```

机器人会形成类似：

```
map
 ↓
odom
 ↓
base_link
 ├── FL_hip
 │    └── FL_thigh
 │         └── FL_calf
 │
├── FR_hip
├── RL_hip
└── RR_hip
```

以后你学 SLAM/Nav2 时会一直碰到：

```
map
odom
base_link
laser
```

这些坐标系。

---

## 四十、运行 LiDAR 版本

这个 Go2 项目还提供 Velodyne 配置：

```
ros2 launch go2_config gazebo_velodyne.launch.py
```

Gazebo + RViz：

```
ros2 launch \
    go2_config \
    gazebo_velodyne.launch.py \
    rviz:=true
```

点云 topic 按项目文档是：

````
/velodyne_points
``` :chatgpt-content-reference{index="15"}


检查：

```bash
ros2 topic list | grep velodyne
````

然后：

```
ros2 topic hz /velodyne_points
```

---

## 四十一、这样就为你后面的 SLAM/Nav2 铺好了路

最后整体数据链会变成：

```
                     Gazebo
                        │
         ┌──────────────┼──────────────┐
         │              │              │
       IMU            LiDAR        joint_states
         │              │              │
         └───────┬──────┘              │
                 │                     │
                 ↓                     ↓
              SLAM                Odometry
                 │                     │
                 └──────────┬──────────┘
                            ↓
                           map
                            │
                            ↓
                          Nav2
                            │
                            ↓
                         /cmd_vel
                            │
                            ↓
                         CHAMP
                            │
                            ↓
                         ros2_control
                            │
                            ↓
                          Go2
```

这就是你之后课程设计真正需要掌握的系统结构。

---

## 四十二、公共机器特别建议加入资源限制

RTX 3060 只有一张的时候：

```
GPU 显存没法像 CPU 一样简单按 Docker 用户严格平分
```

所以你的 Gazebo/PyTorch 任务仍可能占别人的 GPU。

运行前：

```
nvidia-smi
```

看别人有没有任务。

CPU 可以限制。

在 `compose.yaml` 添加：

```
    cpus: 6
    mem_limit: 12g
```

例如：

```
services:
  go2:
    ...
    cpus: 6
    mem_limit: 12g
    shm_size: 2gb
```

这样不会随便吃光：

```
32核 CPU
64GB RAM
```

---

## 四十三、不要给容器这些权限

在你的纯仿真阶段，尽量不要：

```
privileged: true
```

也不要：

```
--privileged
```

不要把：

```
/dev
/
 /home
```

整个挂进去。

当前只挂：

```
./ws:/workspace/ws
```

即可。

这样即使容器里面某个程序出问题，它看到的宿主数据也非常有限。

---

## 四十四、不要修改宿主 `.bashrc` 加 ROS

网上很多 ROS 教程都会让：

```
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
```

在我们的方案中：

**宿主不要这么干。**

因为宿主压根没有：

```
/opt/ros/humble
```

ROS 只存在 Docker。

只在 Docker 镜像内部：

```
source /opt/ros/humble/setup.bash
```

这样以后你完全可以同时拥有：

```
Docker A
ROS2 Humble

Docker B
ROS2 Jazzy

Docker C
Isaac Sim

Docker D
YOLO + CUDA

Docker E
Fast-LIO2
```

互不影响。

---

## 四十五、宇树官方 ROS2 环境以后怎么接

等你把 Gazebo 基础跑通之后，我建议第二阶段再加入：

```
unitreerobotics/unitree_ros2
```

这是宇树官方 ROS2 repo。

宇树目前明确写的是：

```
Ubuntu 20.04 → Foxy
Ubuntu 22.04 → Humble（recommend）
```

并使用 CycloneDDS；对于 Humble，官方还特别指出不需要像 Foxy 那样自行编译特定 CycloneDDS 版本。[GitHub](https://github.com/unitreerobotics/unitree_ros2)

安装主要依赖：

```
sudo apt install \
    ros-humble-rmw-cyclonedds-cpp \
    ros-humble-rosidl-generator-dds-idl \
    libyaml-cpp-dev
```

以后真机链路会变：

```
你的 ROS2 node
      ↓
CycloneDDS
      ↓
Unitree ROS2 message
      ↓
Unitree Go2
```

---

## 四十六、宇树官方还有一个很值得你后面装的仿真器：MuJoCo

宇树现在还有官方：

```
unitree_mujoco
```

而且已经支持：

```
Go2
B2
G1
H1
...
```

例如官方 Go2 仿真启动方式：

```
./unitree_mujoco -r go2 -s scene_terrain.xml
```

并且官方已经提供：

```
Unitree SDK2
Unitree SDK2 Python
Unitree ROS2
```

与仿真的连接方式。官方建议仿真使用：

```
loopback lo
ROS_DOMAIN_ID=1
```

并给出了 ROS2 Go2 仿真控制示例。[GitHub](https://github.com/unitreerobotics/unitree_mujoco/blob/main/readme.md?utm_source=chatgpt.com)

所以以后你的环境可以形成两个模拟器：

```
                    ROS2 Humble
                         │
             ┌───────────┴───────────┐
             ↓                       ↓
          Gazebo                   MuJoCo
             │                       │
      SLAM / Nav2 / Sensor     Dynamics / Control
             │                       │
             └───────────┬───────────┘
                         ↓
                     Real Go2
```

我建议：

**Gazebo 用来学习 ROS/传感器/SLAM/Nav2。**

**MuJoCo 用来学习 Go2 低层运动控制和 sim-to-real。**

---

## 四十七、为什么我暂时不推荐你上 Isaac Sim

RTX 3060 12GB 并不是完全不能用 Isaac Sim，但对于你当前阶段：

```
刚开始 ROS2
刚开始 Go2
要学习 SLAM
还要理解 Nav2
```

直接上：

```
Isaac Sim
Isaac Lab
CUDA
Omniverse
ROS bridge
```

会把问题复杂度突然提高很多。

而且宇树官方当前的 Isaac Lab 仿真项目主要列出的已验证显卡是 RTX 3080、3090、4090 等，更适合你把 ROS 基础打通后再尝试。[GitHub](https://github.com/unitreerobotics/unitree_sim_isaaclab?utm_source=chatgpt.com)

RTX 3060 12G 当前更适合先跑：

```
Gazebo
+
RViz2
+
ROS2 Humble
+
CHAMP
+
SLAM
+
Nav2
```

---

## 四十八、出现问题时按照这个顺序排查

不要看到 Gazebo 启动失败就立刻重装系统。

按层次检查：

```
① PCIe 能看到 GPU 吗？
        ↓
lspci | grep NVIDIA

② NVIDIA Driver 正常吗？
        ↓
nvidia-smi

③ Docker 正常吗？
        ↓
docker run hello-world

④ Docker 能看到 GPU 吗？
        ↓
docker run --gpus all ... nvidia-smi

⑤ CUDA 正常吗？
        ↓
nvcc --version

⑥ OpenGL GPU 正常吗？
        ↓
glxinfo -B

⑦ ROS2 正常吗？
        ↓
ros2 run demo_nodes_cpp talker

⑧ DDS 正常吗？
        ↓
talker → listener

⑨ Gazebo 正常吗？
        ↓
Gazebo empty world

⑩ Go2 URDF 正常吗？
        ↓
robot_state_publisher / RViz

⑪ controller 正常吗？
        ↓
ros2 control list_controllers

⑫ /cmd_vel 正常吗？
        ↓
ros2 topic echo /cmd_vel

⑬ Go2 能走吗？
```

这样出现问题时可以准确知道是哪一层。

---

## 四十九、最终建议你的实际实施顺序

如果你现在就坐在这台 Ubuntu 22.04 机器前，我建议严格按照这个顺序，不要跳步骤：

```
第 1 步
确认 Ubuntu22.04 / RTX3060 / SecureBoot / 当前用户

        ↓

第 2 步
ubuntu-drivers install

        ↓

第 3 步
重启

        ↓

第 4 步
nvidia-smi

        ↓

第 5 步
安装 Docker CE

        ↓

第 6 步
docker hello-world

        ↓

第 7 步
安装 NVIDIA Container Toolkit

        ↓

第 8 步
docker --gpus all nvidia-smi

        ↓

第 9 步
创建 CUDA12.6 + Ubuntu22.04 Docker image

        ↓

第10步
Docker 内安装 ROS2 Humble

        ↓

第11步
测试 ROS2 talker/listener

        ↓

第12步
测试 glxinfo

        ↓

第13步
安装 Go2 + CHAMP + Gazebo

        ↓

第14步
Gazebo 出现 Go2

        ↓

第15步
teleop 控制 Go2

        ↓

第16步
检查 /cmd_vel /joint_states /tf

        ↓

第17步
加入模拟 LiDAR / IMU

        ↓

第18步
SLAM

        ↓

第19步
Nav2

        ↓

第20步
接真实 Go2 + Unitree ROS2
```

---

## 五十、你最终应该达到的第一个里程碑

暂时不要把目标定成：

> “部署完整 Go2 自主导航系统。”

你的**第一个成功标准**应该非常明确：

```
Ubuntu 22.04
     │
     ├── nvidia-smi ✓
     │
     ├── Docker GPU ✓
     │
     └── Docker
           │
           ├── CUDA ✓
           ├── ROS2 Humble ✓
           ├── Gazebo ✓
           ├── RViz2 ✓
           │
           └── Unitree Go2
                  │
                  ├── 站立 ✓
                  ├── 前进 ✓
                  ├── 后退 ✓
                  ├── 转向 ✓
                  ├── /cmd_vel ✓
                  ├── /joint_states ✓
                  ├── /tf ✓
                  └── LiDAR ✓
```

这套环境一旦跑通，**宿主机基本就不用再动了**。以后你做 Fast-LIO2、SLAM Toolbox、Nav2、YOLO、深度相机、点云处理，都可以继续改 Dockerfile 或新建镜像，而不会去污染实验室 Ubuntu 的 ROS、CUDA、Python 和 CMake 环境。

尤其针对你后面准备做的 **Go2 室内 SLAM + 自动导航项目**，我建议下一步就在这个 Docker 基础上继续搭：

```
Go2 Gazebo
→ ROS2 topic/TF 教学
→ 模拟雷达
→ RViz
→ SLAM Toolbox
→ 保存地图
→ Nav2
→ 设置导航目标点
→ Go2 自动走过去
→ 障碍物避障
```

这是目前最适合从零开始、同时又能自然过渡到真实 Go2 的路线。