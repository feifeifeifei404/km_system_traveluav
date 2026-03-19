# TravelUAV + SUPER 集成修复说明

## 问题描述

在之前的集成中，慢系统（TravelUAV）使用大模型输出子目标点，然后将这个点发送给快系统（SUPER）执行精细化飞行。但是存在一个**关键问题**：

**SUPER执行完飞行后，轨迹信息没有送回慢系统，导致慢系统的状态无法正确更新，后续的判断逻辑（如是否到达目标、是否终止等）无法正常工作。**

### 原来的流程（使用小模型）

```
慢系统大模型决策 → 小模型轨迹精细化 → makeActions(执行飞行) → 更新sim_states → 判断是否终止
                                                      ↓
                                            记录轨迹、步数、碰撞等信息
```

### 之前的集成（问题版本）

```
慢系统大模型决策 → 发送给SUPER → wait_for_arrival(等待到达) → ❌ sim_states未更新 → 判断逻辑失效
```

### 现在的修复版本

```
慢系统大模型决策 → 发送给SUPER → wait_for_arrival(等待+记录轨迹) → ✅ 手动更新sim_states → 判断是否终止
                                           ↓
                                   返回轨迹、碰撞信息
```

---

## 修改内容

### 1. 修改 `wait_for_arrival_in_airsim()` 函数

**位置**: `TravelUAV/src/vlnce_src/eval.py`

**主要改动**:
- **新增轨迹记录**: 在等待飞行过程中，按时间间隔采样记录无人机的位置、姿态、速度等信息
- **新增碰撞检测**: 实时检测飞行过程中是否发生碰撞
- **修改返回值**: 从单一的 `bool` 改为 `(success, trajectory, collision_detected)` 三元组

**函数签名变化**:
```python
# 之前
def wait_for_arrival_in_airsim(env, target_pos, threshold=2.0, timeout=60.0):
    ...
    return True/False

# 现在
def wait_for_arrival_in_airsim(env, target_pos, threshold=2.0, timeout=60.0, record_interval=0.5):
    ...
    return success, trajectory, collision_detected
```

**记录的轨迹数据格式**（与原makeActions完全一致）:
```python
trajectory = [
    {
        'sensors': {
            'state': {
                'position': [x, y, z],
                'orientation': [x, y, z, w],  # 四元数
                'linear_velocity': [vx, vy, vz],
                'angular_velocity': [wx, wy, wz]
            }
        }
    },
    ...
]
```

**重要**: 这个格式与原 `move_path_by_waypoints()` 返回的格式**完全相同**，确保：
- `sim_states.trajectory.extend(super_trajectory)` 后数据结构兼容
- `sim_states.state` 属性可以正确访问 `trajectory[-1]['sensors']['state']`
- `sim_states.pose` 属性可以正确访问 `position + orientation`

---

### 2. 修改主循环中的状态更新逻辑

**位置**: `TravelUAV/src/vlnce_src/eval.py` 核心替换区域

**主要改动**:

#### 2.1 接收SUPER返回的轨迹数据
```python
arrival_success, super_trajectory, collision_detected = wait_for_arrival_in_airsim(
    eval_env, sub_goal
)
```

#### 2.2 手动更新 `sim_states`，模拟原来 `makeActions()` 的行为

以下是对 `sim_states` 的**完整更新**（这些步骤原本由 `makeActions` 自动完成）：

##### a) 更新轨迹信息
```python
eval_env.sim_states[batch_idx].trajectory.extend(super_trajectory)
```
- 将SUPER执行的轨迹添加到慢系统的轨迹记录中
- 这样慢系统可以追踪整个飞行历史

##### b) 更新步数
```python
eval_env.sim_states[batch_idx].step += 1
```
- 每次执行一个子目标点，步数+1
- 用于判断是否超过最大步数限制

##### c) 更新碰撞状态
```python
eval_env.sim_states[batch_idx].is_collisioned = collision_detected
```
- 记录本次飞行是否发生碰撞

##### d) 更新 waypoints 记录
```python
eval_env.sim_states[batch_idx].pre_waypoints = waypoint_list
```
- 记录本次发送给SUPER的目标点
- 用于调试和分析

##### e) 检查是否成功到达目标
```python
target_position = eval_env.batch[batch_idx]['object_position']
current_position = eval_env.sim_states[batch_idx].pose[0:3]
dist_to_target = np.linalg.norm(np.array(current_position) - np.array(target_position))

if dist_to_target < eval_env.sim_states[batch_idx].SUCCESS_DISTANCE:
    eval_env.sim_states[batch_idx].oracle_success = True
```
- **关键逻辑**: 每次飞行后都检查是否已经到达最终目标
- 如果距离小于阈值，标记为成功

##### f) 检查是否超过最大步数
```python
if eval_env.sim_states[batch_idx].step >= int(args.maxWaypoints):
    eval_env.sim_states[batch_idx].is_end = True
```
- **关键逻辑**: 防止无限循环，超过最大步数时终止episode

##### g) 更新距离测量
```python
eval_env.update_measurements()
```
- 计算当前位置到目标的距离
- 打印调试信息

---

## 关键改进点

### ✅ 1. 轨迹连续性
- SUPER执行的轨迹现在会被记录到 `sim_states.trajectory` 中
- 慢系统可以完整追踪整个飞行历史

### ✅ 2. 状态一致性
- `sim_states` 的所有关键状态（步数、碰撞、waypoints等）都会正确更新
- 与原来使用小模型时的行为保持一致

### ✅ 3. 终止条件正确触发
- **成功到达**: 检查距离 < SUCCESS_DISTANCE
- **超过步数**: 检查 step >= maxWaypoints
- **碰撞失败**: 记录 is_collisioned

### ✅ 4. 后续流程不变
- 更新观测: `outputs = eval_env.get_obs()`
- 更新批次状态: `batch_state.update_from_env_output(outputs)`
- 预测终止: `batch_state.predict_dones = model_wrapper.predict_done(...)`
- 更新指标: `batch_state.update_metric()`

---

## 数据流对比

### 原来的 `makeActions` 做的事情：

```python
def makeActions(self, waypoints_list):
    # 1. 发送waypoints给模拟器
    results = self.simulator_tool.move_path_by_waypoints(...)
    
    # 2. 获取飞行结果
    batch_results = [...]      # 轨迹状态
    batch_iscollision = [...]  # 碰撞信息
    
    # 3. 更新 sim_states
    for index, waypoints in enumerate(waypoints_list):
        # 检查成功条件
        if distance_to_target < SUCCESS_DISTANCE:
            self.sim_states[index].oracle_success = True
        
        # 检查终止条件
        elif self.sim_states[index].step >= maxWaypoints:
            self.sim_states[index].is_end = True
        
        # 更新状态
        self.sim_states[index].step += 1
        self.sim_states[index].trajectory.extend(batch_results[index])
        self.sim_states[index].pre_waypoints = waypoints
        self.sim_states[index].is_collisioned = batch_iscollision[index]
    
    # 4. 更新距离测量
    self.update_measurements()
    
    return batch_results
```

### 现在的 SUPER 集成做的事情：

```python
# 1. 发送目标点给SUPER
send_goal_to_super(sub_goal[0], sub_goal[1], sub_goal[2])

# 2. 等待并获取飞行结果
arrival_success, super_trajectory, collision_detected = wait_for_arrival_in_airsim(...)

# 3. 获取最新观测
outputs = eval_env.get_obs()

# 4. 手动更新 sim_states (模拟 makeActions 的行为)
eval_env.sim_states[batch_idx].trajectory.extend(super_trajectory)
eval_env.sim_states[batch_idx].step += 1
eval_env.sim_states[batch_idx].is_collisioned = collision_detected
eval_env.sim_states[batch_idx].pre_waypoints = waypoint_list

# 5. 检查成功条件
if dist_to_target < SUCCESS_DISTANCE:
    eval_env.sim_states[batch_idx].oracle_success = True

# 6. 检查终止条件
if step >= maxWaypoints:
    eval_env.sim_states[batch_idx].is_end = True

# 7. 更新距离测量
eval_env.update_measurements()
```

---

## 测试建议

### 1. 验证轨迹记录
```python
# 在飞行后检查轨迹长度
print(f"Trajectory length: {len(eval_env.sim_states[0].trajectory)}")
print(f"Latest position: {eval_env.sim_states[0].trajectory[-1]['position']}")
```

### 2. 验证步数更新
```python
# 每次飞行后检查步数
print(f"Current step: {eval_env.sim_states[0].step}/{args.maxWaypoints}")
```

### 3. 验证成功判断
```python
# 检查成功标志
print(f"Oracle success: {eval_env.sim_states[0].oracle_success}")
print(f"Distance to target: {dist_to_target:.2f}m")
```

### 4. 验证终止条件
```python
# 检查终止标志
print(f"Is end: {eval_env.sim_states[0].is_end}")
print(f"Reason: {'max steps' if step >= maxWaypoints else 'success' if oracle_success else 'ongoing'}")
```

---

## 注意事项

### 1. 批次大小
- 当前代码假设 `batch_size=1`
- 如果需要支持多批次，需要遍历所有 `batch_idx`

### 2. 坐标系一致性
- 确保 SUPER 和 TravelUAV 使用相同的坐标系（AirSim世界坐标系）
- 子目标点已经是世界坐标，无需转换

### 3. 环境隔离
- TravelUAV 在 `llama` conda 环境中
- SUPER 在 `base` 环境中
- 通过 HTTP/gRPC 通信保持环境隔离

### 4. 轨迹采样率
- 默认每 0.5 秒记录一个轨迹点（可通过 `record_interval` 调整）
- 采样率越高，轨迹越精细，但数据量越大

---

## 总结

通过这次修复，我们实现了：

1. ✅ **完整的轨迹回传**: SUPER 执行的轨迹现在会被记录并送回慢系统
2. ✅ **状态正确更新**: `sim_states` 的所有关键信息都会正确更新
3. ✅ **判断逻辑正常**: 成功/终止条件现在可以正确触发
4. ✅ **流程保持一致**: 与原来使用小模型时的行为完全一致

**核心思想**: 用 SUPER 替换掉小模型精细化的部分，但保持慢系统原有的**决策循环和判断逻辑**不变。
