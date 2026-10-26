# PR #542 AirSim 手动复测记录

日期：2026-10-26（Asia/Shanghai）

## 第一轮环境

- Windows OpenHUTB Air 场景，WSL2 Ubuntu 22.04
- ROS 2 Humble，Python 3.10
- AirSim Python 客户端 1.8.1
- AirSim RPC：Windows 主机 `172.29.240.1:41451`
- ROS 域：`ROS_DOMAIN_ID=93`，`ROS_LOCALHOST_ONLY=1`

## 第一轮操作与结果

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

## 第一轮证据与边界

第一轮观察到 AirSim 队列 `COMPLETED`（`3/3`）和返航后的 `LANDED` 状态。起飞返回 `false` 已如实记录，不能作为本次 PR 修复内容。

本轮 AirSim 手动复测覆盖三航点执行和返航落地；暂停/继续、取消和非法高度输入由自动测试覆盖，未将其宣称为本轮 AirSim 实测结果。首次发送任务时出现过一次 YAML 引号错误，修正命令格式后任务成功执行，该次错误不计入功能失败。

## 第二轮补充复测（2026-10-26 上午，Asia/Shanghai）

同一台机器的 Windows OpenHUTB Air / WSL2 Ubuntu 22.04 / ROS 2 Humble；ROS_DOMAIN_ID=93、ROS_LOCALHOST_ONLY=1。使用 `~/venvs/airsim_humble/bin/python3` 直接调用 `uav_mission_safety_manager.main.main()` 启动节点，明确使用已安装 AirSim 的 Python 环境。

最初通过 ROS 可执行入口启动时，节点使用的 Python 找不到 `airsim`，连接进入 FAILSAFE；切换解释器并重启节点后连接成功。这是启动环境问题，未计为成功飞行，也未修改程序来掩盖错误。

本轮起点为 `[0.0, 0.0, 11.196127891540527]`。三航点为 `[3,0,8.2]`、`[3,3,8.2]`、`[0,3,8.2]`，各点指令速度为 0.2 m/s。

以下行号对应原始状态日志，从 1 开始，不表示时间或独立测试次数：

| 项目 | 原始日志行号与结果 |
| --- | --- |
| 连接 | 1305：ARMING，记录上述起点 |
| 首次三航点完成 | 3779 RUNNING → 4177 COMPLETED，completed=3 |
| 已完成后暂停 | 4785：顶层 PAUSED，但队列已 COMPLETED；不计为执行中暂停 |
| 有效执行中暂停 | 7123 RUNNING → 7157 PAUSED，completed=0、current_index=0、remaining=3；目标 `[3,0,8.2]`、速度 0.2 保留 |
| 恢复并完成 | 9489 RUNNING，目标与速度保持；9623/9725 的 completed 为 1/2，9849 COMPLETED、completed=3 |
| 执行中取消 | 10971 RUNNING → 11005 CANCELED；completed=0、current_index=null、remaining=0，目标与速度清空 |
| 返航自动落地 | 11995 RETURNING → 12115 LANDING → 12399 LANDED；队列保持 CANCELED |

最终位置 `[-0.01274413987994194, -0.030296020209789276, 11.20584487915039]`，与记录起点的三维距离约 **0.0343 m**；这是本轮测量值，不是任意场景的精度保证。

### 本轮记录与边界

- 状态日志仅在复测机器本地保存，不随源码提交。上表汇总了实际观察到的状态转换与计数。

原始日志位于复测机器 `~/pr542_manual/status_20261026_081156.txt`，共 6880 条 JSON 状态；本地另有 75 条状态节选。原文件 SHA-256：`32f03976bd04a150b14fca44f14197874b5f40fe03a8d2ad585da98f68a06098`。原始流不含逐条时间戳或命令终端输出，行号仅用于追溯采样顺序。截图不随源码提交，PR 描述保留运行截图展示；暂停/恢复结果依据本地保存的状态流核对。

本轮补充验证了执行中暂停/恢复、三航点完成、执行中取消和返航自动落地。非法高度输入仍仅由自动测试覆盖；未新增 AirSim 非法输入、断连、碰撞、避障或真机实测。起飞 RPC 仍返回 `false`，遥测确认离地，该既有异常未修复。此次为作者同机复测，不代替独立审核。
