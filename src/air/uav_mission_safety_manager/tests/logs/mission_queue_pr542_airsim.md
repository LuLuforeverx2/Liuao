# PR #542 AirSim 手动复测记录

日期：2026-10-26（Asia/Shanghai）

## 环境

- Windows OpenHUTB Air 场景，WSL2 Ubuntu 22.04
- ROS 2 Humble，Python 3.10
- AirSim Python 客户端 1.8.1
- AirSim RPC：Windows 主机 `172.29.240.1:41451`
- ROS 域：`ROS_DOMAIN_ID=93`，`ROS_LOCALHOST_ONLY=1`

## 操作与结果

1. `connect` 后进入 `ARMING`，记录起点约为 `[0.0, 0.0, 11.1961]`。
2. `armed` 后进入 `TAKEOFF`。
3. `airborne` 后进入 `EXECUTING`，遥测高度约为 `z=9.6633`。`takeoff_rpc_result=false` 及对应 `last_error` 为既有 AirSim 起飞返回异常；遥测确认已离地，因此继续执行复测。
4. 提交三航点任务：

   ```json
   {"event":"mission","waypoints":[
     {"x":1.0,"y":0.0,"z":8.2,"speed":0.3},
     {"x":2.0,"y":0.0,"z":8.2,"speed":0.3},
     {"x":2.0,"y":1.0,"z":8.2,"speed":0.3}
   ]}
   ```

   队列最终进入 `COMPLETED`，`total=3`、`completed=3`、`current_index=null`、`remaining=0`，`target_ned=null`。
5. 调用 `return_home` 后进入 `LANDED`。最终位置约为 `[-0.0383, -0.0077, 11.2053]`，相对记录起点的三维误差约为 `0.039 m`。

## 证据与边界

两张终端截图分别保留 AirSim 队列 `COMPLETED`（`3/3`）和返航后的 `LANDED` 状态。截图中的起飞返回 `false` 已如实保留，不能作为本次 PR 修复内容。

本轮 AirSim 手动复测覆盖三航点执行和返航落地；暂停/继续、取消和非法高度输入由自动测试覆盖，未将其宣称为本轮 AirSim 实测结果。首次发送任务时出现过一次 YAML 引号错误，修正命令格式后任务成功执行，该次错误不计入功能失败。
