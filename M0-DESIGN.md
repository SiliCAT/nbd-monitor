# M0-Design：A1 / A2 / A3 最小退出诊断
日期：2026-11-26。路线 C，用户已确认三类核心异常；依据 PROJECT-SCOPE.md。替代此前广泛观察研究（ddeb155）。本文件锁定第一阶段范围，不是生产代码或主动处置授权。审核代码基线 8f16a33，正向回归基线 1a19ae2。

## 1. 唯一目标

先判断 detach 有没有发生、是否在本地生效、是否在服务端完成。重点是客户端已经解除而服务端长期残留，阻断下一轮接入。**等待真实故障检验这个最小集合，缺什么，再加什么。**

| 分类 | 必要事实 | 初始诊断 |
|---|---|---|
| A1 退出动作缺失 | 本轮连接建立过，进入退出/关机，有直接证据表明 detach 未执行 | 记录动作事实和退出上下文，不采集额外设备信息 |
| A2 本地退出未完成 | detach 已执行，本地仍连接或解除无法确认 | 记录命令实际结果、本地状态及异常时的 size/PID、原生 NBD 错误、既有补做结果 |
| A3 服务端退出未完成（核心） | detach 已执行、本地已解除、服务端本轮退出仍未完成 | 通过 close、既有超时及 finish 取得旧 TCP、QEMU、协商状态和必要的入口证据 |
| 正常对照 | detach 执行、本地解除、服务端退出，完成既有协商 | 少量结果记录，不作设备快照 |

unknown 不等于 no；上述是故障维度，不要求所有异常互斥或强行归类。A2 的未解除与未知必须区分；A3 若只取得客户端 zero 声明而无本地直接日志，注明 reported。原生返回非零与本地实际解除也可并存。突然断电、手工绕过 systemd、日志缺失时 detach 事实可能未知，不能凭无日志断言 A1。

## 2. 五类最小事实及采集点

| 编号 | 事实 | 客户端 | 服务端 |
|---|---|---|---|
| M01 | 本轮身份 | 连接建立时复用 A.2 必要身份、session/boot/native InvocationID 与连接 PID | export run、boot、QEMU PID/starttime、register 关联 |
| M02 | detach 是否发生及结果 | 原生 ExecStopPost/systemd 日志：执行与否、开始/结束、退出码/信号或未知 | 不推断客户端原生命令 |
| M03 | 本地是否解除 | 复用既有 device_status、退出钩子及归零/补做结果；异常时补必要 size/PID | zero=yes 是客户端报告 |
| M04 | 双端确认 | 实际 close 尝试、有效回复或未确认、最终结果 | 实际收到/匹配请求、已有状态结果、生成回复与最终协商结果 |
| M05 | 旧轮是否解除 | 复用实际 previous/下一次连接结果 | close 尚未完成、scan timeout、finish 等既有触发点的 QEMU、必要 TCP/队列证据、已有处置结果 |

共享身份、时间与结果；原生事实独立于关联成功。合并记录，不要求五套 worker、独立文件或固定字段格式。正常心跳不逐次记录；不新增正常运行采样。不抢 detach 前快照，也不要求客户端建立 TCP 四元组才能退出。确需不可重建的 socket 锚点时，先证明诊断必要性，允许一次性有限取证，不预设扩展。

## 3. 客户端：A1 与 A2

保留原有 systemd 停止顺序：native detach → ExecStopPost → monitor disconnect/close。复用 A.2 原生 ExecStop 执行事实解析，核验本轮 InvocationID、命令及 PID/时间/code/status；未绑定也记录独立原生事实。SERVICE_RESULT 等主进程/服务结果不冒充 ExecStop 结果；ExecStopPost 可能因启动失败触发。未执行、开始未结束、失败、成功、未知分别保留。

取消前置观察单元、detach 前实时快照、前置握手、ActiveState 观察轮询以及 M0-E01。连接前身份锚点不等于 detach 前快照，保留必要 A.2 成果。观察不得改变原生命令、停止顺序、既有 device_status 和 detach 安全授权。

A2 先用实际退出码/信号/时间、本地归零检查、size/PID 和已有补做结果。内核 NBD disconnect / Send disconnect failed / shutting down sockets 等线索优先事后从原生 Journal 关联；记录缺口，不将未发现错误说成没有错误。Linux 6.12 的 send_disconnects() 错误可能不经 nbd_disconnect() 返回给用户空间；DISC 无协议应答。无需周期 kernel.cursor/recent、重复 raw、挂载/holders/inflight、存储拓扑和全量 I/O 统计。未来实际 A2 证据不足再申请扩展。

## 4. 服务端：A3 的最小独立观察

优先复用两个异常入口：① 收到本轮有效 close/zero=yes，旧 QEMU 尚未退出；② 既有 heartbeat/registration/closing timeout。finish 保留实际进程退出事实；实际下一轮接入复用既有事件。第一次 wait 可是正常收尾时间差，超过既有期限仍未解除才确定持续异常。客户端突然消失时无需其最终报告；只能证明服务端观察到的失联及残留，不证明客户端死亡或 detach 是否执行。

| A3 子类 | 需要区分 |
|---|---|
| A3.1 | 同一旧 run 的已接受数据 TCP 仍存在、QEMU 仍运行 |
| A3.2 | 旧 TCP 未见或已确认解除，但 QEMU 仍运行；查询未知不得当作已解除 |
| A3.3 | QEMU 已退出，但控制协商未完成或下一轮实际接入失败 |

重点先跑通 A3.1。服务端以已知 run/boot/PID+starttime 核验实际持有的非 LISTEN socket，异常窗口有限 ss 查询本地/对端地址端口、TCP 状态和可得 inode/cookie；必要时仅核对目标 QEMU 的 socket fd。监听 Recv-Q 仅在判断后续接入受阻时记录，区别监听与已接受连接。不常态采样，不转储完整 TCP_INFO，不跨主机比较 inode/cookie。register 不能独立证明 TCP 归属；多个候选、代次变化、查询失败等保留 candidate/unknown，不强配。先验证仅凭服务器 run/进程及异常时查询是否已足够；如实际不足再论证建立时 socket 身份锚点。TCP 状态不证明收到 DISC。

外部查询不在资源锁内进行，先保存必要控制上下文、释放锁、前后复核代次；不能阻塞原生退出、回复或 scan。记录首次异常与已观察到的终态，不把一次快照冒充永久状态；不承诺观察到所有中间 TCP 转换。后续连接是否成功只依据真实连接结果，不主动探测。

## 5. 双向确认

沿用 previous/register/heartbeat/close。现有客户端仅在确认本地解除后发送 close(session, zero=yes)。服务端按现有状态生成 complete/abnormal/wait 等结果，客户端记录实际收到的结果或未确认。首次 wait 与最终超时不同；complete 不是文件系统一致性或下一轮接入保证；abnormal 不能被 monitor 退出码 0 掩盖。

初始仅记录请求尝试、服务端实际收到、生成回复及客户端实际收到/无有效回复、最终结果；不预先扩展 wire 全阶段遥测或请求 ID。客户端写入不证明送达，服务端生成回复不证明客户端收到；现有协议不能逐次精确关联时保留未知。若真实故障无法区分控制连接、写入和读取环节，再按准入原则增补。

## 6. 验收、约束和停止条件

先验证正常对照、A1（未执行/未知）、A2（真实返回、本地未解除/未知）、A3（本地归零而服务端残留、服务端独立失联取证和双向确认）；同时回归并行设备身份、观察失败、短时失联、原生成功但内核错误、后续接入。T01～T13 是历史细分验收参考，不是新采集业务清单。难以安全复现的情况使用隔离模拟，不破坏在用设备。

保留既有控制业务及安全检查；默认 detach_enabled=no、reset_on_timeout=no。不新增持久化观察文件、独立历史、daemon、协议、设备监测或主动恢复权限。日志复用 Journal，必要临时文件仅限确认易失目录；不承诺失电时最后日志必然持久保存。观察成本及异常失败隔离以实测验证。先冻结 M0，投入真实运行；不预设第二、第三阶段功能。任何真实故障暴露的新缺口均按 PROJECT-SCOPE.md §3 重新准入。

下一步只设计/实现最小 A1/A2/A3 记录及回归，不开展前置观察实验。服务端异常时 socket 归属、原生日志关联及实际执行成本尚需验证；具体事件格式与实现可随最小实验收敛，不能据此扩大范围。

## 7. 研究依据

- [NBD 协议：Terminating transmission / NBD_CMD_DISC](https://github.com/NetworkBlockDevice/nbd/blob/master/doc/proto.md)
- [Linux Debian 6.12.94 驱动 nbd.c](https://sources.debian.org/src/linux/6.12.94-1/drivers/block/nbd.c)（同 6.12 系列参考，不冒充实际运行内核的完整构建源码）
- [nbd-client 3.26.1 上游源码](https://raw.githubusercontent.com/NetworkBlockDevice/nbd/nbd-3.26.1/nbd-client.c)
- [QEMU 上游 qemu-nbd.c 参考实现](https://qemu.googlesource.com/qemu/+/a6b1625d6107675f1bd7e3c660814e74ae6d21b6/qemu-nbd.c)
- [systemd.service 257](https://www.freedesktop.org/software/systemd/man/257/systemd.service.html)、[systemd.unit 257](https://www.freedesktop.org/software/systemd/man/257/systemd.unit.html)
- [Debian Trixie ss(8)](https://manpages.debian.org/trixie/iproute2/ss.8.en.html)
- 项目历史《10911 遗留连接事故复盘》《NBD 异常退出与服务端残留会话故障调查及架构修订备忘录》《P2～P5 研发推进计划》及 A.2 双设备验收记录。
