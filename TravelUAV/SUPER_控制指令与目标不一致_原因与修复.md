# 控制指令与发给 SUPER 的目标位置不一致 — 原因与修复

## 现象

- **发给 SUPER 的目标**（TravelUAV 日志）：`AirSim(341.50, 132.79, -2.87) -> ROS(132.79, 341.50, 2.87)`，即目标高度 **z=2.87m**（ENU）。
- **桥接收到的控制指令**（第二个终端）：`控制指令ROS(ENU): (133.25, 341.75, 1.05)`、`(132.83, 341.50, 1.47)` 等，高度只有 **z≈1.0~1.5m**。
- 即：**xy 基本一致，但 z 被改成了约 1.5m，与目标 2.87m 不一致**。

---

## 根本原因：SUPER 的 `click_height` 覆盖了目标 z

SUPER 的 FSM 在接收点击/目标点（`/goal_pose`）时，会做一次高度覆盖：

- **代码位置**：`SUPER-master/super_planner/src/super_core/fsm.cpp` 的 `setGoalPosiAndYaw`：
  ```cpp
  auto click_point = p;
  if (cfg_.click_height > -5) {
      click_point.z() = cfg_.click_height;  // 用配置高度覆盖我们发过去的 z
  }
  ```
- **配置**：`super_planner/config/click_smooth_ros2.yaml` 里默认 `click_height: 1.5`。
- **结果**：无论 TravelUAV 发的是 z=2.87 还是其他值，SUPER 都会把**目标高度**改成 **1.5m**，所以规划/控制指令的 z 一直是 ~1.0–1.5，与“发给 SUPER 的指令位置”不一致。

设计上这是为了“在 Rviz 里点选时固定飞行高度”；但在与 TravelUAV 集成时，我们需要**使用消息里的真实 z**，因此要关掉这层覆盖。

---

## 修复方法

让 SUPER **不要覆盖 z**：把 `click_height` 设成 **≤ -5**（例如 **-10**），这样 `cfg_.click_height > -5` 为假，就不会执行 `click_point.z() = cfg_.click_height`，目标点的 z 会保持为你发过去的 2.87。

### 已修改的文件

- **`/mnt/data/SUPER-master/super_planner/config/click_smooth_ros2.yaml`**  
  - 将 `click_height: 1.5` 改为 `click_height: -10`，并加了注释说明用途。

### 若你实际用的是 super_ws 里的配置

- 路径一般为：`super_ws/src/SUPER/super_planner/config/click_smooth_ros2.yaml`  
- 该文件里若仍是 `click_height: 1.5`，请同样改为 **`click_height: -10`**（或任意 ≤ -5 的值）。  
- 若已经是 `-10.0`，则无需再改。

改完后**重启 SUPER 节点**（或重新 launch），再发目标，控制指令的 z 就会与目标 z 一致。

---

## 关于 “到达目标 0.0s” 的说明

你这边日志里有：`[SUPER] ✓ 到达目标！距离=1.91m, 耗时=0.0s`。

- 当时无人机高度约 **1.02m**，目标高度 **2.87m**，高度差约 **1.85m**。
- 若使用**带高度阈值的到达判断**（例如 `height_threshold=1.0`），不应在 0s 就判“到达”。
- 若当前运行的 `super_ros2_client` 是**只按 3D 距离 < 2.0m 就判到达**的旧逻辑（没有单独的高度差条件），则 1.91m < 2.0m 会立刻判到达，就会出现“0.0s 到达”但实际高度还差很多的情况。

建议确认 TravelUAV 侧的 `super_ros2_client.wait_for_arrival` 已使用“3D 距离 + 高度差”双条件（与当前仓库中的实现一致），这样只有**平面和高度都接近**时才判到达，避免误判。

---

## 小结

| 问题 | 原因 | 处理 |
|------|------|------|
| 控制指令 z 与目标 z 不一致 | SUPER 用 `click_height: 1.5` 覆盖了目标 z | 将 `click_height` 改为 **-10**（或任意 ≤ -5） |
| 0.0s 就判到达 | 可能只按 3D 距离判断，未用高度差 | 使用带 `height_threshold` 的 `wait_for_arrival` |

按上述修改并重启 SUPER 后，控制指令与发给 SUPER 的目标位置（含 z）应会一致。
