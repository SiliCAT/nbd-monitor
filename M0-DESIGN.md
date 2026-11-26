# M0-Design：双端最小退出证据契约（研究稿）
日期：2026-11-26。依据 PROJECT-SCOPE.md 的路线 C 与 T01～T13。本文为研究设计候选，不是生产代码或新处置授权；先验证关键接线与采样，再冻结实现。审核生产基线 8f16a33，正向回归基线 1a19ae2。

## 1. 研究结论与边界

NBD_CMD_DISC 没有协议应答。Linux 6.12 的 nbd_disconnect() 调用 send_disconnects() 后返回 0，发送失败可能仅出现在内核日志。nbd-client 3.26.1 的传统 ioctl 退出依次执行 NBD_DISCONNECT、NBD_CLEAR_SOCK。因此命令返回、本地容量/PID、内核错误、TCP 关闭、QEMU 退出、控制协商、下一轮接入都是独立事实。非 persistent QEMU 的退出还受已协商连接及内部状态制约；不能将进程返回 0 或监听 active 视作 DISC 收到或接入恢复证明。部署 QEMU 精确版本待实机核对；源码结论不替代实机验证。

10911 事故证明：旧 QEMU 持有失去真实对端的 ESTAB，新的 TCP 在监听队列积压；人工解除旧连接后 QEMU 退出，积压又触发新实例。NIX 事故缺失客户端历史日志。不能断言两次事故的客户端 DISC 发送失败或 QEMU 死锁已经查明。

## 2. 最小事件：按事实而不是按属性组织

事件共享最小身份：端（client/server）、资源（dev 或 port/export）、本机 boot、同轮 observation/session 或 run、事件种类、发生时间（单调时间与墙钟）、结果/原因。身份未知不禁止独立事实记录。事件时间取观察时刻，区分原生开始/结束时间及 Journal 接收时间。两端单调时钟不直接比较。

| 事件族 | 触发时机 | 最小有效载荷 |
|---|---|---|
| C1 client_generation | 原生启动前、连接确认、register 结果；必要身份补齐 | 客户端/endpoint、boot、observation/session、native InvocationID、连接 PID/容量、启动/登记结果；preconnect 锚点及可取得的 TCP 候选身份 |
| C2 client_detach | 可取得的退出前触发；原生 ExecStopPost；已有补做或归零判定 | phase=pre/post、执行事实、开始/结束、退出码/信号、local state 与 zero、PID、观察来源/缺口；原生内核日志按设备/代次/窗口关联 |
| C3 client_close | 退出业务开始、请求写入/响应/状态变化、最终结束 | session、zero、实际传输阶段/结果、首次与最终尝试时间、次数、收到的回复、退出原因；未尝试/不确认/异常单独表达 |
| S1 server_generation | export 建轮、register、可取得的首次接受连接 | run、boot、QEMU PID+starttime、session/登记结果；接受 socket 四元组、inode/cookie、身份置信；未登记也保留 run |
| S2 server_exit | 首次 close、首次登记/心跳/closing 超时、状态变化、finish、既有处置结果 | old socket 存在/状态/未知、QEMU 活性及实际退出结果、最后匹配心跳时间、timeout/finish 原因、listener 状态/队列、已有处置结果 |
| S3 server_close | 请求收到/匹配/拒绝、状态保存结果、生成回复、最终协商结果 | run/session、zero（客户端声明）、request/commit/reply 事实、complete/abnormal/wait/error、次数和时间 |
| 双端下一轮 | 现有 previous、真实原生启动、export 建轮/登记；无需主动探测 | 上轮结果、旧 run 是否已释放、新 observation/session/run 及实际原生启动/登记结果 |

以上是语义族，非七套 worker，也不是所有字段在每次事件中必填。可合并事件。无状态变化不新增正常心跳历史；关键失败/恢复及初次、终态不丢失。正常重复 wait 或超时按同轮汇总，不能掩盖最终失败。M0 以原生日志和现有控制状态为优先来源。

## 3. 客户端关键窗口与原生接线

现有 monitor Before/PartOf=nbd@，原生 nbd@ Requires/After=monitor；因此关机时 native 先停止，monitor disconnect 随后。现有 ExecStopPost=observe-stop 可复用已验收 A.2 解析；查询本轮 InvocationID，匹配实际原生 ExecStop 路径/参数及 PID、start/stop、code/status。SERVICE_RESULT、EXIT_CODE、EXIT_STATUS 属于服务/主进程结果，不能直接当原生 ExecStop 返回值。ExecStopPost 也可能由启动失败触发。未执行、开始但未结束、结果未知不可混淆；不能仅凭没有日志断言未执行。

关键难点是 detach 前真实现场。ActiveState=deactivating 轮询不保证发生在 ExecStop 之前；ExecStop.start_time 是事后事实，不是事前状态。优先保留连接建立身份，再验证确实需要的前置采样：仅本轮身份、size/pid 三态、必要的已知 TCP 候选，不读块设备、不查挂载、不联网握手、不重新执行 detach。直接手工 detach、突发断电、启动未完成时该窗口可缺失。

候选接线 A（首选实验）：独立、弱依赖、有界的一次性观察单元，启动顺序 After=nbd@，停止与 nbd@ 共同发生时先观察；通过 PartOf 传播停止/重启并检验 Wants 拉起与失败隔离。不能把单独停止观察单元当作原生 detach 已执行，不保证非 systemd 路径；需验证 mount、网络和 shutdown 无环。候选 B：显式在原生 ExecStop 列表前插入观察命令，必须清空并重列厂商命令；systemd 257 的停止命令超时会跳过后续 ExecStop，单纯 '-' 不足以证明安全，故不作为首选。候选 C：不增加接线，仅保留建立时身份及原生事后记录，pre 标为 unknown。先用隔离实验评估 A 的实际诊断增益及成本，未经确认不改生产单元。即使有 timeout 也不能承诺不可中断内核等待必定结束。

退出后复用原生 hook、既有 device_status/close 条件和结果；观察用 A.2 sysfs 三态，控制层检查保持原样。不为观察额外重复控制命令。内核关键消息优先直接从现有 Journal 取回，按本机 boot、设备名、代次和原生退出窗口限定；有重名/时钟/截断/日志缺失则 candidate/unknown。第一阶段先验证离线提取能否满足要求，不默认保留周期 kernel.cursor/recent 或重复 raw 日志。

## 4. 服务端独立观察：在旧实例还存在时取证

nbd-export@.socket Accept=no；监听 fd 交给 qemu-nbd，现有 export 记录 run/PID/starttime 后 exec QEMU，实际接受 socket 稍后才建立；ExecStopPost=finish 在进程停止后执行，不能作为残留现场的唯一观察点。

S1 优先在成功 register 或随后已有 heartbeat/scan 的有限补齐窗口取得接受 socket，未登记 run 在首次 scan/登记超时时取证。若连接极短或采样失败，记录 unknown，不新增常态 TCP 轮询。S2 在首次有效 close（尤其 wait）、已有 scan 判定 timeout、旧轮占用阻止新 export、finish、已有 reset/入口恢复的实际结果处取证。复用现有 scan.timer（5s 配置，不等于固定 5s 实际周期）。已记录同轮同状态不反复采集；如需判断之后是否释放，复用已有状态变化/finish/下一轮触发，避免一次失联快照被当成永久状态。没有对应触发时不能承诺观察到每次 TCP 转换或实际新接入尝试。

观察不得在资源锁内执行耗时外部命令、改变原有 reply/finish/scan 时间预算或阻塞原生 exec。先捕获已校验 run/PID/boot 与控制结果，释放锁后有界查询并前后核验身份；发生代次变化时仅保留带时间的候选。优先复用既有 .state 字段；不改变 .state schema 或新增持久历史。

## 5. TCP 最小身份与判定

服务器优先使用有界 ss 查询目标端口，核对非 LISTEN 连接的实际 QEMU 进程归属（PID+启动身份；必要时核对该 PID 的 socket fd/inode），记录本地/对端 IP:port、TCP state、可取得的 inode 或 sk cookie、scope/置信；监听记录只需是否存在及 Recv-Q。不要混淆监听队列连接数与已接受 socket 的数据字节队列；不采集 RTT/吞吐/拥塞/完整 TCP_INFO。

客户端内核 socket 不一定显示用户空间进程归属；nbd PID 与 TCP PID 不应强等同。用实际四元组候选及服务器反向四元组、session/run、时间窗口交叉关联；同 endpoint 唯一候选也非内核直接证明。cookie/inode 只用于本机 boot/netns 内追踪，不跨主机比较；NAT、代理、多个候选、端口复用或查询失败降为 candidate/unknown。现有 register 以端口/current run 建立控制身份，并不验证该 TCP 是该客户端的真实数据连接，不得把 session 登记当 TCP 确证。M0 不增加线上协议、不追踪全机 socket、不采包或改 QEMU trace 配置。

仅在已关联的旧 socket 仍可见时肯定残留；成功查询未见目标只说明本观察窗口/范围内未见。已知 socket 消失、TIME-WAIT 等 TCP 收尾和 QEMU 退出分别记录，不能一律视为残留。旧 QEMU PID 不可读时区分已确认不在、PID 复用、权限/查询未知。独立日志可以定位阻塞环节，但不能由 ESTAB 或心跳超时单独判定客户端死亡或授权强制恢复。

## 6. 双向确认：复用现有 close，不新增协议

现有 close 在原生服务 inactive/failed 且本地 device_status 确认 disconnected 后才发送 zero=yes。服务端按 session/run、zero 声明、process=exited 及既有 fault/exit_kind 状态返回结果；客户端收到 complete 或 abnormal 均可能把本地 session 标记 closed。此控制语义保持不变。

当前 wire() 将打开连接失败、写入失败、读超时、无效响应合并为失败。M0 需要在实际 wire 阶段区分：未尝试/连接失败/请求本地写入失败/写入完成但未取得有效回复/收到有效回复/整体超时。写入完成不是网络送达。服务端分别记录有效请求收到、身份校验、状态保存成功/失败、生成或尝试写出回复；不能从服务端 printf 成功推断客户端收到。保持 stdout 原有 wire 结果，日志走原有 Journal 通道；不得因日志影响状态保存、锁和回复。现有协议没有逐次请求 ID，重试只能按 session/run、双方时间及次数关联；丢失位置无法判明则 unknown，不为每次重试新增协议字段。

## 7. 最小诊断输出与验收

一次退出保留独立维度：native=success/failure/not_executed/started_no_result/unknown；local=connected/disconnected/unknown；server_socket=present/not_observed/unknown；server_process=alive/exited/unknown；close_client/close_server 各自结果；next_connection=success/failure/not_attempted/unknown；证据范围与缺口。值是语义候选，不是新控制状态机。

优先验证 T01 正常、T04 原生成功与内核错误并存、T06 本地归零与服务器残留、T09 请求/回复故障、T10 突发消失、T11 旧实例释放后接入仍受阻；再覆盖其余 T02～T13。隔离注入不能代替真实偶发故障捕获。正常路径、观察器失败/超时、并发代次及停止顺序均须回归。第一阶段完成后冻结证据集合，只有真实故障仍无法区分才提出扩展。无需在 M0 阶段强制证明 DISC 已收到或实现自动安全恢复。

## 8. 轻量化与未决事项

不采集挂载、Btrfs/LVM 等拓扑、常规 I/O 统计，不新增 daemon、周期全量采集、持久化文件、依赖、协议或主动处置权限。日志优先原生 Journal；评估直接 stderr 的单行 key=value 输出是否足够（避免每条 logger 外部命令），保留原生 systemd 元数据和显式身份。日志投递不等于持久化确认；突发失电可能损失尾部证据。必要临时状态只在确认易失的 /run 或 /tmp 中复用/有界保存；服务端 /run 尚需核验。默认 detach_enabled=no、reset_on_timeout=no 不变。所有预算通过实机测量冻结，不凭正常运行耗时推定冷启动或关机上限。

未决：前置观察单元能否满足失败隔离；最小 socket 归属及关联在真实部署的可行性；wire 阶段记录如何做到不改变控制行为；离线 Journal 关联能否保留关键内核线索；部署 QEMU 精确版本及服务端日志保留策略。下一步先完成小型隔离实验与只读现场测量，再决定生产重建实现，不自动延伸第二、第三阶段。

## 9. 研究依据

- [NBD 协议：Terminating transmission / NBD_CMD_DISC](https://github.com/NetworkBlockDevice/nbd/blob/master/doc/proto.md)
- [Linux Debian 6.12.94 驱动 nbd.c](https://sources.debian.org/src/linux/6.12.94-1/drivers/block/nbd.c)（同 6.12 系列参考，不冒充实际运行内核的完整构建源码）
- [nbd-client 3.26.1 上游源码](https://raw.githubusercontent.com/NetworkBlockDevice/nbd/nbd-3.26.1/nbd-client.c)
- [QEMU 上游 qemu-nbd.c 参考实现](https://qemu.googlesource.com/qemu/+/a6b1625d6107675f1bd7e3c660814e74ae6d21b6/qemu-nbd.c)
- [systemd.service 257](https://www.freedesktop.org/software/systemd/man/257/systemd.service.html)、[systemd.unit 257](https://www.freedesktop.org/software/systemd/man/257/systemd.unit.html)
- [Debian Trixie ss(8)](https://manpages.debian.org/trixie/iproute2/ss.8.en.html)
- 项目历史《10911 遗留连接事故复盘》《NBD 异常退出与服务端残留会话故障调查及架构修订备忘录》《P2～P5 研发推进计划》及 A.2 双设备验收记录。
