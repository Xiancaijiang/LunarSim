# LunarSim -  ROS 2-Connected Lunar Rover Simulation

![header](./README_FILES/header.png)

## About


## Installation

```bash
# clone the repo
git clone https://github.com/PUTvision/LunarSim

# change the working directory to LunarSim
cd ./LunarSim

# build docker image
docker build -t lunarsim:latest .
```

## Run simulator

```bash
# run docker container
bash ./run.sh

# run the simulator
./LunarSim.x86_64
```

> **Note:** DO NOT SOURCE ROS IN TERMINAL WHERE YOU RUN LUNARSIM

## Attach to container and activate ROS (in new terminal)

```bash
# connect to container
docker exec -it lunarsim bash

# source ROS environment
source /opt/ros/humble/setup.bash

# then you can run sanity checks and display topics
ros2 wtf
ros2 topic list
```

## Data generation mode

You can find sample image sequences generated with the simulator [here](https://chmura.put.poznan.pl/s/9CQiMdU6O9Olyli).

We also provide our real images recorded during analogue lunar missions ([link](https://chmura.put.poznan.pl/s/HAybyHz1rgpyy9r)). 

## ROS 2 mode

### Topics

#### Subscribed topics

* `/cmd_vel` (geometry_msgs/Twist)

    The velocity command for the robot.

#### Published topics

* `/lunarsim/gt/pose` (geometry_msgs/PoseStamped)

    The ground truth pose of the robot.

  > For example, you can use terminal command `ros2 topic pub --once /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 1.0, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.0}}"`

* `/lunarsim/imu` (sensor_msgs/Imu)

    The IMU data.

* `/lunarsim/camera_left/raw` (sensor_msgs/Image)

    or

   `/lunarsim/camera_left/compressed` (sensor_msgs/CompressedImage)

    The left camera image.

* `/lunarsim/camera_right/raw` (sensor_msgs/Image)

    or

   `/lunarsim/camera_right/compressed` (sensor_msgs/CompressedImage)

    The right camera image.
    
* `/lunarsim/camera_depth/depth` (sensor_msgs/Image)

    The depth image.


## Examples

* Object detection – embedded device

    ```
    ./examples/jetson_nx_od_yolo
    ```

    ![object_detection](./examples/jetson_nx_od_yolo/lunarsim_od_jestson_result.png)

* Visual odometry – x64-based mini-PC

    ```
    ./examples/amd64_stella_vslam
    ```

    ![visual_odometry](./examples/amd64_stella_vslam/lunarsim_traj_points.png)

* Simple moving-average filter – microcontroller

    ```
    ./examples/raspberry_pico_smaf
    ```

    ![moving_average](./examples/raspberry_pico_smaf/lunarsim_smf_rpi_z.png)

* image segmentation – FPGA

    ```
    ./examples/versal_vck190_segment
    ```

    ![image_segmentation](./examples/versal_vck190_segment/lunarsim_seg_fpga_result.png)

## Citation

If you use this code for your research, please cite our paper:

```

@Article{app132212401,
AUTHOR = {Pieczyński, Dominik and Ptak, Bartosz and Kraft, Marek and Drapikowski, Paweł},
TITLE = {LunarSim: Lunar Rover Simulator Focused on High Visual Fidelity and ROS 2 Integration for Advanced Computer Vision Algorithm Development},
JOURNAL = {Applied Sciences},
VOLUME = {13},
YEAR = {2023},
NUMBER = {22},
ARTICLE-NUMBER = {12401},
URL = {https://www.mdpi.com/2076-3417/13/22/12401},
ISSN = {2076-3417},
DOI = {10.3390/app132212401}
}
```

# LunarSim（PUTvision）部署与操控复现手册

> 适用环境：Ubuntu 22.04 + NVIDIA 显卡 + Docker + ROS 2 Humble（容器内）
> 项目路径：`~/Desktop/lunar/LunarSim`
> 版本：v1.0

 **Unity 独立可执行程序** 打进镜像，运行后由程序内部的 ROS 2 Bridge 对外发布传感器话题。

- **启动仿真** = 在容器内直接执行 `./LunarSim.x86_64`
- **绝对不能用** `ros2 launch lunarsim ...`（没有这个包，必然报 `package 'lunarsim' not found`）
- **跑仿真的那个终端不要 `source /opt/ros/humble/setup.bash`**（官方提示 `DO NOT SOURCE ROS IN TERMINAL WHERE YOU RUN LUNARSIM`，加载 ROS 环境会与内置 Bridge 冲突）
- 只有**另外新开的终端**（`docker exec` 进去的那个）才 source ROS，用来遥控 / 可视化 / 录包

---

## 1. 前置环境检查（宿主机，只做一次）

### 1.1 确认显卡驱动

```bash
nvidia-smi
```

要求：能正常打印显卡型号、驱动版本、显存占用表。

### 1.2 确认 Docker 与 GPU 容器运行时

```bash
docker --version
docker info | grep -i runtime
```

安装 Docker 与 NVIDIA Container Toolkit（Ubuntu 22.04）：

```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
```

NVIDIA Container Toolkit

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

**验证 GPU 透传是否真的生效**：

```bash
docker run --rm --gpus all nvidia/cuda:12.1.0-base-ubuntu22.04 nvidia-smi
```

- 能打印显卡信息 → GPU 容器运行时 OK

### 1.3 确认显卡渲染节点存在

```bash
ls -l /dev/dri/
```

应看到 `card0`、`renderD128` 之类。**这个节点必须挂进容器**，否则 Unity 一初始化渲染管线就 `Segmentation fault`。

---

## 2. 首次部署

### 2.1 获取源码并赋权

```bash
cd ~/Desktop/lunar
git clone https://github.com/PUTvision/LunarSim.git
cd LunarSim
chmod +x create.sh run.sh run_once.sh
```

### 2.2 报错解决

- 原始 Dockerfile 里用 `curl` 拉 `raw.githubusercontent.com/ros/rosdistro/master/ros.key`，国内网络必超时，构建直接失败，报：

```
curl: (28) Failed to connect to raw.githubusercontent.com port 443 ... timeout
ERROR: failed to solve ...
```

打开 Dockerfile：

```bash
gedit Dockerfile
```

找到下面两行（大约在第 42、43 行）：

```dockerfile
RUN curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg
RUN echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
```

**整体替换为**（清华 TUNA 镜像），把 ROS 源换成国内镜像（绕过 `curl 28` 超时）即可：

```dockerfile
RUN curl -sSL https://mirrors.tuna.tsinghua.edu.cn/rosdistro/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg
RUN echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] https://mirrors.tuna.tsinghua.edu.cn/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | tee /etc/apt/sources.list.d/ros2.list > /dev/null
```

可选：顺手把 Ubuntu 基础源也换成清华源（放在 `FROM ubuntu:22.04` 之后）：

```dockerfile
RUN sed -i 's/archive.ubuntu.com/mirrors.tuna.tsinghua.edu.cn/g' /etc/apt/sources.list
RUN sed -i 's/security.ubuntu.com/mirrors.tuna.tsinghua.edu.cn/g' /etc/apt/sources.list
```

> 构建时会出现的几条 `UndefinedVar` / `LegacyKeyValueFormat` 警告是 Dockerfile 语法提示，**不影响构建结果，忽略即可**。

- 原始 `run.sh` 缺 `--device=/dev/dri`，且 root 用户 + 远程桌面下非常容易段错误。**推荐直接用下面的完整版本覆盖 `run.sh`**：

```bash
#!/bin/bash
xhost +local:root

docker run -it \
  --gpus all \
  --privileged \
  --net=host \
  --name=lunarsim \
  -e DISPLAY=$DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  -v $HOME/.Xauthority:/root/.Xauthority:rw \
  --device=/dev/dri:/dev/dri \
  lunarsim:humble bash
```

逐条说明，修 `run.sh`：补齐 GPU / 渲染节点 / X11 转发：

| 参数                               | 作用                                 | 缺失后果                           |
| ---------------------------------- | ------------------------------------ | ---------------------------------- |
| `--gpus all`                       | 容器内可用 NVIDIA 显卡               | Unity 拿不到 GPU，直接段错误       |
| `--device=/dev/dri:/dev/dri`       | 挂载显卡渲染节点                     | `Segmentation fault (core dumped)` |
| `-e DISPLAY` + `-v /tmp/.X11-unix` | X11 图形窗口转发                     | 图形窗口弹不出来                   |
| `-v $HOME/.Xauthority`             | X11 认证                             | `cannot open display`              |
| `--privileged`                     | 放开设备权限（调试期建议保留）       | 偶发渲染/设备访问失败              |
| `--net=host`                       | 与宿主机共享网络，ROS 2 DDS 才好发现 | 容器内外话题互相看不到             |
| `--name=lunarsim`                  | 固定容器名，方便 `docker exec`       | 每次要 `docker ps` 现查            |
| `xhost +local:root`                | 宿主机放行 X11 访问                  | 窗口弹不出来                       |

- 项目原脚本自相矛盾 —— `create.sh` 只打 `lunarsim:humble` 标签，而 `run.sh` 里要的是 `lunarsim:latest`，于是启动时报：

```
Unable to find image 'lunarsim:latest' locally
docker: Error response from daemon: pull access denied for lunarsim
```

两种修法，**任选一种**（推荐方案 A，长期维护干净）：

**方案 A（推荐）：统一用 `humble` 标签**
把上面的 `run.sh` 里的 `lunarsim:humble` 保持一致，`create.sh` 不动。

**方案 B：构建时双标签**
改 `create.sh` 里的 build 行：

```bash
# 原来
docker build -t lunarsim:humble .
# 改为
docker build -t lunarsim:humble -t lunarsim:latest .
```

> 注意：`latest` 不会自动跟随最新构建，只有你显式 `-t xxx:latest` 才会更新，别把它当成"自动最新版"。



### 2. 3 构建镜像

```bash
cd ~/Desktop/lunar/LunarSim

# 若之前构建失败过，先清掉失败缓存，避免复用坏层
docker builder prune -f

./create.sh
```

成功标志（终端末尾出现）：

```
naming to docker.io/library/lunarsim:humble
naming to docker.io/library/lunarsim:         ← 或者 lunarsim:latest
```

验证镜像存在：

```bash
docker images | grep lunarsim
```

---

## 3. 每次启动标准流程

- 宿主机：放行 X11 并进入容器

```bash
cd ~/Desktop/lunar/LunarSim
xhost +local:root
./run.sh
```

执行后你会进入容器内部，提示符类似：

```
root@xxx:~/lunarsim#
```

> 如果提示 `Conflict. The container name "/lunarsim" is already in use`，说明上次容器没删干净：

```bash
docker rm -f lunarsim
./run.sh
```

- 容器内（主终端，**禁止 source ROS**）：启动仿真

```bash
./LunarSim.x86_64
```

若提示 `Permission denied`：先 `chmod +x LunarSim.x86_64`

若文件名不同，先 `ls` 确认可执行文件名

**启动成功的表现**：Unity 窗口弹出并稳定显示月面场景与月球车，不再闪退；终端日志中 ROS Bridge 初始化完成。

> 如果仍然 `Segmentation fault`，按下表逐条排查（按成功率从高到低）：

| 排查动作           | 命令                                                         | 说明                                                         |
| ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 脱离远程桌面       | 在本机显示器登录 Ubuntu 桌面操作                             | ToDesk / 向日葵的 X11 转发会让 Unity OpenGL 必崩，这是第一杀手 |
| 确认渲染节点已挂载 | 容器内 `ls /dev/dri`                                         | 空目录 = 没挂载，回到 2.3 修 `run.sh`                        |
| 绕过 root 检测     | `export UNITY_DISABLE_ROOT_CHECK=1`                          | Unity Linux 版对 root 用户兼容性差                           |
| 强制指定渲染后端   | `./LunarSim.x86_64 -force-glcore`                            | 或 `-force-vulkan`，逐个试                                   |
| 无头模式（兜底）   | `./LunarSim.x86_64 -nographics`                              | 无画面但 ROS 话题照常发布，可正常跑算法与采数据              |
| 验证 GPU 透传      | 宿主机 `docker run --rm --gpus all nvidia/cuda:12.1.0-base-ubuntu22.04 nvidia-smi` | 失败则 nvidia-container-toolkit 未生效                       |
| 排查内存           | `free -h`                                                    | 月面大场景加载建议 ≥ 16 GB 可用内存                          |

- 另开宿主机终端：接入同一容器做操控与可视化

**新开一个宿主机终端**（不要动跑仿真的那个）：

```bash
docker exec -it lunarsim bash
```

进入后，**这个终端才可以 source ROS**：

```bash
source /opt/ros/humble/setup.bash
```

确认话题是否正常

```bash
ros2 topic list
```

查话题的确切名字与类型：

```bash
ros2 topic list
ros2 topic info /odom
ros2 topic echo /odom --once
```

---

## 4. 操控机器人

### 4.1 键盘遥控

```bash
apt update
apt install -y ros-humble-teleop-twist-keyboard
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

按键映射（光标必须停在该终端窗口内才生效）：

| 键                    | 动作                        |
| --------------------- | --------------------------- |
| `i`                   | 前进                        |
| `,`                   | 后退                        |
| `j`                   | 左转                        |
| `l`                   | 右转                        |
| `k`                   | 立即停止                    |
| `u` / `o` / `m` / `.` | 组合转向（前进/后退同时转） |

### 4.2 命令行发速度指令

单次发布（注意：默认只发一帧，车会很快停下）：

```bash
ros2 topic pub --once /cmd_vel geometry_msgs/msg/Twist \
  "{linear: {x: 0.2, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.0}}"
```

持续前进（10 Hz 循环，Ctrl+C 可终止）：

```bash
while true; do
  ros2 topic pub --once /cmd_vel geometry_msgs/msg/Twist \
    "{linear: {x: 0.2, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.0}}"
  sleep 0.1
done
```

刹车（任何时候都能用）：

```bash
ros2 topic pub --once /cmd_vel geometry_msgs/msg/Twist \
  "{linear: {x: 0.0, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.0}}"
```

参数含义：

| 字段        | 单位  | 说明                         |
| ----------- | ----- | ---------------------------- |
| `linear.x`  | m/s   | 正为前进，负为后退           |
| `angular.z` | rad/s | 正为左转（逆时针），负为右转 |

### 4.3 传感器数据查看

```bash
# 里程计
ros2 topic echo /odom

# IMU
ros2 topic echo /imu

# 图像（可视化）
apt install -y ros-humble-rqt-image-view
rqt_image_view
# 下拉框选 /camera/image_raw 或 /camera/depth/image_raw
```

### 4.4 录制数据（喂给 VLM / SLAM 训练）

```bash
ros2 bag record -o lunarsim_run1 /camera/image_raw /camera/depth/image_raw /odom /imu /tf /tf_static
```

---

## 5. 关闭

按以下顺序收尾：

1. **跑仿真的终端**：`Ctrl + C` 关闭 Unity 程序
2. **操控 / 可视化终端**（`docker exec` 进去的那些）：逐一 `exit` 退出
3. **主容器终端**（`./run.sh` 进去的那个）：输入 `exit`
	- `run.sh` 是前台 `-it` 交互容器，`exit` 之后容器自动停止
4. 回到**宿主机终端**确认：

```bash
docker ps          # 应看不到 lunarsim
```

若还有残留：

```bash
docker stop lunarsim
docker rm lunarsim
```

> 注意：`docker stop` / `docker rm` **只能在宿主机执行**，在容器内部敲必然报 `docker: command not found`。

---

关闭后再次启动

```bash
# ① 宿主机：进入项目目录、放行 X11、启动容器
cd ~/Desktop/lunar/LunarSim
xhost +local:root
./run.sh

# ② 容器内主终端（不要 source ROS）：启动仿真
./LunarSim.x86_64

# ③ 宿主机新开一个终端：接入容器做操控
docker exec -it lunarsim bash
source /opt/ros/humble/setup.bash
ros2 topic list
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

若容器名冲突

```bash
docker rm -f lunarsim
./run.sh
```

若镜像丢了（重启后 `run.sh` 报 pull access denied）

```bash
docker images | grep lunarsim     # 确认镜像是否还在
cd ~/Desktop/lunar/LunarSim
./create.sh                        # 重新构建（有缓存会快很多）
```

镜像保存在 Docker 本地仓库，**正常重启机器不会丢**，只有 `docker rmi` 或 `system prune` 后才会没。

---

## 6. 命令速查卡

```bash
# ───────── 宿主机 ─────────
cd ~/Desktop/lunar/LunarSim
xhost +local:root
./run.sh                                  # 进入容器（主终端，不 source ROS）
docker exec -it lunarsim bash             # 另开终端接入容器
docker ps                                 # 查看容器状态
docker rm -f lunarsim                     # 清理残留容器
docker images | grep lunarsim             # 查看镜像
docker builder prune -f                   # 清理构建缓存
./create.sh                               # 重新构建镜像
docker run --rm --gpus all nvidia/cuda:12.1.0-base-ubuntu22.04 nvidia-smi   # 验证 GPU 透传

# ───────── 容器内 · 主终端（禁止 source ROS）─────────
./LunarSim.x86_64                          # 启动仿真（GUI）
./LunarSim.x86_64 -force-glcore            # 强制 OpenGL 后端
./LunarSim.x86_64 -nographics              # 无头模式（无画面，有话题）
exit                                       # 退出容器

# ───────── 容器内 · 操控终端（需要 source ROS）─────────
source /opt/ros/humble/setup.bash
ros2 topic list
ros2 topic echo /odom
ros2 run teleop_twist_keyboard teleop_twist_keyboard
rqt_image_view
rviz2
ros2 bag record -o run1 /camera/image_raw /odom /imu /tf
```

---

## 7. 与上层算法对接

数据链路：

```
LunarSim (Unity, 容器内)
   │  ROS 2 Bridge
   ├─ /camera/image_raw ──┐
   ├─ /camera/depth/image_raw
   ├─ /odom, /imu, /tf
   └─ /cmd_vel ◄──────────┘
        ▲
        │  速度指令
  上位算法（VLM 地形分级 → 规划 → cmd_vel）
```

对接要点：

1. **话题名以 `ros2 topic list` 输出为准**，不同版本镜像可能有 `/cmd_vel` 或 `/robot/cmd_vel` 的区别；
2. 若算法跑在**宿主机**而非容器内：`run.sh` 已加 `--net=host`，宿主机 source 本机 ROS 2 Humble 后即可直接订阅容器内话题（要求两端 `ROS_DOMAIN_ID` 一致）；
3. 若算法用 Python 推理（VLM 等）：在**操控终端**那一侧运行最稳妥，避免污染仿真终端的 ROS 环境；
4. 控制频率建议 ≥ 10 Hz 持续发布，单帧 `--once` 会被仿真器的看门狗逻辑迅速归零。
