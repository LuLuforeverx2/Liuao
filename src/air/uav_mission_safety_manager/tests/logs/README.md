# AirSim 原始测试日志

这些文件是 2026-10-19 的实际测试证据，不是运行时输出目录。新测试请输出到 `/tmp` 或工作空间外，审核后再选择归档。

完整结果与限制见 [AirSim 实测记录](../../../../../docs/air/uav_mission_safety_manager/airsim_validation.md)。`missed_window` 虽有 PASS 行，但没有中断返航，不算该场景通过；`yield_timeout` 是已弃用方案的失败记录。最终版本以三个 `verified` 日志为准。断线后的独立恢复不算自动故障降落。

2026-10-20 整理文件位置并移除文件名中的日期；移动前后已逐份检查内容一致，日志正文未改写。以下 SHA256 使用 Git 存储的 LF 换行计算；Windows autocrlf 检出为 CRLF 时，应先归一化换行再核对。

| 文件 | SHA256 |
| --- | --- |
| [airsim_retest.jsonl](airsim_retest.jsonl) | `807bf66d1c5f778a8e556316ba20c97feb2c4db9f5d6606d1f7960615a4471c2` |
| [airsim_retest_final.jsonl](airsim_retest_final.jsonl) | `a894f6a8cded86d82045f235755a02e7f45d0f45d56e8b48b39dddcbdf60c70d` |
| [airsim_retest_stable.jsonl](airsim_retest_stable.jsonl) | `255ccc0d27be91ce6cfa8bdf43cc677521e43b96b70e8a8f9fc395488f6554fe` |
| [horizontal_return.log](horizontal_return.log) | `aff05621967c6b4cb9b565b43eea08249053c5c32e18609fbfc2d5b28cad031d` |
| [ros_airsim_disconnect.jsonl](ros_airsim_disconnect.jsonl) | `adbd0c3d854574ddd4b7950554b6c3cf9e6c3d1f6749fc4d86678a0403dcf020` |
| [ros_airsim_disconnect_verified.jsonl](ros_airsim_disconnect_verified.jsonl) | `0f2c7256a7f6892523c3db7397f2f076d77c31a3c750c12d378e34ec292dea7c` |
| [ros_airsim_mission.jsonl](ros_airsim_mission.jsonl) | `1220cb1577b0a6ac31b2c8cb052b36d31afd2f7ebad75591997cc97b39e1d4c0` |
| [ros_airsim_mission_verified.jsonl](ros_airsim_mission_verified.jsonl) | `26b33bd67ae7947b62c590d08f279ad51b244abde2c4e7a24f32992348798f5d` |
| [ros_airsim_mission_yield_timeout.jsonl](ros_airsim_mission_yield_timeout.jsonl) | `b90435023f41c6b844806f7c0b85f7bed49aae819ec2901f21bd2fbc81b81f70` |
| [ros_airsim_return_land.jsonl](ros_airsim_return_land.jsonl) | `fc0b283bda20e28b9f62dbf44c45851d1e60e90201188f0b8d50e13e2cf65683` |
| [ros_airsim_return_land_missed_window.jsonl](ros_airsim_return_land_missed_window.jsonl) | `6f270a8b0a042ff9d961f342212e9585cd0db0a2e0553bef7bda0f61dee57666` |
| [ros_airsim_return_land_verified.jsonl](ros_airsim_return_land_verified.jsonl) | `a6a249a891b1100c22a54c0e7e5d451a16e42eb721432586f97e977de62f62a3` |
