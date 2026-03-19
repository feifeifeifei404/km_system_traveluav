# TravelUAV + SUPER 集成 — 当前代码现状总结

## 一、目标与约束（你的需求）

- **目标**：用 SUPER 完全替代 TravelUAV 中的「小模型精化 + 轨迹执行」部分，只保留大模型输出一个子目标点 → SUPER 规划并飞过去 → 到达后继续 TravelUAV 的终止/失败判断。
- **坐标系**：TravelUAV 用 AirSim 世界坐标（NED）；SUPER/ROS2 用 ENU；需要在一处做 NED↔ENU 转换。
- **环境隔离**：TravelUAV 跑在 `llamanew`（或 llama）conda；SUPER 侧 `conda deactivate`，用系统/ROS2 环境；两套环境独立，通过 ROS2 通信。

---

## 二、TravelUAV 侧现状

### 2.1 主流程入口与评估循环

- **主入口**：`TravelUAV/src/vlnce_src/eval.py`
- **评估循环**：`eval()` → `eval_env.next_minibatch()` 取 batch → 每步 `for t in range(maxWaypoints+1)`：
  1. `batch_state.check_batch_termination(t)` → 是否终止
  2. 获取观测、准备输入
  3. **得到 `final_refined_waypoints`**（见下）
  4. **核心替换区**：不再调用 `eval_env.makeActions(final_refined_waypoints)`，改为「发子目标给 SUPER → 等到达 → 更新 sim_states」
  5. `batch_state.update_from_env_output(outputs)`、`predict_done`、`update_metric` 等（保持不变）

### 2.2 `final_refined_waypoints` 的来源（被替换的逻辑）

- **标准推理**（无 budget forcing）  
  - `model_wrapper.run(inputs, episodes, rot_to_targets)`  
  - 内部：`waypoints_llm_new = run_llm_model(...)` → `refined_waypoints, ... = run_traj_model(episodes, waypoints_llm_new, rot_to_targets)`  
  - `run_traj_model`：小模型精化 → `transform_to_world(refined_waypoints, episodes)` → 得到世界坐标下的 `refined_waypoints`  
  - 返回的 `final_refined_waypoints` 即**小模型精化后的世界坐标航点**（AirSim NED）。

- **Budget forcing 模式**  
  - 并行/串行多候选 → `score_and_select_best_waypoint(...)` 得到 `best_waypoint`（世界坐标）。  
  - `final_refined_waypoints = [best_waypoint]`。  
  - 此处**已经不再调用小模型**，直接是「大模型思考后的一个点」；你要的就是用这个点作为 SUPER 的子目标。

结论：**要替换的，就是「小模型精化 + `makeActions(waypoints)」整段**；替换成「大模型给出的一个点 → 发给 SUPER → 等到达 → 更新状态」。

### 2.3 原 `makeActions` 做了什么（被 SUPER 替代的部分）

- 文件：`TravelUAV/src/vlnce_src/env_uav.py`，`makeActions(waypoints_list)`  
- 逻辑概要：
  - 用 `simulator_tool.move_path_by_waypoints(waypoints_list, start_states)` 在仿真里按航点移动；
  - 拿回 `batch_results`（轨迹）、`batch_iscollision`；
  - 更新 `sim_states[index].trajectory`、`step`、`pre_waypoints`、`is_collisioned`，判断 `oracle_success`、`is_end`；
  - 调用 `update_measurements()`。

集成后：**不再调用 `makeActions`**，改为在 `eval.py` 的「核心替换区」里：发一个子目标给 SUPER、等到达、然后**按同样语义**更新 `sim_states`（轨迹、步数、碰撞、成功/结束等）。

### 2.4 当前 eval.py 里「核心替换区」已实现的内容

- 位置：约 627–751 行，注释标明了「替换掉原本的小模型 refine 和 env.makeActions」。
- 流程概括：
  1. 若 `eval_env.sim_states[batch_idx].is_end`，直接 `get_obs()` 并 continue。
  2. 从 `final_refined_waypoints[0]` 取出**子目标**：
     - 若是多点轨迹 `(N,3)`，取 `trajectory[min(1, len(trajectory)-1)]`（第 2 个点或最后一点）；
     - 否则当作单点 `sub_goal`。
  3. 用 `get_super_ros2_client().send_goal(sub_goal[0], sub_goal[1], sub_goal[2])` 发给 SUPER。
  4. `wait_for_arrival_in_airsim(eval_env, sub_goal, threshold=2.0, timeout=120.0)` 阻塞等待到达。
  5. 根据 `arrival_success`、`super_trajectory`、`collision_detected` 更新：
     - `sim_states[batch_idx].trajectory`、`step`、`is_collisioned`、`pre_waypoints`；
     - `oracle_success`（到目标距离 < SUCCESS_DISTANCE）、`is_end`（步数≥maxWaypoints 或按需）；
     - `update_measurements()`；
  6. 最后 `outputs = eval_env.get_obs()`，供后面 `update_from_env_output` 等使用。

也就是说：**慢系统（TravelUAV）大模型只出一个子目标点，快系统（SUPER）负责飞过去，到达后的判断和 TravelUAV 原有逻辑已经接上**；唯一缺的是「标准推理」时仍会走小模型，若你要「完全用 SUPER 替代小模型精化」，需要让标准模式也只输出大模型的一个点并走同一套 SUPER 逻辑（见下文建议）。

### 2.5 TravelUAV 侧 SUPER 通信：super_ros2_client.py

- 文件：`TravelUAV/src/vlnce_src/super_ros2_client.py`
- 作用：在 TravelUAV 进程内起一个 ROS2 节点（单例），与 SUPER 侧通过话题通信。
- 接口：
  - **发布**：`/goal_pose`（`geometry_msgs/PoseStamped`），目标点。
  - **订阅**：`/lidar_slam/odom`（`nav_msgs/Odometry`），用于当前位置和 `wait_for_arrival`。
- 坐标系：
  - `send_goal(x, y, z)` 的 x,y,z 约定为 **AirSim NED**；
  - 内部转换为 ROS ENU：`ros_x = y, ros_y = x, ros_z = -z`，再填到 `PoseStamped` 发布。
- `wait_for_arrival(target_pos_airsim, threshold, timeout, height_threshold=1.0, check_interval=0.5)`：
  - 把 `target_pos_airsim` 转成 ROS 坐标，与 `/lidar_slam/odom` 的当前位置比较；
  - 返回 `(success, status)`，status 可为 `"ARRIVED"` / `"TIMEOUT"` / `"STUCK"`。
- 单例与线程：`get_super_ros2_client()` 内部 `rclpy.init()` + 创建节点 + 在 daemon 线程里 `rclpy.spin()`，保证 TravelUAV 主线程可同步调用 `send_goal` 和 `wait_for_arrival`。

依赖：TravelUAV 环境需能 `import rclpy` 等（即 llama 环境里装 ROS2 的 rclpy），与 SUPER 进程通过 DDS 通信，无需同进程、同 conda。

### 2.6 eval.py 中的 wait_for_arrival_in_airsim

- 实现：调用 `super_ros2_client.wait_for_arrival(target_pos, threshold, timeout, check_interval=record_interval)`，把返回的 `(success, status)` 转成：
  - `collision_detected = (status == "STUCK")`
  - `trajectory = []`（当前 ROS2 版不回传轨迹）
- 与 `super_ros2_client` 的约定：**传入的 target_pos 为 AirSim NED**，与 `send_goal` 一致。

---

## 三、SUPER 侧现状

### 3.1 你当前使用的 SUPER 与桥接

- **SUPER 规划器**：订阅目标点、发布控制指令；ROS2 配置里目标点为 `click_goal_topic: "/goal_pose"`（如 `super_planner/config/click_smooth_ros2.yaml`）。
- **AirSim–SUPER 桥接**：`airsim_super_integration/scripts/airsim_super_bridge_ros2.py`（或 `TravelUAV/jicheng/` 下的同源版本）：
  - 从 AirSim 读 LiDAR、状态 → 发布 `/cloud_registered`、`/lidar_slam/odom`（ENU）；
  - 订阅 `/planning/pos_cmd` 把 SUPER 的控制指令下发给 AirSim；
  - 桥接里还订阅了 `/goal`，用于打印/调试；**实际给 SUPER 用的目标点是 TravelUAV 发的 `/goal_pose`**，不冲突。

### 3.2 坐标系统一

- **TravelUAV / AirSim 世界坐标**：NED（X 前，Y 右，Z 下）。
- **ROS/SUPER**：ENU（X 东，Y 北，Z 上）。
- **转换**（已在 `super_ros2_client.send_goal` 和 `wait_for_arrival` 里实现）：
  - NED → ENU：`x_enu = y_ned, y_enu = x_ned, z_enu = -z_ned`。
  - 到达判断时，目标点也按同样方式转成 ENU 再和 odom 比较。

因此：**只要 TravelUAV 传给 SUPER 的始终是「AirSim 世界坐标 (NED)」的一个点**，当前转换就是一致的；大模型/score 给出的 `best_waypoint` 或 `final_refined_waypoints[0]` 已经是世界坐标，直接当 `sub_goal` 使用即可。

---

## 四、两套环境与 ROS2 通信

- **TravelUAV**：conda `llamanew`（或 llama），跑 `eval.py` + 大模型 + `super_ros2_client`（仅 rclpy + geometry_msgs + nav_msgs）。
- **SUPER + 桥接**：`conda deactivate`，用系统 Python + ROS2 Humble，跑：
  - `airsim_super_bridge_ros2.py`（发布 odom/点云，订阅 pos_cmd，可选订阅 /goal 做调试）；
  - SUPER 的 fsm/规划节点（订阅 `/goal_pose`）。

两进程通过 ROS2 DDS 通信，**环境完全独立**；只要两边都 source 同一套 ROS2 工作空间且网络/ DDS 配置正确即可。

---

## 五、当前已接好的部分 vs 需要你确认/改动的部分

### 5.1 已经接好的部分

- 大模型（或 budget forcing 择优）得到一个世界坐标点 → 作为 `sub_goal`。
- `super_ros2_client.send_goal(sub_goal)` 发到 `/goal_pose`（NED→ENU 已做）。
- `wait_for_arrival_in_airsim(...)` 通过 `/lidar_slam/odom` 判断到达/超时/STUCK。
- 到达后：更新 `sim_states`（trajectory、step、is_collisioned、pre_waypoints、oracle_success、is_end）、`update_measurements()`、`get_obs()`，再走原来的 `update_from_env_output`、`predict_done`、`update_metric`。

即：**「大模型一个点 → SUPER 飞 → 到达后接上 TravelUAV 判断」这条链已经打通**，且坐标系在 TravelUAV 侧统一为 NED，在 ROS2 侧统一为 ENU。

### 5.2 需要你确认或小改的地方

1. **标准推理（非 budget forcing）**  
   - 当前标准模式仍调用 `model_wrapper.run(...)`，内部会跑小模型 `run_traj_model`，得到多段精化轨迹。  
   - 若要「只用 SUPER 替代小模型精化」：  
     - 要么在标准模式里也改为「只取 LLM 的一个点」再走同一套 SUPER 逻辑（例如从 `waypoints_llm_new` 取第一个点并 `transform_to_world` 得到世界坐标一个点）；  
     - 要么加一个开关（如 `args.use_super_for_execution`），当为 True 时标准模式也只输出一个点并走 SUPER 分支。

2. **子目标取点方式**  
   - 当前对「多点轨迹」取的是 `trajectory[min(1, len(trajectory)-1)]`（第 2 个或最后一点）。  
   - 若你希望**始终只发一个点**且就是「大模型说的下一个子目标」，在完全去掉小模型后，`final_refined_waypoints` 应只含一个点，这里就不会再有多点分支，逻辑会更简单。

3. **话题与配置**  
   - TravelUAV 发的是 `/goal_pose`，SUPER 的 ROS2 配置为 `click_goal_topic: "/goal_pose"`，一致。  
   - 桥接的 `/goal` 仅作调试用，不影响集成。

4. **wait_for_arrival 参数**  
   - `super_ros2_client.wait_for_arrival` 的 `height_threshold` 目前用默认 1.0m；若你对高度有更严/更松要求，可在 `eval.py` 或 client 里暴露为参数。

---

## 六、流程小结（当前实现）

```
1. TravelUAV (eval.py, conda llama)
   ├─ 大模型（+ 可选 budget forcing）→ 得到 final_refined_waypoints（世界坐标，NED）
   ├─ 取一个子目标 sub_goal
   ├─ super_ros2_client.send_goal(sub_goal)  → 发布 /goal_pose (ENU)
   └─ wait_for_arrival_in_airsim()  → 订阅 /lidar_slam/odom，判断到达/超时/STUCK

2. SUPER 侧 (conda deactivate, ROS2)
   ├─ 订阅 /goal_pose，规划轨迹，发布 /planning/pos_cmd
   └─ airsim_super_bridge_ros2 订阅 pos_cmd，控制 AirSim

3. 到达后 (仍在 eval.py)
   └─ 更新 sim_states、update_measurements、get_obs() → 继续原有终止/失败判断与下一步
```

**总结**：  
- 慢系统（TravelUAV）用大模型输出**一个**目标点，交给快系统（SUPER）规划并飞行，已经替代了原先「小模型精化 + makeActions」的执行部分；到达后的流程已接上 TravelUAV，坐标系在 NED/ENU 间已统一。  
- 若你要「完全」去掉小模型精化，只需在标准推理分支也改为只使用大模型的一个点并走同一段 SUPER 发送/等待/更新逻辑即可；其余失败判断和 TravelUAV 流程可保持不变。

如果你愿意，下一步可以具体改哪一版 eval（例如只改带 budget forcing 的 eval.py，或加 `use_super_for_execution` 开关）我可以按文件+行号给出修改片段。
