# mockamap on Ubuntu 22.04 (Docker + ROS Noetic) 复现笔记（可直接复制粘贴）

> 目标：在 **Ubuntu 22.04** 宿主机上，用 **Docker** 拉起 **ROS Noetic** 环境，编译并运行 `mockamap`，在 **RViz** 显示点云地图，并把颜色调得更明显。  
> 说明：以下步骤按“第一次完整复现”写，后面也提供“下次快速复现”最短命令。

---

## 0. 宿主机准备（一次性 / 每次登录后可能需要）

### 0.1 允许容器使用 X11 显示（RViz GUI）
```bash
xhost +local:root
```

> 如果你使用的是 Wayland，会遇到 GUI 转发不稳定。建议在登录界面选择 **“Ubuntu on Xorg”** 再执行上面命令。

---

## 1. 拉取 ROS Noetic 镜像（一次性）
```bash
docker pull osrf/ros:noetic-desktop-full
```

---

## 2. 创建工作区并拉取 mockamap 源码（宿主机）
> 该目录会挂载到容器 `/root/catkin_ws`，用于持久化源码与编译结果。

```bash
mkdir -p ~/ws_mockamap/src
cd ~/ws_mockamap/src
git clone https://github.com/HKUST-Aerial-Robotics/mockamap.git
```

---

## 3. 启动容器（推荐：带 DNS、带 GUI、可 `docker exec` 进入）
> `--net=host`：省去 ROS1 网络坑  
> `--dns`：避免容器内 DNS 解析失败（你之前遇到过 `Temporary failure resolving ...`）

```bash
docker run -it --rm \
  --net=host \
  --dns=1.1.1.1 --dns=8.8.8.8 \
  -e DISPLAY=$DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v ~/ws_mockamap:/root/catkin_ws \
  --name ros_noetic_mockamap \
  osrf/ros:noetic-desktop-full \
  bash
```

---

## 4. 容器内：安装依赖并编译（第一次需要）

### 4.1 安装常用构建工具 + PCL/ROS 依赖
```bash
apt update
apt install -y \
  build-essential cmake git \
  python3-rosdep python3-catkin-tools \
  ros-noetic-pcl-ros ros-noetic-pcl-conversions
```

### 4.2 初始化 rosdep（容器第一次）
```bash
rosdep init || true
rosdep update
```

### 4.3 安装 mockamap 的依赖（如果 rosdep 能解析到）
```bash
cd /root/catkin_ws
rosdep install --from-paths src --ignore-src -r -y
```

### 4.4 编译（关键：强制 C++14，避免 PCL 头文件报错）
> 你之前遇到的 PCL lambda/模板报错，几乎都是 **编译标准太低（例如 c++11）** 导致。
```bash
cd /root/catkin_ws
rm -rf build devel
catkin_make -DCMAKE_CXX_STANDARD=14 -DCMAKE_CXX_STANDARD_REQUIRED=ON
```

### 4.5 每个新终端都要 source（建议写入 .bashrc）
```bash
source /opt/ros/noetic/setup.bash
source /root/catkin_ws/devel/setup.bash
```

可选：自动 source（以后每次进容器就自动生效）：
```bash
echo "source /opt/ros/noetic/setup.bash" >> ~/.bashrc
echo "source /root/catkin_ws/devel/setup.bash" >> ~/.bashrc
```

---

## 5. 运行 mockamap（容器内）

### 5.1 运行任意 demo（示例：Perlin 3D）
```bash
source /opt/ros/noetic/setup.bash
source /root/catkin_ws/devel/setup.bash

roslaunch mockamap perlin3d.launch
```

> 你也可以换成：  
> - `roslaunch mockamap post2d.launch`  
> - `roslaunch mockamap maze2d.launch`  
> - `roslaunch mockamap maze3d.launch`

---

## 6. 新开终端进入同一个容器（用于 rostopic / 调试）

在 **宿主机** 新开一个终端：
```bash
docker exec -it ros_noetic_mockamap bash
```

进入后（容器内）：
```bash
source /opt/ros/noetic/setup.bash
source /root/catkin_ws/devel/setup.bash
```

### 6.1 查看话题
```bash
rostopic list
rostopic hz /mock_map
rostopic echo -n 1 /mock_map/header
```

---

## 7. RViz 显示与“颜色更明显”的设置（UI 操作）

### 7.1 添加点云显示
1. 打开 RViz（如果 launch 没带 RViz，可手动运行）：
   ```bash
   rviz
   ```
2. 左下角 **Add** → 选择 **PointCloud2**
3. 在 PointCloud2 的 **Topic** 里选择：
   - `/mock_map`（如果你的话题名不同，以 `rostopic list` 为准）

### 7.2 Fixed Frame
- 左侧 **Global Options** → **Fixed Frame**：设置为 `map`

> 如果出现 “No TF data” 警告，但点云能显示，一般不影响看地图；后续叠加机器人/轨迹才需要 TF。

### 7.3 让颜色更明显（推荐：按高度 Z 上色）
在左侧 **PointCloud2** 显示项中：

- **Color Transformer**：从 `Intensity` 改为 **`AxisColor`**
- **Axis**：选择 **`Z`**
- **Use rainbow**：勾选 ✅
- **Autocompute Intensity Bounds**：取消勾选（如果有）
- 手动调整范围（示例，按实际地图高度可改）：
  - **Min**：`-2`
  - **Max**：`2`

### 7.4 让点更清楚（大小/对比度）
- **Style**：`Flat Squares`（或 `Points`）
- **Size (m)**：建议 `0.03 ~ 0.08`（你之前 `0.01` 会比较细）
- **Alpha**：`1`（不透明）
- **Background Color**：可改为更黑或更白以增加对比

---

## 8. 下次快速复现（最短命令版）

### 8.1 宿主机：一键启动容器 + 进入
```bash
xhost +local:root

docker run -it --rm \
  --net=host \
  --dns=1.1.1.1 --dns=8.8.8.8 \
  -e DISPLAY=$DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v ~/ws_mockamap:/root/catkin_ws \
  --name ros_noetic_mockamap \
  osrf/ros:noetic-desktop-full \
  bash
```

### 8.2 容器内：如果你已经编译过（直接运行）
```bash
source /opt/ros/noetic/setup.bash
source /root/catkin_ws/devel/setup.bash

roslaunch mockamap perlin3d.launch
```

### 8.3 宿主机另开终端：进入容器做调试
```bash
docker exec -it ros_noetic_mockamap bash
source /opt/ros/noetic/setup.bash
source /root/catkin_ws/devel/setup.bash

rostopic list
rostopic hz /mock_map
```

---

## 9. 常见问题速查

### 9.1 `apt update` 报：`Temporary failure resolving ...`
- 本质是 DNS 解析失败  
- 解决：启动容器时加 `--dns=1.1.1.1 --dns=8.8.8.8`（本笔记已默认加上）

### 9.2 编译时报 PCL 相关模板/lambda 大量报错
- 解决：用 C++14 编译
```bash
cd /root/catkin_ws
rm -rf build devel
catkin_make -DCMAKE_CXX_STANDARD=14 -DCMAKE_CXX_STANDARD_REQUIRED=ON
```

### 9.3 新开终端进容器后 `rostopic/rviz` 找不到
- 解决：source 环境
```bash
source /opt/ros/noetic/setup.bash
source /root/catkin_ws/devel/setup.bash
```

---