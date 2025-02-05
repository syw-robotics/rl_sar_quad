# rl_sar_quad

**ROS版本: [ROS-Noetic](https://github.com/fan-ziqi/rl_sar/tree/main)**

本仓库修改自[rl_sar](https://github.com/fan-ziqi/rl_sar), 提供了四足机器人强化学习算法的sim2sim验证(gazebo)与实物部署框架(以HIMLoco为例)

特性：
- 支持IaacGym, IsaacLab，用`framework`加以区分, 主要区别是四元数wxyz和xyzw顺序的convention差异
- 集成plotjuggler, 用于实时监控仿真中关节数据(gazebo_a1_xxx.launch文件中, only cpp)
- 代码有python和cpp两个版本，python版本可以在`src/rl_sar/scripts`中找到
  - `cpp`: sim2sim & 实机部署
  - `python`: sim2sim

## Prerequisites

```bash
git clone https://github.com/fan-ziqi/rl_sar.git
```

## Dependencies

```bash
sudo apt install ros-noetic-teleop-twist-keyboard ros-noetic-controller-interface ros-noetic-gazebo-ros-control ros-noetic-joint-state-controller ros-noetic-effort-controllers ros-noetic-joint-trajectory-controller
```

- `libtorch`
在任意位置下载并配置环境变量

```bash
cd /path/to/your/torchlib
wget https://download.pytorch.org/libtorch/cpu/libtorch-cxx11-abi-shared-with-deps-2.0.1%2Bcpu.zip
unzip libtorch-cxx11-abi-shared-with-deps-2.0.1+cpu.zip -d ./
echo 'export Torch_DIR=/path/to/your/torchlib' >> ~/.bashrc
```

- `yaml-cpp` & `lcm`
Ubuntu可直接apt install进行安装

```bash
sudo apt install liblcm-dev libyaml-cpp-dev
```

<details>

<summary>也可以使用源码安装</summary>

安装yaml-cpp

```bash
git clone https://github.com/jbeder/yaml-cpp.git
cd yaml-cpp && mkdir build && cd build
cmake -DYAML_BUILD_SHARED_LIBS=on .. && make
sudo make install
sudo ldconfig
```

安装lcm

```bash
git clone https://github.com/lcm-proj/lcm.git
cd lcm && mkdir build && cd build
cmake .. && make
sudo make install
sudo ldconfig
```
</details>

- `tbb`

使用Intel TBB（Threading Building Blocks）库进行线程间数据交换

Ubuntu可直接apt install安装

```bash
sudo apt install libtbb-dev
```

## Compilation


在项目根目录编译

```bash
cd <path_to_rl_sar_quad>
catkin build
```

若 catkin build 报错: `Unable to find either executable 'empy' or Python module 'em'`, 在`catkin build` 之前执行 `catkin config -DPYTHON_EXECUTABLE=/usr/bin/python3`

## Run

下文中使用 **\<ROBOT\>_\<FRAMEWORK\>** 代替表示不同的环境，如`a1_isaacgym` 、 `a1_isaacsim` 、 `go2_isaacgym`

运行前请将训练好的pt模型文件拷贝到`rl_sar/src/rl_sar/models/<ROBOT>_<FRAMEWORK>`中，并配置`config.yaml`中的参数

### 仿真

打开一个终端，启动gazebo仿真环境

```bash
source devel/setup.bash
roslaunch rl_sar gazebo_<ROBOT>_<FRAMEWORK>.launch
```

打开一个新终端，启动控制程序

```bash
source devel/setup.bash
(for cpp version)    rosrun rl_sar rl_sim
(for python version) rosrun rl_sar rl_sim.py
```

控制：

* 按 **\<Enter\>** 切换仿真器运行/停止
* **W** 和 **S** 控制x轴，**A** 和 **D** 控制yaw轴，**J** 和 **L** 控制y轴，按下空格重置控制指令
* 按 **\<Space\>** 将所有控制指令设置为零
* 如果机器人摔倒，按 **R** 重置Gazebo环境

### 真实机器人
> 实机部署不依赖ROS

#### Unitree A1

与Unitree A1连接可以使用无线与有线两种方式

* 无线：连接机器人发出的Unitree开头的WIFI **（注意：无线连接可能会出现丢包断联甚至失控，请注意安全）**
* 有线：用网线连接计算机和机器人的任意网口，配置计算机ip为192.168.123.162，网关255.255.255.0

新建终端，启动控制程序

```bash
source devel/setup.bash
rosrun rl_sar rl_real_a1
```

按下遥控器的**R2**键让机器人切换到默认站起姿态，按下**R1**键切换到RL控制模式，任意状态按下**L2**切换到最初的趴下姿态。左摇杆上下控制x左右控制yaw，右摇杆左右控制y

或者按下键盘上的**0**键让机器人切换到默认站起姿态，按下**P**键切换到RL控制模式，任意状态按下**1**键切换到最初的趴下姿态。WS控制x，AD控制yaw，JL控制y

#### Unitree Go2

1. 用网线的一端连接Go2机器人，另一端连接用户电脑，并开启电脑的 USB Ethernet 后进行配置。机器狗机载电脑的 IP 地地址为 192.168.123.161，故需将电脑 USB Ethernet 地址设置为与机器狗同一网段，如在 Address 中输入 192.168.123.222 (“222”可以改成其他)
2. 通过`ifconfig`命令查看123网段的网卡名字，如`enxf8e43b808e06`，下文用 \<YOUR_NETWORK_INTERFACE\> 代替
3. 新建终端，启动控制程序
    ```bash
    source devel/setup.bash
    rosrun rl_sar rl_real_go2 <YOUR_NETWORK_INTERFACE>
    ```
4. Go2支持手柄与键盘控制，方法与上面a1相同

### 训练执行器网络

下面拿A1举例

1. 取消注释`rl_real_a1.cpp`中最上面的`#define CSV_LOGGER`，你也可以在仿真程序中修改对应部分采集仿真数据用来测试训练过程
2. 运行控制程序，程序会在执行后记录所有数据
3. 停止控制程序，开始训练执行器网络。注意，下面的路径前均省略了`rl_sar/src/rl_sar/models/`
    ```bash
    rosrun rl_sar actuator_net.py --mode train --data a1/motor.csv --output a1/motor.pt
    ```
4. 验证已经训练好的训练执行器网络
    ```bash
    rosrun rl_sar actuator_net.py --mode play --data a1/motor.csv --output a1/motor.pt
    ```

## 添加你的机器人

下文中使用 **\<ROBOT\>_\<FRAMEWORK\>** 代替表示你的机器人环境

1. 在`rl_sar/src/robots`路径下创建名为`<ROBOT>_description`的模型包，将模型的urdf放到`rl_sar/src/robots/<ROBOT>_description/urdf`路径下并命名为`<ROBOT>.urdf`，并在`rl_sar/src/robots/<ROBOT>_description/config`路径下创建命名空间为`<ROBOT>_gazebo`的关节配置文件
2. 将训练好的RL模型文件放到`rl_sar/src/rl_sar/models/<ROBOT>_<FRAMEWORK>`路径下，并在此路径中新建config.yaml文件，参考`rl_sar/src/rl_sar/models/a1_isaacgym/config.yaml`文件修改其中参数
3. 按需修改代码中的`forward()`函数，以适配不同的模型
4. 若需要运行仿真，则参考`rl_sar/src/rl_sar/launch`路径下的launch文件自行修改
5. 若需要运行实物，则参考`rl_sar/src/rl_sar/src/rl_real_a1.cpp`文件自行修改

