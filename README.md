# nbd-monitor

**v0.0.8 — Status 只读化（候选版，待实机验收）**

使用 Bash、systemd、Debian nbd-client 和 qemu-nbd，记录每个 NBD Run 的退出事实。一次正常退出需要同一 Run 的原生 detach 成功、本地设备确认解除、QEMU 正常退出并释放导出，以及在截止前完成并保存双端确认。实际 Run 未满足上述条件则保守失败，保留原因和已有证据；尚未形成 Run 的预约失效独立记录，不计入 Run 退出失败。

## 启动与退出

- 当前面向一对一资源映射和受控网络；admit 仅为轻量资源预约，不认证真实客户端或绑定实际数据连接。服务端在资源锁内确认预约，原生 `nbd@` 启动前置检查收到合法确认后才调用 nbd-client，一次启动只执行一次原生连接。
- 数据 socket 保留 `Accept=no`。服务端执行导出前复核有效准入，连接身份依靠客户端守约。应限制两个端口的网络可达范围，仅允许受管客户端访问。用 `nbd-monitor report DEVICE` 通过控制通道查询状态。控制通道 TCP 11120 未认证、未加密。
- 上层先安全卸载，原生 `nbd@` 执行 detach，再由原生 `ExecStopPost=finalize` 采集并交换退出事实。monitor 只承担运行期心跳。服务端从首次有效 close 或 QEMU finish 取较早者开始计时，120 秒内完成判定；客户端停止阶段的通信上限为 150 秒。结构性证据缺失、明确业务失败及未恢复的通信故障按失败处理。原生 stop 作业等待 finalize 结束，但 stop 返回码不是双端 normal 的凭据；检查本轮 `/run/nbd-monitor/DEVICE/final` 与 Journal，缺少有效终态不得判为正常。
- 旧的实际 Run 未收口或被保护性阻断时拒绝新一轮；未形成 Run 的预约自然失效后允许重新预约。人工核实旧 QEMU、镜像和数据连接均已释放后，可在服务端执行 `nbd-server unlock PORT SESSION`。Session 可从 Journal 中确认，解锁记录与原始失败结果一并保留。
- 客户端与服务端的运行状态在 `/run`。整机重启使旧准入和 Session 失效，重新按当前资源状态准入。日志进入 Journal，历史是否跨重启保留取决于系统的 Journal 配置。

- 心跳仍为 10 秒、观测缺口阈值 30 秒、扫描 5 秒。心跳只用于观测，不参与成败裁定；active 无业务截止。服务端独立记录缺口/恢复，客户端记录通信异常/恢复及首次可信 Run 绑定，正常心跳不逐条记日志。

- v0.0.3 仅为 `confirm` 增加有界通信重试：单次请求 2s，失败后等待 1s，使用原始 finalize 150s 截止及 0.5s 退出余量，不另设次数上限。只重试传输故障与明确请求超时；非法回复、身份错误及业务拒绝立即收口。请求保持同一 Session / Run，重试恢复后可 normal。admit、heartbeat、close 的失败处理不变。服务端状态机、120s 截止、历史重放窗口、线协议和状态格式不变；最后确认始终未取得时客户端失败、服务端已提交的 normal 仍有效。

- v0.0.4 移除本地 systemctl 查询的额外 1s 超时；使用原生 IPC 期限和外层停止保护，记录查询耗时、失败及解析拒绝原因。finalize 的 150s 预算从进入函数起算，覆盖 close.lock 等待；heartbeat.lock 等待共用剩余预算，并保留已有单次请求及退出余量。服务端资源锁取消独立 1s 限制，正常阻塞等待，各入口仍受已有外层保护（export 启动及人工 unlock 无新增期限）。事实分类、失败保护、120s 服务端窗口及通信重试策略不变。真实关机与锁竞争专项验收尚待完成。

- v0.0.5（T01）将既有 confirm 有界通信重试复用于 close：仅传输故障与明确请求超时可重试；原样重发冻结的退出事实或 Run 确认，单次 2s、重试间隔 1s、正常 waiting 轮询 5s，共用原始 finalize 150s 截止及 0.5s 余量，不增设次数上限，不重置服务端 120s 截止。合法回复结束当次重试；close/confirm 分别记录重试事件。明确失败、拒绝、协议或身份错误立即结束；admit、heartbeat、status 不变。服务端仅更新版本号。真实关机与丢包专项验收待完成。
- 项目总体设计契约采用[超时最小化原则](docs/CLOSE-EXCHANGE-DESIGN.md#总体设计契约超时最小化原则)：超时约束业务生命周期，不替代真实业务事实；所有新增时限须说明必要性及失败后果。

- v0.0.6（T02）取消 heartbeat 的客户端 fault 锁存、服务端 active 失联失败裁定及 finalize 心跳交接锁等待。新版 close 的 client_fault 固定为 -，保留线协议和原有状态格式；已有可信 Run 快照仍用于退出身份校验。服务端另存每资源当前 Session/Run 的观测记录，30s 仅为缺口观测阈值。原有准入/退出保护和人工 unlock 条件不变；active 残留若没有 finish 事实会持续独占，不自动释放。须在安全空闲时双端升级，禁止活动 Session 混用版本。长期实机观察后再决定是否移除心跳。
 
- v0.0.7（T03）admit 复用 2s 请求保护、1s 重试及 systemd ExecStartPre 启动保护；仅 transport/timeout 原样重试同一 Session，不续期、不新增超时或取消协议。未形成 Run 的 30s 预约过期记 released/not_started；reserved close 的 disconnected 结束预约、unknown 保留原期限但禁止再导出、connected 或 Run 身份矛盾保守 blocked。保留 QEMU 释放字段原值及实际 Run 保护；新旧状态不自动迁移。迟到数据连接可能落入下一预约，当前不增加清队列、冷却或真正认证机制。辅助机制的成本与收益须匹配，详见上位契约 D01～D07；双端安全空闲升级，待专项实机验收。

- v0.0.8（T04）status 仅在原资源锁内读取已保存的权威业务状态，直接回复，不调用 expire、不推进或虚拟推导状态、不生成业务终态。保留单次请求、原有超时、协议、stdout 格式及 stderr/非零退出码语义。已过期但尚未被业务入口或 scan 裁定的状态仍原样返回；available 不是准入承诺或数据通道健康证明。客户端业务代码不变，待专项实机验收。

## 仓库与验证

- `client/`、`server/` 对应目标主机的根文件系统路径。两个脚本分别位于 `usr/local/libexec/nbd-monitor` 和 `usr/local/libexec/nbd-server`。
- `etc/*.conf` 和 systemd 单元采用正式文件名，供审核后的隔离环境实机验收；部署前须按目标环境填写业务配置。客户端从原生 `/etc/nbdtab` 读取设备、主机、数据端口和 export。首验使用独立 QCOW2 镜像、端口、设备和挂载点。真实 systemd 停止顺序、QEMU 与内核 NBD 联动仍需实机验证。
- [Close Exchange 契约](docs/CLOSE-EXCHANGE-DESIGN.md) 和 [最小实机部署设计](docs/MINIMAL-DEPLOYMENT-DESIGN.md) 保留完整设计背景。首版以本 README 的单一成功路径、120 秒服务端截止和 `/run` 状态规则为实施范围。
- 旧版代码位于 [archive/v0.4.2-M0](https://github.com/SiliCAT/nbd-monitor/tree/archive/v0.4.2-M0)。本版本使用自己的线协议和本次开机的状态。
