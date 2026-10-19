# 无人机任务管理与安全状态机原型

ROS2 Foxy / Ubuntu 20.04，支持离线状态机验证和 Windows AirSim + WSL2 联动。
当前提供单航点、暂停/继续、返航自动降落和超时检测；尚未实现任务队列或 ROS2 Action。

## 构建

```bash
source /opt/ros/foxy/setup.bash
cd /mnt/d/ros作业/ros2-contrib
rosdep install --from-paths src/air/uav_mission_safety_manager --ignore-src --rosdistro foxy -r -y
colcon build --symlink-install --packages-select uav_mission_safety_manager
source install/setup.bash
```

可选 AirSim 客户端（先安装其安装程序需要的依赖）：

```bash
python3 -m pip install --user numpy msgpack-rpc-python
python3 -m pip install --user airsim==1.8.1
```

## 启动与连接

先启动 Windows OpenHUTB Air 场景，确保 AirSim 41451 端口对 WSL 可达。
只启动一个状态机节点。每个持续运行的命令使用独立终端；各终端先 source 环境。

```bash
ros2 launch uav_mission_safety_manager main.launch.py \
  use_airsim:=true airsim_host:=172.29.240.1 airsim_port:=41451
```

IP 是示例，WSL2 NAT 模式下用 `ip route | awk '/default/ {print $3}'` 查询当前主机地址。
启动节点不会自动连接、解锁或起飞。连接前无人机必须已落地且停稳，以记录起点。
先检查 `ros2 param get /uav_mission_safety_manager use_airsim` 为 True。
AirSim 模式忽略外部 `/uav/telemetry`，防止模拟遥测掩盖真实断线。

在命令终端中依次执行，每步查看状态后再执行下一步：

```bash
ros2 topic pub --once /uav_mission_safety_manager/command std_msgs/msg/String '{data: "{\"event\":\"connect\"}"}'
# 等待 ARMING
ros2 topic pub --once /uav_mission_safety_manager/command std_msgs/msg/String '{data: "{\"event\":\"armed\"}"}'
# 等待 TAKEOFF，此时只完成解锁
ros2 topic pub --once /uav_mission_safety_manager/command std_msgs/msg/String '{data: "{\"event\":\"airborne\"}"}'
# 此命令请求起飞；遥测确认离地后才进入 EXECUTING
```

另一个终端观察：

```bash
ros2 topic echo /uav_mission_safety_manager/status
```

状态 JSON 包含 `state`、`reason`、`position_ned`、`target_ned`、`target_speed`、
`takeoff_rpc_result`、`command_pending`、`last_error`。
`takeoff_rpc_result` 为 null 表示尚无结果；False 会保留在状态和错误信息中。
起飞后的 EXECUTING 只表示遥测确认离地，不能据此声称起飞接口返回 True。
`last_error` 保留最近一次 RPC/遥测错误，用于排障。

## 航点与控制

航点是 AirSim 本地 NED 绝对坐标（米），不是相对当前位置的位移；不自动转换 ROS ENU。
NED 的 x 向北、y 向东、z 向下，`z=-10` 是相对于 AirSim 原点向上 10 米。
安全高度根据连接时起点的 z 计算，默认上限 50 米。

```bash
ros2 topic pub --once /uav_mission_safety_manager/command std_msgs/msg/String '{data: "{\"event\":\"goto\",\"x\":10,\"y\":0,\"z\":-10,\"speed\":3}"}'
ros2 service call /uav_mission_safety_manager/pause std_srvs/srv/Trigger '{}'
ros2 service call /uav_mission_safety_manager/resume std_srvs/srv/Trigger '{}'
ros2 service call /uav_mission_safety_manager/return_home std_srvs/srv/Trigger '{}'
```

暂停取消当前任务并请求悬停；继续重新派发剩余目标点及原速度。
普通航点保持 1 米到达容差。返航使用实际捕获的起点 x/y，速度 0.5 米/秒，
保持当前高度或至少起点上方 5 米（受高度上限约束）。进入最后 1 米后，
通过最长 0.3 秒、合速度不超过 0.3 米/秒的速度指令修正位置。
只有距目标不超过 0.08 米且合速度不超过 0.05 米/秒持续 1 秒，才执行
`RETURNING → LANDING` 并下发降落；遥测确认落地后进入 `LANDED`。
这些阈值属于待场景验证的控制参数，不代表任意场景都能保证该精度。
降落后先解除解锁，再等待新遥测确认落地。部分 OpenHUTB/SimpleFlight 场景在屋顶接触后，
需要解除解锁才更新 `landed_state`：本原型仅在降落 RPC 明确返回 True、距连接时起点不超过
0.15 米、三轴合速度不超过 0.1 米/秒并连续满足 3 秒后，允许尝试解除解锁。
遥测缺失、无效速度或偏离条件会中断静止确认。该兼容逻辑限于已记录的起降点，
不能用于确认任意位置的地面接触，也不能作为真实无人机的落地检测方案。
降落返回 True 本身不会让状态变成 LANDED，最终仍需解除解锁成功及新遥测的落地反馈。
返航沿直线飞行，没有避障或路径规划。

可在 TAKEOFF、EXECUTING、PAUSED、RETURNING、LANDING 或 FAILSAFE 请求降落：

```bash
ros2 service call /uav_mission_safety_manager/land std_srvs/srv/Trigger '{}'
```

服务成功表示指令已被本地接受，异步 RPC 失败会进入 FAILSAFE；成功不代表飞行已完成。
非法状态的请求在调用 AirSim 前拒绝。AirSim 模式禁止手动发送 `connected`、`at_home`、`landed`，
这些事件由 RPC/遥测结果决定。LANDED 后可发送 `reset` 回到 IDLE，下一轮重新连接记录起点。

## 并发与安全边界

* ROS 回调只派发请求、轮询 Future，不等待飞行完成。
* 指令派发、遥测及飞行结果等待分别使用独立单线程池，各线程拥有自己的 RPC 客户端，避免共享 msgpack/Tornado 事件循环。
* 指令/遥测 RPC 超时设置为 1 秒；专用飞行线程的 RPC 超时为 125 秒，航点/降落操作超时为 120 秒，起飞为 20 秒。
  专用飞行线程调用 Future 的 `get()`，持续处理响应；ROS 状态发布和安全检测仍可运行。
* 替换飞行任务前请求取消旧任务，并最多等待 2 秒确认旧 Future 结束；新任务随后在同一飞行线程启动。
  旧任务无法结束时拒绝发出新飞行指令并报告故障，不让降落与旧航点并发执行。
* 同时最多一个指令派发、一个遥测请求；正在派发时拒绝新指令，请稍后重试。
* 用户请求到来时先处理已完成的派发 Future，避免因等待定时器收尾而错误拒绝服务；仍在派发中的指令继续拒绝重入。
* 飞行期间仍可接受暂停/降落，不需要等待上一段飞行结束。
* 旧指令前发起的遥测样本不会完成新指令的状态转换。
* 遥测超过 2 秒未更新、超高或飞行阶段超过 120 秒会进入 FAILSAFE。超时计时使用单调时钟。
* FAILSAFE 目前记录故障并允许显式重试降落，不保证自动悬停或降落，也不表示已中止 AirSim 原有飞行任务。
* AirSim 模式尚无电子围栏或碰撞检测；不能把外部布尔值当作真实围栏实现。
* 停止 ROS 节点不会自动降落。请在关闭前确认无人机已落地。

## 测试与离线模式

```bash
# source install/setup.bash 后，在仓库根目录执行
python3 -m unittest discover -s src/air/uav_mission_safety_manager/tests -v
# 使用独立 ROS domain，不连接模拟器、不影响正常飞行节点
ROS_DOMAIN_ID=87 python3 src/air/uav_mission_safety_manager/tests/ros_smoke.py
```

单元测试覆盖非阻塞遥测、起飞确认、暂停恢复目标、真实起点返航、自动降落、非法状态无副作用、
RPC 失败、超时及非法航点。ROS 通信测试验证未完成 RPC 不阻塞服务、状态发布和安全检测。

离线模式用 `use_airsim:=false`（默认）启动，手动发送状态事件；进入飞行状态前需持续发布模拟遥测：

```bash
ros2 topic pub -r 2 /uav/telemetry std_msgs/msg/String '{data: "{\"altitude\":10,\"inside_fence\":true}"}'
```

实际模拟器中已完成连接→解锁→起飞→航点中暂停/继续→返航→自动落地的联动测试，
详见 [AIRSIM_VALIDATION.md](AIRSIM_VALIDATION.md)，其中保留失败轮次和起飞 False 异常。
补充的 ROS 接口完整流程、起点上方返航中显式降落及 RPC 传输中断结果亦记录在该文档中。
不把离线测试当作实际模拟器飞行验证。

也可在没有其他控制节点运行时，用实际适配器与控制器执行自动模拟器复测：

```bash
# 会重置并起飞模拟无人机，水平移动 3 米，最后返航降落。
python3 src/air/uav_mission_safety_manager/tests/live_airsim_validation.py \
  --host "$(ip route | awk '/default/ {print $3; exit}')" \
  --log /tmp/uav_airsim_retest.jsonl --fly --reset
```

该脚本记录控制器与真实 AirSim 的联动，未经过 ROS 话题/服务；ROS 通信另由冒烟测试验证。
重置后须连续 3 秒确认落地且停稳，才记录起点并解锁。失败后的恢复降落不计为流程通过。

### 通过 ROS 接口执行实际模拟器补测

以下脚本用于实际飞行验证。先确认 Air 场景已加载、41451 端口可达，
并停止其他飞行控制节点；脚本自行启动已安装的 launch，通过命令话题与 Trigger 服务控制。
每次会重置模拟无人机并起飞，三种用例必须依次执行。

```bash
source /opt/ros/foxy/setup.bash
source install/setup.bash
# 仓库根目录执行；每条命令结束并确认落地后再执行下一条。
ROS_DOMAIN_ID=93 python3 src/air/uav_mission_safety_manager/tests/ros_airsim_validation.py \
  --host "$(ip route | awk '/default/ {print $3; exit}')" \
  --case mission --log /tmp/ros_airsim_mission.jsonl --fly
ROS_DOMAIN_ID=93 python3 src/air/uav_mission_safety_manager/tests/ros_airsim_validation.py \
  --host "$(ip route | awk '/default/ {print $3; exit}')" \
  --case return_land --log /tmp/ros_airsim_return_land.jsonl --fly
ROS_DOMAIN_ID=93 python3 src/air/uav_mission_safety_manager/tests/ros_airsim_validation.py \
  --host "$(ip route | awk '/default/ {print $3; exit}')" \
  --case disconnect --log /tmp/ros_airsim_disconnect.jsonl --fly
```

`mission` 检查水平航点、暂停停稳、继续保留指令速度及返航自动降落；
`return_land` 在返航到起点上方后、自动降落前显式请求降落，不能代表远离起点的任意位置降落；
`disconnect` 切断节点专用 TCP 转发连接，检查 FAILSAFE 和持续状态发布，模拟器本身保持运行。
后者是 RPC 传输中断测试，不能替代模拟器进程崩溃测试，也不证明断线后会自动降落。
脚本记录服务响应耗时、状态及 PASS/FAIL；启动日志写入同名 `.node.log`。
停止被测节点后的独立恢复降落单独记录，不算被测流程通过。
