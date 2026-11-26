# nbd-monitor

**v0.0.1 — 第二代研发基底（不可部署）**

本仓库基于 Bash、systemd、Debian nbd-client 与 qemu-nbd，面向逐轮退出结果交叉分析。以 Run 为基本业务实体，服务端收集双端退出事实；close 是退出阶段的特殊心跳，不是成功退出证明。

本次重建仅保留工程结构与设计契约，未实现连接、监护、退出、保护、维护、指标导出或任何旧版协议。不得复制到运行中的系统，不得安装、启用、重启现有 NBD 服务或替换现有 libexec 文件。

## 结构

- `client/`、`server/`：各自对应目标 Linux 根文件系统的工程路径。
- `usr/local/libexec/nbd-monitor`、`usr/local/libexec/nbd-server`：最小入口骨架。仅帮助和版本查询可成功；所有业务操作明确返回未实现错误，不执行外部命令、不读写状态、不发起连接。
- `etc/*.conf.in`：配置结构模板，尚无冻结的配置格式。
- `etc/systemd/system/**/*.in`：非活动 systemd 模板，保留原生 nbd@ 集成、Type=notify、socket activation、数据与控制通道分离和扫描结构；未验收，不可直接部署。不得将模板直接改名安装。客户端 monitor 停止预算模板为 180s；其他历史参数不是新版已冻结值。
- [Close Exchange 设计契约](docs/CLOSE-EXCHANGE-DESIGN.md)：E01–E12、E08 细则、W01–W09，设计版本 v0.1 与工程版本 v0.0.1 独立。字段候选和未决事项仍须审核。
- [最小实机接入、功能边界及部署验收](docs/MINIMAL-DEPLOYMENT-DESIGN.md)：已审核的首个最小实机系统范围、MD01–MD15 及 D01–D14 验收条件。独立控制端口为 TCP 11120；首个版本不实现主动补做 detach、自动 Reset 或镜像维护。

## 研发边界

- 不要求兼容 v0.4 / M0 行为或线协议；从历史代码按需复用已验证的实现。
- 继承 A1/A2/A3 诊断目标、安全不变量与原生 NBD 的职责。诊断事实不是 detach、Reset、镜像维护的新增授权。默认不自动处置；unknown 不等于成功、失败或安全释放。
- 服务端阶段性退出观察 120s，客户端主动通信硬截止 150s，E08 满足即结束；外部 systemd 超时按 30s 梯度留余量。尚未实现计时或状态机。
- Journal 为详细诊断日志；临时状态使用 /run（或 /tmp）。不新增持久化文件、字段、数据库或真实文件系统写入，除非先讨论并批准；现有 Run 档案的扩充亦须批准。
- 不增加常驻组件或生产依赖。云端能完成的测试尽量在云端消化，真实 systemd、NBD、QEMU 和文件系统联动须实机验收。云端自动化、CI 不是本基底的正式工作项目。
- 旧版不变地保存在 [archive/v0.4.2-M0](https://github.com/SiliCAT/nbd-monitor/tree/archive/v0.4.2-M0)，提交 `174a5b101c92ab95202cf0007dd83a205cc0589a`。Git 历史未重写。旧版设计与手册不再作为新版活动契约。

本基底不是 Release，不是旧版的升级包；没有安装器、自动发布流程或生产可用性承诺。
