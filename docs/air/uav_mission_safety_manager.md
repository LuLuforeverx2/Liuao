# 无人机任务管理与安全状态机

本模块面向 ROS2 Foxy 与 AirSim，提供单航点控制、暂停继续、返航自动降落以及高度/遥测超时检测。

源码与完整操作步骤：[uav_mission_safety_manager](https://github.com/OpenHUTB/ros2/tree/master/src/air/uav_mission_safety_manager)。

## 状态与返航流程

```text
IDLE → CONNECTING → ARMING → TAKEOFF → EXECUTING
EXECUTING ↔ PAUSED
EXECUTING / PAUSED → RETURNING → LANDING → LANDED
异常 → FAILSAFE
```

连接时记录已落地无人机的实际位置；返航到起点上空后自动下发降落，遥测确认落地后才进入 LANDED。
暂停/继续保留原航点速度。返航最后 1 米采用低速位置修正，距目标不超过 0.08 米、
合速度不超过 0.05 米/秒持续 1 秒后才开始降落；新飞行任务等待旧任务取消完成后再启动。
降落收尾包括解除解锁。对于屋顶接触后仍报告 Flying 的 OpenHUTB/SimpleFlight 场景，
仅在降落命令成功、距记录起点不超过 0.15 米且速度不超过 0.1 米/秒持续 3 秒后尝试解除解锁，
随后仍须新遥测确认落地。这是模拟器起降点兼容逻辑，不是任意地形或真机落地检测。
飞行命令派发、遥测及飞行结果等待分别使用独立工作线程，ROS 回调不等待整段飞行完成。
起飞接口实际返回值通过 `takeoff_rpc_result` 记录；进入 EXECUTING 依据离地遥测，不等同于接口返回 True。
暂停、返航和降落服务的成功响应表示接受请求，动作完成需查看状态话题。

## 验证

在安装并构建包、加载工作空间后：

```bash
python3 -m unittest discover -s src/air/uav_mission_safety_manager/tests -v
ROS_DOMAIN_ID=87 python3 src/air/uav_mission_safety_manager/tests/ros_smoke.py
```

测试可在没有模拟器时执行。合入前还需验证真实 AirSim 中的暂停响应、返航自动降落、返航中降落及断线行为。

当前是原型，尚未实现任务队列、ROS2 Action、避障和实际电子围栏。FAILSAFE 不保证自动降落，
关闭节点不会停止模拟器里的飞行任务。坐标使用 AirSim 本地 NED，不能直接当作 ROS ENU。
