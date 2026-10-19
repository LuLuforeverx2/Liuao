# AirSim 实测与验证边界

[返回模块概览](overview.md) · [构建与复测命令](usage.md)

日期：2026-10-19。早期过程依据操作者提供的终端与模拟器截图记录；后续自动复测另有连续 JSONL 日志，见各节。
环境：Windows OpenHUTB（CarlaUE4.exe）、WSL2 Ubuntu 20.04、ROS2 Foxy。
以下结果仅适用于本次模拟环境，不能作为真机验证结果。

## 已观察到的接口行为

- 全新模拟器会话中，未运行 ROS 任务节点时，直接调用
  `takeoffAsync(timeout_sec=20).get()` 返回 `False`，实测耗时约 3.407 秒。
  随后三次遥测分别显示升高约 1.536、1.505、1.518 米，`landed_state=1`。
  这说明返回 False 时仍可能发生实际起飞；原因尚未确定，不能认定起飞接口验证通过。
- 同一新会话的独立降落调用 `landAsync(timeout_sec=60).get()` 返回 `True`，
  耗时约 7.422 秒。随后五次采样 z 约 11.2053、垂直速度 0，
  但 `landed_state` 仍为 1。
- 在上述降落成功后，连续检查距本轮起点约 0.0092 米、三轴合速度为 0，
  再调用 `armDisarm(False)` 返回 True，随后三次 `landed_state=0`。
- 更早的独立降落曾立即返回 False（约 0.001 秒），且未确认落地。
  完整重启模拟器后出现上述 True 结果；尚不能确定重启与行为差异的因果关系。

## 新版 ROS 垂直航点及自动降落收尾

本轮截图所示过程：

1. IDLE → ARMING，捕获起点约 `(0, 0, 11.20531177520752)`。
2. 解锁后 TAKEOFF；发送 airborne 后进入 EXECUTING，z 约 9.6725。
3. 下发绝对 NED 航点 `(0, 0, 6.2)`、速度 1 米/秒。
   随后位置 z 约 6.573，目标被清除，状态保持 EXECUTING，无错误。
   航点到达采用 1 米容差，不能描述为精确抵达 6.2。
4. 请求 return_home 后观察到 LANDING，随后持续 LANDED；最终位置回到起点，
   `command_pending=false`、`last_error` 为空，期间未人工解除解锁。

结论：这轮起点上方的自动降落收尾通过。根据当前控制器逻辑，LANDED 需要
解除解锁成功及后续新遥测确认落地。截图未单独保存每个内部 RPC 返回值。
本轮没有水平位移，不能作为水平返航验证；起飞结果 Future 未被现有适配器跟踪，
ROS 进入 EXECUTING 依据离地遥测，不能据此宣称起飞 RPC 返回 True。

## 修复前的水平航点与暂停补测

第二轮 ROS 测试已重新连接、解锁并进入 EXECUTING；垂直航点后位置约
`(0, 0, 6.568263053894043)`，目标已清除，无错误。
下发水平航点 `(3, 0, 6.2)`、速度 0.5 米/秒，等待约 2 秒后请求 pause。
操作者随后提供的截图显示持续 PAUSED，原目标 `(3, 0, 6.2)` 保留，
`command_pending=false`、`last_error` 为空。可见采样中 x 从约 1.2678
逐渐变为 1.2416，z 约 6.6568 至 6.6562，说明移动已显著减缓；
截图没有速度字段或逐样本时间戳，不能据此量化制动时间或确认速度阈值已满足。
随后独立只读遥测连续三次（间隔约 1 秒）显示三轴合速度均约 0.0001 米/秒，
位置 x 约 1.249058246612548、z 约 6.656608581542969，y 接近 0。
结合 PAUSED 状态及目标保留，暂停后的停稳检查通过；尚未量化暂停指令到停稳的延迟。
随后请求 resume，截图显示恢复 EXECUTING，位置约
`(3.1778, 0, 6.3051)`，距原目标 `(3, 0, 6.2)` 约 0.207 米，
目标已清除，`command_pending=false`、`last_error` 为空。
恢复原目标并到达容差内的行为通过；完整服务响应未保存，未测实际飞行速度。
随后请求水平返航，状态日志中采集到 210 条 LANDING，首条位置约
`(-0.2581, 0, 5.8981)`，末条约 `(-0.4732, 0, 5.9074)`。
该日志未捕获 RETURNING，不能给出完整转换时序。
进一步只读检查确认模拟未暂停、API 控制已启用、landed_state=1，
连续三次位置 z 约 5.9074，垂直速度约 0.0000363 米/秒。
因此已返回起点水平位置附近，但仍在记录起点上方约 5.298 米，
本轮水平返航后的自动降落尚未完成，不能记为完整流程通过。
原因尚未确定，不应仅凭静止或状态名 LANDING 判定已接地或解除解锁。

停止 ROS 节点并确认无残留进程后，操作者执行独立
`landAsync(timeout_sec=60).get()`，返回 True，耗时约 25.992 秒。
随后位置约 `(-0.47319335, 0.00001709, 11.20531178)`，三轴速度均为 0，
但 landed_state 仍为 1。该独立恢复操作不计入 ROS 自动流程通过证据。
此位置距原起点约 0.473 米，超过自动解除解锁要求的 0.15 米，
即使节点降落成功，也不能预期当前兼容收尾条件一定满足。
已确认返航到达容差（1 米）与收尾位置阈值（0.15 米）存在不匹配；
但本轮此前悬停在屋顶上方的根因仍未定位，不能将其仅归因于这个阈值。

修复前实现限制：controller 的 resume 重新派发目标时没有传入原速度，
适配器默认使用 3 米/秒。因此继续不能描述为恢复原先的 0.5 米/秒速度。

## 2026-10-19 修复及实际 AirSim 自动复测

本次已实现：继续时保留原速度；返航进入最后 1 米后采用限速修正，
距目标不超过 0.08 米且速度不超过 0.05 米/秒持续 1 秒后才启动降落；
飞行 Future 在专用线程等待，替换任务需先取消并确认旧任务结束。
解除解锁的 0.15 米、0.1 米/秒、连续 3 秒条件未放宽。
起飞 Future 现在保留实际结果并写入状态，不把 False 当作接口成功。

复测使用 `tests/live_airsim_validation.py` 直接调用同一适配器和控制器，
连接实际 Windows OpenHUTB/AirSim。没有其他 ROS 飞行节点运行。
这是实际模拟器联动测试，未经过 ROS 话题/服务；ROS 通信通过单独冒烟测试验证。

保留三份日志：

1. [`airsim_retest.jsonl`](https://github.com/OpenHUTB/ros2/blob/master/src/air/uav_mission_safety_manager/tests/logs/airsim_retest.jsonl)：首次脚本 reset 后只等 1 秒，
   模拟器短暂报告 Landed，但无人机仍在下降。错误捕获 home z=3.6369，
   起飞抛出正在移动（约 7.016 米/秒）的异常，本次失败，不计入通过。
   随后补充预检：连续 3 秒落地、位置稳定且速度不超过 0.1 米/秒才开始；
   控制器连接也拒绝正在移动的落地遥测。后续会话以 reset 重新开始。
2. [`airsim_retest_stable.jsonl`](https://github.com/OpenHUTB/ros2/blob/master/src/air/uav_mission_safety_manager/tests/logs/airsim_retest_stable.jsonl)：完整流程通过，最终 LANDED，
   起点 `(0, 0, 11.196149826)`，最终约 `(-0.020503, 0, 11.205310)`，
   三维误差约 0.022456 米。暂停后独立遥测合速度约 0.0611 米/秒。
   该轮起飞还未返回便被航点替换，因此记录的 False 可能来自取消，
   不能用作原始起飞异常的独立复现证据。
3. [`airsim_retest_final.jsonl`](https://github.com/OpenHUTB/ros2/blob/master/src/air/uav_mission_safety_manager/tests/logs/airsim_retest_final.jsonl)：等待起飞 Future 自行完成后再发航点。
   起点 `(0, 0, 11.196171761)`，起飞约 3.44 秒后返回 False，
   此时遥测 z 约 9.660652，已升高约 1.5355 米；未用新航点取消起飞。
   原异常在本轮再次复现，原因仍未确定。
   随后发送起点上方 5 米的垂直航点及向北 3 米水平航点（0.5 米/秒），
   暂停保留目标，独立遥测合速度约 0.0607 米/秒；继续保留 0.5 米/秒指令值。
   该数值是指令速度，不是对整段实际速度的测量。
   返航自动完成 RETURNING → LANDING → LANDED，最终约
   `(-0.024734, -0.000002, 11.205307)`，距起点约 0.026367 米。
   期间没有人工降落或人工解除解锁；脚本在新遥测确认 Landed 后记录 PASS。

普通航点仍采用 1 米到达容差，向北 3 米是指令目标，并非确认精确飞满 3 米。
本次没有复现修复前的空中停滞，但不能单凭这两轮成功判定之前的根因。
离线检查：28 项单元测试、Foxy colcon 构建、ROS 单线程服务/状态/超时冒烟测试均通过。
实际 AirSim 最终复测完成后，无人机已落地，脚本已退出。

## 本轮开始前的补测清单（完成情况见续测节）

- 水平航点飞行中暂停：PAUSED、目标保留及停稳遥测已观察通过，完整服务响应未保存。
- 继续：新版已验证恢复原航点及原指令速度；未量化完整速度曲线。
- 水平返航：新版已通过两轮实际模拟器联动，最终误差约 2.25 和 2.64 厘米，日志已保存。
- 返航中显式降落、模拟器断线：尚未完成本轮验证。

当时返航中显式降落及断线仅有离线覆盖，不计为实际模拟器故障场景通过。

## 提交检查续测（2026-10-19）

- 再次执行 Foxy `colcon build --symlink-install --packages-select uav_mission_safety_manager`：通过。
- 再次执行 28 项单元测试：全部通过；`ROS_DOMAIN_ID=87` ROS 冒烟测试：通过。
- `mkdocs serve --livereload`：构建成功，已在浏览器预览模块页面，确认标题、中文、状态流程及代码块正常显示。
  全站存在其他模块的既有链接告警，本模块未出现链接告警；不将构建成功写成全站无告警。
- 新增 [`tests/ros_airsim_validation.py`](https://github.com/OpenHUTB/ros2/blob/master/src/air/uav_mission_safety_manager/tests/ros_airsim_validation.py)，通过实际 ROS launch、命令话题与 Trigger 服务补测，命令及边界见 [运行教程](usage.md)。
- 最初 Windows 未监听 41451；操作者启动 Air 场景后，端口恢复可达并完成下列测试。

### ROS 接口实际 AirSim 续测

1. [`ros_airsim_mission.jsonl`](https://github.com/OpenHUTB/ros2/blob/master/src/air/uav_mission_safety_manager/tests/logs/ros_airsim_mission.jsonl)：完整流程通过。
   暂停服务响应约 0.00350 秒，3 秒后独立遥测速度约 0.03093 米/秒，原目标保留；
   继续响应约 0.00342 秒，保留 0.5 米/秒指令速度；返航响应约 0.00301 秒。
   最终自动 LANDED，三维起点误差约 0.031468 米，未人工降落或解除解锁。
   此轮在下述返航服务调度修复前执行；修复后完整流程另行重跑。
2. [`ros_airsim_return_land_missed_window.jsonl`](https://github.com/OpenHUTB/ros2/blob/master/src/air/uav_mission_safety_manager/tests/logs/ros_airsim_return_land_missed_window.jsonl)：第一次补测漏过返航中断窗口。
   虽然脚本输出 PASS 且最终落地，但显式 land 前已自动进入 LANDING，
   只能证明落地，不能计为返航中显式降落通过。原始日志保留，不改写其 PASS 行。
   复核发现末段修正可在同一次 tick 内完成旧派发又立即发出新派发，持续显示 pending，
   不利于用户服务插入。最初尝试派发完成后留出一个定时器周期，增加回归测试；
   该尝试随后出现返航超时，已弃用，最终处理方式见下文。
   补测脚本同时记录服务请求前状态，并要求显式 land 前仍处于 RETURNING。
3. [`ros_airsim_return_land.jsonl`](https://github.com/OpenHUTB/ros2/blob/master/src/air/uav_mission_safety_manager/tests/logs/ros_airsim_return_land.jsonl)：上述留出周期方案下，该用例通过。
   land 请求前明确记录 RETURNING，位置约 `(0.02498, 0.000001, 6.41338)`；
   服务约 0.00293 秒返回 Accepted: LANDING，最终 LANDED，三维起点误差约 0.119233 米。
   该用例在起点水平位置附近中断，不能推广为远离起点或任意地形显式降落。
4. [`ros_airsim_disconnect.jsonl`](https://github.com/OpenHUTB/ros2/blob/master/src/air/uav_mission_safety_manager/tests/logs/ros_airsim_disconnect.jsonl)：RPC 传输中断检查通过。
   在起点上方飞行后切断节点专用 TCP 转发，约 2.4003 秒进入 FAILSAFE，
   故障原因为 telemetry timeout 2.2s；切断后仍收到 12 条状态消息。
   模拟器进程保持运行，此结果不等同于模拟器进程崩溃或自动故障降落通过。
   独立恢复降落在停止被测节点后执行并单独记录，不计入 FAILSAFE 自动恢复能力。

本轮起飞 Future 自行完成后再发航点，仍返回 False；此已知异常没有被消除。
留出周期方案的完整任务重跑见 [`ros_airsim_mission_yield_timeout.jsonl`](https://github.com/OpenHUTB/ros2/blob/master/src/air/uav_mission_safety_manager/tests/logs/ros_airsim_mission_yield_timeout.jsonl)：
返航末段高度停在目标上方约 0.14 米，120 秒后进入 FAILSAFE，不能计为通过。
停止节点后的独立恢复降落返回 True，解除解锁后新遥测确认 Landed。
最终改为服务请求到来时先处理已完成的派发 Future，不改变连续修正节奏；
仍在派发中的请求保持拒绝，脚本在 RETURNING 内有界重试。
最终版本 Foxy 构建、29 项单元测试、ROS 冒烟测试均通过，实际模拟器结果另见 verified 日志。

环境补充：Ubuntu 20.04.6、Python 3.8.10、airsim 1.8.1、msgpack-rpc-python 0.4.1、
numpy 1.17.4、tornado 4.5.3；ros-foxy-rclpy 1.0.13、ros-foxy-ros2launch 0.11.7。
模拟器未提供明确产品版本；为便于识别本次二进制，记录 SHA256：

- `CarlaUE4.exe`：`D3FA6944043AE994EE3FB79127ED84C1366F1A5B7A618B59A7579690F9D80EAE`
- `CarlaUE4-Win64-Shipping.exe`：`0CA2D9518841A23894FB0B75E8FE52936F39B52FEFD14291012B07D44B88E962`

哈希仅识别上述可执行文件，未覆盖全部资源或构建来源。

首次断线用例的恢复日志中，解除解锁后立即读取仍是 Flying；稍后独立只读检查确认
Landed、速度为 0，已追加 `recovery_followup_observation`，没有改写原采样。
脚本已补充解除解锁后最多等待 3 秒的新遥测确认。

### 最终实现的连续三项复核

三项依次执行，前一脚本退出后才重置并启动下一项，均通过实际 ROS 接口驱动。

| 日志 | 检查结果 |
| --- | --- |
| [`ros_airsim_mission_verified.jsonl`](https://github.com/OpenHUTB/ros2/blob/master/src/air/uav_mission_safety_manager/tests/logs/ros_airsim_mission_verified.jsonl) | PASS；暂停服务约 2.90 毫秒，暂停后合速度约 0.03085 米/秒；继续约 2.13 毫秒，原目标及 0.5 米/秒指令保留；返航约 3.33 毫秒；自动 LANDED，三维起点误差 0.024154 米。 |
| [`ros_airsim_return_land_verified.jsonl`](https://github.com/OpenHUTB/ros2/blob/master/src/air/uav_mission_safety_manager/tests/logs/ros_airsim_return_land_verified.jsonl) | PASS；请求前为 RETURNING，位置约 `(0.05286, 0.000002, 6.32169)`；land 服务约 2.13 毫秒接受并进入 LANDING；自动 LANDED，三维起点误差 0.114227 米。 |
| [`ros_airsim_disconnect_verified.jsonl`](https://github.com/OpenHUTB/ros2/blob/master/src/air/uav_mission_safety_manager/tests/logs/ros_airsim_disconnect_verified.jsonl) | PASS；RPC 传输中断后约 2.4000 秒进入 FAILSAFE，期间收到 12 条状态消息；不代表断线自动降落通过。 |

最终断线用例停止节点后独立降落返回 True，受保护解除解锁返回 True，新遥测确认 `landed_state=0`。
三个脚本均已退出，没有留下 ROS 飞行控制节点。

全部起飞结果仍为 False。服务耗时为该测试脚本观测到的请求响应时间，非硬实时保证。
普通航点有 1 米容差，指令速度不等同于完整实际速度曲线。
模拟器进程崩溃、远离起点的显式降落、任意地形或真机仍未验证。


## 文件整理说明（2026-10-20）

教程和验证记录集中在本模块文档目录，原始日志迁移到源码包的 `tests/logs/`，文件名不包含日期，测试日期保留在记录正文中。日志内容未修改，文件索引与 SHA256 见 [日志索引](https://github.com/OpenHUTB/ros2/blob/master/src/air/uav_mission_safety_manager/tests/logs/README.md)。目录整理后的构建和离线检查不算新增实际飞行验证；上述飞行结果仍来自 2026-10-19。

整理后重新通过 Foxy colcon 构建、29 项单元测试、ROS 冒烟测试。
已分别启动并停止 `ros2 run ... mission_manager` 和离线 `main.launch.py`，确认新 `main.py` 入口可用；
安装目录包含 launch、config、README 与 LICENSE。MkDocs 构建成功，已预览三个模块页面，
模块未产生路径告警，全站其他模块既有告警仍存在。
