# SUPER单独测试 - 快速参考

## 🎯 测试目标

单独测试SUPER规划器在AirSim环境中的性能，不依赖TravelUAV的其他组件。

---

## 🚀 方法一：手动启动（推荐学习）

### ⚠️ 远程连接服务器用户注意

如果你是**远程SSH连接**服务器，AirSim窗口需要在**服务器本地启动**（有显示器的机器上）。

**操作流程：**
1. 在服务器本地（有显示器）启动AirSim
2. 在远程SSH终端运行其他组件（SUPER、桥接节点等）

---

按顺序在**不同终端**中执行以下命令：

### 1️⃣ 启动AirSim环境

**在服务器本地（有显示器）执行：**
```bash
cd /mnt/data/TravelUAV/envs/closeloop_envs
./ModularPark.sh -ResX=1280 -ResY=720 -windowed
```
⏱️ 等待环境加载完成（看到无人机）

**在远程SSH终端验证（可选）：**
```bash
# 检查进程
ps aux | grep ModularPark

# 检查API端口
netstat -tuln | grep 41451
```

### 2️⃣ 启动SUPER规划器
```bash
conda deactivate
cd /mnt/data/airsim_super_integration/scripts
./start_super_airsim.sh
```

### 3️⃣ 启动桥接节点
```bash
conda deactivate
cd /mnt/data/airsim_super_integration/scripts
./run_bridge.sh
```

### 4️⃣ 发送目标点
```bash
conda deactivate
cd /mnt/data/airsim_super_integration/scripts

# 查看当前位置
ros2 topic echo /lidar_slam/odom --once | grep -A 3 "position:"

# 发送目标点 (x, y, z)
./send_goal.sh 3 7 -3
```

### 5️⃣ RViz可视化（可选）
```bash
conda deactivate
cd /mnt/data/airsim_super_integration/scripts
./start_rviz.sh
```

---

## ⚡ 方法二：快速启动脚本（实验性）

```bash
cd /mnt/data/airsim_super_integration/scripts
./quick_test.sh
```

这个脚本会自动：
- 启动AirSim环境
- 在新终端中启动SUPER和桥接节点
- 准备好测试环境

⚠️ 注意：需要GUI环境和终端模拟器（gnome-terminal/xterm/konsole）

---

## 📊 验证测试

### 检查话题
```bash
# 点云
ros2 topic hz /cloud_registered

# 里程计
ros2 topic hz /lidar_slam/odom

# 控制指令
ros2 topic hz /planning/pos_cmd
```

### 观察AirSim
- ✅ 无人机起飞
- ✅ 按照规划路径飞行
- ✅ 避障（如果有障碍物）
- ✅ 到达目标点

---

## 🛑 结束测试

### 正常关闭
1. Ctrl+C 停止发送目标的终端
2. 关闭RViz窗口（如果打开）
3. Ctrl+C 停止桥接节点
4. Ctrl+C 停止SUPER
5. 关闭AirSim窗口（服务器本地）

### 强制结束（远程终端）
```bash
# 方法1：使用脚本
cd /mnt/data/airsim_super_integration/scripts
./stop_airsim.sh

# 方法2：直接命令
pkill -9 -f ModularPark
```

---

## 📚 详细文档

- **完整测试指南**: `SUPER测试指南.md`
- **脚本说明**: 查看 `scripts/` 目录下各脚本的注释

---

## 🔧 常见问题

| 问题 | 解决方案 |
|-----|---------|
| SUPER启动失败 | 检查 `/home/wuyou/super_ws` 是否编译 |
| 点云无数据 | 确认桥接节点正常运行 |
| 无人机不响应 | 检查SUPER状态和控制指令话题 |
| RViz无法启动 | 确认已安装 rviz2 |

详细问题排查请参考 `SUPER测试指南.md`

---

## 📝 测试目标点建议

```bash
# 近距离测试
./send_goal.sh 3 7 -3

# 中距离测试
./send_goal.sh 10 5 -2

# 远距离测试
./send_goal.sh 20 10 -4
```

---

## 📞 系统要求

- ✅ Ubuntu 22.04
- ✅ ROS2 Humble
- ✅ SUPER工作空间已编译
- ✅ AirSim环境已安装
- ✅ Python3 环境

---

**最后更新**: 2026-02-10
