# SUPER单独测试指南

本指南用于单独测试SUPER系统（不包含TravelUAV的其他组件）

## 📋 测试流程概览

1. **终端1**: 启动AirSim环境
2. **终端3**: 启动SUPER规划器
3. **终端4**: 启动桥接节点
4. **终端5**: 启动RViz可视化（可选）
5. **终端6**: 发送测试目标点

---

## 🚀 详细步骤

### 终端1: 启动AirSim环境

```bash
cd /mnt/data/TravelUAV/envs/closeloop_envs
./ModularPark.sh -ResX=1280 -ResY=720 -windowed
```

**说明**: 
- 启动ModularPark仿真环境
- 窗口化模式，分辨率1280x720
- 等待环境完全加载（看到无人机出现）

---

### 终端2: ~~暂时不用~~

~~原本用于操作API的终端，当前测试不需要~~

---

### 终端3: 启动SUPER规划器 ⭐

```bash
conda deactivate
cd /mnt/data/airsim_super_integration/scripts
./start_super_airsim.sh
```

**说明**: 
- 启动SUPER规划器核心节点
- 订阅话题：`/cloud_registered`（点云）、`/lidar_slam/odom`（里程计）
- 发布话题：`/planning/pos_cmd`（控制指令）、`/fsm/path`（路径）
- 等待看到"SUPER规划器已启动"相关信息

---

### 终端4: 启动桥接节点

```bash
conda deactivate
cd /mnt/data/airsim_super_integration/scripts
./run_bridge.sh
```

**说明**: 
- 启动AirSim与ROS2的桥接节点
- 从AirSim读取传感器数据并发布到ROS2话题
- 从ROS2订阅控制指令并发送到AirSim
- 等待看到"桥接节点已启动"相关信息

---

### 终端5: 启动RViz可视化（可选）

```bash
conda deactivate
cd /mnt/data/airsim_super_integration/scripts
./start_rviz.sh
```

**说明**: 
- 启动RViz2进行可视化
- 显示内容：实时点云、占据地图、规划路径
- 这一步是可选的，主要用于可视化调试

---

### 终端6: 发送测试目标点

#### 查看当前位置

```bash
conda deactivate
cd /mnt/data/airsim_super_integration/scripts
ros2 topic echo /lidar_slam/odom --once | grep -A 3 "position:"
```

#### 发送目标点

```bash
# 语法: ./send_goal.sh <x> <y> <z>
./send_goal.sh 3 7 -3
```

**说明**: 
- 发送目标点 (3, 7, -3) 到SUPER规划器
- 观察AirSim中的无人机是否开始规划和飞行
- 可以多次发送不同的目标点进行测试

---

## 🔍 测试验证

### 1. 检查话题是否正常发布

```bash
# 检查点云话题
ros2 topic hz /cloud_registered

# 检查里程计话题
ros2 topic hz /lidar_slam/odom

# 检查控制指令话题
ros2 topic hz /planning/pos_cmd

# 检查规划路径话题
ros2 topic echo /fsm/path --once
```

### 2. 观察AirSim仿真

- 无人机是否起飞
- 是否按照规划路径飞行
- 是否避障（如果有障碍物）
- 是否到达目标点

### 3. 观察RViz（如果启动）

- 点云是否正常显示
- 占据地图是否更新
- 规划路径是否合理

---

## ⚠️ 常见问题

### 问题1: SUPER启动失败

**解决方案**:
```bash
# 检查SUPER工作空间是否编译
ls -la /home/wuyou/super_ws/install/

# 重新编译SUPER（如果需要）
cd /home/wuyou/super_ws
colcon build
```

### 问题2: 桥接节点无法连接AirSim

**解决方案**:
```bash
# 确认AirSim已经启动
# 检查settings.json配置
cat ~/Documents/AirSim/settings.json

# 重启AirSim环境
```

### 问题3: 点云话题没有数据

**解决方案**:
- 确认桥接节点正常运行
- 检查AirSim中的Lidar传感器是否配置
- 查看桥接节点的日志输出

### 问题4: 无人机不响应目标点

**解决方案**:
```bash
# 检查SUPER是否收到目标点
ros2 topic echo /goal --once

# 检查SUPER状态
ros2 topic echo /fsm/state --once

# 检查控制指令是否发布
ros2 topic echo /planning/pos_cmd --once
```

---

## 🛑 结束测试

### 正常关闭顺序

1. **终端6**: Ctrl+C（如果还在运行）
2. **终端5**: 关闭RViz窗口
3. **终端4**: Ctrl+C 停止桥接节点
4. **终端3**: Ctrl+C 停止SUPER
5. **终端1**: 关闭AirSim窗口

### 强制结束AirSim（如果卡死）

```bash
pkill -9 -f ModularPark
```

---

## 📝 测试记录

### 测试日期: ___________

- [ ] AirSim环境启动成功
- [ ] SUPER规划器启动成功
- [ ] 桥接节点启动成功
- [ ] RViz可视化正常（可选）
- [ ] 话题数据正常发布
- [ ] 目标点发送成功
- [ ] 无人机规划路径正常
- [ ] 无人机飞行正常
- [ ] 到达目标点

### 问题记录

1. 
2. 
3. 

---

## 🔗 相关文件

- SUPER启动脚本: `/mnt/data/airsim_super_integration/scripts/start_super_airsim.sh`
- 桥接节点脚本: `/mnt/data/airsim_super_integration/scripts/run_bridge.sh`
- RViz配置: `/mnt/data/airsim_super_integration/config/airsim_rviz.rviz`
- Launch文件: `/mnt/data/airsim_super_integration/launch/airsim_super.launch.py`

---

## 📞 技术支持

如有问题，请检查：
1. ROS2环境是否正确配置
2. SUPER工作空间是否编译成功
3. AirSim配置文件是否正确
4. 所有依赖包是否安装完整
