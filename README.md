# nbd-monitor

**v0.0.3 — confirm 有界通信重试（候选版，待实机验收）**

使用 Bash、systemd、Debian nbd-client 和 qemu-nbd，记录每个 NBD Run 的退出事实。一次正常退出需要同一 Run 的原生 detach 成功、本地设备确认解除、QEMU 正常退出并释放导出，以及在截止前完成并保存双端确认。其余记录为失败，保留原因和已有证据供人工分析。

## 启动与退出

- 多个受管客户端可竞争同一资源。服务端在资源锁内确认第一个有效准入，原生 `nbd@` 的启动前置检查收到确认后才调用 nbd-client。一次准入只执行一次原生连接。
- 数据 socket 保留 `Accept=no`。服务端执行导出前复核有效准入，连接身份依靠客户端守约。应限制两个端口的网络可达范围，仅允许受管客户端访问。用 `nbd-monitor report DEVICE` 通过控制通道查询状态。控制通道 TCP 11120 未认证、未加密。
- 上层先安全卸载，原生 `nbd@` 执行 detach，再由原生 `ExecStopPost=finalize` 采集并交换退出事实。monitor 只承担运行期心跳。服务端从首次有效 close 或 QEMU finish 取较早者开始计时，120 秒内完成判定；客户端停止阶段的通信上限为 150 秒。结构性证据缺失、明确业务失败及未恢复的通信故障按失败处理。原生 stop 作业等待 finalize 结束，但 stop 返回码不是双端 normal 的凭据；检查本轮 `/run/nbd-monitor/DEVICE/final` 与 Journal，缺少有效终态不得判为正常。
- 旧轮未收口或发生失败时拒绝新一轮。人工核实旧 QEMU、镜像和数据连接均已释放后，可在服务端执行 `nbd-server unlock PORT SESSION`。Session 可从 Journal 中确认，解锁记录与原始失败结果一并保留。
- 客户端与服务端的运行状态在 `/run`。整机重启使旧准入和 Session 失效，重新按当前资源状态准入。日志进入 Journal，历史是否跨重启保留取决于系统的 Journal 配置。

- 心跳仍为 10 秒、失联阈值 30 秒、扫描 5 秒；control 与 scan 单元仅过滤 info/debug 常规启停日志，脚本事实使用 notice，告警与失败保留。客户端每轮首次绑定 Run 留一条 heartbeat_bound，之后正常心跳不逐条记日志。

- v0.0.3 仅为 `confirm` 增加有界通信重试：单次请求 2s，失败后等待 1s，使用原始 finalize 150s 截止及 0.5s 退出余量，不另设次数上限。只重试传输故障与明确请求超时；非法回复、身份错误及业务拒绝立即收口。请求保持同一 Session / Run，重试恢复后可 normal。admit、heartbeat、close 的失败处理不变。服务端状态机、120s 截止、历史重放窗口、线协议和状态格式不变；最后确认始终未取得时客户端失败、服务端已提交的 normal 仍有效。

## 仓库与验证

- `client/`、`server/` 对应目标主机的根文件系统路径。两个脚本分别位于 `usr/local/libexec/nbd-monitor` 和 `usr/local/libexec/nbd-server`。
- `etc/*.conf` 和 systemd 单元采用正式文件名，供审核后的隔离环境实机验收；部署前须按目标环境填写业务配置。客户端从原生 `/etc/nbdtab` 读取设备、主机、数据端口和 export。首验使用独立 QCOW2 镜像、端口、设备和挂载点。真实 systemd 停止顺序、QEMU 与内核 NBD 联动仍需实机验证。
- [Close Exchange 契约](docs/CLOSE-EXCHANGE-DESIGN.md) 和 [最小实机部署设计](docs/MINIMAL-DEPLOYMENT-DESIGN.md) 保留完整设计背景。首版以本 README 的单一成功路径、120 秒服务端截止和 `/run` 状态规则为实施范围。
- 旧版代码位于 [archive/v0.4.2-M0](https://github.com/SiliCAT/nbd-monitor/tree/archive/v0.4.2-M0)。本版本使用自己的线协议和本次开机的状态。
