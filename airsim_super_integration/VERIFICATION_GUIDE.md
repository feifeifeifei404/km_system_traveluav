# 🚀 验证工具使用指南

## 问题已解决！

你遇到的 **QoS 不兼容问题** 已修复。

### 修改内容

**文件**: `/mnt/data/airsim_super_integration/scripts/airsim_super_bridge_ros2.py`

**改动**: 将 QoS 从 `BEST_EFFORT` 改为 `RELIABLE`

```python
# 修改前 ❌
qos_profile = QoSProfile(
    reliability=ReliabilityPolicy.BEST_EFFORT,  # 不兼容
    history=HistoryPolicy.KEEP_LAST,
    depth=1,
    durability=DurabilityPolicy.VOLATILE
)

# 修改后 ✅
qos_profile = QoSProfile(
    reliability=ReliabilityPolicy.RELIABLE,  # 兼容 SUPER
    history=HistoryPolicy.KEEP_LAST,
    depth=10,
    durability=DurabilityPolicy.VOLATILE
)
```

---

## 📋 快速重启步骤

### 方法 1: 使用重启脚本（推荐）

```bash
cd /mnt/data/airsim_super_integration/scripts
bash restart_bridge.sh
```

### 方法 2: 手动重启

```bash
# 终端 1: 杀死旧进程
pkill -f airsim_super_bridge_ros2
sleep 2

# 启动新的桥接节点
cd /mnt/data/airsim_super_integration/scripts
source /opt/ros/humble/setup.bash
/usr/bin/python3 airsim_super_bridge_ros2.py
```

```bash
# 终端 2: 启动 SUPER
source /opt/ros/humble/setup.bash
source ~/super_ws/install/local_setup.bash
cd /mnt/data/airsim_super_integration/launch
ros2 launch airsim_super.launch.py
```

---

## ✅ 验证方法

### 方法 1: 查看桥接节点输出

启动桥接节点后，应该看到：

```
[INFO] [1773397387.695166544] [airsim_super_bridge_ros2]: ======================================================================
[INFO] [1773397387.695166544] [airsim_super_bridge_ros2]: AirSim-SUPER 集成桥接节点启动 (ROS2 Humble)
[INFO] [1773397387.695166544] [airsim_super_bridge_ros2]: ======================================================================
[INFO] [1773397387.695166544] [airsim_super_bridge_ros2]: ✓ AirSim连接成功: ip=127.0.0.1, port=25001
[INFO] [1773397387.695166544] [airsim_super_bridge_ros2]: 桥接节点初始化完成，开始数据传输...
```

**关键**: 不应该再看到 `QoS` 警告！

### 方法 2: 查看 SUPER 输出

启动 SUPER 后，应该看到：

```
[fsm_node-1] Load config file: /home/wuyou/super_ws/src/SUPER/super_planner/config/click_smooth_ros2.yaml
[fsm_node-1]  Load param rog_map/ros_callback/cloud_topic success: /cloud_registered
[fsm_node-1]  Load param rog_map/ros_callback/odom_topic success: /lidar_slam/odom
```

**关键**: 不应该再看到 `No point cloud input` 警告！

### 方法 3: 运行验证工具

```bash
# 在新终端中
source /opt/ros/humble/setup.bash
cd /mnt/data/airsim_super_integration/scripts
python3 verify_coordinate_system.py
```

应该看到：

```
📊 点云信息:
  Frame ID: world
  点数: 1000+
  X 范围: [-5.00, 5.00]
  Y 范围: [-5.00, 5.00]
  Z 范围: [-2.00, 2.00]

🚁 无人机状态:
  Frame ID: world
  位置: (161.51, -2.93, 1.33)
  速度: (0.00, 0.00, 0.00)

🔍 对齐验证 ✅ 对齐正确:
  平均距离: 2.50 m
  最小距离: 0.10 m
  最大距离: 5.00 m
```

---

## 🎯 完整工作流程

```bash
# 终端 1: 启动 AirSim
cd /mnt/data/TravelUAV/airsim_plugin
python AirVLNSimulatorServerTool.py --port 25000 --root_path /mnt/data/TravelUAV/envs

# 终端 2: 启动桥接节点
cd /mnt/data/airsim_super_integration/scripts
bash restart_bridge.sh

# 终端 3: 启动 SUPER
source /opt/ros/humble/setup.bash
source ~/super_ws/install/local_setup.bash
cd /mnt/data/airsim_super_integration/launch
ros2 launch airsim_super.launch.py

# 终端 4: 验证（可选）
source /opt/ros/humble/setup.bash
cd /mnt/data/airsim_super_integration/scripts
python3 verify_coordinate_system.py

# 终端 5: 发送测试目标点
source /opt/ros/humble/setup.bash
ros2 topic pub /goal_pose geometry_msgs/PoseStamped \
  "{header: {frame_id: 'world'}, pose: {position: {x: 5.0, y: 5.0, z: 0.0}}}"
```

---

## 📊 预期结果

✅ **修复前**:
- ❌ QoS 不兼容警告
- ❌ 点云无法发送
- ❌ SUPER 无法接收点云

✅ **修复后**:
- ✅ 没有 QoS 警告
- ✅ 点云正常发送
- ✅ SUPER 正常接收点云
- ✅ 地图正常更新
- ✅ 规划正常工作

---

## 🔧 如果还有问题

1. **检查桥接节点是否有错误输出**
   - 查看启动桥接节点的终端

2. **检查 AirSim 是否正在运行**
   - 确保 AirSim 服务器在运行

3. **检查端口是否正确**
   - 默认端口: 25000 (AirSim), 25001 (桥接节点连接)

4. **重启所有组件**
   - 关闭 SUPER
   - 关闭桥接节点
   - 关闭 AirSim
   - 重新启动（按顺序）

---

**现在可以安心使用了！** ✅
